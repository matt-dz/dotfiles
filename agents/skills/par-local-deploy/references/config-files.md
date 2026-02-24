# PAR Local Deploy — Configuration File Reference

All configuration files live at `~/dd/par-tools/operator/`. These are the source-of-truth files that get applied to the Kind cluster.

## values.yaml — Operator Helm Values

**Path:** `~/dd/par-tools/operator/values.yaml`

Controls how the Datadog Operator Helm chart is installed. Key fields:

- `image.repository` — Image repository (use `datadog-operator` for locally loaded images)
- `image.tag` — Image tag (custom tags require `doNotCheckTag: true`)
- `image.pullPolicy` — Set to `Never` for locally loaded images (no registry pull)
- `image.doNotCheckTag` — Must be `true` for non-semver tags
- `watchNamespaces` — Set to `[""]` to watch all namespaces

## rbac.yaml — Operator RBAC

**Path:** `~/dd/par-tools/operator/rbac.yaml`

ClusterRoleBinding granting the operator's service account `cluster-admin`. Required for the operator to create ClusterRoles/ClusterRoleBindings for agent components.

Key fields to verify:
- `subjects[0].namespace` — Must match the namespace where the operator runs (typically `default`)
- `subjects[0].name` — Must match the operator's service account name (typically `datadog-operator`)

## datadog-agent.yaml — DatadogAgent CR

**Path:** `~/dd/par-tools/operator/datadog-agent.yaml`

The DatadogAgent custom resource that the operator reconciles into the actual agent deployment.

### PAR Annotations

PAR is enabled entirely through annotations:

```yaml
metadata:
  annotations:
    agent.datadoghq.com/private-action-runner-enabled: "true"
    agent.datadoghq.com/private-action-runner-configdata: |
      private_action_runner:
        enabled: true
        self_enroll: true
        actions_allowlist:
          - com.datadoghq.script.testConnection
          - com.datadoghq.ddagent.testConnection
          - ...
```

To add new actions to PAR, add the fully-qualified action name to the `actions_allowlist` array.

### Image Configuration

```yaml
spec:
  override:
    nodeAgent:
      image:
        name: localhost/datadog-agent:local    # Local build or CI arm64 image
    clusterAgent:
      image:
        name: docker.io/datadog/cluster-agent:7.76.0-rc.3    # Public RC image
```

- **Node agent**: Must contain the PAR binary. Use locally built images or arm64 CI images.
- **Cluster agent**: The public RC image works fine since it doesn't need PAR.

### Required Environment Variables

Both `agent` and `private-action-runner` containers need `DD_HOSTNAME` on Kind:

```yaml
containers:
  agent:
    env:
      - name: DD_HOSTNAME
        valueFrom:
          fieldRef:
            fieldPath: spec.nodeName
  private-action-runner:
    env:
      - name: DD_HOSTNAME
        valueFrom:
          fieldRef:
            fieldPath: spec.nodeName
```

### Volume Mounts for Predefined Scripts

```yaml
volumes:
  - name: par-scripts
    secret:
      secretName: par-script-credentials
containers:
  private-action-runner:
    volumeMounts:
      - name: par-scripts
        mountPath: /etc/dd-action-runner/config/credentials
        readOnly: true
```

### Global Configuration

```yaml
spec:
  global:
    credentials:
      apiSecret:
        secretName: datadog-secret
        keyName: api-key
      appSecret:
        secretName: datadog-secret
        keyName: app-key
    clusterName: kind-par-dev
    site: datad0g.com
```

## scripts.yaml — Predefined Scripts

**Path:** `~/dd/par-tools/operator/scripts.yaml`

Defines scripts available to the `runPredefinedScript` action. Mounted into the PAR container as a Kubernetes secret at `/etc/dd-action-runner/config/credentials/script.yaml`.

Schema:

```yaml
schemaId: script-credentials-v1
runPredefinedScript:
  <script-name>:
    command: ["<binary>", "<args>"]
    parameterSchema:      # Optional: JSON Schema for parameters
      properties:
        <param-name>:
          type: string
      required:
        - <param-name>
```

Parameters are injected via Go template syntax: `{{ parameters.<name> }}`.

### Adding a New Predefined Script

1. Add the script definition to `~/dd/par-tools/operator/scripts.yaml`
2. Delete and recreate the secret:
   ```bash
   limactl shell par-dev -- kubectl delete secret par-script-credentials --ignore-not-found
   limactl shell par-dev -- kubectl create secret generic par-script-credentials \
     --from-file=script.yaml=$HOME/dd/par-tools/operator/scripts.yaml
   ```
3. Restart agent pods to pick up the new mount:
   ```bash
   limactl shell par-dev -- kubectl delete pods -l agent.datadoghq.com/component=agent
   ```
