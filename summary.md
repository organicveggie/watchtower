# Watchtower — Feature Summary

Watchtower is a daemon that automatically monitors running Docker containers and restarts them when their images are
updated. It polls or responds to triggers, compares local images against their registries, and recreates stale
containers in-place — preserving their original configuration, networks, and volumes.

---

## Entry Point and CLI (`cmd`)

- **Root command**: Cobra-based CLI that wires together all subsystems — Docker client, scheduler, notifier, HTTP API,
and metrics — and drives the main execution loop.
- **Execution modes**:
  - **Scheduled**: Runs update cycles on a cron schedule or polling interval, then blocks until a shutdown signal.
  - **Run-once** (`--run-once`): Performs a single update cycle, sends notifications, and exits.
  - **HTTP API mode** (`--http-api-update`): Waits for external HTTP triggers rather than polling; optional periodic
  polling can be re-enabled alongside it.
- **Startup sequence**: Processes flag aliases, configures logging, resolves file-based secrets, creates the Docker
client, registers the notification hook, builds the container filter, validates configuration, checks for duplicate
Watchtower instances, registers API handlers, and starts the scheduler.
- **Graceful shutdown**: Blocks on OS signals; closes the notifier and scheduler cleanly on exit.
- **Notification migration tool** (`notify-upgrade` subcommand): Converts legacy per-service notification flags (email,
Slack, MSTeams, Gotify) into equivalent Shoutrrr URLs, writes them to a temporary file inside the container, and logs
the `docker cp` command to retrieve them. The file is auto-deleted after 5 minutes.

---

## Configuration and Flags (`internal/flags`)

All CLI flags and environment variable bindings in one place.

- **Docker connection flags**: `--host`, `--tlsverify`, `--api-version` (env: `DOCKER_HOST`, `DOCKER_TLS_VERIFY`,
`DOCKER_API_VERSION`). Defaults to the Unix socket at `/var/run/docker.sock`.
- **Scheduling**: `--interval` (seconds, default 86400) or `--schedule` (cron expression). Mutually exclusive;
`--interval` is converted to a cron expression internally.
- **Container selection**: `--label-enable`, `--disable-containers`, `--scope`, `--include-stopped`,
`--include-restarting`.
- **Update strategy**: `--no-pull`, `--no-restart`, `--monitor-only`, `--cleanup`, `--rolling-restart`,
`--revive-stopped`, `--label-take-precedence`.
- **Lifecycle hooks**: `--enable-lifecycle-hooks`.
- **Output**: `--log-level`, `--log-format` (`auto`/`json`/`logfmt`/`pretty`), `--porcelain v1`, `--debug`, `--trace`.
- **HTTP API**: `--http-api-update`, `--http-api-metrics`, `--http-api-token`, `--http-api-periodic-polls`.
- **Notifications**: 29 flags covering Shoutrrr URLs, template selection, title/hostname, delay, log level filtering,
and legacy per-service config for email, Slack, Teams, and Gotify.
- **Secret file resolution**: Sensitive flag values that are file paths are transparently replaced with the file's
contents at startup, enabling Docker secrets and mounted credential files.
- **Flag aliases**: `--porcelain v1` expands into the appropriate notification URL, template, and report flags;
`--debug`/`--trace` promote to `--log-level`.

---

## Update Engine (`internal/actions`)

The core business logic for a single update cycle.

- **Preflight checks**: Rejects rolling-restart + linked-container combinations as incompatible. Detects multiple
Watchtower instances sharing a scope and stops all but the most recently created one (optionally removing their images).
- **Scan phase**: Iterates matched containers, checks each for a stale image, and marks failures as skipped rather than
aborting the cycle. Verifies that stale containers can be safely recreated before proceeding.
- **Dependency ordering**: Topologically sorts containers by their declared dependencies so stop and start sequences
respect link order.
- **Implicit restart propagation**: Marks containers for restart when any of their dependencies is also being restarted.
- **Stop and restart**:
  - *Standard*: Stops all stale containers in reverse dependency order, then starts them in forward order.
  - *Rolling*: Stops and restarts one container at a time; cleans up the old image after each successful update.
- **Lifecycle hooks**: Pre/post-check hooks bracket the scan phase; pre-update hooks can signal skip-without-failure
via exit code `75` (`EX_TEMPFAIL`); post-update hooks run after a successful restart.
- **Image cleanup**: Removes old images after successful updates when `--cleanup` is enabled.
- **Self-update**: Renames the running Watchtower container before starting its replacement so the new instance can
reuse the original name.
- **Session report**: Returns a structured result classifying every container as scanned, updated, failed, skipped,
stale, or fresh.

---

## Docker Client and Container Abstraction (`pkg/container`)

All communication with the Docker daemon.

- **Listing and inspection**: Fetches containers with configurable status filters. Resolves
`network_mode: container:<id>` references to names. Tolerates missing image info gracefully.
- **Staleness detection**: Compares `Docker-Content-Digest` headers against local `RepoDigests` before pulling,
skipping the pull when digests match. Rejects digest-pinned images.
- **Stop, remove, recreate**: Full container lifecycle — stop signal, wait, remove, recreate with config delta
(image-default env vars, labels, and ports are subtracted before recreation), multi-network workaround, start.
- **Command execution**: `docker exec` with exit-code inspection; `EX_TEMPFAIL` signals a graceful skip.
- **Label-driven behaviour**: 14 `com.centurylinklabs.watchtower.*` labels control enable/disable, monitor-only,
no-pull, stop signal, scope, dependencies, and four lifecycle hook commands with per-hook timeouts.
- **Label precedence**: When enabled, per-container labels override global flags for `monitor-only` and `no-pull`.
- **Dependency sources**: `depends-on` label, Docker `--link` entries, and `network_mode: container:` references are
unified into a single dependency list.
- **Self-identification**: Reads `/proc/<pid>/cgroup` to detect the running container's own ID.

---

## Registry Integration (`pkg/registry`)

Authentication and image staleness checking against Docker registries.

- **Credential loading**: `REPO_USER`/`REPO_PASS` environment variables take precedence; falls back to the Docker
config file (`DOCKER_CONFIG`, defaulting to `/`). Supports native OS credential stores and plain file stores.
- **Pull option construction**: Encodes credentials for the Docker SDK's `ImagePull` call. Falls back to unauthenticated
retry if the initial pull is rejected.
- **Challenge-response authentication**: Full Docker Registry HTTP API V2 flow — probes `/v2/`, reads
`WWW-Authenticate`, handles Basic and Bearer challenges. Bearer token exchange constructs a scoped token endpoint URL
 and returns a formatted `Authorization` header.
- **Docker Hub normalisation**: Resolves `docker.io` to `index.docker.io`, prepends `library/` for official images, and
strips vanity host prefixes from scope paths.
- **Digest comparison**: Authenticated HEAD request to the manifest endpoint; reads `Docker-Content-Digest`; compares
against all local `RepoDigests`. Accepts `v1`, `v2`, `list.v2`, and OCI index manifest types. TLS verification disabled
for self-hosted registries.
- **HEAD warning strategy**: Configurable per-registry warning behaviour for failed HEAD requests (`always`, `never`,
`auto`).

---

## Notification System (`pkg/notifications`)

Sends update summaries to external services after each update cycle.

- **Shoutrrr-based dispatch**: Sends to any Shoutrrr-supported service via URLs. Async background goroutine with
configurable send delay.
- **Logrus hook**: Automatically collects log entries during an update session; batches them with the session report for
delivery at `SendNotification`. Outside a batch, entries are dispatched immediately.
- **Template system**: Go `text/template` with four built-in templates:
  - `default`: Scan summary + per-container lines; suppressed if nothing updated or failed.
  - `default-legacy`: One line per log entry.
  - `porcelain.v1.summary-no-log`: Machine-readable one-line-per-container output.
  - `json.v1`: Full session data as pretty-printed JSON.
- **Custom templates**: Any Go template string accepted via `--notification-template`. Functions available: `ToUpper`,
`ToLower`, `Title`, `ToJSON`.
- **Empty message suppression**: Notifications with empty rendered output are silently dropped.
- **Legacy adapters**: Email (SMTP + STARTTLS), Slack (auto-detects Discord), Microsoft Teams, and Gotify — each
converts its CLI flags to a Shoutrrr URL.
- **Template preview**: Offline `Render` function produces deterministic preview output from synthetic data, allowing
users to validate custom templates without running an update cycle.

---

## HTTP API (`pkg/api`)

Optional HTTP server for external integration, listening on `:8080`.

- **Bearer token authentication**: All endpoints require `Authorization: Bearer <token>`; returns `401` otherwise.
- **`GET /v1/update`**: Triggers an immediate update cycle. Accepts `?image=` to restrict to specific images. Concurrent
full-scan requests are dropped if an update is already running; targeted requests always execute. Lock is shared with
the scheduler to prevent overlap.
- **`GET /v1/metrics`**: Serves Prometheus metrics in the standard text exposition format. Tracks scans total, scans
skipped, containers scanned, containers updated, and containers failed — as gauges (per-scan) and counters (cumulative).

---

## Build Metadata (`internal/meta`)

- **Version**: Injected at build time via `-ldflags`; defaults to `"v0.0.0-unknown"`. Used in startup log messages.
- **User-Agent**: `"Watchtower/<version>"` — sent with all outbound registry HTTP requests for identifiability in access
logs.
