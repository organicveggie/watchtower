# `pkg/sorter` Package

This package provides two sorting mechanisms for slices of `types.Container`: a creation-date sort used when cleaning up
duplicate Watchtower instances, and a topological dependency sort used to determine the correct stop and start order
during an update cycle. It is consumed by `internal/actions/check.go` (`ByCreated`) and `internal/actions/update.go`
(`SortByDependencies`).

---

## Files

### `sort.go`

Contains both sorting implementations in a single file.

---

**Types:**

#### `ByCreated []types.Container`

A `[]types.Container` type alias that implements `sort.Interface` by comparing containers' `Created` timestamps. Used
with `sort.Sort` to order containers oldest-first. Consumed by `internal/actions/check.go` in `cleanupExcessWatchtowers`
to identify the most recently created Watchtower instance (which is the one kept running).

| Method | Description |
| ------ | ----------- |
| `Len() int` | Returns the number of containers in the slice. |
| `Swap(i, j int)` | Swaps the containers at indices `i` and `j`. |
| `Less(i, j int) bool` | Parses the `Created` field of each container as RFC3339Nano and returns `true` if container `i` was created before container `j`. If parsing fails for container `i`, `t1` is set to `time.Now()`, effectively sorting unparseable entries to the end. Note: due to a bug, the error check for container `j` re-uses `err` from container `i`, so a parse failure on `j` is silently ignored and `t2` retains its zero value. |

---

**Public Functions:**

---

#### `SortByDependencies(containers []types.Container) ([]types.Container, error)`

Performs a topological sort of the container slice based on each container's declared dependencies (as returned by
`container.Links()`). Returns a new slice in which every container appears after all containers it depends on. This
ordering ensures that during an update cycle, linked containers are stopped in reverse order and started in forward
order without violating dependency constraints.

Delegates to the unexported `dependencySorter` type. Returns an error if a circular dependency is detected.

---

**Internal Helpers:**

#### `dependencySorter`

Implements the topological sort using an iterative depth-first search with cycle detection. Maintains three fields:

| Field | Type | Description |
| ----- | ---- | ----------- |
| `unvisited` | `[]types.Container` | Containers not yet placed in the sorted output. |
| `marked` | `map[string]bool` | Containers currently on the DFS call stack, used to detect cycles. |
| `sorted` | `[]types.Container` | The output slice, built up as containers are fully visited. |

| Method | Description |
| ------ | ----------- |
| `Sort(containers)` | Entry point. Initialises `unvisited` and `marked`, then repeatedly calls `visit` on the first unvisited container until none remain. Returns `sorted` when complete, or an error if a cycle is found. |
| `visit(c)` | Visits a single container. Returns an error immediately if `c.Name()` is already in `marked` (cycle detected). Marks `c`, recursively visits each of its unvisited linked containers via `findUnvisited`, then removes `c` from `unvisited` and appends it to `sorted`. Unmarks `c` via a deferred `delete` when the frame returns, so the mark is only active for the duration of the current DFS path. |
| `findUnvisited(name)` | Searches `unvisited` for a container with the given name. Returns a pointer to it if found, or `nil` if it has already been sorted or does not exist in the set. |
| `removeUnvisited(c)` | Removes a container by name from the `unvisited` slice by finding its index and splicing it out. |

---

## Test Coverage

This package has no dedicated test file. Its behaviour is exercised indirectly through
`internal/actions/update_test.go`, which tests `SortByDependencies` via the `Update` function across scenarios including
linked containers, rolling restarts, and dependency chains. `ByCreated` is exercised indirectly through
`internal/actions/actions_suite_test.go`, which tests `CheckForMultipleWatchtowerInstances`.
