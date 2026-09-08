# taxdocs-api/INFRA.md

The AWS substrate for the taxdocs capstone is authored as raw CloudFormation
YAML in `cfn/`. Floci is the local execution/testing target only; it is not
treated as equivalent to the AWS CloudFormation control plane, and CDK (if
encountered) is read-only reference material, not a replacement authoring
path.

Previous work (W6 D1, W6 D2) shipped CI via GitHub Actions + OIDC and moved
deployment to Argo CD against a shared/platform-provided EKS cluster. This
work provisions the AWS infrastructure substrate those workloads depend on:
bootstrap IAM/storage, network, and application (RDS) resources, plus the
validation tooling that gates changes to all of it.

## Stacks and deployment order

1. `taxdocs-bootstrap-dev` (Task 1)
   - Retained bootstrap bucket + access-log bucket (KMS/AES256, PAB on all
     four toggles, versioning, lifecycle, deny-non-TLS bucket policy).
   - `CfnDeployRole` (`taxdocs-api-cfn-deploy`): the GitHub OIDC role every
     later stack's ChangeSet is created and executed under.
2. `taxdocs-artifacts-dev` (Task 3)
   - Independent hardened artifact bucket (KMS, PAB, versioning, lifecycle
     tiering to `STANDARD_IA`/`GLACIER_IR`, deny-non-TLS). No dependency on
     the network or bootstrap stacks; can deploy any time after bootstrap.
3. `taxdocs-network-dev` (Task 2)
   - 3-AZ VPC, 3 public + 3 private subnets, Internet Gateway, public route
     table, per-AZ private route tables, NAT gateways gated by an
     `IsProdLike`/`IsDev` Condition pair (1 NAT in dev, 3 in staging/prod),
     and the application security group.
4. `taxdocs-app-dev` (Task 3)
   - RDS PostgreSQL instance, DB subnet group, DB security group, and the
     app-to-DB TCP 5432 rule. Consumes the network stack's exports via
     `Fn::ImportValue`. Requires the `taxdocs/dev/db-master` Secrets Manager
     secret to already exist (see below) — it is not created by this stack.

Deploy order is bootstrap → (artifacts, network, in either order) → app. The
app stack depends on the network stack's exports and on the out-of-band
Secrets Manager secret; it has no dependency on the artifacts stack.

## ChangeSet CREATE/UPDATE flow

There is no `CREATE_OR_UPDATE` ChangeSet type — `--change-set-type` is either
`CREATE` (new stack) or `UPDATE` (existing stack), decided before the call:

```bash
# first deploy of a stack
aws cloudformation create-change-set \
  --stack-name taxdocs-network-dev \
  --change-set-name initial \
  --change-set-type CREATE \
  --template-body file://cfn/taxdocs-network-dev.yaml \
  --parameters ParameterKey=EnvName,ParameterValue=dev \
  --region us-east-1

aws cloudformation describe-change-set \
  --stack-name taxdocs-network-dev --change-set-name initial \
  --region us-east-1
# review every Add / Modify / Remove / Replacement before proceeding

aws cloudformation execute-change-set \
  --stack-name taxdocs-network-dev --change-set-name initial \
  --region us-east-1
```

Subsequent changes use `--change-set-type UPDATE` against the same
stack/template. The bootstrap stack additionally requires
`--capabilities CAPABILITY_NAMED_IAM` because it names `CfnDeployRole`
explicitly. `describe-change-set` output is the PR review artifact — every
Replace, Modify, Add, and Remove is read before `execute-change-set` runs.

## Export/import naming

The network stack exports (as literal, stack-specific names):

- `taxdocs-network-dev-VpcId`
- `taxdocs-network-dev-PublicSubnets`
- `taxdocs-network-dev-PrivateSubnets`
- `taxdocs-network-dev-AppSgId`

The application stack imports `VpcId`, `PrivateSubnets` (split with
`Fn::Split`/`Fn::Select` into three individual subnet IDs), and `AppSgId` via
`Fn::ImportValue`. No subnet or security-group ID is hardcoded in the app
template. The bootstrap and artifacts stacks export their own bucket/role
values under their own stack name (e.g. `taxdocs-bootstrap-dev-BootstrapBucketArn`,
`taxdocs-artifacts-dev-BucketArn`) — nothing outside the network stack is
consumed cross-stack today.

Deviation from the reference shape: exports use literal strings
(`taxdocs-network-dev-VpcId`) rather than `!Sub "${AWS::StackName}-VpcId"`.
This is a portability note, not a security finding — it means renaming the
stack breaks every consumer's `Fn::ImportValue`, since the export name no
longer follows the stack automatically. Left as-is because the stack names
are fixed per environment in this capstone; flagged here so a future
multi-instance deployment doesn't get surprised by it.

## Secrets Manager password handling

No template in `cfn/` declares a password `Parameter`, and none uses
`NoEcho`. `cfn/taxdocs-app-dev.yaml` resolves the RDS master password with a
Secrets Manager dynamic reference, evaluated at deploy time, never stored in
template parameters or CloudFormation state:

```yaml
MasterUserPassword: "{{resolve:secretsmanager:taxdocs/dev/db-master:SecretString:password}}"
```

Unlike the cfn-author reference shape, the app stack does **not** create the
`AWS::SecretsManager::Secret` or `AWS::SecretsManager::SecretTargetAttachment`
resources itself — `taxdocs/dev/db-master` is treated as an out-of-band
prerequisite that must exist before the app stack's ChangeSet is executed.
This is a deliberate, not accidental, deviation: it keeps the secret's
lifecycle independent of the app stack (a stack delete/replace never risks
recreating or rotating it), at the cost of an operational precondition —
deploying `taxdocs-app-dev` before the secret exists fails at
`CREATE_IN_PROGRESS` on `DbInstance`. This precondition is recorded here so
it isn't rediscovered as a deploy-time surprise.

## Retention rules

Every stateful resource in `cfn/` carries both policies, not just one:

```yaml
DeletionPolicy: Retain
UpdateReplacePolicy: Retain
```

Applies to: `AccessLogBucket`, `BootstrapBucket` (`taxdocs-bootstrap-dev.yaml`),
`ArtifactBucket` (`taxdocs-artifacts-dev.yaml`), and `DbInstance`
(`taxdocs-app-dev.yaml`). `UpdateReplacePolicy: Retain` matters as much as
`DeletionPolicy: Retain` here: a property change that forces replacement
(e.g. changing `DBSubnetGroupName` or a bucket's `BucketName`) without the
former would silently delete the live resource on a successful `UPDATE`,
even though the stack itself was never deleted.

## Validation workflow

`.github/workflows/cfn-validate.yml` runs on any PR touching `cfn/` or the
workflow file itself, against a Floci `2.0.1` service container:

1. `cfn-lint 1.56.0` against `cfn/*.yaml` — passes clean (0 errors) as of
   this audit.
2. `cfn-nag 0.8.10` via `cfn_nag_scan --fail-on-warnings` — passes clean
   (0 failures, 0 warnings) across all four templates as of this audit.
3. A Floci-backed `aws cloudformation validate-template` loop over all four
   templates, explicitly labeled as a wire-compatibility check only —
   `cfn-lint`/`cfn-nag` are the authoritative local gates, Floci's
   `validate-template` response is not treated as proof of real AWS
   acceptance.

## Drift verification

Local drift probe attempted against Floci:

```bash
aws --endpoint-url http://localhost:4566 cloudformation detect-stack-drift \
  --stack-name taxdocs-network-dev --region us-east-1
```

Floci does not implement `DetectStackDrift` (`UnknownAction`). No `DRIFTED`
or `IN_SYNC` result was produced or is claimed here. The full drift
exercise — deliberate console mutation, `detect-stack-drift`,
`describe-stack-drift-detection-status` polling, per-resource
`describe-stack-resource-drifts` review, and reversion via ChangeSet — is
documented as a procedure above but has not been executed against a real
AWS account, and this document does not assert that it has.

## cfn-author deviations found and fixed

Audited all four implemented stacks (`cfn/taxdocs-bootstrap-dev.yaml`,
`cfn/taxdocs-network-dev.yaml`, `cfn/taxdocs-app-dev.yaml`,
`cfn/taxdocs-artifacts-dev.yaml`) plus the validation workflow against the
cfn-author Skill's conventions (`.claude/cfn-author/SKILL.md`) and the
reference shapes.

**Found and fixed in this audit:**

- **OIDC `sub` claim over-broadened.** `CfnDeployRole`'s trust policy used
  `StringLike` with a single repo-wide wildcard,
  `repo:${GitHubOrg}/${GitHubRepo}:*`, matching any ref, environment, or
  workflow in the repo. Fixed to the two exact patterns the deploy workflow
  actually needs: `repo:${GitHubOrg}/${GitHubRepo}:ref:refs/heads/main` and
  `repo:${GitHubOrg}/${GitHubRepo}:pull_request`. `aud` was already correctly
  pinned with `StringEquals` in both the original and fixed versions — that
  part was never wrong.
- **CloudFormation action `Resource` used wildcard region/account.** The
  `TaxdocsStackChangeSets` statement scoped `Resource` to
  `arn:aws:cloudformation:*:*:stack/taxdocs-*` — name-scoped to the capstone
  but not account/region-scoped. Fixed to
  `!Sub "arn:aws:cloudformation:${AWS::Region}:${AWS::AccountId}:stack/taxdocs-*"`,
  matching the reference's pattern of pinning pseudo parameters wherever the
  account/region are known statically.

**Checked and confirmed correct (no fix needed):**

- No password `Parameter` anywhere in `cfn/`; no `NoEcho: true` on any
  parameter. RDS resolves the master password only via a Secrets Manager
  dynamic reference (see above).
- Every `DeletionPolicy: Retain` resource also carries
  `UpdateReplacePolicy: Retain` — no unpaired occurrence found.
- No literal `Action: "*"` anywhere in `cfn/`. The only `Action: "s3:*"`
  occurrences (`taxdocs-bootstrap-dev.yaml`, `taxdocs-artifacts-dev.yaml`)
  are on `Effect: Deny` statements scoped to a specific bucket ARN pair and
  gated by `Condition: {Bool: {aws:SecureTransport: "false"}}` — the
  deny-non-TLS hardening pattern, not a broad grant.
  No bare `Resource: "*"` exists anywhere in `cfn/`.
- IAM `Allow` actions are enumerated explicitly (never a service wildcard)
  and every `Allow` `Resource` is scoped to `taxdocs-*` stack/changeSet ARNs,
  the bootstrap bucket's own ARN, or `role/taxdocs-*` — no broad grant
  reaches beyond this capstone's own resources.
- Both S3 buckets that need it (`BootstrapBucket`, `ArtifactBucket`) pair
  `DeletionPolicy: Retain` with `UpdateReplacePolicy: Retain`; no data
  resource in `cfn/` has one without the other.

Post-fix, `cfn-lint 1.56.0` reports 0 errors and `cfn-nag 0.8.10
--fail-on-warnings` reports 0 failures / 0 warnings across all four
templates.

## AWS versus Floci limitations

Explicit, so nothing here is mistaken for real AWS evidence:

- GitHub OIDC federation and real STS trust enforcement are not exercised
  locally — Floci does not validate the `sub`/`aud` condition logic the way
  AWS STS does.
- Floci's `validate-template` is a wire-compatibility check only; it is not
  authoritative AWS `ValidateTemplate` behavior and is documented as such in
  the CI workflow output itself.
- `DetectStackDrift` and related drift APIs are unsupported by Floci
  (`UnknownAction`); no drift result has been produced or claimed against
  this environment.
- `cfn-lint` and `cfn-nag` are the meaningful, authoritative local
  validation gates in this setup — Floci is execution/smoke-test only, per
  the environment constraints for this capstone (no real AWS account is
  available).
- The CloudFormation templates in `cfn/` are authored as real
  AWS-compatible CloudFormation throughout; nothing here has been rewritten
  into emulator-specific shapes to make Floci happy.
