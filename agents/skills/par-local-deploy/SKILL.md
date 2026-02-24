---
name: par-local-deploy
description: This skill should be used when the user asks to load a Docker image onto the Lima VM, deploy an agent image to the Kind cluster, redeploy or upgrade the Datadog operator, update a DatadogAgent CR, apply operator configuration changes, or restart the PAR local development environment. Covers building images, loading them into the par-dev Lima VM's Kind cluster, and managing the operator and DatadogAgent CR lifecycle.
---

# PAR Local Deploy

This skill manages the local PAR development environment: loading Docker images into the `par-dev` Lima VM's Kind cluster and deploying/redeploying the Datadog Operator with updated configuration.

## Environment

- **Lima VM:** `par-dev` (Apple Silicon / arm64)
- **Kind cluster:** `par-dev` (runs inside the Lima VM)
- **Helm releases:** `datadog-operator` (operator), `datadog-par` (standalone PAR, if used)
- **Config files:** `~/dd/par-tools/operator/`
- **Datadog site:** `datad0g.com` (staging)

## Configuration Files

| File | Purpose |
|------|---------|
| `~/dd/par-tools/operator/values.yaml` | Helm values for the custom operator |
| `~/dd/par-tools/operator/rbac.yaml` | ClusterRoleBinding granting operator cluster-admin |
| `~/dd/par-tools/operator/datadog-agent.yaml` | DatadogAgent CR with PAR enabled via annotations |
| `~/dd/par-tools/operator/scripts.yaml` | Predefined scripts for PAR (mounted as secret) |

When the user requests changes to the operator config, DatadogAgent CR, RBAC, or scripts, edit the appropriate file above before redeploying. Read `references/config-files.md` for the current contents and schema details.

## Workflows

### Loading a Node Agent Image

Use this workflow when the user has built a new `datadog-agent` image and needs it loaded onto the Kind cluster.

**Step 1: Determine the image source.**

Ask the user which image to load. Common scenarios:
- **Locally built image** — Built with `dda inv omnibus.docker-build --arch=arm64`, tagged as `localhost/datadog-agent:local`
- **CI image** — A specific CI registry path (e.g., `registry.ddbuild.io/ci/datadog-agent/agent:v<pipeline>-<commit>-7-arm64`). Must be arm64.
- **Custom tag** — The user may have tagged an image differently

**Step 2: Save, copy, and load the image.**

```bash
# Save the Docker image to a tar file
docker save <image:tag> -o /tmp/agent-image.tar

# Copy the tar file into the Lima VM
limactl copy /tmp/agent-image.tar par-dev:/tmp/agent-image.tar

# Inside the Lima VM: load into Docker, then into Kind, then clean up
limactl shell par-dev -- bash -c 'docker load -i /tmp/agent-image.tar && kind load docker-image <image:tag> --name par-dev && rm /tmp/agent-image.tar'

# Clean up local tar file
rm /tmp/agent-image.tar
```

**Step 3: Update the DatadogAgent CR if the image tag changed.**

If the image name/tag differs from what's in `~/dd/par-tools/operator/datadog-agent.yaml`, update the `spec.override.nodeAgent.image.name` field and re-apply:

```bash
limactl shell par-dev -- kubectl apply -f ~/dd/par-tools/operator/datadog-agent.yaml
```

If the image tag is the same (just a rebuilt image), delete the existing node agent pods to force a pull of the new image:

```bash
limactl shell par-dev -- kubectl delete pods -l agent.datadoghq.com/component=agent
```

### Loading a Custom Operator Image

Use this workflow when the user has built a custom `datadog-operator` image.

**Step 1: Build and tag the operator image.**

```bash
# From the datadog-operator repo:
make docker-build
docker tag gcr.io/datadoghq/operator:v1.23.0 datadog-operator:<custom-tag>
```

**Step 2: Load into Kind cluster.**

```bash
docker save datadog-operator:<custom-tag> -o /tmp/datadog-operator.tar
limactl copy /tmp/datadog-operator.tar par-dev:/tmp/datadog-operator.tar
limactl shell par-dev -- bash -c 'docker load -i /tmp/datadog-operator.tar && kind load docker-image datadog-operator:<custom-tag> --name par-dev && rm /tmp/datadog-operator.tar'
rm /tmp/datadog-operator.tar
```

**Step 3: Install or upgrade the operator via Helm.**

For a fresh install:
```bash
limactl shell par-dev -- helm install datadog-operator datadog/datadog-operator \
  --set image.repository=datadog-operator \
  --set image.tag=<custom-tag> \
  --set image.pullPolicy=Never \
  --set image.doNotCheckTag=true
```

For an upgrade (operator already installed):
```bash
limactl shell par-dev -- helm upgrade datadog-operator datadog/datadog-operator \
  --set image.repository=datadog-operator \
  --set image.tag=<custom-tag> \
  --set image.pullPolicy=Never \
  --set image.doNotCheckTag=true
```

Alternatively, use the values file:
```bash
limactl shell par-dev -- helm upgrade datadog-operator datadog/datadog-operator -f ~/dd/par-tools/operator/values.yaml
```

### Redeploying with Updated Configuration

Use this workflow when the user wants to change the DatadogAgent CR, RBAC, secrets, or operator settings.

**Step 1: Edit the configuration file(s).**

Make the requested changes to the relevant file(s) in `~/dd/par-tools/operator/`. Common changes include:
- Adding actions to the allowlist in `datadog-agent.yaml`
- Changing the agent or cluster-agent image
- Updating PAR annotations or environment variables
- Modifying RBAC permissions
- Adding/updating predefined scripts

**Step 2: Apply the changes.**

For DatadogAgent CR changes:
```bash
limactl shell par-dev -- kubectl apply -f ~/dd/par-tools/operator/datadog-agent.yaml
```

For RBAC changes:
```bash
limactl shell par-dev -- kubectl apply -f ~/dd/par-tools/operator/rbac.yaml
```

For script secret changes (must delete and recreate):
```bash
limactl shell par-dev -- kubectl delete secret par-script-credentials --ignore-not-found
limactl shell par-dev -- kubectl create secret generic par-script-credentials --from-file=script.yaml=$HOME/dd/par-tools/operator/scripts.yaml
```

**Step 3: Verify the changes propagated.**

```bash
# Check operator logs for reconciliation
limactl shell par-dev -- kubectl logs -l app.kubernetes.io/name=datadog-operator --tail=20

# Check node agent pods are running
limactl shell par-dev -- kubectl get pods -l agent.datadoghq.com/component=agent

# Check PAR container logs specifically
limactl shell par-dev -- kubectl logs <pod-name> -c private-action-runner --tail=20
```

If annotation changes don't propagate to the `DatadogAgentInternal` resource, force a re-sync by deleting the DDAI (the operator will recreate it):
```bash
limactl shell par-dev -- kubectl delete datadogagentinternal --all
```

### Full Fresh Deployment

Use when setting up from scratch or doing a full reset.

```bash
# 1. Ensure Lima VM is running
limactl start par-dev

# 2. Install operator (or upgrade if exists)
limactl shell par-dev -- helm install datadog-operator datadog/datadog-operator -f ~/dd/par-tools/operator/values.yaml

# 3. Apply RBAC
limactl shell par-dev -- kubectl apply -f ~/dd/par-tools/operator/rbac.yaml

# 4. Create secrets
limactl shell par-dev -- kubectl create secret generic datadog-secret \
  --from-literal api-key=$DD_API_KEY \
  --from-literal app-key=$DD_APP_KEY
limactl shell par-dev -- kubectl create secret generic par-script-credentials \
  --from-file=script.yaml=$HOME/dd/par-tools/operator/scripts.yaml

# 5. Apply DatadogAgent CR
limactl shell par-dev -- kubectl apply -f ~/dd/par-tools/operator/datadog-agent.yaml
```

### Lima VM Management

```bash
# Start the VM
limactl start par-dev

# Stop the VM
limactl stop par-dev

# Shell into the VM
limactl shell par-dev

# Run kubectl commands from the host
limactl shell par-dev -- kubectl get pods
```

## Critical Gotchas

Read `references/gotchas.md` for the full list. The most important ones:

1. **arm64 only**: CI publishes amd64 images. The Lima VM is arm64. Always build locally or use arm64-tagged CI images.
2. **`doNotCheckTag: true`**: Required for non-semver custom image tags in the operator Helm chart.
3. **RBAC first**: Without cluster-admin RBAC, the operator reconciles silently with no pods created — only errors in operator logs.
4. **`DD_HOSTNAME` required on Kind**: Both `agent` and `private-action-runner` containers need `DD_HOSTNAME` set via `spec.override.nodeAgent.containers`, or the core agent exits with "unable to reliably determine the host name."
5. **Do NOT mount hostPath over `/etc/datadog-agent/auth`**: Breaks Remote Config with TLS handshake failures.
6. **DDAI annotation sync lag**: If ConfigMap doesn't update after annotation changes, delete the DDAI to force re-sync.
