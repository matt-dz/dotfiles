# datadog-agent PAR Bundle Reference

Detailed file templates and patterns for creating a new PAR bundle in the `datadog-agent` repository.

## Files to Create

### `pkg/privateactionrunner/bundles/<bundlename>/entrypoint.go`

Bundle struct with constructor that accepts FX-injected dependencies and registers actions.

```go
package <bundlename>

import (
    "github.com/DataDog/datadog-agent/pkg/privateactionrunner/types"
    // Import FX deps as needed, e.g.:
    // "github.com/DataDog/datadog-agent/comp/core/ipc"
)

// BundleNameBundle is the PAR bundle for <description>.
type BundleNameBundle struct {
    // Store FX-injected dependencies here
    // client ipc.HTTPClient
}

// NewBundleName creates a new bundle instance.
// Accept FX-injected dependencies as parameters.
func NewBundleName(/* client ipc.HTTPClient */) *BundleNameBundle {
    return &BundleNameBundle{
        // client: client,
    }
}

// Register registers all actions for this bundle.
func (b *BundleNameBundle) Register() map[string]types.ActionHandler {
    return map[string]types.ActionHandler{
        "actionName": &ActionNameHandler{
            // client: b.client,
        },
    }
}
```

### `pkg/privateactionrunner/bundles/<bundlename>/<action_name>.go`

Action handler implementation. Uses `types.ExtractInputs` for typed input extraction.

```go
package <bundlename>

import (
    "context"
    "encoding/json"
    "fmt"

    "github.com/DataDog/datadog-agent/pkg/privateactionrunner/types"
    // Import IPC packages if calling agent API:
    // "github.com/DataDog/datadog-agent/comp/core/ipc"
    // ipchttp "github.com/DataDog/datadog-agent/comp/core/ipc/ipcclient/ipchttp"
)

// ActionNameInputs defines the typed inputs for this action.
type ActionNameInputs struct {
    // Define fields matching the action's input schema.
    // Use pointers for optional fields.
    Verbose *bool   `json:"verbose,omitempty"`
    Section *string `json:"section,omitempty"`
}

// ActionNameHandler handles the actionName action.
type ActionNameHandler struct {
    // client ipc.HTTPClient
}

// Run executes the action.
func (h *ActionNameHandler) Run(ctx context.Context, task *types.Task) (interface{}, error) {
    inputs, err := types.ExtractInputs[ActionNameInputs](task)
    if err != nil {
        return nil, fmt.Errorf("failed to extract inputs: %w", err)
    }

    // Implementation here. Example using IPC:
    //
    // if h.client == nil {
    //     return nil, fmt.Errorf("IPC client is not available")
    // }
    //
    // path := "/agent/status"
    // if inputs.Section != nil {
    //     path = fmt.Sprintf("/agent/%s/status", *inputs.Section)
    // }
    //
    // endpoint := h.client.NewIPCEndpoint(path)
    //
    // v := url.Values{}
    // v.Set("format", "json")
    // if inputs.Verbose != nil && *inputs.Verbose {
    //     v.Set("verbose", "true")
    // }
    //
    // resp, err := endpoint.DoGet(
    //     ipchttp.WithContext(ctx),
    //     ipchttp.WithValues(v),
    // )
    // if err != nil {
    //     return nil, fmt.Errorf("IPC request failed: %w", err)
    // }
    //
    // var result map[string]interface{}
    // if err := json.Unmarshal(resp, &result); err != nil {
    //     return nil, fmt.Errorf("failed to parse response: %w", err)
    // }
    //
    // return result, nil

    _ = inputs
    return nil, fmt.Errorf("not implemented")
}
```

### `pkg/privateactionrunner/bundles/<bundlename>/<action_name>_test.go`

Unit tests. Use `ipcmock` when testing IPC-based actions.

```go
package <bundlename>

import (
    "context"
    "encoding/json"
    "net/http"
    "testing"

    "github.com/DataDog/datadog-agent/pkg/privateactionrunner/types"
    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/require"
    // For IPC-based actions:
    // "github.com/DataDog/datadog-agent/comp/core/ipc/ipcmock"
)

func makeTask(inputs interface{}) *types.Task {
    inputBytes, _ := json.Marshal(inputs)
    var rawInputs map[string]json.RawMessage
    json.Unmarshal(inputBytes, &rawInputs)
    return &types.Task{
        Inputs: rawInputs,
    }
}

func TestActionName_Success(t *testing.T) {
    // For IPC-based actions, use ipcmock:
    //
    // ipcComp := ipcmock.New(t)
    // ipcComp.SetHandler(func(w http.ResponseWriter, r *http.Request) {
    //     assert.Equal(t, "/agent/status", r.URL.Path)
    //     w.WriteHeader(http.StatusOK)
    //     json.NewEncoder(w).Encode(map[string]string{"status": "ok"})
    // })
    //
    // handler := &ActionNameHandler{client: ipcComp.GetClient()}
    // task := makeTask(ActionNameInputs{})
    // result, err := handler.Run(context.Background(), task)
    // require.NoError(t, err)
    // assert.NotNil(t, result)

    t.Skip("implement test")
}

func TestActionName_NilClient(t *testing.T) {
    handler := &ActionNameHandler{}
    task := makeTask(ActionNameInputs{})
    _, err := handler.Run(context.Background(), task)
    require.Error(t, err)
}

func TestBundleRegistration(t *testing.T) {
    bundle := NewBundleName()
    actions := bundle.Register()
    assert.Contains(t, actions, "actionName")
}
```

## Files to Modify

### `pkg/privateactionrunner/bundles/registry.go` (non-kubeapiserver build)

Add the bundle import and registration:

```go
import (
    // ... existing imports ...
    bundlename "github.com/DataDog/datadog-agent/pkg/privateactionrunner/bundles/<bundlename>"
)

// In NewRegistry(), add to the bundles map:
bundles["com.datadoghq.ddagent.<bundlename>"] = bundlename.NewBundleName(/* deps */)
```

### `pkg/privateactionrunner/bundles/registry_kubeapiserver.go` (kubeapiserver build)

If new FX deps were added to `NewRegistry()`, add matching `_` parameters:

```go
// NewRegistry signature must match registry.go
func NewRegistry(/* ..., _ ipc.HTTPClient */) *Registry {
    // kubeapiserver variant does not register agent-specific bundles
}
```

### Threading New FX Dependencies

Only required if the bundle needs FX components not already threaded through.

**`comp/privateactionrunner/impl/privateactionrunner.go`**:

```go
// Add to Requires struct:
type Requires struct {
    // ... existing fields ...
    NewDep newdep.Component
}

// In NewComponent(), extract the client:
newDepClient := reqs.NewDep.GetClient()

// Pass through NewPrivateActionRunner() and store on struct:
type PrivateActionRunner struct {
    // ... existing fields ...
    newDepClient newdep.Client
}

// In Start(), pass to NewWorkflowRunner():
NewWorkflowRunner(..., p.newDepClient)
```

**`pkg/privateactionrunner/runners/workflow_runner.go`**:

```go
// Add parameter to NewWorkflowRunner():
func NewWorkflowRunner(..., newDepClient newdep.Client) *WorkflowRunner {
    // Pass through to NewRegistry():
    registry := privatebundles.NewRegistry(..., newDepClient)
}
```

**`cmd/cluster-agent/subcommands/start/command.go`**:

```go
// Update startPrivateActionRunner() to accept and pass the new dep.
// This is called outside FX wiring, so extract the dep from
// the already-available FX component at the call site.
```

## Checklist

- [ ] Bundle package created with entrypoint, handler(s), and tests
- [ ] Bundle registered in `registry.go`
- [ ] `registry_kubeapiserver.go` signature matches `registry.go`
- [ ] FX dependencies threaded through entire chain (if new deps needed)
- [ ] All callers of modified functions updated (grep for function names)
- [ ] Unit tests pass
- [ ] Cluster agent build compiles (check `registry_kubeapiserver.go` and `cmd/cluster-agent/`)
