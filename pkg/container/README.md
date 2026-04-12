# `pkg/container` Package

This package is the primary abstraction layer between Watchtower and the Docker daemon. It defines the `Client` interface and its production implementation, the `Container` type and all of its behaviour, label-driven metadata accessors, cgroup-based self-identification, and the sentinel errors used across the package. It is consumed by `internal/actions`, `cmd/root.go`, and the registry packages.

---

## Files

### `client.go`

Implements the `Client` interface using the official Docker SDK. Handles all communication with the Docker daemon: listing and inspecting containers, pulling images, starting and stopping containers, executing commands, and removing images.

**Types:**

#### `Client` _(interface)_

The interface through which all of Watchtower's Docker interactions are performed. Implemented by `dockerClient` and by the mock client in `internal/actions/mocks`.

| Method | Description |
|---|---|
| `ListContainers(types.Filter) ([]types.Container, error)` | Returns all containers matched by the supplied filter. |
| `GetContainer(containerID types.ContainerID) (types.Container, error)` | Returns the full details of a single container by ID. |
| `StopContainer(types.Container, time.Duration) error` | Sends a stop signal to a container and waits for it to exit, then removes it. |
| `StartContainer(types.Container) (types.ContainerID, error)` | Creates and starts a new container using the configuration of the supplied container. |
| `RenameContainer(types.Container, string) error` | Renames a container. |
| `IsContainerStale(types.Container, types.UpdateParams) (stale bool, latestImage types.ImageID, err error)` | Determines whether a newer image is available for the container. |
| `ExecuteCommand(containerID types.ContainerID, command string, timeout int) (SkipUpdate bool, err error)` | Runs a shell command inside a container via `docker exec`. |
| `RemoveImageByID(types.ImageID) error` | Removes a Docker image by ID. |
| `WarnOnHeadPullFailed(container types.Container) bool` | Returns whether a failed HEAD request for this container's registry should be logged as a warning rather than silently ignored. |

---

#### `ClientOptions`

Configuration struct passed to `NewClient` to control the behaviour of the Docker client wrapper.

| Field | Type | Description |
|---|---|---|
| `RemoveVolumes` | `bool` | Remove anonymous volumes when a container is removed. |
| `IncludeStopped` | `bool` | Include created and exited containers in listings. |
| `ReviveStopped` | `bool` | Start stopped containers that have had their image updated. |
| `IncludeRestarting` | `bool` | Include restarting containers in listings. |
| `WarnOnHeadFailed` | `WarningStrategy` | Controls when to warn about failed HEAD requests to registries. |

---

#### `WarningStrategy`

A string type that controls warning behaviour for failed registry HEAD requests.

| Constant | Description |
|---|---|
| `WarnAlways` | Always emit a warning when a HEAD request fails. |
| `WarnNever` | Never emit a warning when a HEAD request fails. |
| `WarnAuto` | Emit a warning only for registries known to rate-limit (Docker Hub, ghcr.io). |

---

**Public Functions:**

---

#### `NewClient(opts ClientOptions) Client`

Factory function that creates and returns a production `dockerClient` by reading connection parameters from the standard Docker environment variables (`DOCKER_HOST`, `DOCKER_TLS_VERIFY`, `DOCKER_API_VERSION`). Calls `log.Fatalf` if the SDK client cannot be initialised.

---

**`dockerClient` Method Implementations:**

---

##### `WarnOnHeadPullFailed(container types.Container) bool`

Delegates to `registry.WarnOnAPIConsumption` when the strategy is `WarnAuto`, and returns a fixed `true` or `false` for `WarnAlways` and `WarnNever` respectively.

---

##### `ListContainers(fn types.Filter) ([]types.Container, error)`

Queries the Docker daemon for containers matching the configured status filters (running, and optionally stopped and/or restarting). For each returned container, calls `GetContainer` to fetch full details, then applies the user-supplied filter function `fn`. Returns only the containers for which `fn` returns `true`.

---

##### `GetContainer(containerID types.ContainerID) (types.Container, error)`

Inspects a single container by ID. If the container uses `network_mode: container:<id>`, resolves the referenced container's name so that the network mode reference remains valid after the supplier is recreated. Fetches the container's image info via `ImageInspectWithRaw` and returns a fully populated `Container`. If image info cannot be fetched, returns a `Container` with a nil `imageInfo` rather than an error.

---

##### `StopContainer(c types.Container, timeout time.Duration) error`

Stops a running container by sending its configured stop signal (defaulting to `SIGTERM`), waits for it to exit, then removes it. Respects the `AutoRemove` host config flag — if set, skips the explicit `ContainerRemove` call. After removal, waits again to confirm the container is gone, returning an error if it persists.

---

##### `GetNetworkConfig(c types.Container) *network.NetworkingConfig`

Returns the container's current network endpoint configuration, with the container's own short ID removed from the aliases list of each network endpoint. This prevents stale container ID aliases from accumulating across updates.

---

##### `StartContainer(c types.Container) (types.ContainerID, error)`

Recreates a container from its current configuration. To work around a Docker API limitation with multiple networks, it creates the container connected to only one network, then disconnects and reconnects to all networks in the full config. Respects `ReviveStopped` — if the original container was not running and `ReviveStopped` is false, returns after creation without starting.

---

##### `RenameContainer(c types.Container, newName string) error`

Renames the given container to `newName` via the Docker API.

---

##### `IsContainerStale(container types.Container, params types.UpdateParams) (bool, types.ImageID, error)`

Determines whether a container's image is out of date. Unless `container.IsNoPull(params)` is true, pulls the latest image first via `PullImage`. Then calls `HasNewImage` to compare the current and latest image IDs.

---

##### `RemoveImageByID(id types.ImageID) error`

Removes a Docker image by ID with `Force: true`. At debug log level, reports which image layers were deleted and untagged.

---

##### `ExecuteCommand(containerID types.ContainerID, command string, timeout int) (SkipUpdate bool, err error)`

Runs a shell command (`sh -c <command>`) inside a container via the Docker exec API. Attaches to the exec session to capture output, then inspects the exit code. An exit code of `75` (`EX_TEMPFAIL`) signals that the update should be skipped without being treated as a failure. Any other non-zero exit code returns an error.

---

**Internal Helpers in `client.go`:**

| Function | Description |
|---|---|
| `createListFilter()` | Builds a Docker API filter argument including `running` and optionally `created`, `exited`, and `restarting` statuses based on `ClientOptions`. |
| `HasNewImage(ctx, container)` | Inspects the image referenced by the container's image name and compares its ID to the container's current image ID. |
| `PullImage(ctx, container)` | Pulls the latest image for a container. First attempts a digest comparison via `digest.CompareDigest` to skip the pull if the image is already up to date. Rejects pinned (`sha256:`) images with an error. |
| `doStartContainer(ctx, c, creation)` | Calls `ContainerStart` after the container has been created. |
| `waitForExecOrTimeout(ctx, ID, output, timeout)` | Polls `ContainerExecInspect` until the exec finishes or the context times out. |
| `waitForStopOrTimeout(c, waitTime)` | Polls `ContainerInspect` until the container's `Running` state is false or the timeout elapses. |

---

### `container.go`

Defines the `Container` struct and implements the `types.Container` interface. Provides all introspection and configuration-building behaviour for individual containers.

**Types:**

#### `Container`

The core container type. Wraps the Docker SDK's `ContainerJSON` and `ImageInspect` structs and exposes a higher-level interface over them.

| Field | Type | Description |
|---|---|---|
| `LinkedToRestarting` | `bool` | Set by `UpdateImplicitRestart` when a dependency of this container is being restarted. |
| `Stale` | `bool` | Set during the update scan when a newer image is found for this container. |
| `containerInfo` | `*types.ContainerJSON` | The raw Docker container inspection result. |
| `imageInfo` | `*types.ImageInspect` | The raw Docker image inspection result. May be `nil` if image info was unavailable. |

**Public Functions:**

---

#### `NewContainer(containerInfo *types.ContainerJSON, imageInfo *types.ImageInspect) *Container`

Factory function. Returns a new `Container` wrapping the supplied Docker SDK structs.

---

**`Container` Methods:**

| Method | Return type | Description |
|---|---|---|
| `ContainerInfo()` | `*types.ContainerJSON` | Returns the raw Docker container inspection data. |
| `ID()` | `types.ContainerID` | Returns the container's full ID. |
| `IsRunning()` | `bool` | Returns `true` if `State.Running` is true. |
| `IsRestarting()` | `bool` | Returns `true` if `State.Restarting` is true. |
| `Name()` | `string` | Returns the container name as reported by Docker (including the leading `/`). |
| `ImageID()` | `types.ImageID` | Returns the ID of the image the container was started from. Panics if `imageInfo` is nil. |
| `SafeImageID()` | `types.ImageID` | Returns the image ID, or an empty string if `imageInfo` is nil. |
| `ImageName()` | `string` | Returns the image name from the container config, falling back to the zodiac label if present. Appends `:latest` if no tag is specified. |
| `Enabled()` | `(bool, bool)` | Returns the parsed value of the `com.centurylinklabs.watchtower.enable` label and whether it was set. |
| `IsMonitorOnly(params)` | `bool` | Returns whether this container should be monitored but not updated, combining the per-container label with the global flag and label-precedence setting. |
| `IsNoPull(params)` | `bool` | Returns whether image pulling should be skipped for this container, combining the per-container label with the global flag and label-precedence setting. |
| `Scope()` | `(string, bool)` | Returns the value of the `com.centurylinklabs.watchtower.scope` label and whether it was set. |
| `Links()` | `[]string` | Returns the names of all containers this container depends on, sourced from the `depends-on` label, Docker `--link` host config, or implicit network mode links. |
| `ToRestart()` | `bool` | Returns `true` if the container is stale or linked to a restarting container. |
| `IsWatchtower()` | `bool` | Returns `true` if the container has the `com.centurylinklabs.watchtower=true` label. |
| `StopSignal()` | `string` | Returns the value of the `com.centurylinklabs.watchtower.stop-signal` label, or an empty string. |
| `HasImageInfo()` | `bool` | Returns `true` if image info is available (i.e. `imageInfo` is not nil). |
| `ImageInfo()` | `*types.ImageInspect` | Returns the raw image inspection data. |
| `PreUpdateTimeout()` | `int` | Returns the pre-update command timeout in minutes from the container label, defaulting to `1`. |
| `PostUpdateTimeout()` | `int` | Returns the post-update command timeout in minutes from the container label, defaulting to `1`. |
| `GetLifecyclePreCheckCommand()` | `string` | Returns the `pre-check` lifecycle label value, or empty string. |
| `GetLifecyclePostCheckCommand()` | `string` | Returns the `post-check` lifecycle label value, or empty string. |
| `GetLifecyclePreUpdateCommand()` | `string` | Returns the `pre-update` lifecycle label value, or empty string. |
| `GetLifecyclePostUpdateCommand()` | `string` | Returns the `post-update` lifecycle label value, or empty string. |
| `VerifyConfiguration()` | `error` | Checks that all fields required to recreate the container are present and non-nil. If port bindings exist but `ExposedPorts` is nil, initialises it to an empty map rather than returning an error. |
| `GetCreateConfig()` | `*dockercontainer.Config` | Returns a container config suitable for passing to the Docker create API. Subtracts image defaults from the container config so that only user-supplied overrides are carried forward. |
| `GetCreateHostConfig()` | `*dockercontainer.HostConfig` | Returns the container's host config with link aliases rewritten into the format expected by the Docker create API. |
| `SetStale(bool)` | — | Sets the `Stale` field. |
| `IsStale()` | `bool` | Returns the `Stale` field. |
| `SetLinkedToRestarting(bool)` | — | Sets the `LinkedToRestarting` field. |
| `IsLinkedToRestarting()` | `bool` | Returns the `LinkedToRestarting` field. |

---

**Internal Helpers in `container.go`:**

| Function | Description |
|---|---|
| `getContainerOrGlobalBool(globalVal, label, contPrecedence)` | Combines a global boolean flag with a per-container label value, respecting the label-precedence setting. Used by `IsMonitorOnly` and `IsNoPull`. |

---

### `metadata.go`

Declares all Docker label key constants used by Watchtower and provides low-level label accessor helpers. Also contains `ContainsWatchtowerLabel`, the function used to identify Watchtower containers.

**Constants:**

| Constant | Label Key |
|---|---|
| `watchtowerLabel` | `com.centurylinklabs.watchtower` |
| `signalLabel` | `com.centurylinklabs.watchtower.stop-signal` |
| `enableLabel` | `com.centurylinklabs.watchtower.enable` |
| `monitorOnlyLabel` | `com.centurylinklabs.watchtower.monitor-only` |
| `noPullLabel` | `com.centurylinklabs.watchtower.no-pull` |
| `dependsOnLabel` | `com.centurylinklabs.watchtower.depends-on` |
| `zodiacLabel` | `com.centurylinklabs.zodiac.original-image` |
| `scope` | `com.centurylinklabs.watchtower.scope` |
| `preCheckLabel` | `com.centurylinklabs.watchtower.lifecycle.pre-check` |
| `postCheckLabel` | `com.centurylinklabs.watchtower.lifecycle.post-check` |
| `preUpdateLabel` | `com.centurylinklabs.watchtower.lifecycle.pre-update` |
| `postUpdateLabel` | `com.centurylinklabs.watchtower.lifecycle.post-update` |
| `preUpdateTimeoutLabel` | `com.centurylinklabs.watchtower.lifecycle.pre-update-timeout` |
| `postUpdateTimeoutLabel` | `com.centurylinklabs.watchtower.lifecycle.post-update-timeout` |

**Public Functions:**

---

#### `ContainsWatchtowerLabel(labels map[string]string) bool`

Returns `true` if the supplied label map contains the key `com.centurylinklabs.watchtower` with the value `"true"`. Used by `IsWatchtower()` and by `filters.WatchtowerContainersFilter`.

---

**Internal Helpers in `metadata.go`:**

| Function | Description |
|---|---|
| `getLabelValueOrEmpty(label)` | Returns the value of a label from the container's config, or an empty string if the label is absent. |
| `getLabelValue(label)` | Returns the value of a label and a boolean indicating whether it was present. |
| `getBoolLabelValue(label)` | Parses a label value as a boolean. Returns `errorLabelNotFound` if the label is absent, or a `strconv` error if the value cannot be parsed. |

---

### `cgroup_id.go`

Provides self-identification for the running Watchtower container. Used by `cmd/notify-upgrade.go` to determine the container ID so it can print the correct `docker cp` command for the user.

**Public Functions:**

---

#### `GetRunningContainerID() (cid types.ContainerID, err error)`

Reads `/proc/<pid>/cgroup` for the current process and attempts to extract a Docker container ID from the contents. Returns an empty `ContainerID` if no match is found (e.g. when running outside a container), or an error if the file cannot be read.

---

**Internal Helpers in `cgroup_id.go`:**

| Function | Description |
|---|---|
| `getRunningContainerIDFromString(s string)` | Applies `dockerContainerPattern` (a regexp matching 64-character hex strings following `/docker/`) to a cgroup file's contents and returns the first captured container ID, or an empty string. |

---

### `errors.go`

Declares the package-level sentinel errors returned by `Container` methods and `VerifyConfiguration`.

| Error | Description |
|---|---|
| `errorNoImageInfo` | Returned when an operation requires image info but `imageInfo` is nil. |
| `errorNoContainerInfo` | Returned by `VerifyConfiguration` when `containerInfo` is nil. |
| `errorInvalidConfig` | Returned by `VerifyConfiguration` when the container or host config within `containerInfo` is nil. |
| `errorLabelNotFound` | Returned by `getBoolLabelValue` when the requested label is not present in the container's config. |

---

## Test Coverage

| File | Description |
|---|---|
| `container_suite_test.go` | Bootstraps the Ginkgo test suite for the `container_test` package. |
| `container_mock_test.go` | Defines `MockContainer` and a set of `MockContainerUpdate` option functions (`WithPortBindings`, `WithImageName`, `WithLinks`, `WithLabels`, `WithContainerState`, `WithHealthcheck`, `WithImageHealthcheck`) used to construct `Container` values for tests without needing a live Docker daemon. |
| `container_test.go` | Tests `Container` methods covering: `VerifyConfiguration` (all nil-field error cases and the port binding compatibility fix), `GetCreateConfig` (healthcheck delta computation including matching, differing, empty, and nil configs), label accessors (`Name`, `ID`, `Enabled`, `IsWatchtower`, `StopSignal`, `ImageName`, `Links`, `IsNoPull`, `PreUpdateTimeout`, `PostUpdateTimeout`), and the zodiac label fallback. |
| `client_test.go` | Tests the `dockerClient` implementation against a `ghttp` mock server. Covers `WarnOnHeadPullFailed` (all three `WarningStrategy` values), `PullImage` (pinned image rejection), `StopContainer` (container present after stop, container absent after stop), `RemoveImageByID` (debug log output, not-found error), `ListContainers` (no filter, name filter, watchtower filter, include-stopped, include-restarting, exclude-restarting), `ExecuteCommand` (container ID in log output), `GetNetworkConfig` (short ID alias removal), and container networking mode resolution (valid and missing supplier). |
| `cgroup_id_test.go` | Tests `getRunningContainerIDFromString` for a full multi-line cgroup file with a matching container ID, and for a file with no matching entry. |
| `util_test.go` | Tests `types.ImageID.ShortID` for IDs with and without the `sha256:` prefix, short IDs, and IDs with unknown prefixes. |
