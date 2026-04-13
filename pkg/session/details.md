# `pkg/session` Package

This package models the state of a single Watchtower update session. It defines the container state enum, a per-container status type that implements `types.ContainerReport`, a mutable `Progress` map that accumulates container statuses as an update cycle runs, and the `report` type that converts a completed `Progress` into an immutable `types.Report` for consumption by the notification and metrics systems. It is consumed by `internal/actions/update.go`, `pkg/metrics`, `pkg/notifications`, and `internal/actions/mocks`.

---

## Files

### `container_status.go`

Defines the `State` enum and the `ContainerStatus` type that implements the `types.ContainerReport` interface.

**Types:**

#### `State`

An integer enum representing the lifecycle state of a container within a session.

| Constant | Description |
|---|---|
| `UnknownState` | Default zero value. Used to represent an uninitialised state; should not appear in a completed report. |
| `SkippedState` | The container was explicitly skipped (e.g. via a lifecycle hook exit code or `monitor-only`). |
| `ScannedState` | The container was inspected but its final state has not yet been determined. Transitional — resolved to `Fresh`, `Updated`, `Failed`, or `Stale` by `NewReport`. |
| `UpdatedState` | The container was successfully updated and restarted with a new image. |
| `FailedState` | The container's update attempt failed. |
| `FreshState` | The container's image was already up to date; no action was taken. Set by `NewReport` when `oldImage == newImage`. |
| `StaleState` | The container's image is outdated but it was not restarted (e.g. `monitor-only` or `no-restart`). Set by `NewReport` as the default for non-updated, non-failed scanned containers. |

---

#### `ContainerStatus`

Holds the per-container state captured during a session and implements `types.ContainerReport`. All fields are unexported; values are set by `UpdateFromContainer` and mutated by `Progress` methods.

| Field | Type | Description |
|---|---|---|
| `containerID` | `wt.ContainerID` | The container's full ID. |
| `oldImage` | `wt.ImageID` | The image ID the container was running at the start of the session. |
| `newImage` | `wt.ImageID` | The newest image ID found during the session. Equals `oldImage` if no newer image was found. |
| `containerName` | `string` | The container name. |
| `imageName` | `string` | The image name and tag the container uses. |
| `error` | `error` | Any error encountered during the update attempt. Embedded directly so it does not conflict with the `Error() string` method. |
| `state` | `State` | The container's current state within the session. |

**Public Methods on `ContainerStatus`:**

| Method | Return type | Description |
|---|---|---|
| `ID()` | `wt.ContainerID` | Returns `containerID`. |
| `Name()` | `string` | Returns `containerName`. |
| `CurrentImageID()` | `wt.ImageID` | Returns `oldImage` — the image the container was running at session start. |
| `LatestImageID()` | `wt.ImageID` | Returns `newImage` — the newest image found during the session. |
| `ImageName()` | `string` | Returns `imageName`. |
| `Error()` | `string` | Returns the error message string, or an empty string if no error occurred. |
| `State()` | `string` | Returns the human-readable state name (`"Skipped"`, `"Scanned"`, `"Updated"`, `"Failed"`, `"Fresh"`, `"Stale"`, or `"Unknown"`). |

---

### `progress.go`

Defines the mutable `Progress` map used to accumulate container statuses as an update cycle runs, and provides the constructor function for creating `ContainerStatus` values.

**Types:**

#### `Progress`

A `map[types.ContainerID]*ContainerStatus`. Used as the primary accumulator throughout `internal/actions/update.go`. Converted to an immutable `types.Report` at the end of a session via `Report()`.

**Public Functions and Methods:**

---

#### `UpdateFromContainer(cont types.Container, newImage types.ImageID, state State) *ContainerStatus`

Constructs and returns a new `ContainerStatus` populated from the fields of `cont`. Sets `oldImage` from `cont.SafeImageID()` (which returns an empty ID rather than panicking if image info is unavailable) and `newImage` from the provided argument. Used internally by `AddSkipped` and `AddScanned`.

---

#### `(m Progress) AddSkipped(cont types.Container, err error)`

Adds a container to the progress map with `SkippedState` and the provided error. The `newImage` is set to the container's current image ID (i.e. no new image was found). Used when a container is explicitly excluded from the update cycle.

---

#### `(m Progress) AddScanned(cont types.Container, newImage types.ImageID)`

Adds a container to the progress map with `ScannedState` and the provided newest image ID. The final state (Fresh, Updated, Failed, or Stale) is determined later by `NewReport` or by calling `MarkForUpdate` / `UpdateFailed`.

---

#### `(m Progress) UpdateFailed(failures map[types.ContainerID]error)`

Iterates over the `failures` map and, for each container ID, sets its state to `FailedState` and records the associated error. Called after a batch of stop/restart operations to record which containers could not be updated.

---

#### `(m Progress) Add(update *ContainerStatus)`

Inserts a `ContainerStatus` into the map using `update.containerID` as the key. Used directly by `AddSkipped` and `AddScanned`; also available for callers that construct a `ContainerStatus` manually.

---

#### `(m Progress) MarkForUpdate(containerID types.ContainerID)`

Sets the state of the container identified by `containerID` to `UpdatedState`. Called after a container has been successfully stopped and restarted with a new image.

---

#### `(m Progress) Report() types.Report`

Converts the completed `Progress` map into an immutable `types.Report` by delegating to `NewReport`. This is the terminal operation of the session accumulation lifecycle.

---

### `report.go`

Implements `types.Report` via the unexported `report` struct, provides the `NewReport` constructor that classifies containers from a `Progress` into their final result buckets, and defines the `sortableContainers` helper used to sort all report slices by container ID.

**Public Functions:**

---

#### `NewReport(progress Progress) types.Report`

Constructs an immutable `types.Report` from a completed `Progress` map by classifying each `ContainerStatus` into its final result bucket. The classification logic is:

- **`SkippedState`** — placed directly into `skipped`. Not added to `scanned`.
- **All others** — added to `scanned` first, then further classified:
  - If `newImage == oldImage`: state is overridden to `FreshState` and the container is added to `fresh`.
  - If `UpdatedState`: added to `updated`.
  - If `FailedState`: added to `failed`.
  - **Default** (e.g. `ScannedState` with a new image but no update or failure): state is overridden to `StaleState` and the container is added to `stale`.

All six result slices are sorted by container ID (lexicographic on `types.ContainerID`) before the report is returned.

---

**`report` Methods (implementing `types.Report`):**

| Method | Description |
|---|---|
| `Scanned()` | Returns all containers that were inspected (excludes skipped). |
| `Updated()` | Returns containers successfully updated. |
| `Failed()` | Returns containers whose update failed. |
| `Skipped()` | Returns containers explicitly skipped. |
| `Stale()` | Returns containers with a newer image available but not updated. |
| `Fresh()` | Returns containers whose image was already up to date. |
| `All()` | Returns a deduplicated, sorted union of all six buckets. Priority order for deduplication: updated → failed → skipped → stale → fresh → scanned. A container that appears in multiple buckets (which can occur if `scanned` overlaps with a more specific bucket) is included only once, in its most specific category. |

---

**Internal Helpers in `report.go`:**

#### `sortableContainers`

A `[]types.ContainerReport` type alias that implements `sort.Interface` by comparing `ID()` values lexicographically. Used to sort all six result slices in `NewReport` and the combined slice in `All()`.

---

## Test Coverage

This package has no dedicated test file. Its behaviour is exercised through:

- `internal/actions/mocks/progress.go` — `CreateMockProgressReport` constructs `Progress` instances covering all states (`SkippedState`, `FreshState`, `UpdatedState`, `FailedState`) and calls `Progress.Report()`, exercising the full `NewReport` classification path.
- `internal/actions/update_test.go` — verifies the end-to-end integration between the update engine and the report produced at the end of a session.
- `pkg/notifications/shoutrrr_test.go` — renders notification templates against mock reports, exercising the `types.Report` interface methods on `report`.
