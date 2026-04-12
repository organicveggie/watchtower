# `pkg/notifications/preview/data` Package

This package provides all synthetic data used to render notification template previews. It has no runtime role — its sole purpose is to supply realistic-looking session state (container statuses, log entries, report summaries, and rendered preview strings) so that users and tests can validate custom `--notification-template` values without needing to run a real update cycle. It is consumed by `pkg/notifications/preview`.

---

## Files

### `data.go`

Declares the synthetic `types.Report` used as the top-level input to template rendering.

**Package-level Variables:**

#### `SlimReport types.Report`

A synthetic `types.Report` instance populated in `init()` with a representative set of container outcomes across all result categories. Constructed by calling `types.Report.All()` with a slice of `types.ContainerStatus` values. The containers and their outcomes are:

| Container name | Image | Result |
|---|---|---|
| `approvals` | `containrrr/watchtower:mytag-1` | Fresh |
| `epicer` | `containrrr/watchtower:mytag-2` | Updated |
| `contaner` | `containrrr/watchtower:mytag-3` | Failed |
| `postfix` | `containrrr/watchtower:mytag-4` | Skipped |
| `oauth2` | `containrrr/watchtower:mytag-5` | Scanned |

Each `ContainerStatus` is built using `types.ContainerStatus.WithImageInfo()` to attach image name metadata, ensuring the rendered template has access to both the container name and its image reference.

---

### `logs.go`

Declares the synthetic log entries used to populate the legacy (non-report) template data path.

**Package-level Variables:**

#### `Entries []*log.Entry`

A slice of pre-built Logrus `log.Entry` values populated in `init()`. Represents the log output that would be produced during a typical update session, covering a variety of log levels (`info`, `warn`, `error`) and messages. Used by `pkg/notifications/preview` when rendering templates that consume log entries rather than the structured report.

---

### `preview_strings.go`

Declares the expected rendered output strings for the built-in notification templates. These are used in tests to assert that a given template, when applied to `SlimReport` or `Entries`, produces the correct output.

**Package-level Variables:**

| Variable | Description |
|---|---|
| `LegacyTemplate string` | The expected rendered output of the default legacy (log-entry-based) notification template when applied to `Entries`. |
| `ReportTemplate string` | The expected rendered output of the default report-based notification template when applied to `SlimReport`. |
| `PorcelainTemplate string` | The expected rendered output of the `--porcelain v1` machine-readable template when applied to `SlimReport`. |

All three are declared as package-level `var` strings and initialised as string literals.

---

### `report.go`

Provides a constructor for building synthetic `types.ContainerStatus` values used in `data.go`.

**Public Functions:**

---

#### `NewContainerStatus(name string, image string, result string) types.ContainerStatus`

Creates and returns a `types.ContainerStatus` with the given container name, image name, and result string. Internally constructs a minimal `ContainerJSON` to satisfy the `ContainerStatus` interface, attaches image metadata via `WithImageInfo()`, and sets the result state. Used exclusively by `data.go` to populate `SlimReport`.

---

### `status.go`

Declares the string constants used as result identifiers when constructing synthetic container statuses in `report.go`.

**Package-level Constants:**

| Constant | Value | Description |
|---|---|---|
| `UpdatedStatus` | `"updated"` | Identifies a container that was successfully updated. |
| `FreshStatus` | `"fresh"` | Identifies a container whose image was already up to date. |
| `FailedStatus` | `"failed"` | Identifies a container whose update attempt failed. |
| `SkippedStatus` | `"skipped"` | Identifies a container that was explicitly skipped. |
| `ScannedStatus` | `"scanned"` | Identifies a container that was scanned but not updated. |

---

## Test Coverage

This package has no dedicated test file. The expected output strings declared in `preview_strings.go` are consumed by `pkg/notifications/preview` tests to assert correct template rendering.
