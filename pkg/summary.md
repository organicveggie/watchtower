# `pkg` Packages — Feature Summary

## HTTP API (`pkg/api`)

Watchtower's optional HTTP API server, enabled via `--http-api-update` and/or `--http-api-metrics`.

- **Bearer token authentication**: All endpoints are automatically protected by `RequireToken` middleware, which validates the `Authorization: Bearer <token>` header and returns `401 Unauthorized` on mismatch.
- **Conditional startup**: The server only launches when at least one handler is registered; requires a non-empty token or terminates with a fatal error.
- **Blocking and non-blocking modes**: Can run in the current goroutine or as a background goroutine. Listens on `:8080` (hardcoded).
- **Prometheus metrics endpoint** (`/v1/metrics`): Wires the Prometheus default registry's HTTP handler to the API, serving all Watchtower metrics in the standard text exposition format.
- **On-demand update trigger** (`/v1/update`): Accepts an optional `?image=` query parameter to restrict the update to specific images. Untargeted requests are dropped silently if an update is already running. A shared lock channel makes HTTP-triggered and scheduled updates mutually exclusive.

---

## Docker Client and Container Abstraction (`pkg/container`)

The primary interface between Watchtower and the Docker daemon.

### Docker Client
- **Container listing**: Queries the daemon with configurable status filters (running, stopped, restarting). Applies a user-supplied filter function before returning.
- **Container inspection**: Fetches full details including image info. Resolves `network_mode: container:<id>` references to names so links remain valid after recreation.
- **Staleness detection**: Attempts a digest comparison before pulling to avoid unnecessary network traffic. Rejects digest-pinned images. Respects per-container and global no-pull settings.
- **Stop and removal**: Sends the configured stop signal (default `SIGTERM`), waits for exit, then removes. Respects `AutoRemove`. Confirms removal before returning.
- **Container recreation**: Recreates from existing config, working around a Docker API multi-network limitation by attaching one network at a time. Respects `ReviveStopped`.
- **Self-update support**: Renames the running Watchtower container to free its name before starting the updated instance.
- **Image removal**: Force-removes by ID; reports deleted and untagged layers at debug level.
- **Command execution**: Runs shell commands via `docker exec`. Exit code `75` (`EX_TEMPFAIL`) signals skip-without-failure.
- **Network alias sanitisation**: Strips the container's own short ID from network endpoint aliases to prevent accumulation across updates.
- **HEAD request warning strategy**: Three modes — always warn, never warn, or warn only for Docker Hub and ghcr.io.

### Container Type
- **Label-driven configuration**: Reads the full `com.centurylinklabs.watchtower.*` label set for enable/disable, monitor-only, no-pull, stop signal, scope, dependency declaration, and lifecycle hook commands with timeouts.
- **Label precedence**: Per-container labels can override global flags for `monitor-only` and `no-pull` when `LabelPrecedence` is enabled.
- **Dependency resolution**: Combines `depends-on` label, Docker `--link` entries, and `network_mode: container:` references into a unified dependency list.
- **Configuration delta**: Subtracts image-default env vars, labels, and exposed ports from the container config before recreation, carrying forward only user-supplied overrides.
- **Configuration verification**: Validates all fields needed for recreation; auto-initialises `ExposedPorts` when port bindings exist but the map is nil.
- **Self-identification**: Reads `/proc/<pid>/cgroup` to extract the running container's Docker ID for use in CLI output.

### Supported Labels

| Category | Labels |
|---|---|
| Identity | `com.centurylinklabs.watchtower` |
| Control | `enable`, `monitor-only`, `no-pull`, `stop-signal`, `scope`, `depends-on` |
| Lifecycle hooks | `lifecycle.pre-check`, `lifecycle.post-check`, `lifecycle.pre-update`, `lifecycle.post-update`, `lifecycle.pre-update-timeout`, `lifecycle.post-update-timeout` |
| Legacy | `com.centurylinklabs.zodiac.original-image` |

---

## Notification System (`pkg/notifications`)

Complete notification system bridging the update cycle with external services via Shoutrrr.

### Template System
- **Template data model**: `Data` struct carries per-session Logrus log entries and a structured `Report`, plus static `Title` and `Host` fields set at startup.
- **Four built-in templates**: `default-legacy` (log-entry lines), `default` (scan summary with per-container lines, suppressed when nothing updated or failed), `porcelain.v1.summary-no-log` (machine-readable one-line-per-container), `json.v1` (full data as pretty-printed JSON).
- **Custom templates**: Users may supply any Go `text/template` string via `--notification-template`. Available template functions: `ToUpper`, `ToLower`, `Title`, `ToJSON`.

### Core Engine
- **Shoutrrr integration**: Dispatches to any Shoutrrr-supported service (Slack, Teams, Discord, email, Gotify, and more) via service URLs.
- **Logrus hook**: Receives log entries automatically at or below the configured level. Ignores entries tagged `notify: "no"` to prevent recursive loops.
- **Batched sending**: Accumulates log entries during an update session; flushes together with the session report at `SendNotification`. Entries outside a batch are sent immediately.
- **Async dispatch**: Background goroutine reads rendered messages from a buffered channel, applies a configurable delay, and dispatches. `Close` drains and waits for completion.
- **Empty message suppression**: Notifications with empty rendered output are silently dropped.

### Notifier Construction
- **Legacy service conversion**: `AppendLegacyUrls` converts `--notifications` entries for email, Slack, MSTeams, and Gotify into Shoutrrr URLs.
- **Title composition**: Combines `"Watchtower updates"` with an optional tag prefix and hostname suffix. Omitted entirely when `--notification-skip-title` is set.
- **Delay resolution**: Per-notifier delay (email only) takes precedence over the global `--notifications-delay` flag.

### Legacy Adapters
- **Email**: SMTP with `STARTTLS` by default; configurable server, port, credentials, and per-send delay.
- **Slack**: Webhook-based; auto-detects Discord URLs and produces a Discord Shoutrrr URL instead.
- **Microsoft Teams**: Parses incoming webhook URL to extract token components; applies brand colour.
- **Gotify**: URL + token; sets `DisableTLS` automatically for `http://` URLs.

### Template Preview (`pkg/notifications/preview`)
- **Offline preview**: `Render` parses a template, populates it with synthetic data matching specified container states and log levels, and returns the rendered string — no Docker daemon required.
- **Deterministic**: Fixed random seed ensures identical inputs always produce identical output.
- **Synthetic data generator**: Produces realistic container entries (pooled names, random IDs, state-appropriate error messages) and log entries (level-appropriate messages, monotonically advancing timestamps) without any runtime state.

---

## Registry Integration (`pkg/registry`)

All logic for authenticating with and querying Docker image registries.

### Credential Management
- **Source priority**: Environment variables (`REPO_USER`/`REPO_PASS`) take precedence over the Docker config file. Config credentials are per-registry; env credentials apply globally.
- **Docker config loading**: Reads from `DOCKER_CONFIG` (defaulting to `/`). Supports both native OS credential stores (e.g. `docker-credential-osxkeychain`) and plain file stores.
- **Auth encoding**: URL-safe base64-encodes a JSON `AuthConfig` for use as the `X-Registry-Auth` header.
- **Pull option construction**: Combines credentials with a `PrivilegeFunc` fallback that retries without auth if the initial pull is rejected, allowing public images on strict registries to succeed.

### Challenge-Response Authentication
- **Full V2 auth flow**: `GetToken` sends an unauthenticated request to `/v2/`, reads the `WWW-Authenticate` challenge, and dispatches to Basic or Bearer handling.
- **Bearer token exchange**: Constructs the token endpoint URL from the challenge's `realm` and `service` fields, adds a `pull`-scoped repository claim, sends credentials, and returns `"Bearer <token>"`.
- **Docker Hub scope handling**: Prepends `library/` for official single-segment images; strips `docker.io`/`index.docker.io` vanity prefixes without affecting other registries.
- **Malformed challenge robustness**: Handles trailing commas and valueless keys in challenge strings without panicking.

### Digest Comparison
- **Staleness check**: Transforms credentials to the registry-expected format, obtains a token, builds the manifest URL, fetches the `Docker-Content-Digest` header via HEAD request, and compares against all local `RepoDigests`.
- **Broad compatibility**: Accepts all four manifest media types (`v1`, `v2`, `list.v2`, OCI index) and disables TLS verification for self-hosted registries with self-signed certificates.

### URL Utilities
- **Registry address resolution**: Parses any image reference and returns the registry hostname, normalising `docker.io` to `index.docker.io`.
- **Manifest URL construction**: Builds `https://<host>/v2/<image>/manifests/<tag>` from a container's image name. Defaults to `:latest` when no tag is present. Rejects digest-pinned references.
