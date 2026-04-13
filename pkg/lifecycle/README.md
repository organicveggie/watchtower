# `pkg/lifecycle` Package

This package implements Watchtower's lifecycle hook system. It provides functions that execute user-defined shell commands inside containers at specific points in the update cycle: before and after the full scan (pre/post-check), and before and after an individual container is updated (pre/post-update). Lifecycle hook execution is an optional feature enabled by the `--enable-lifecycle-hooks` flag and is invoked by `internal/actions/update.go`.

---

## Files

### `lifecycle.go`

All lifecycle hook logic is contained in a single file. Each function follows the same pattern: retrieve the relevant command from the container's labels, skip silently if none is configured, and delegate execution to `client.ExecuteCommand`.

---

**Public Functions:**

---

#### `ExecutePreChecks(client container.Client, params types.UpdateParams)`

Runs the pre-check lifecycle hook for every container currently matched by `params.Filter`. Retrieves the full container list via `client.ListContainers` and calls `ExecutePreCheckCommand` for each. If the container list cannot be fetched, returns silently without error. Invoked once at the start of each update session when lifecycle hooks are enabled.

---

#### `ExecutePostChecks(client container.Client, params types.UpdateParams)`

Runs the post-check lifecycle hook for every container currently matched by `params.Filter`. Follows the same pattern as `ExecutePreChecks`. Invoked once at the end of each update session when lifecycle hooks are enabled.

---

#### `ExecutePreCheckCommand(client container.Client, container types.Container)`

Executes the pre-check command for a single container. Reads the command from the `com.centurylinklabs.watchtower.lifecycle.pre-check` label via `container.GetLifecyclePreCheckCommand()`. If the label is absent or empty, logs a debug message and returns. The execution timeout is hardcoded to 1 minute. Errors from `client.ExecuteCommand` are logged but do not propagate — a failing pre-check hook does not interrupt the update cycle.

---

#### `ExecutePostCheckCommand(client container.Client, container types.Container)`

Executes the post-check command for a single container. Reads the command from the `com.centurylinklabs.watchtower.lifecycle.post-check` label via `container.GetLifecyclePostCheckCommand()`. Follows the same silent-skip, hardcoded 1-minute timeout, and error-logging behaviour as `ExecutePreCheckCommand`.

---

#### `ExecutePreUpdateCommand(client container.Client, container types.Container) (SkipUpdate bool, err error)`

Executes the pre-update command for a single container before it is stopped and recreated. Reads the command from the `com.centurylinklabs.watchtower.lifecycle.pre-update` label via `container.GetLifecyclePreUpdateCommand()`. Differs from the check hooks in two important ways:

- **State guard**: If the container is not currently running or is in a restarting state, the command is skipped entirely. This prevents exec failures against containers that cannot accept commands.
- **Return values**: The result of `client.ExecuteCommand` is returned directly to the caller. An exit code of `75` (`EX_TEMPFAIL`) causes `SkipUpdate` to be `true`, signalling to `internal/actions` that this container should be skipped for the current cycle without being treated as a failure. Any other non-zero exit code returns a non-nil error that causes the container to be skipped and logged as a failure.

The timeout used for execution is read from the container's `com.centurylinklabs.watchtower.lifecycle.pre-update-timeout` label via `container.PreUpdateTimeout()`.

---

#### `ExecutePostUpdateCommand(client container.Client, newContainerID types.ContainerID)`

Executes the post-update command inside a newly started container after it has been recreated. Accepts the new container's ID rather than a `types.Container`, since the original container object refers to the now-removed instance. Fetches the new container's details via `client.GetContainer` to resolve the command and timeout from its labels. If the container cannot be fetched, or if the post-update label is absent, logs and returns without error. Errors from `client.ExecuteCommand` are logged but do not propagate.

The timeout used for execution is read from the new container's `com.centurylinklabs.watchtower.lifecycle.post-update-timeout` label via `newContainer.PostUpdateTimeout()`.

---

## Test Coverage

This package has no dedicated test file. Its behaviour is exercised indirectly through the lifecycle hook tests in `internal/actions/update_test.go`, which verify the integration between the update engine and the hook execution functions using the mock client from `internal/actions/mocks`.
