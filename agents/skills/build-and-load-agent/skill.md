---
name: build-and-load-agent
description: Use this skill when the user asks to build the datadog-agent locally and load it onto the Kind cluster in the Lima VM. Covers building the arm64 Docker image with dda inv, saving and copying the tar to the Lima VM, loading it into Kind, and restarting the agent pods.
---

# Build and Load Agent

This skill builds the `datadog-agent` Docker image locally for arm64, loads it into the `par-dev` Kind cluster inside the Lima VM, and restarts the node agent pods.

## Environment

- **Agent repo:** `~/go/src/github.com/DataDog/datadog-agent`
- **Lima VM:** `par-dev` (Apple Silicon / arm64)
- **Kind cluster:** `par-dev` (runs inside the Lima VM)
- **Image tag after build:** `localhost/datadog-agent:local`

## Workflow

### Step 1: Build the image

Run from the `datadog-agent` repo root:

```bash
cd ~/go/src/github.com/DataDog/datadog-agent
dda inv omnibus.docker-build --arch=arm64
```

This produces a Docker image tagged `localhost/datadog-agent:local`. The build takes several minutes.

### Step 2: Save, copy, and load into Kind

```bash
# Save the image to a tar file
docker save localhost/datadog-agent:local -o /tmp/agent-image.tar

# Copy into the Lima VM
limactl copy /tmp/agent-image.tar par-dev:/tmp/agent-image.tar

# Load into Docker inside the VM, then into Kind, then clean up
limactl shell par-dev -- bash -c 'docker load -i /tmp/agent-image.tar && kind load docker-image localhost/datadog-agent:local --name par-dev && rm /tmp/agent-image.tar'

# Clean up local tar
rm /tmp/agent-image.tar
```

### Step 3: Restart agent pods

If the image tag (`localhost/datadog-agent:local`) is already set in the DatadogAgent CR, just delete the pods to force a fresh pull from the local image:

```bash
limactl shell par-dev -- kubectl delete pods -l agent.datadoghq.com/component=agent
```

If the CR's `spec.override.nodeAgent.image.name` needs to be updated to `localhost/datadog-agent:local`, edit `~/dd/par-tools/operator/datadog-agent.yaml` first, then apply:

```bash
limactl shell par-dev -- kubectl apply -f ~/dd/par-tools/operator/datadog-agent.yaml
```

### Step 4: Verify

```bash
# Watch pods come back up
limactl shell par-dev -- kubectl get pods -l agent.datadoghq.com/component=agent

# Check PAR container is running
limactl shell par-dev -- kubectl logs <pod-name> -c private-action-runner --tail=20
```

## Critical Gotchas

1. **arm64 only**: The Lima VM is arm64. Always pass `--arch=arm64` to the build command.
2. **`imagePullPolicy: Never`**: The Kind cluster cannot pull from `localhost`. The DatadogAgent CR must set `imagePullPolicy: Never` (or `IfNotPresent`) for the node agent, otherwise the pod will fail with `ErrImagePull`.
3. **`doNotCheckTag: true`**: Required in the Datadog Operator Helm values for non-semver image tags.
4. **`DD_HOSTNAME` required**: Both `agent` and `private-action-runner` containers must have `DD_HOSTNAME` set, or the core agent exits on startup.
5. **Save/copy/load is required**: Kind clusters inside Lima cannot access the host Docker daemon directly. The tar-based copy is the only reliable method.
