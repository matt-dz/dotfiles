# PAR Local Deploy — Gotchas

Critical pitfalls discovered during local PAR development on the `par-dev` Lima VM and Kind cluster.

## Image and Architecture

### 1. arm64 vs amd64
The Lima VM and Kind cluster run arm64 (Apple Silicon). CI only publishes amd64 images to the E2E and public registries. Must build the operator image locally with `make docker-build` for arm64. For agent images, use arm64-tagged CI images or build locally with `dda inv omnibus.docker-build --arch=arm64`.

### 2. `doNotCheckTag`
The Datadog Operator Helm chart validates the image tag as semver. Custom tags (like `par-privileged`) need `doNotCheckTag: true` in the Helm values, or the template rendering fails.

### 3. Agent image must include the PAR binary
The standard public agent image (`gcr.io/datadoghq/agent:7.75.0`) does not contain `/opt/datadog-agent/embedded/bin/privateactionrunner`. The PAR binary is only present in CI builds or release candidates that include the `privateactionrunner` package. Only the CI image `registry.ddbuild.io/ci/datadog-agent/agent:v<pipeline>-<commit>-7-arm64` or locally-built images are confirmed to work.

### 4. Stock operator image lacks PAR
The PAR feature was added to the operator in `main` branch (post-v1.22.0). The default Helm chart installs v1.22.0 which doesn't have PAR support. Must use a custom image built from a branch that includes PAR.

## RBAC and Permissions

### 5. RBAC must be applied before the DatadogAgent CR
The operator service account needs cluster-admin to create all the ClusterRoles the agent components require. Without it, reconciliation fails silently — no pods are created, errors only appear in operator logs.

### 6. RBAC namespace must match the operator
The `rbac.yaml` ClusterRoleBinding must reference the correct namespace where the operator's service account lives. If the operator is in `default`, the binding must say `namespace: default`, not `datadog-operator`. Verify with:

```bash
limactl shell par-dev -- kubectl get serviceaccount datadog-operator -o yaml
```

## Kind Cluster Specifics

### 7. `DD_HOSTNAME` required on Kind
Kind clusters can't reliably detect the hostname. Both the `agent` and `private-action-runner` containers need the `DD_HOSTNAME` env var set via `spec.override.nodeAgent.containers`:

```yaml
env:
  - name: DD_HOSTNAME
    valueFrom:
      fieldRef:
        fieldPath: spec.nodeName
```

Without it, the core agent exits with "unable to reliably determine the host name" and PAR fails self-enrollment with the same error.

## Remote Config and Identity

### 8. Do NOT mount hostPath over `/etc/datadog-agent/auth`
Mounting a hostPath at `/etc/datadog-agent/auth` to persist the PAR identity file breaks Remote Config. The PAR and core agent end up with different IPC certs, causing TLS handshake failures (`x509: ECDSA verification failure`). The PAR cannot receive work from Datadog. Identity persistence is not worth breaking RC.

## Operator Reconciliation

### 9. Deleting DatadogAgentInternal cascades
Deleting the DDAI resource removes ALL managed resources (daemonsets, deployments, clusterroles). The operator will recreate them, but only if RBAC is correct. Be careful — this is a full teardown.

### 10. DDAI annotation sync lag
Changes to annotations on the `DatadogAgent` CR may not immediately propagate to the `DatadogAgentInternal` resource. If the ConfigMap doesn't update after an annotation change, delete the DDAI to force a full re-sync:

```bash
limactl shell par-dev -- kubectl delete datadogagentinternal --all
```

Be aware this triggers gotcha #9 — all managed resources will be deleted and recreated.

## Secrets

### 11. Secret updates require delete + recreate
Kubernetes secrets created with `kubectl create secret` are immutable in the sense that `kubectl apply` won't update them the way you'd expect. To update a secret (e.g., changing predefined scripts):

```bash
limactl shell par-dev -- kubectl delete secret par-script-credentials --ignore-not-found
limactl shell par-dev -- kubectl create secret generic par-script-credentials \
  --from-file=script.yaml=$HOME/dd/par-tools/operator/scripts.yaml
```

After updating the secret, restart the affected pods so they pick up the new mount:

```bash
limactl shell par-dev -- kubectl delete pods -l agent.datadoghq.com/component=agent
```
