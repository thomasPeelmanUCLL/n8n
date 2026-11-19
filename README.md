# n8n on Kubernetes with private Ollama

Plain Kubernetes manifests deployed through Argo CD. The companion AI VM is documented in [nixos-cloudstack-ollama](https://github.com/thomasPeelmanUCLL/nixos-cloudstack-ollama).

## Verified status (2026-11-19)

Operator output confirmed Argo Synced/Healthy at 608d2df5dda7144efe507dac2058aa112d8447c2, both application/database pods Ready with zero restarts, and bound PVCs. The observed running versions were n8n 2.48.6 and PostgreSQL 17.12. A Basic-authenticated POST to Ollama model qwen2.5:1.5b returned Hello!, and Edit Fields successfully extracted message.content. These are point-in-time operator observations, not ongoing monitoring or automated deployment tests.

## Architecture

```text
Browser -> HTTPS Nginx Proxy Manager -> nginx Ingress -> n8n:5678
n8n -> n8n-postgres.n8n.svc.cluster.local:5432
n8n -> HTTPS 10.1.1.209:443 -> nginx Basic Auth -> Ollama 127.0.0.1:11434
```

Public editor: https://n8n.iswleuven.be. Public TLS is terminated by Nginx Proxy Manager; the Ingress manifest uses HTTP to Service n8n port 80, targeting container port 5678. The private Ollama TLS connection is separate.

## Repository map and current settings

- argocd/n8n-application.yaml: Application n8n in argocd; tracks main, path k8s; automated sync enabled, prune false, selfHeal false, CreateNamespace true.
- k8s/namespace.yaml: namespace n8n.
- k8s/n8n.yaml: one n8n replica, Service, 5Gi ReadWriteOnce PVC n8n-data mounted at /home/node/.n8n.
- k8s/postgres.yaml: one PostgreSQL StatefulSet replica, Service and 10Gi ReadWriteOnce claim template. Existing claim: data-n8n-postgres-0.
- k8s/ingress.yaml: nginx Ingress for n8n.iswleuven.be.
- k8s/ollama-ca-configmap.yaml: public CA in ConfigMap ollama-ca, key ca.crt. Do not reintroduce a duplicate ConfigMap manifest.

Current images are n8nio/n8n:latest and postgres:17-alpine, not immutable pins. n8n requests 100m CPU/256Mi memory and has a 2Gi memory limit. All probes use /healthz with 5s timeouts. Startup allows 30 failures at 10s intervals; readiness 3 at 10s; liveness 5 at 15s. Requests are scheduling reservations, not guarantees of adequate capacity. PostgreSQL currently has no resource requests/limits or explicit probes in this repository. Review capacity using real measurements.

## Deployment prerequisites

Use an existing Kubernetes cluster, Argo CD and nginx Ingress controller; working storage provisioning; DNS/public proxy routing; and private connectivity from pods to 10.1.1.209:443. A Kubernetes Service does not establish that private route. The observed PVC storage class was local-path; preserve node storage and maintain off-node backups.

Provision Secret n8n-secrets in namespace n8n through your approved secret-management process before starting workloads. Required keys:

- DB_POSTGRESDB_PASSWORD: same database password consumed by n8n and PostgreSQL.
- N8N_ENCRYPTION_KEY: stable encryption key; preserve it with database backups or stored n8n credentials may become unusable.

Never commit Secret values, passwords, private keys, htpasswd hashes, tokens or kubeconfig. Do not rotate database passwords or encryption keys casually. On an initialized PostgreSQL volume, changing POSTGRES_PASSWORD alone is not a database password migration.

In Argo CD UI create/review the application using this repository, main and k8s, matching argocd/n8n-application.yaml. Keep pruning and self-healing disabled as configured. Review sync changes before applying. Documentation does not enable either option.

## Public proxy: required WebSocket support

Enable Websockets Support on the Nginx Proxy Manager Proxy Host for n8n.iswleuven.be. This fixed the observed Lost connection to the server error during workflow execution. This setting is on the public n8n proxy, not the private Ollama nginx service.

The live logs also showed ERR_ERL_UNEXPECTED_X_FORWARDED_FOR. The repository currently does not set N8N_PROXY_HOPS or N8N_EDITOR_BASE_URL; their addition remains a recommendation, not a completed manifest change. For a verified path through Nginx Proxy Manager and Kubernetes nginx Ingress, review adding:

```yaml
- name: N8N_PROXY_HOPS
  value: "2"
- name: N8N_EDITOR_BASE_URL
  value: https://n8n.iswleuven.be/
```

Use the actual number of trusted reverse proxies, not a blanket trust-all setting. Ensure forwarded headers are correctly supplied by trusted proxies and direct untrusted access is restricted. Existing N8N_HOST, N8N_PROTOCOL and N8N_WEBHOOK_URL already reference the public HTTPS hostname. After any Deployment template change, let Argo roll out the updated pod and inspect new logs.

## Operations and health checks

```sh
kubectl -n argocd get application n8n -o json | jq '{sync: .status.sync.status, revision: .status.sync.revision, health: .status.health.status}'
kubectl -n n8n get pods,pvc
kubectl -n n8n logs deployment/n8n -c n8n --since=10m --timestamps --tail=100
kubectl -n n8n logs statefulset/n8n-postgres --tail=80
kubectl -n n8n exec statefulset/n8n-postgres -- psql -U n8n -d n8n -c 'SELECT version();'
```

Inspect live pod specs as well as Git; a successful sync operation does not alone demonstrate application readiness. If an intentional restart is needed after syncing trust changes:

```sh
kubectl -n n8n rollout restart deployment/n8n
kubectl -n n8n rollout status deployment/n8n --timeout=300s
```

Do not repeatedly restart a healthy pod as a substitute for diagnosis. Exit code 137 alone does not prove OOM; check termination reason, events, memory and node pressure.

## Ollama CA trust

ConfigMap ollama-ca is mounted read-only at /etc/ollama-ca/ca.crt. NODE_EXTRA_CA_CERTS points there. Correct SHA-256 fingerprint:

```text
96:37:75:CA:D6:89:42:BA:8D:8A:E7:F0:98:E2:EA:79:A7:61:C8:B2:BE:05:C1:83:21:AB:7E:01:EE:DD:4A:ED
```

The source public certificate is /root/ollama-ca/ca.crt on ollama-ai. A different /root/ca.crt had the same CN but the wrong signing key. Use signature verification, not issuer-name equality, to select a CA. Never copy CA/server private keys into n8n. Sync changes before restarting Node to load updated trust.

Run this unauthenticated test from the pod:

```sh
kubectl -n n8n exec deployment/n8n -c n8n -- node -e '
const fs = require("fs");
const { X509Certificate } = require("crypto");
const https = require("https");
console.log("CA fingerprint:", new X509Certificate(fs.readFileSync("/etc/ollama-ca/ca.crt")).fingerprint256);
const req = https.get("https://10.1.1.209/api/chat", res => {
  console.log("HTTP status:", res.statusCode);
  console.log("TLS authorized:", res.socket.authorized);
  res.resume();
  if (res.statusCode !== 401 || !res.socket.authorized) process.exitCode = 1;
});
req.setTimeout(10000, () => req.destroy(new Error("Connection timed out")));
req.on("error", err => { console.error(err.code || "ERROR", err.message); process.exitCode = 1; });
'
```

Expected: correct fingerprint, HTTP status 401, TLS authorized true. The GET is intentionally only an unauthenticated TLS/auth challenge test, not a generation call. Keep certificate verification enabled. A certificate verification error is not fixed by opening public firewall ports.

## Working connection-test workflow

```text
Manual Trigger -> HTTP Request -> Edit Fields
```

HTTP Request settings:

| Setting | Value |
|---|---|
| Method | POST |
| URL | https://10.1.1.209/api/chat |
| Authentication | Generic Credential Type -> Basic Auth |
| Credential | Create/save/select an HTTP Basic Auth credential, e.g. Ollama Proxy |
| Username | n8n |
| Password | Plaintext password chosen for the proxy htpasswd entry; not a hash or editor login |
| Send Body | On |
| Body Content Type / Specify Body | JSON / Using JSON |
| Timeout | 180000 ms |
| Ignore SSL Issues | Off |

```json
{
  "model": "qwen2.5:1.5b",
  "messages": [{"role": "user", "content": "Say hello in one short sentence."}],
  "stream": false
}
```

Check ollama list on the VM if changing models. stream false requests a complete response. In Edit Fields add a String field answer with expression:

```js
{{ $json.message.content }}
```

To output only answer, disable inclusion of other input fields. Save the working workflow as a baseline. It is stored in the database; this repository does not currently include a workflow JSON export. If exporting later, remove sensitive data and stale credential references before committing.

## Troubleshooting learned from this deployment

| Symptom | Next check |
|---|---|
| Lost connection to server | Public Nginx Proxy Manager Websockets Support; then pod restarts, logs and browser /rest/push connection |
| Credentials not found | Create/select an accessible saved Basic Auth credential; old workflow IDs can refer to deleted credentials |
| HTTP 405 | Inspect actual Request.method: the resolved failure sent GET with a body. Set POST in the editable node, rerun and check the new timestamp |
| HTTP 401 on generation | Verify the proxy credential; unauthenticated TLS-test 401 is expected |
| UNABLE_TO_VERIFY_LEAF_SIGNATURE | Verify mounted CA fingerprint and server signature; do not disable TLS |
| ERR_ERL_UNEXPECTED_X_FORWARDED_FOR | Review proxy hops and trusted forwarded headers |
| Database connection refused | Check PostgreSQL pod, Service endpoints and recent logs; distinguish recovered startup errors from recurring failures |
| Task runner grant-token error | Check whether a later Registered runner message appears; do not adjust TTL based only on a recovered startup error |
| Pending pod or volume attachment | Inspect pod events, resource availability and ReadWriteOnce storage topology before restarting |

PostgreSQL 17 was observed initializing a fresh database, not upgrading old PostgreSQL 16 files. Do not claim an automatic migration occurred. Major-version upgrades require a planned compatible migration and backups. Never delete a PVC merely to clear a version error.

## Pending maintenance, not implemented by these docs

- Pin the tested n8n image/version or digest instead of latest; review database image pinning too.
- Add and verify appropriate proxy-hop/editor URL settings through Git.
- Back up PostgreSQL, n8n persistent data and the existing encryption key; keep secret backups protected and off-node, and test restoration.
- Review single-replica rollout strategy and local ReadWriteOnce storage to avoid overlapping instances or volume-attachment problems.
- Review database probes/resources, node capacity and actual memory use.
- Track private server certificate renewal; observed expiry was 2027-11-18.
- Validate model outputs and require human approval for consequential actions. Do not grant the model cluster-admin credentials.

See the companion [private API runbook](https://github.com/thomasPeelmanUCLL/nixos-cloudstack-ollama/blob/main/docs/private-api.md) for VM certificate verification, nginx operations and password rotation. Documentation-only commits do not apply these pending configuration changes.
