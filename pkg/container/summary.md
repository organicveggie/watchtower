# `pkg/container` Packages — Feature Summary

## `pkg/container` — Docker Client and Container Abstraction

The primary abstraction layer between Watchtower and the Docker daemon.

### Docker Client (`client.go`)

- **Container listing**: Queries the Docker daemon for containers matching configurable status filters. Supports including stopped (`created`/`exited`) and restarting containers in addition to running ones. Applies a user-supplied filter function to the results before returning.
- **Container inspection**: Fetches full container details by ID, including image info. Resolves `network_mode: container:<id>` references to container names so that network mode links remain valid after a supplier container is recreated. Continues without error if image info or network supplier lookup fails.
- **Staleness detection**: Determines whether a container's image is out of date. First attempts a digest comparison to skip the pull when the image is already current. Rejects digest-pinned (`sha256:`) images. Supports skipping the pull entirely via a per-container label or global flag.
- **Container stop and removal**: Sends the container's configured stop signal (defaulting to `SIGTERM`) and waits for it to exit before removing it. Skips explicit removal if `AutoRemove` is set. Confirms the container is gone after removal.
- **Container recreation**: Recreates a container from its existing configuration. Works around a Docker API limitation with multiple networks by attaching to networks one at a time. Respects `ReviveStopped` — if the original container was not running and the option is disabled, creates without starting.
- **Container renaming**: Renames a container via the Docker API. Used when Watchtower needs to free its own name before starting a new instance.
- **Image removal**: Removes a Docker image by ID with force. Reports deleted and untagged layers at debug log level.
- **Command execution**: Runs an arbitrary shell command inside a container via `docker exec`. Captures output and inspects the exit code. Exit code `75` (`EX_TEMPFAIL`) signals a skip-without-failure; any other non-zero code returns an error.
- **Network config sanitisation**: Strips the container's own short ID from network endpoint aliases when retrieving network config, preventing stale aliases from accumulating across updates.
- **Warning strategy**: Controls whether a failed registry HEAD request is logged as a warning. Supports three strategies — always warn, never warn, or warn only for known rate-limiting registries (Docker Hub and ghcr.io).

### Container Type (`container.go`)

- **Docker label integration**: Reads a comprehensive set of `com.centurylinklabs.watchtower.*` labels to control per-container behaviour, including enable/disable, monitor-only, no-pull, stop signal, scope, dependency declaration, and all four lifecycle hook commands with configurable timeouts.
- **Label precedence**: Combines per-container label values with global flag values. When `LabelPrecedence` is enabled, the container label takes priority over the global flag for `monitor-only` and `no-pull`.
- **Dependency resolution**: Derives container dependency links from three sources in combination: the `depends-on` label, Docker `--link` host config entries, and implicit `network_mode: container:` references.
- **Image name resolution**: Returns the container's image name from the container config, falling back to the Zodiac `original-image` label if present. Appends `:latest` if no tag is specified.
- **Configuration delta computation**: `GetCreateConfig` subtracts image-default environment variables, labels, and exposed ports from the container's config before passing it to the Docker create API, ensuring only user-supplied overrides are carried forward when a container is recreated.
- **Configuration verification**: Validates that all fields required to recreate the container are present and non-nil. Automatically initialises `ExposedPorts` to an empty map when port bindings exist but `ExposedPorts` is nil, for Docker API compatibility.
- **Stale and restart state tracking**: Exposes `SetStale`/`IsStale` and `SetLinkedToRestarting`/`IsLinkedToRestarting` flags, set by the update engine during a session.

### Label Metadata (`metadata.go`)

Declares all Docker label key constants used by Watchtower:

| Category | Labels |
|---|---|
| Identity | `com.centurylinklabs.watchtower` |
| Control | `enable`, `monitor-only`, `no-pull`, `stop-signal`, `scope`, `depends-on` |
| Lifecycle hooks | `lifecycle.pre-check`, `lifecycle.post-check`, `lifecycle.pre-update`, `lifecycle.post-update`, `lifecycle.pre-update-timeout`, `lifecycle.post-update-timeout` |
| Legacy | `com.centurylinklabs.zodiac.original-image` |

### Self-Identification (`cgroup_id.go`)

- **Container ID detection**: Reads `/proc/<pid>/cgroup` for the current process and extracts the running container's Docker ID by matching a 64-character hex string following a `/docker/` path segment. Returns an empty ID when running outside a container.

### Sentinel Errors (`errors.go`)

Defines package-level sentinel errors for missing image info, missing container info, invalid container config, and label-not-found, used across `Container` methods and `VerifyConfiguration`.

---

## `pkg/container/mocks` — Test Support Library

A test-only package providing mock implementations, a simulated Docker daemon, and JSON fixture data. Not used in production.

### Mock Docker API Server (`ApiServer.go`)

- **Simulated Docker daemon**: Provides `http.HandlerFunc` constructors for individual Docker API endpoints (`GET /containers/{id}/json`, `GET /images/{id}/json`, `GET /containers/json`, `POST /containers/{id}/kill`, `DELETE /containers/{id}`, `DELETE /images/{id}`), composable onto a `ghttp.Server` to form a complete mock daemon.
- **Fixture-backed responses**: `GetContainerHandlers` constructs the full set of handlers for a named container fixture, including any referenced containers (e.g. network suppliers) and their images. `ListContainersHandler` reads a shared `containers.json` fixture and filters by container status.
- **Named container references**: Declares canonical `ContainerRef` variables (`Watchtower`, `Running`, `Stopped`, `Restarting`, `NetConsumerOK`, `NetConsumerInvalidSupplier`) that map container names to their fixture files and serve as the primary handle for constructing test scenarios.

### Mock FilterableContainer (`FilterableContainer.go`)

- **Testify mock**: Auto-generated `mock.Mock` implementation of `types.FilterableContainer`, allowing filter tests to set up per-method return values and assert call expectations without constructing real container objects.

### JSON Fixture Data (`data/`)

- Seven container fixtures covering: running Watchtower, running non-Watchtower, stopped, restarting, network supplier, and two network consumer variants (valid and missing supplier).
- Four image fixtures: `portainer/portainer:latest`, `containrrr/watchtower:latest`, `qmcgaw/gluetun:latest`, `nginx:latest`.
- One container-list fixture used by `ListContainersHandler`, filtered by status at request time.
