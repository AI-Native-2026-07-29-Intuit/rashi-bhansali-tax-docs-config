# Taxdocs infrastructure

The AWS substrate is authored as raw CloudFormation YAML in `cfn/`. Floci is
the local execution target; it is not treated as equivalent to the AWS control
plane.

## Stacks and deployment order

1. `taxdocs-bootstrap-dev`
   - Retained bootstrap and access-log S3 buckets.
   - GitHub OIDC CloudFormation deployment role.
2. `taxdocs-artifacts-dev`
   - Independent retained artifact bucket with KMS encryption, versioning,
     public-access blocking, lifecycle transitions, and a non-TLS deny policy.
3. `taxdocs-network-dev`
   - Three-AZ VPC, three public and three private subnets, internet gateway,
     environment-conditioned NAT gateways, routes, and the application
     security group.
4. `taxdocs-app-dev`
   - RDS PostgreSQL, its subnet and security groups, and the cross-stack
     application-to-database TCP 5432 rule.

Create `taxdocs/dev/db-master` in Secrets Manager before deploying the
application stack. The artifact and network stacks are independent, but the
network stack must exist before the application stack.

## ChangeSet workflow

For local execution, every AWS CLI command is explicitly directed to Floci:

```bash
export FLOCI_ENDPOINT=http://localhost:4566
export AWS_ACCESS_KEY_ID=test
export AWS_SECRET_ACCESS_KEY=test
export AWS_DEFAULT_REGION=us-east-1
```

Create a new stack:

```bash
aws --endpoint-url "$FLOCI_ENDPOINT" cloudformation create-change-set \
  --stack-name taxdocs-network-dev \
  --change-set-name initial-network \
  --change-set-type CREATE \
  --template-body file://cfn/taxdocs-network-dev.yaml \
  --parameters ParameterKey=EnvName,ParameterValue=dev \
  --region us-east-1

aws --endpoint-url "$FLOCI_ENDPOINT" cloudformation describe-change-set \
  --stack-name taxdocs-network-dev \
  --change-set-name initial-network \
  --region us-east-1

aws --endpoint-url "$FLOCI_ENDPOINT" cloudformation execute-change-set \
  --stack-name taxdocs-network-dev \
  --change-set-name initial-network \
  --region us-east-1
```

Use `--change-set-type UPDATE` for an existing stack. Review every Add, Modify,
Remove, and Replacement value from `describe-change-set` before execution.
The bootstrap stack additionally requires `CAPABILITY_NAMED_IAM`.

Task 4 added a `ManagedBy=CloudFormation` VPC tag. Floci's ChangeSet reported
one VPC Modify, no Add or Remove, and `Replacement: False`. Execution reached
`UPDATE_COMPLETE`, but Floci changed the VPC ID and did not expose the new tag.
The reviewed change had no intended replacement, but the emulator result
cannot prove AWS replacement behavior.

## Exports and imports

The network stack exports:

- `taxdocs-network-dev-VpcId`
- `taxdocs-network-dev-PublicSubnets`
- `taxdocs-network-dev-PrivateSubnets`
- `taxdocs-network-dev-AppSgId`

The application stack imports the VPC, private subnet list, and application
security group. No generated subnet or security-group ID is hardcoded. The
private subnet export is split and selected into the RDS subnet group.

The bootstrap, artifact, and application stacks also export their bucket,
role, database endpoint, port, and database security-group values under
stack-prefixed names.

## Secrets and retained data

The database password is never a CloudFormation parameter and no password
parameter uses `NoEcho`. RDS resolves only the `password` JSON key at deploy
time:

```yaml
MasterUserPassword: "{{resolve:secretsmanager:taxdocs/dev/db-master:SecretString:password}}"
```

The out-of-band secret is not recreated or owned by the application stack.
The bootstrap buckets, artifact bucket, and RDS instance each carry both:

```yaml
DeletionPolicy: Retain
UpdateReplacePolicy: Retain
```

The artifact bucket transitions objects to `STANDARD_IA` after 90 days and
`GLACIER_IR` after 365 days. Its four public-access-block settings are enabled,
default encryption uses `alias/aws/s3`, versioning is enabled, and its bucket
policy explicitly denies non-TLS S3 requests.

## Validation workflow

`.github/workflows/cfn-validate.yml` runs on pull requests that change `cfn/`
or the workflow itself. It pins and runs:

- `cfn-lint 1.56.0` against all four templates.
- `cfn-nag 0.8.10` with `--fail-on-warnings`.
- `aws cloudformation validate-template` against a local Floci service.

The Floci validation step is a wire-compatibility check only. `cfn-lint` and
`cfn-nag` are the meaningful local validation gates; Floci output is not
represented as authoritative AWS `ValidateTemplate` behavior.

Documented cfn-nag suppressions cover deliberate constraints: required fixed
resource names, required internet TCP 443 egress, an explicit empty database
SG egress list, and logging destinations not defined by these tasks. The final
local scan completes with zero unsuppressed failures or warnings.

## Drift verification

The local drift probe was:

```bash
aws --endpoint-url "$FLOCI_ENDPOINT" cloudformation detect-stack-drift \
  --stack-name taxdocs-network-dev \
  --region us-east-1
```

Floci returned `UnknownAction` because `DetectStackDrift` is unsupported.
There is no local AWS Console in which to make the curriculum's deliberate
mutation. No `DRIFTED` or `IN_SYNC` result was observed or claimed.

The full mutation, detection, per-resource review, reversion, and final
`IN_SYNC` exercise must be performed later in a real authorized AWS account.

## cfn-author reference audit

All six supplied reference shapes were compared with the implemented files:

- Bootstrap: the OIDC `aud` claim already used `StringEquals`; the final
  template preserves it and uses repo-scoped `StringLike` only for `sub`.
  The reference non-TLS bucket policy covered only selected S3 actions; the
  implementation denies all insecure S3 operations. IAM allow actions remain
  explicitly enumerated and resource scopes are limited to taxdocs stacks,
  bootstrap objects, and taxdocs roles.
- Network: the reference derived only six CIDRs and used a generic `IsHA`
  condition. The implementation derives eight CIDRs as required and defines
  `IsProdLike` plus its `IsDev` inverse. NAT B and C and their routes are
  consistently conditional. No database resource is imported into Network.
- Application: the reference attempted to create a secret that is an
  out-of-band prerequisite and lacked explicit application-SG egress to RDS.
  The implementation consumes the existing secret through a password dynamic
  reference and owns both sides of the TCP 5432 rule without a circular stack
  dependency.
- Artifacts: the reference changed the required bucket name, omitted the
  `GLACIER_IR` transition, and denied non-TLS access for only selected actions.
  The implementation uses the required name, both transitions, and a complete
  non-TLS deny.
- Validation workflow: the reference pinned an older-than-required cfn-lint,
  omitted bootstrap validation, did not fail on cfn-nag warnings, and attempted
  real OIDC with a placeholder AWS account. The implementation validates all
  templates and uses only a local Floci compatibility endpoint.
- Infrastructure document: the reference used the nonexistent
  `CREATE_OR_UPDATE` ChangeSet type, described the secret as stack-owned, and
  asserted fabricated AWS Console drift results. This document separates
  CREATE from UPDATE, records the out-of-band secret, and reports the
  unsupported drift API honestly.

Across the implemented templates there is no literal `Action: "*"` and no
unscoped `Resource: "*"`. Wildcard S3 actions appear only in explicit Deny
statements for insecure transport. Every data resource requiring retention has
the matching deletion and update-replacement policies.

## AWS versus Floci limitations

Observed Floci limitations include:

- GitHub OIDC federation and real STS trust enforcement are not exercised.
- `validate-template` is compatibility-only, not authoritative AWS validation.
- drift and `list-imports` APIs are unsupported.
- deletion of an in-use network export was allowed; AWS export-in-use
  protection was not proven. The network stack was recreated afterward.
- S3 CloudFormation provisioning omitted some declared KMS, lifecycle,
  public-access-block, and bucket-policy behavior.
- RDS provisioning did not faithfully expose all declared encryption and
  security-group properties.
- security groups retained emulator-default egress rules not declared by the
  templates.
- the network tag update changed physical IDs despite a no-replacement
  ChangeSet report.
- `UpdateReplacePolicy: Retain` enforcement was not behaviorally verified.

These limitations do not change the AWS-compatible source templates and must
not be presented as real AWS evidence.
