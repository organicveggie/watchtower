# `internal/actions` Package

This package contains the core business logic that drives Watchtower's container update lifecycle. It is responsible for
checking preconditions, determining which containers need to be updated, coordinating stop/start sequences (including
rolling restarts), and managing implicit restart propagation through container dependency chains.

The package is consumed primarily by `cmd/root.go` and is tested via `actions_suite_test.go` and `update_test.go`.

---

## Files

### `check.go`

Contains preflight checks that run before any update cycle begins. These guards prevent Watchtower from operating in
configurations that are known to be unsafe or contradictory.

**Public Functions:**

---

#### `CheckForSanity(client container.Client, filter types.Filter, rollingRestarts bool) error`

Validates that the current configuration is safe to proceed with. Currently enforces one constraint: if rolling restarts
are enabled, none of the containers matched by `filter` may have links to other containers (Docker `--link`). Linked
containers require a specific shutdown order that is incompatible with the one-at-a-time rolling restart strategy.

Returns an error describing the offending container if the check fails, or `nil` if everything is safe to proceed.

---

#### `CheckForMultipleWatchtowerInstances(client container.Client, cleanup bool, scope string) error`

Ensures that only one Watchtower instance is managing containers at a time within the same scope. It lists all running
containers that match the Watchtower label (optionally filtered by `scope`) and, if more than one is found, stops all
but the most recently created one.

- If `cleanup` is `true`, it also attempts to remove the Docker image used by each stopped instance.
- If a scope is provided, only Watchtower containers within that scope are considered — allowing multiple Watchtower
instances to coexist on the same host when each manages a distinct scope.

Returns an error if any containers could not be stopped, summarising the count of failures.

---

### `update.go`

The heart of the Watchtower update engine. Orchestrates the full lifecycle of a single update session: scanning
containers for stale images, sorting them by dependency order, stopping stale containers, and restarting them with
updated images. Also manages optional lifecycle hook execution and post-update image cleanup.

**Public Functions:**

---

#### `Update(client container.Client, params types.UpdateParams) (types.Report, error)`

Runs a complete container update session. The sequence of operations is:

1. **Pre-check hooks** — If `params.LifecycleHooks` is enabled, executes the pre-check command inside every matched
container.
2. **Staleness check** — Iterates over all containers matched by `params.Filter`, calling `client.IsContainerStale` for
each. Containers that cannot be checked (e.g. due to a pull error) are marked as skipped rather than causing a hard
failure.
3. **Configuration verification** — For containers that are stale and eligible for update, calls `VerifyConfiguration`
to ensure the container can be safely recreated.
4. **Dependency sort** — Sorts all containers topologically by their declared dependencies using
`sorter.SortByDependencies`, so that linked containers are stopped and started in the correct order.
5. **Implicit restart propagation** — Calls `UpdateImplicitRestart` to mark containers as needing a restart if any
container they depend on is also being restarted.
6. **Stop and restart** — Either performs a rolling restart (one container at a time) or stops all stale containers in
reverse dependency order and then restarts them in forward order.
7. **Image cleanup** — If `params.Cleanup` is enabled, removes the old images of successfully updated containers.
8. **Post-check hooks** — If `params.LifecycleHooks` is enabled, executes the post-check command inside every matched
container.

Returns a `types.Report` summarising which containers were scanned, updated, failed, skipped, stale, or fresh.

---

#### `UpdateImplicitRestart(containers []types.Container)`

Iterates through a slice of containers and marks any container as `LinkedToRestarting` if at least one container it
depends on (via `Links()`) is already marked for restart. This ensures that containers sharing a network or dependency
with a container being updated are also restarted, keeping the dependency graph consistent.

Operates in-place on the slice. Containers that are already marked for restart are skipped.

---

## Internal Helpers

These unexported functions implement the detailed mechanics of the update sequence:

| Function | Description |
| -------- | ----------- |
| `performRollingRestart(containers, client, params)` | Stops and restarts containers one at a time in reverse dependency order. Collects failures and optionally cleans up old images after each successful update. |
| `stopContainersInReversedOrder(containers, client, params)` | Stops all containers marked for restart, iterating in reverse dependency order. Returns a map of failures and a set of image IDs that were running before the stop. |
| `restartContainersInSortedOrder(containers, client, params, stoppedImages)` | Restarts containers in forward dependency order, but only for images that were confirmed stopped. Optionally cleans up old images afterwards. |
| `stopStaleContainer(container, client, params)` | Stops a single container. Skips the Watchtower container itself, containers not marked for restart, and containers where the pre-update lifecycle hook returns a skip signal or error. |
| `restartStaleContainer(container, client, params)` | Restarts a single container. If the container is the Watchtower instance itself, renames the old container first so the new one can reuse the name. Executes the post-update lifecycle hook after a successful start. |
| `cleanupImages(client, imageIDs)` | Removes a set of Docker images by ID, logging any errors encountered. |
| `cleanupExcessWatchtowers(containers, client, cleanup)` | Stops all but the last container in the provided list (sorted oldest-first). Used by `CheckForMultipleWatchtowerInstances`. |
| `linkedContainerMarkedForRestart(links, containers)` | Returns the name of the first container in `links` that is marked for restart, or an empty string if none are. |

---

## Test Coverage

| File | Description |
| ---- | ----------- |
| `actions_suite_test.go` | Bootstraps the Ginkgo test suite. Contains tests for `CheckForMultipleWatchtowerInstances` covering empty input, single-instance, multi-instance, and image cleanup scenarios. |
| `update_test.go` | Covers `Update` and `UpdateImplicitRestart` across a wide range of scenarios including cleanup deduplication, monitor-only mode, label precedence, rolling restarts, lifecycle hook exit codes, linked container propagation, and stopped/restarting container handling. |
