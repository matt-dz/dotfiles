---
name: create-par-bundle
description: This skill should be used when the user asks to create a new Private Action Runner (PAR) bundle, add a new action to the PAR, scaffold a PAR bundle on datadog-agent or dd-source, or mentions creating a workflow automation action for the Datadog agent. Covers both the Go implementation in datadog-agent and the Action Platform registration in dd-source.
---

# Create PAR Bundle

This skill guides the creation of a new Private Action Runner (PAR) bundle across two repositories: `datadog-agent` (Go implementation) and `dd-source` (Action Platform registration). PAR bundles expose agent-side capabilities as actions in the Workflow Automation UI.

## Architecture Overview

PAR bundles follow a dependency-threading pattern through the FX container:

```
FX Container
  comp/privateactionrunner/impl   -> Requires struct (FX deps)
    runners/workflow_runner.go    -> NewWorkflowRunner(..., deps)
      bundles/registry.go        -> NewRegistry(..., deps)
        bundles/<bundlename>/    -> NewBundleName(deps)
          action handlers        -> actual implementation
```

New FX-injected dependencies must be threaded through the entire chain: component -> runner -> registry -> bundle.

## Workflow

### Step 0: Set Up Worktree and Sync with Main

Before making any changes, create a **unique git worktree** and ensure the branch is up to date:

1. **Pull the latest from `main`** to ensure the base is current:
   ```bash
   git checkout main && git pull origin main
   ```

2. **Create a unique git worktree** for the new bundle work. Use the `EnterWorktree` tool to create an isolated worktree. Name it descriptively based on the bundle being created (e.g., `par-bundle-<bundlename>`).

All subsequent changes must be made **within the worktree**—do not modify the main working tree.

### Step 1: Gather Requirements

Before creating a bundle, clarify the following:

- **Bundle name**: The fully-qualified bundle ID (e.g., `com.datadoghq.ddagent.agentstatus`)
- **Action names**: The actions the bundle exposes (e.g., `getStatus`)
- **Action inputs/outputs**: The typed inputs and outputs for each action
- **FX dependencies**: Any FX components the bundle needs (e.g., `ipc.HTTPClient`, `traceroute.Component`)
- **Target repositories**: Whether changes are needed on `datadog-agent`, `dd-source`, or both

### Step 2: Implement on datadog-agent

Read `references/datadog-agent-bundle.md` for detailed file templates and patterns.

Summary of changes:

1. **Create the bundle package** at `pkg/privateactionrunner/bundles/<bundlename>/`
   - `entrypoint.go` - Bundle struct, constructor, action registration
   - `<action_name>.go` - Action handler implementation
   - `<action_name>_test.go` - Unit tests using `ipcmock` if IPC is involved

2. **Register the bundle** in `pkg/privateactionrunner/bundles/registry.go` (non-kubeapiserver build)
   - Add import for the new bundle package
   - Add entry in the bundles map

3. **Match signature** in `pkg/privateactionrunner/bundles/registry_kubeapiserver.go`
   - If new FX deps were added to `NewRegistry()`, add `_` params here to match the signature

4. **Thread FX dependencies** (only if the bundle requires new FX components not already in the chain):
   - `comp/privateactionrunner/impl/privateactionrunner.go` - Add to `Requires` struct, extract in `NewComponent()`, pass through
   - `pkg/privateactionrunner/runners/workflow_runner.go` - Add parameter to `NewWorkflowRunner()`
   - `cmd/cluster-agent/subcommands/start/command.go` - Update the non-FX caller of `NewPrivateActionRunner`

### Step 3: Implement on dd-source

Read `references/dd-source-bundle.md` for detailed file templates and patterns.

Summary of changes:

1. **Go backend (private-runner)** at `domains/actionplatform/apps/private-runner/src/private-bundles/com/datadoghq/ddagent/<bundlename>/`
   - `entrypoint.go` - Stub bundle struct and constructor
   - `<action_name>.go` - Stub action handler (real work done on agent side)
   - `BUILD.bazel` - Bazel build file

2. **Register in Go backend**:
   - `domains/actionplatform/apps/private-runner/src/private-bundles/registry.go` - Add to bundles map
   - `domains/actionplatform/apps/private-runner/src/private-bundles/BUILD.bazel` - Add dep

3. **Update pool test**:
   - `domains/actionplatform/shared/libs/go/pool/pools_test.go` - Add bundle to pool assignment test

4. **TypeScript frontend (wf-actions-worker)** at `domains/workflow/actionplatform/apps/wf-actions-worker/src/runner/bundles/<bundle.id>/`
   - `actions.ts` - Bundle metadata (title, icon, stability, environments)
   - `actions/<action-name>.ts` - Action stub with `resultErr('Not implemented')`
   - `entrypoint.gen.ts` - Entry point implementing `Bundle` interface
   - `manifest.gen.json` - JSON Schema types, action metadata, access modes
   - `keywords.json` - Keywords keyed by action name
   - `sentence-templates.json` - Natural language template
   - `access-control.json` - Access modes and level
   - `release-dates.json` - Empty for new bundles

5. **Register in TypeScript frontend**:
   - `bundles.gen.ts` - Add lazy import entry
   - `runner/BUILD.bazel` - Add `manifest.gen.json` to embedsrcs

## Critical Gotchas

1. **Two registry files must match**: `registry.go` and `registry_kubeapiserver.go` must have matching `NewRegistry()` signatures. The kubeapiserver variant uses `_` for unused deps.

2. **Cluster agent caller**: `NewPrivateActionRunner` is called directly from `cmd/cluster-agent/subcommands/start/command.go` outside FX wiring. Always grep for all callers when changing a function signature.

3. **IPC is already in the FX graph**: `ipcfx.ModuleReadWrite()` is already included in `cmd/privateactionrunner/subcommands/run/command.go`, so no changes are needed at the command entrypoint level for IPC.

4. **Stability flag**: New bundles should use `stability: "dev"` in the TypeScript metadata. To see the action on staging, add the staging org to the dev actions allow-list in Consul (`[dd.wf.bundles_allow_list_for_dev_actions]`).

5. **PAR sync**: The dd-source private-runner code is synced to `datadog-agent` via `domains/actionplatform/tools/sync_par_to_agent/sync_config.yaml`. The synced registry is the stub version; the actual agent uses its own registry with FX-wired dependencies.

6. **Do NOT modify `registry_kubeapiserver.go` in dd-source**: The kubeapiserver variant in dd-source is a slimmed-down registry for the Cluster Agent and doesn't need agent-specific bundles.

## Template PRs

Use these PRs as direct templates for the file structure and patterns:

- **datadog-agent networkpath bundle**: [#45704](https://github.com/DataDog/datadog-agent/pull/45704)
- **dd-source networkpath bundle**: [#358733](https://github.com/DataDog/dd-source/pull/358733)
- **datadog-agent agentstatus bundle**: [#46389](https://github.com/DataDog/datadog-agent/pull/46389)
- **dd-source agentstatus bundle**: [#359062](https://github.com/DataDog/dd-source/pull/359062)

## Key Utilities

- `types.ExtractInputs[T](task)` - Typed input extraction from task
- `ipc.HTTPClient.NewIPCEndpoint(path)` - Create endpoint for agent API
- `ipchttp.WithValues(v)` - Set query parameters on IPC requests
- `ipchttp.WithContext(ctx)` - Propagate context to IPC requests
- `ipcmock.New(t)` - Mock IPC component for tests with TLS server

## Local Testing

```bash
# Build local docker image
dda inv omnibus.docker-build --arch=arm64

# Load onto kind cluster in lima VM
docker save localhost/datadog-agent:local -o /tmp/datadog-agent-local.tar
limactl copy /tmp/datadog-agent-local.tar par-dev:/tmp/datadog-agent-local.tar
limactl shell par-dev kind load image-archive /tmp/datadog-agent-local.tar --name par-dev
```

## Create Draft PR

After all changes are committed, create a **draft** pull request using the GitHub CLI:

```bash
gh pr create --draft --title "<PR title>" --body "<PR description>"
```

- The PR must be created in **draft mode** (`--draft` flag)
- The PR title should clearly describe the new bundle being added
- The PR body should summarize the changes made (files added, bundle registered, dependencies threaded, etc.)
- If changes span multiple repositories (`datadog-agent` and `dd-source`), create a separate draft PR in each repository and cross-link them in the PR descriptions

## Integrate dd-source PR to Staging

After the **dd-source** draft PR has been created, integrate it into staging by commenting on the PR:

```bash
gh pr comment <PR_NUMBER> --repo DataDog/dd-source --body "/integrate -f"
```

- This step applies **only** to the `dd-source` PR—do not run `/integrate` on the `datadog-agent` PR
- The `/integrate -f` command forces integration of the PR into the staging environment
- Wait for the draft PR to be fully created before commenting
