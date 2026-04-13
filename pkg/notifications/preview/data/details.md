# `pkg/notifications/preview/data` Package

This package generates synthetic data for rendering notification template previews. It has no runtime role — its sole
purpose is to produce realistic-looking session state (container report entries, log entries, and associated metadata)
so that users and tests can validate custom `--notification-template` values without running a real update cycle. It is
consumed by `pkg/notifications/preview`.

---

## Files

### `data.go`

Defines the central `previewData` builder and its `staticData` helper. All synthetic data flows through a `previewData`
instance.

**Types:**

#### `previewData`

The main builder struct. Holds a seeded random source, a monotonically advancing timestamp, a lazily-initialised
`*report`, a running container count, a slice of generated log entries, and a `staticData` block.

| Field | Type | Description |
| ----- | ---- | ----------- |
| `rand` | `*rand.Rand` | Seeded with `1` for reproducible output. Used by all random-selection helpers. |
| `lastTime` | `time.Time` | Initialised to 30 minutes before `New()` is called. Advanced by a random 0–29 second step each time `generateTime()` is called. |
| `report` | `*report` | Lazily initialised on the first call to `addContainer`. |
| `containerCount` | `int` | Incremented by `addContainer`; used by `generateName` and `generateImageName` to select names from the pool. |
| `Entries` | `[]*logEntry` | Accumulated log entries, appended by `AddLogEntry`. |
| `StaticData` | `staticData` | Fixed title (`"Title"`) and host (`"Host"`) strings available to templates. |

#### `staticData`

Plain struct with `Title string` and `Host string`.

---

**Public Functions:**

#### `New() *previewData`

Returns a freshly initialised `previewData` with the random source seeded to `1`, `lastTime` set to 30 minutes in the
past, empty `Entries`, and `StaticData` set to `{Title: "Title", Host: "Host"}`.

---

**Methods:**

#### `(pb *previewData) AddFromState(state State)`

Generates a synthetic container entry and appends it to the internal report. Produces random hex container and image
IDs, selects a container name from the pool (cycling with a numeric suffix once exhausted), and derives an image name
from the organisation name pool. For `FailedState` and `SkippedState`, a random error message is selected from the
corresponding pool and stored as the container's error. Delegates to `addContainer`.

#### `(pb *previewData) AddLogEntry(level LogLevel)`

Appends a `logEntry` to `Entries`. Selects a message from `logErrors` for `FatalLevel`, `ErrorLevel`, and `WarnLevel`;
selects from `logMessages` for all other levels. The entry's timestamp advances monotonically via `generateTime`.

#### `(pb *previewData) Report() types.Report`

Returns the internal `*report` as a `types.Report`. Returns `nil` if no containers have been added.

---

### `logs.go`

Defines the log-entry type and log level constants used when generating synthetic log output.

**Types:**

#### `logEntry`

| Field | Type | Description |
| ----- | ---- | ----------- |
| `Message` | `string` | The log message text. |
| `Data` | `map[string]any` | Additional structured fields (always an empty map in generated entries). |
| `Time` | `time.Time` | The log entry timestamp. |
| `Level` | `LogLevel` | The severity level. |

#### `LogLevel`

A `string` type representing a log severity level.

| Constant | Value |
| -------- | ----- |
| `TraceLevel` | `"trace"` |
| `DebugLevel` | `"debug"` |
| `InfoLevel` | `"info"` |
| `WarnLevel` | `"warning"` |
| `ErrorLevel` | `"error"` |
| `FatalLevel` | `"fatal"` |
| `PanicLevel` | `"panic"` |

**Public Functions:**

#### `LevelsFromString(str string) []LogLevel`

Parses a compact string of level characters and returns the corresponding `LogLevel` slice. Character mapping: `p` →
Panic, `f` → Fatal, `e` → Error, `w` → Warn, `i` → Info, `d` → Debug, `t` → Trace. Unrecognised characters are silently
skipped.

#### `(level LogLevel) String() string`

Returns the level's underlying string value.

---

### `preview_strings.go`

Declares the string pool arrays used by `data.go` when generating random container names, image names, and log messages.

| Variable | Description |
| -------- | ----------- |
| `containerNames []string` | 40 synthetic container names (e.g. `"cyberscribe"`, `"quantumquill"`). Selected round-robin (with numeric suffixes when exhausted) by `generateName`. |
| `organizationNames []string` | 39 synthetic organisation names (e.g. `"techwave"`, `"codecrafters"`). Used as image name prefixes by `generateImageName`. |
| `errorMessages []string` | 42 error strings (e.g. `"Error 404: Resource not found"`). Used for `FailedState` containers. |
| `skippedMessages []string` | 20 skip-reason strings (e.g. `"Fear of introducing new bugs"`). Used for `SkippedState` containers. |
| `logMessages []string` | 13 informational log strings (e.g. `"Checking for available updates..."`). Used by `AddLogEntry` for non-error levels. |
| `logErrors []string` | 5 error log strings (e.g. `"Update package download failed."`). Used by `AddLogEntry` for Warn/Error/Fatal levels. |

---

### `report.go`

Defines the `State` type, the internal `report` struct that implements `types.Report`, and a sort helper.

**Types:**

#### `State`

A `string` type representing the outcome of a container in a session report.

| Constant | Value | Character in `StatesFromString` |
| -------- | ----- | ------------------------------- |
| `ScannedState` | `"scanned"` | `c` |
| `UpdatedState` | `"updated"` | `u` |
| `FailedState` | `"failed"` | `e` |
| `SkippedState` | `"skipped"` | `k` |
| `StaleState` | `"stale"` | `t` |
| `FreshState` | `"fresh"` | `f` |

#### `report`

Package-private struct implementing `types.Report`. Holds six slices of `types.ContainerReport`, one per state.
Implements `Scanned()`, `Updated()`, `Failed()`, `Skipped()`, `Stale()`, `Fresh()`, and `All()`. `All()` merges all six
slices, deduplicates by container ID (keeping the first occurrence in the order: updated → failed → skipped → stale →
fresh → scanned), and sorts the result by container ID.

**Public Functions:**

#### `StatesFromString(str string) []State`

Parses a compact string of state characters and returns the corresponding `State` slice. Uses the character mapping in
the table above. Unrecognised characters are silently skipped.

---

### `status.go`

Defines the `containerStatus` struct, which implements the `types.ContainerReport` interface for use in preview reports.

**Types:**

#### `containerStatus`

| Field | Type | Description |
| ----- | ---- | ----------- |
| `containerID` | `wt.ContainerID` | Randomly generated hex ID. |
| `oldImage` | `wt.ImageID` | Randomly generated hex ID representing the current image. |
| `newImage` | `wt.ImageID` | Randomly generated hex ID representing the latest image. |
| `containerName` | `string` | Selected from the `containerNames` pool. |
| `imageName` | `string` | Derived from `organizationNames` + container name + `":latest"`. |
| `error` | `error` | Non-nil for `FailedState` and `SkippedState` entries; `nil` otherwise. |
| `state` | `State` | The container's outcome state. |

Implements `types.ContainerReport` via methods: `ID()`, `Name()`, `CurrentImageID()`, `LatestImageID()`, `ImageName()`,
`Error()` (returns `""` when `error` is nil), and `State()`.

---

## Test Coverage

This package has no dedicated test file. Its types and functions are exercised through `pkg/notifications/preview`
tests.
