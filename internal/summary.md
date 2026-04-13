# `internal` Packages — Feature Summary

## `internal/actions` — Container Update Engine

The core business logic driving Watchtower's container update lifecycle.

### Preflight Checks (`check.go`)

- **Sanity check**: Validates that rolling restarts are not enabled when any matched container has Docker `--link`
  dependencies, which are incompatible with one-at-a-time restart ordering.
- **Multiple instance detection**: Scans for more than one running Watchtower container within the same scope. Stops all
  but the most recently created instance. Optionally removes the Docker images of stopped instances. When a scope is
  provided, only Watchtower containers within that scope are considered, allowing multiple Watchtower instances to
  coexist on the same host with non-overlapping scopes.

### Update Engine (`update.go`)

- **Full update session**: Orchestrates the complete lifecycle of a single update cycle:
  1. Executes pre-check lifecycle hooks inside every matched container (if enabled).
  2. Scans all matched containers for stale images; containers that cannot be checked are marked skipped rather than
     causing a hard failure.
  3. Verifies that stale containers can be safely recreated before proceeding.
  4. Sorts containers topologically by dependency order so linked containers are stopped and started correctly.
  5. Propagates restart intent to containers whose dependencies are also being restarted.
  6. Stops and restarts stale containers — either all at once (in dependency order) or one at a time (rolling restart).
  7. Removes old images after successful updates (if cleanup is enabled).
  8. Executes post-check lifecycle hooks inside every matched container (if enabled).
  9. Returns a structured report classifying all containers as scanned, updated, failed, skipped, stale, or fresh.
- **Implicit restart propagation**: Marks containers as needing a restart when any container they depend on (via
  `Links()`) is also being restarted, keeping the dependency graph consistent.
- **Rolling restarts**: Stops and restarts containers one at a time in reverse dependency order, with per-container
  cleanup after each successful update.
- **Watchtower self-update**: When restarting the Watchtower container itself, renames the running instance first so the
  new container can reuse the original name.
- **Lifecycle hooks**: Pre-update hooks can signal that a container should be skipped (via `EX_TEMPFAIL`). Post-update
  hooks run after a successful restart. Pre/post-check hooks bracket the entire scan phase.

---

## `internal/flags` — CLI Flag and Configuration Management

Centralises all command-line flag declarations, environment variable bindings, and configuration processing.

### Flag Registration

- **Docker flags**: `--host`, `--tlsverify`, `--api-version` — connection settings passed to the Docker SDK.
- **System flags**: 29 flags covering scheduling (`--interval`, `--schedule`), container selection (`--label-enable`,
  `--disable-containers`, `--scope`), update strategy (`--no-pull`, `--no-restart`, `--monitor-only`, `--cleanup`,
  `--rolling-restart`, `--revive-stopped`), output (`--log-level`, `--log-format`, `--porcelain`), HTTP API
  (`--http-api-update`, `--http-api-metrics`, `--http-api-token`, `--http-api-periodic-polls`), and miscellaneous
  runtime options.
- **Notification flags**: 29 flags covering the Shoutrrr URL-based system, notification formatting and filtering, and
  legacy per-service adapters for email (SMTP), Slack, Microsoft Teams, and Gotify.

### Configuration Processing

- **Defaults**: Initialises Viper with environment variable auto-binding and sets sensible defaults (e.g. 24-hour poll
  interval, 10-second stop timeout, Docker Unix socket).
- **Environment variable bridging**: Writes parsed Docker flag values back into the `DOCKER_*` environment variables
  that the Docker SDK reads at client construction time.
- **Secret file resolution**: For a fixed list of sensitive flags (SMTP password, Slack hook URL, Teams hook, Gotify
  token, notification URLs, HTTP API token), replaces any flag value that is a file path with the contents of that file,
  enabling secrets to be injected via mounted files rather than as literal strings.
- **Flag aliases and mutual exclusion**: Expands `--porcelain v1` into the appropriate combination of notification URL,
  template, and report flags; converts `--interval N` into a cron expression; enforces mutual exclusion between
  `--interval` and `--schedule`; promotes `--debug` and `--trace` to their `--log-level` equivalents.
- **Logging setup**: Configures the global Logrus logger with the selected format (`auto`, `json`, `logfmt`, `pretty`)
  and level.

---

## `internal/meta` — Build-Time Metadata

Exposes version and identity information injected at build time.

- **Version string**: The `Version` variable defaults to `"v0.0.0-unknown"` and is overridden at build time via
  `-ldflags` with the output of `git describe --tags`. Referenced in startup log messages.
- **User-Agent header**: The `UserAgent` variable is derived from `Version` as `"Watchtower/<version>"` and is used in
  all outbound HTTP requests to image registries, making Watchtower identifiable in registry access logs.

---

## `internal/util` — General-Purpose Utilities

Pure helper functions with no Docker-specific dependencies.

### Collection Diffing (`util.go`)

- **`SliceEqual`**: Compares two string slices for equality (same length, same order). Used when comparing environment
  variable lists, entrypoints, and command arrays between a running container and its source image.
- **`SliceSubtract`**: Returns elements of one slice not present in another. Used to strip image-default environment
  variables from a container's env list before recreation.
- **`StringMapSubtract`**: Returns entries from one string map whose key-value pairs do not exactly match the second
  map. Used to subtract image-default labels from a container's label set.
- **`StructMapSubtract`**: Returns entries from one struct-valued map whose keys are absent from the second. Used to
  subtract image-default exposed ports from a container's port set.

### Random Name Generation (`rand_name.go`)

- **`RandName`**: Generates a random 32-character alphanumeric string for use as a temporary container name when
  Watchtower renames its own running instance before self-updating.

### Random Hash Generation (`rand_sha256.go`)

- **`GenerateRandomSHA256`**: Generates a random 64-character hex string (no prefix), used in tests to produce realistic
  container and image IDs.
- **`GenerateRandomPrefixedSHA256`**: Generates a `sha256:<64 hex chars>` string matching the full image ID format used
  by the Docker API. Uses `crypto/rand` for secure randomness.
