# `pkg/types` Package

This package defines all shared interfaces, type aliases, and data structures used across Watchtower. It is the
dependency-free foundation of the codebase — every other package may import it, but it imports nothing from within
Watchtower itself. Its primary role is to establish the contracts (interfaces) that decouple the major subsystems from
their concrete implementations, enabling testability and clear separation of concerns.

---

## Files

### `container.go`

Defines the two ID type aliases and their shared `ShortID` method, and the `Container` interface that abstracts all
per-container behaviour.

**Types:**

#### `ImageID string`

A named string type representing a Docker image content hash. Typically a full `sha256:<64 hex chars>` string.

| Method | Description |
|---|---|
| `ShortID() string` | Returns a 12-character short form of the image ID, stripping the `sha256:` prefix if present. Delegates to the private `shortID` helper. |

---

#### `ContainerID string`

A named string type representing a Docker container instance ID.

| Method | Description |
|---|---|
| `ShortID() string` | Returns a 12-character short form of the container ID, stripping the `sha256:` prefix if present. Delegates to the private `shortID` helper. |

---

**Internal Helpers:**

#### `shortID(longID string) string`

Shared implementation behind both `ShortID` methods. Parses any `<prefix>:` preamble in the ID string:

- If the prefix is `"sha256"`, strips it and returns the first 12 characters of the hash portion.
- If the prefix is anything else (e.g. an unknown digest algorithm), includes it in the output and returns the first `12
  + len(prefix) + 1` characters.
- If no prefix separator is found, returns the first 12 characters directly.
- If the string is shorter than the required length, returns it unchanged.

---

#### `Container` _(interface)_

The central abstraction for a Docker container. Implemented by `pkg/container.Container` and by mock types in
`internal/actions/mocks` and `pkg/container/container_mock_test.go`. Consumed by virtually every package in the
codebase.

| Method | Return type | Description |
|---|---|---|
| `ContainerInfo()` | `*types.ContainerJSON` | Returns the raw Docker container inspection data. |
| `ID()` | `ContainerID` | Returns the container's full ID. |
| `IsRunning()` | `bool` | Returns whether the container is in the running state. |
| `IsRestarting()` | `bool` | Returns whether the container is in the restarting state. |
| `Name()` | `string` | Returns the container name. |
| `ImageID()` | `ImageID` | Returns the image ID the container was started from. |
| `SafeImageID()` | `ImageID` | Returns the image ID without panicking if image info is unavailable. |
| `ImageName()` | `string` | Returns the container's image name and tag. |
| `Enabled()` | `(bool, bool)` | Returns the value of the enable label and whether it was set. |
| `IsMonitorOnly(UpdateParams)` | `bool` | Returns whether the container should be monitored but not updated. |
| `IsNoPull(UpdateParams)` | `bool` | Returns whether image pulling should be skipped for this container. |
| `Scope()` | `(string, bool)` | Returns the scope label value and whether it was set. |
| `Links()` | `[]string` | Returns the names of containers this container depends on. |
| `ToRestart()` | `bool` | Returns whether the container should be restarted in this cycle. |
| `IsWatchtower()` | `bool` | Returns whether the container is a Watchtower instance. |
| `StopSignal()` | `string` | Returns the custom stop signal label value. |
| `HasImageInfo()` | `bool` | Returns whether image info is available. |
| `ImageInfo()` | `*types.ImageInspect` | Returns the raw image inspection data. |
| `GetLifecyclePreCheckCommand()` | `string` | Returns the pre-check lifecycle hook command. |
| `GetLifecyclePostCheckCommand()` | `string` | Returns the post-check lifecycle hook command. |
| `GetLifecyclePreUpdateCommand()` | `string` | Returns the pre-update lifecycle hook command. |
| `GetLifecyclePostUpdateCommand()` | `string` | Returns the post-update lifecycle hook command. |
| `PreUpdateTimeout()` | `int` | Returns the pre-update command timeout in minutes. |
| `PostUpdateTimeout()` | `int` | Returns the post-update command timeout in minutes. |
| `VerifyConfiguration()` | `error` | Checks that all fields required to recreate the container are present. |
| `SetStale(bool)` | — | Sets whether the container has a stale image. |
| `IsStale()` | `bool` | Returns whether the container has a stale image. |
| `SetLinkedToRestarting(bool)` | — | Marks the container as linked to a restarting container. |
| `IsLinkedToRestarting()` | `bool` | Returns whether the container is linked to a restarting container. |
| `GetCreateConfig()` | `*dc.Config` | Returns the container config to use when recreating the container. |
| `GetCreateHostConfig()` | `*dc.HostConfig` | Returns the host config to use when recreating the container. |

---

### `update_params.go`

Defines the parameter struct passed to `internal/actions.Update` and propagated through the update cycle.

**Types:**

#### `UpdateParams`

Bundles all runtime options that govern the behaviour of a single update session.

| Field | Type | Description |
|---|---|---|
| `Filter` | `Filter` | The container filter function built from CLI flags and container names. |
| `Cleanup` | `bool` | Remove old images after a successful update. |
| `NoRestart` | `bool` | Do not restart containers after updating their image. |
| `Timeout` | `time.Duration` | How long to wait before forcefully stopping a container. |
| `MonitorOnly` | `bool` | Check for updates and notify but do not restart containers. |
| `NoPull` | `bool` | Skip pulling new images; use the local cache only. |
| `LifecycleHooks` | `bool` | Enable pre/post-check and pre/post-update lifecycle hook execution. |
| `RollingRestart` | `bool` | Restart containers one at a time rather than all at once. |
| `LabelPrecedence` | `bool` | Allow per-container labels to override global arguments. |

---

### `filter.go`

Defines the `Filter` function type.

**Types:**

#### `Filter func(FilterableContainer) bool`

A function type used throughout the container listing and filtering pipeline. A `Filter` accepts a `FilterableContainer`
and returns `true` if the container should be included. Filters are composed via the constructor functions in
`pkg/filters`. The zero value (`nil`) should not be passed to `Client.ListContainers`; `filters.NoFilter` is used as the
always-true base case.

---

### `filterable_container.go`

Defines the minimal interface used by the filter system.

**Types:**

#### `FilterableContainer` _(interface)_

A subset of the `Container` interface exposing only the fields needed to evaluate filter predicates. Implemented by
`pkg/container.Container` and mocked by `pkg/container/mocks.FilterableContainer`. Keeping this interface narrow
prevents filters from depending on the full `Container` interface and makes mocking easier.

| Method | Return type | Description |
|---|---|---|
| `Name()` | `string` | The container name. |
| `IsWatchtower()` | `bool` | Whether the container is a Watchtower instance. |
| `Enabled()` | `(bool, bool)` | The enable label value and whether it was set. |
| `Scope()` | `(string, bool)` | The scope label value and whether it was set. |
| `ImageName()` | `string` | The container's image name and tag. |

---

### `notifier.go`

Defines the `Notifier` interface implemented by `pkg/notifications.shoutrrrTypeNotifier`.

**Types:**

#### `Notifier` _(interface)_

The contract for any notification service. Consumed by `cmd/root.go` which holds a `Notifier` instance and calls
`StartNotification` / `SendNotification` around each update cycle, and `Close` on shutdown.

| Method | Description |
|---|---|
| `StartNotification()` | Begins accumulating log entries for the current notification batch. |
| `SendNotification(Report)` | Flushes the accumulated entries and the session report, rendering and dispatching the notification. |
| `AddLogHook()` | Registers the notifier as a Logrus hook so it receives log entries automatically. |
| `GetNames() []string` | Returns the list of notification service names currently configured. |
| `GetURLs() []string` | Returns the list of raw Shoutrrr service URLs currently configured. |
| `Close()` | Waits for all pending notifications to be dispatched, then shuts down. |

---

### `convertible_notifier.go`

Defines the interfaces implemented by the legacy per-service notification adapters in `pkg/notifications`.

**Types:**

#### `ConvertibleNotifier` _(interface)_

Implemented by the four legacy notifier types (`emailTypeNotifier`, `slackTypeNotifier`, `msTeamsTypeNotifier`,
`gotifyTypeNotifier`). Used by `AppendLegacyUrls` in `pkg/notifications/notifier.go` to convert old-style flag-based
configs into Shoutrrr URLs.

| Method | Return type | Description |
|---|---|---|
| `GetURL(c *cobra.Command)` | `(string, error)` | Builds and returns the Shoutrrr URL equivalent of the notifier's current configuration. |

---

#### `DelayNotifier` _(interface)_

An optional extension of `ConvertibleNotifier`. Currently implemented only by `emailTypeNotifier`. Checked via a type
assertion in `AppendLegacyUrls` to determine whether to apply a per-notifier send delay.

| Method | Return type | Description |
|---|---|---|
| `GetDelay()` | `time.Duration` | Returns the delay to apply before sending notifications for this service. |

---

### `report.go`

Defines the two interfaces that make up the session report consumed by the notification and metrics systems.

**Types:**

#### `Report` _(interface)_

Implemented by `pkg/session.report`. Represents the immutable result of a completed update session.

| Method | Return type | Description |
|---|---|---|
| `Scanned()` | `[]ContainerReport` | All containers inspected (excluding skipped). |
| `Updated()` | `[]ContainerReport` | Containers successfully updated. |
| `Failed()` | `[]ContainerReport` | Containers whose update failed. |
| `Skipped()` | `[]ContainerReport` | Containers explicitly skipped. |
| `Stale()` | `[]ContainerReport` | Containers with a newer image available but not updated. |
| `Fresh()` | `[]ContainerReport` | Containers whose image was already up to date. |
| `All()` | `[]ContainerReport` | Deduplicated union of all six buckets, sorted by container ID. |

---

#### `ContainerReport` _(interface)_

Implemented by `pkg/session.ContainerStatus` and `pkg/notifications/preview/data.containerStatus`. Represents the
per-container result within a session report.

| Method | Return type | Description |
|---|---|---|
| `ID()` | `ContainerID` | The container's ID. |
| `Name()` | `string` | The container name. |
| `CurrentImageID()` | `ImageID` | The image ID the container was running at session start. |
| `LatestImageID()` | `ImageID` | The newest image ID found during the session. |
| `ImageName()` | `string` | The image name and tag. |
| `Error()` | `string` | The error message, or empty string if none. |
| `State()` | `string` | The human-readable state name. |

---

### `registry_credentials.go`

Defines the credential pair type used when constructing registry auth tokens.

**Types:**

#### `RegistryCredentials`

A simple struct holding a username and password for basic registry authentication. Used by
`pkg/registry/digest.TransformAuth` when unmarshalling a base64-encoded Docker config auth JSON blob.

| Field | Type | Description |
|---|---|---|
| `Username` | `string` | The registry username. |
| `Password` | `string` | The registry password or access token. |

---

### `token_response.go`

Defines the JSON response type returned by a registry token endpoint.

**Types:**

#### `TokenResponse`

Unmarshalled from the JSON body of a successful bearer token exchange in `pkg/registry/auth.GetBearerHeader`.

| Field | Type | JSON key | Description |
|---|---|---|---|
| `Token` | `string` | `"token"` | The bearer token returned by the registry's auth endpoint. |

---

## Test Coverage

This package has no test files. The types defined here are exercised through the tests of the packages that implement or
consume them — most extensively in `pkg/container/container_test.go` (which tests `ShortID` via `util_test.go`),
`internal/actions/update_test.go` (which exercises `Container`, `UpdateParams`, `Filter`, and `Report`), and
`pkg/notifications/shoutrrr_test.go` (which exercises `Notifier` and `Report`).
