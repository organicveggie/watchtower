# `pkg/notifications/preview` Packages — Feature Summary

## `pkg/notifications/preview` — Template Preview Renderer

Provides a single entry point for rendering notification template previews without running a real update cycle.

- **Template rendering**: `Render` accepts a Go `text/template` string, a list of container states, and a list of log levels. It parses the template (applying the shared Watchtower template function map), populates a synthetic data object from the provided states and log levels, executes the template against that data, and returns the rendered string.
- **Deterministic output**: The random source used for synthetic data generation is seeded to a fixed value (`1`), so identical inputs always produce identical output regardless of when or how many times `Render` is called.
- **No side effects**: Does not interact with the Docker daemon, the filesystem, or any external service.
- **Template validation**: Returns a descriptive error if the template string is syntactically invalid or fails to execute, allowing users to catch template errors before deploying a custom `--notification-template`.

---

## `pkg/notifications/preview/data` — Synthetic Preview Data Generator

Generates realistic-looking session data for template preview rendering. Has no runtime role.

### Data Builder (`data.go`)

- **Incremental construction**: `previewData` is built up by calling `AddFromState` and `AddLogEntry` any number of times before passing the result to a template renderer via `Report()` and the `Entries` field.
- **Synthetic container generation**: `AddFromState` generates a container entry with random hex IDs, a name selected from a pool (with numeric suffixes when the pool is exhausted), and a derived image name. Containers in `FailedState` and `SkippedState` receive a randomly selected error message. Each container is routed into the appropriate report bucket (scanned, updated, failed, skipped, stale, or fresh).
- **Synthetic log entry generation**: `AddLogEntry` appends a log entry at the specified level with a randomly selected message. Error-level messages (Warn, Error, Fatal) are drawn from a separate pool from informational messages. Timestamps advance monotonically with a random 0–29 second step between entries.
- **Static template data**: Each `previewData` instance exposes a `StaticData` block with fixed `Title` and `Host` strings, available to templates that reference those fields.

### Log Levels (`logs.go`)

- **`LogLevel` type**: A string-based enum with seven levels — Trace, Debug, Info, Warn, Error, Fatal, Panic — mirroring Logrus severity levels without a Logrus dependency.
- **`LevelsFromString`**: Parses a compact single-character-per-level string (e.g. `"ewi"`) into a `[]LogLevel` slice, making it convenient to specify log level sets from CLI arguments.

### State and Report Types (`report.go`, `status.go`)

- **`State` type**: A string-based enum with six container outcome states — Scanned, Updated, Failed, Skipped, Stale, Fresh — with a corresponding `StatesFromString` parser using single-character codes (`c`, `u`, `e`, `k`, `t`, `f`).
- **`report`**: An internal `types.Report` implementation backed by six per-state slices. `All()` returns a deduplicated, ID-sorted union across all buckets with priority ordering: updated → failed → skipped → stale → fresh → scanned.
- **`containerStatus`**: An internal `types.ContainerReport` implementation carrying randomly generated IDs, a pooled container name, a derived image name, an optional error, and a state. Used as the element type in all report slices.

### String Pools (`preview_strings.go`)

Pre-populated arrays of synthetic strings used by the data generator:
- 40 container names and 39 organisation names for generating realistic image references.
- 42 error messages and 20 skip-reason messages for failed and skipped container entries.
- 13 informational log messages and 5 error log messages for log entry generation.
