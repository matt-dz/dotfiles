# dd-source PAR Bundle Reference

Detailed file templates and patterns for registering a new PAR bundle in the `dd-source` repository. The dd-source side acts as a stub — it registers the bundle and defines the action metadata so the Action Platform can route tasks to the PAR. The real execution happens on the agent side.

## Go Backend (private-runner)

### Files to Create

#### `domains/actionplatform/apps/private-runner/src/private-bundles/com/datadoghq/ddagent/<bundlename>/entrypoint.go`

Stub bundle struct. Does not accept FX-injected dependencies (those are agent-side only).

```go
package com_datadoghq_ddagent_<bundlename>

import (
    "github.com/nicktrav/dd-source/domains/actionplatform/apps/private-runner/src/private-bundles/types"
)

// BundleNameBundle is the PAR bundle for <description>.
type BundleNameBundle struct{}

// NewBundleName creates a new bundle instance.
func NewBundleName() *BundleNameBundle {
    return &BundleNameBundle{}
}

// Register registers all actions for this bundle.
func (b *BundleNameBundle) Register() map[string]types.ActionHandler {
    return map[string]types.ActionHandler{
        "actionName": &ActionNameHandler{},
    }
}
```

#### `domains/actionplatform/apps/private-runner/src/private-bundles/com/datadoghq/ddagent/<bundlename>/<action_name>.go`

Stub action handler. Extracts inputs but returns them as output — the agent does the real work.

```go
package com_datadoghq_ddagent_<bundlename>

import (
    "context"

    "github.com/nicktrav/dd-source/domains/actionplatform/apps/private-runner/src/private-bundles/types"
)

// ActionNameInputs defines the inputs for the action.
type ActionNameInputs struct {
    // Define input fields matching the action schema.
    Section string `json:"section,omitempty"`
}

// ActionNameOutputs defines the outputs for the action.
type ActionNameOutputs struct {
    // Define output fields.
    Result interface{} `json:"result"`
}

// ActionNameHandler handles the actionName action.
type ActionNameHandler struct{}

// Run extracts inputs and returns them (actual execution is on the agent).
func (h *ActionNameHandler) Run(ctx context.Context, task *types.Task) (interface{}, error) {
    inputs, err := types.ExtractInputs[ActionNameInputs](task)
    if err != nil {
        return nil, err
    }

    return ActionNameOutputs{
        Result: inputs,
    }, nil
}
```

#### `domains/actionplatform/apps/private-runner/src/private-bundles/com/datadoghq/ddagent/<bundlename>/BUILD.bazel`

```python
load("@nicktrav_rules_go//go:def.bzl", "dd_go_library")

dd_go_library(
    name = "go_default_library",
    srcs = [
        "entrypoint.go",
        "<action_name>.go",
    ],
    importpath = "github.com/nicktrav/dd-source/domains/actionplatform/apps/private-runner/src/private-bundles/com/datadoghq/ddagent/<bundlename>",
    visibility = ["//visibility:public"],
    deps = [
        "//domains/actionplatform/apps/private-runner/src/private-bundles/types:go_default_library",
        "//domains/actionplatform/apps/private-runner/src/private-connection:go_default_library",
    ],
)
```

### Files to Modify

#### `domains/actionplatform/apps/private-runner/src/private-bundles/registry.go`

Add import and bundle registration:

```go
import (
    // ... existing imports ...
    com_datadoghq_ddagent_<bundlename> "github.com/nicktrav/dd-source/domains/actionplatform/apps/private-runner/src/private-bundles/com/datadoghq/ddagent/<bundlename>"
)

// In the bundles map:
"com.datadoghq.ddagent.<bundlename>": com_datadoghq_ddagent_<bundlename>.NewBundleName(),
```

> **Note:** Do NOT modify `registry_kubeapiserver.go` in dd-source. It is a slimmed-down registry for the Cluster Agent and doesn't need agent-specific bundles.

#### `domains/actionplatform/apps/private-runner/src/private-bundles/BUILD.bazel`

Add dependency:

```python
deps = [
    # ... existing deps ...
    "//domains/actionplatform/apps/private-runner/src/private-bundles/com/datadoghq/ddagent/<bundlename>:go_default_library",
]
```

#### `domains/actionplatform/shared/libs/go/pool/pools_test.go`

Add the bundle to the pool assignment test:

```go
// In the pool assignments map:
"com.datadoghq.ddagent.<bundlename>": pool.Generic,
```

## TypeScript Frontend (wf-actions-worker)

All files go under:
`domains/workflow/actionplatform/apps/wf-actions-worker/src/runner/bundles/com.datadoghq.ddagent.<bundlename>/`

### Files to Create

#### `actions.ts`

Bundle metadata with JSDoc annotations.

```typescript
/**
 * @bundleTitle <Human-readable bundle title>
 * @bundleIcon <icon-name>
 * @bundleStability dev
 * @bundleEnvironments ON_PREMISES
 */
```

#### `actions/<action-name>.ts`

Action stub. Returns error since actual work is done in Go.

```typescript
import { resultErr } from '@actionplatform/action-runner-core';
import type { ActionContext } from '@actionplatform/action-runner-core';

/**
 * @actionTitle <Human-readable action title>
 */
export default async function actionName(
  context: ActionContext,
  inputs: ActionNameInputs,
): Promise<ActionNameOutputs> {
  return resultErr('Not implemented - execution handled by Private Action Runner');
}

export interface ActionNameInputs {
  /** Optional description of input field */
  section?: string;
}

export interface ActionNameOutputs {
  result: unknown;
}
```

#### `entrypoint.gen.ts`

Generated-style entry point implementing the `Bundle` interface.

```typescript
import type { Bundle } from '@actionplatform/action-runner-core';

const bundle: Bundle = {
  actions: {
    'actionName': () => import('./actions/<action-name>'),
  },
};

export default bundle;
```

#### `manifest.gen.json`

JSON Schema types, action metadata, and access configuration.

```json
{
  "bundleId": "com.datadoghq.ddagent.<bundlename>",
  "actions": {
    "actionName": {
      "title": "<Human-readable action title>",
      "description": "<Action description>",
      "inputSchema": {
        "type": "object",
        "properties": {
          "section": {
            "type": "string",
            "description": "<Input description>"
          }
        }
      },
      "outputSchema": {
        "type": "object",
        "properties": {
          "result": {
            "description": "<Output description>"
          }
        }
      },
      "accessModes": ["read"],
      "accessLevel": "standard"
    }
  }
}
```

#### `keywords.json`

```json
{
  "actionName": ["keyword1", "keyword2", "keyword3"]
}
```

#### `sentence-templates.json`

```json
{
  "actionName": "<Natural language template, e.g. 'Get the {{section}} status of the agent'>"
}
```

#### `access-control.json`

```json
{
  "accessModes": ["read"],
  "accessLevel": "standard"
}
```

#### `release-dates.json`

Empty for new bundles:

```json
{}
```

### Files to Modify

#### `domains/workflow/actionplatform/apps/wf-actions-worker/src/runner/bundles/bundles.gen.ts`

Add lazy import entry:

```typescript
'com.datadoghq.ddagent.<bundlename>': () => import('./com.datadoghq.ddagent.<bundlename>/entrypoint.gen'),
```

#### `domains/workflow/actionplatform/apps/wf-actions-worker/src/runner/BUILD.bazel`

Add `manifest.gen.json` to embedsrcs:

```python
embedsrcs = [
    # ... existing entries ...
    "bundles/com.datadoghq.ddagent.<bundlename>/manifest.gen.json",
]
```

## Visibility

New bundles use `stability: "dev"` and are hidden from the action catalog by default. To make the action visible on staging:

- Add the staging org to the dev actions allow-list in Consul: `[dd.wf.bundles_allow_list_for_dev_actions]`
- Or change stability to `stable` when ready for broader visibility

## PAR Sync

The dd-source private-runner code is synced to `datadog-agent` via:
`domains/actionplatform/tools/sync_par_to_agent/sync_config.yaml`

The sync copies:
- `registry.go` and `registry_kubeapiserver.go` to `pkg/privateactionrunner/bundles/`
- Bundle implementation files to `pkg/privateactionrunner/bundles/ddagent/<bundlename>/`

The synced registry is the **stub** version. The actual agent uses its own registry with FX-wired dependencies (IPC client, etc.).

## Checklist

- [ ] Go backend: bundle package created with entrypoint, handler(s), and BUILD.bazel
- [ ] Go backend: bundle registered in `registry.go`
- [ ] Go backend: BUILD.bazel for registry updated with new dep
- [ ] Go backend: pool test updated
- [ ] TypeScript frontend: all files created (actions.ts, action stubs, entrypoint, manifest, keywords, sentence-templates, access-control, release-dates)
- [ ] TypeScript frontend: registered in `bundles.gen.ts`
- [ ] TypeScript frontend: BUILD.bazel updated with manifest
- [ ] `registry_kubeapiserver.go` NOT modified (intentional)
