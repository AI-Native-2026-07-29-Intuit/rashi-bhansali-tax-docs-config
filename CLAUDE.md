# CLAUDE.md

## k3d cluster bootstrap gotchas (taxdocs-dev)

Recurring issues hit when bringing this cluster/GitOps loop back up after it's been idle (containers stopped, laptop restarted, etc). Check these before debugging from scratch.

### 1. OTel init-container TLS failure behind Zscaler

`taxdocs-api`'s `otel-agent-download` init container (`manifests/observability/taxdocs-api-deployment-patch.yaml` in the app repo) does a plain `curl` to github.com to fetch the OTel Java agent jar. On a machine behind the Zscaler TLS-intercepting proxy, this fails with:

```
curl: (60) SSL certificate problem: unable to get local issuer certificate
```

`curlimages/curl` ships a minimal CA bundle that doesn't trust Zscaler's re-signed cert (confirmed via `curl -vk https://github.com` showing `issuer: ... CN=Zscaler Intermediate Root CA`).

**Fix** (cluster-local workaround, not committed to the app repo's manifest — this is a this-machine issue, not something CI or other developers need):
```bash
# Export the trusted Zscaler root CA from the host keychain
security find-certificate -a -c "Zscaler" -p /Library/Keychains/System.keychain > /tmp/zscaler-root-ca.pem

# Load it into taxdocs-dev as a ConfigMap
kubectl -n taxdocs-dev create configmap zscaler-root-ca --from-file=ca.pem=/tmp/zscaler-root-ca.pem

# Patch the running deployment to mount it and pass --cacert
kubectl -n taxdocs-dev patch deployment taxdocs-api --type=json -p='[
  {"op": "add", "path": "/spec/template/spec/initContainers/0/volumeMounts/-", "value": {"name": "zscaler-ca", "mountPath": "/etc/ssl/zscaler"}},
  {"op": "replace", "path": "/spec/template/spec/initContainers/0/command", "value": ["sh", "-c", "curl -fsSL --cacert /etc/ssl/zscaler/ca.pem -o /otel/opentelemetry-javaagent.jar https://github.com/open-telemetry/opentelemetry-java-instrumentation/releases/download/v2.5.0/opentelemetry-javaagent.jar"]},
  {"op": "add", "path": "/spec/template/spec/volumes/-", "value": {"name": "zscaler-ca", "configMap": {"name": "zscaler-root-ca"}}}
]'
```
This patch lives only on the live deployment object — it will be lost if ArgoCD prunes/recreates the Deployment from the unpatched manifest, since `syncPolicy.automated.selfHeal: true` is on. Re-apply if pods start `Init:CrashLoopBackOff`ing on `otel-agent-download` again.

### 2. Postgres/redis/kafka connection failures — stale bridge IPs

`overlays/dev/05-taxdocs-dev.external-datastores.yaml` hand-manages `Service`+`Endpoints` pairs pointing at the Compose-managed Postgres/Redis/Kafka/Mongo containers via their Docker bridge network IP (`taxdocs_dev_taxdocs_net`). Every time those containers are recreated (`make up` after `make down`, a Docker restart, etc.), Docker assigns new IPs and these Endpoints go stale — pods get `Connection refused` / Hibernate `Unable to determine Dialect without JDBC metadata` errors even though the containers themselves are healthy.

**Diagnose:**
```bash
# Compare what the Endpoints say vs what the containers actually have
for svc in postgres redis kafka mongo; do
  echo "=== $svc ==="
  kubectl -n taxdocs-dev get endpoints $svc
  docker inspect taxdocs_dev-${svc}-1 -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}'
done
```

**Fix:**
1. Edit `overlays/dev/05-taxdocs-dev.external-datastores.yaml` — update each `Endpoints.subsets[].addresses[].ip` to the container's current bridge IP.
2. This repo's ArgoCD app (`taxdocs-api-dev`) tracks branch **`w6d2-implementation`**, not necessarily whatever branch you're working on (`kubectl -n argocd get application taxdocs-api-dev -o jsonpath='{.spec.source.targetRevision}'` to confirm) — commit and push the fix there, or it will never sync.
3. Force ArgoCD to pick it up immediately instead of waiting on its poll interval:
   ```bash
   kubectl -n argocd annotate application taxdocs-api-dev argocd.argoproj.io/refresh=hard --overwrite
   ```
4. Verify: `kubectl -n taxdocs-dev get endpoints postgres redis kafka mongo` should match the containers' current IPs, and `kubectl -n taxdocs-dev get pods -l app.kubernetes.io/name=taxdocs-api` should go to `3/3 Running`.

Don't delete/recreate the k3d nodes' `docker network connect` to `taxdocs_dev_taxdocs_net` — that bridge is what lets the cluster nodes route to these containers at all; only the IPs inside it drift.
