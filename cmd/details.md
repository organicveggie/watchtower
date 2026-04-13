# `cmd` Package

This package contains the entry points for Watchtower's command-line interface. It defines the root command and all
subcommands using [Cobra](https://github.com/spf13/cobra), wires together the application's major subsystems (Docker
client, scheduler, notifier, HTTP API, and metrics), and handles the main execution loop.

---

## Files

### `root.go`

The core of the Watchtower CLI. It defines the root `watchtower` command, registers all flags, initializes all
subsystems, and runs the main update loop — either on a schedule, once, or in HTTP API mode.

**Responsibilities:**

- Registering Docker, system, and notification flags via the `flags` package
- Initializing the Docker client, notifier, and scheduler
- Running container update cycles (scheduled or one-shot)
- Starting the HTTP API server for update triggers and metrics
- Handling graceful shutdown on OS signals

**Public Functions:**

---

#### `NewRootCommand() *cobra.Command`

Factory function that creates and returns the root `watchtower` Cobra command. The command is configured with `Run` and
`PreRun` lifecycle hooks but does not register flags itself — that is done in `init()`.

---

#### `Execute()`

Entry point called from `main.go`. Adds subcommands (e.g. `notify-upgrade`) to the root command and calls `Execute()`
on it. Calls `log.Fatal` on any error.

---

#### `PreRun(cmd *cobra.Command, _ []string)`

Cobra `PreRun` hook that runs before the main `Run` function. Handles all initialization:

- Processes flag aliases (e.g. `--porcelain`, `--debug`, `--trace`)
- Sets up logging format and level
- Reads secrets from files where applicable
- Reads common flags (`--cleanup`, `--no-restart`, `--monitor-only`, `--stop-timeout`)
- Reads system flags (`--label-enable`, `--disable-containers`, `--scope`, etc.)
- Configures Docker environment variables
- Creates the Docker client (`container.NewClient`)
- Creates and registers the notifier log hook

---

#### `Run(c *cobra.Command, names []string)`

The main execution function for the root command. Orchestrates the full Watchtower runtime:

- Builds the container filter from CLI arguments and flags
- Handles the `--health-check` flag (exits 0 if healthy)
- Enforces mutual exclusion of `--rolling-restart` and `--monitor-only`
- Waits for the Docker client to initialize
- Runs a sanity check (e.g. rolling restart + linked containers)
- If `--run-once`: runs one update cycle, sends notifications, and exits
- Checks for multiple Watchtower instances running with the same scope (`actions.CheckForMultipleWatchtowerInstances`)
- If HTTP API mode (`--http-api-update`): registers the `/v1/update` handler and optionally blocks periodic polling
- If metrics mode (`--http-api-metrics`): registers the `/v1/metrics` handler
- Starts the HTTP API server
- Starts the update scheduler

---

### `notify-upgrade.go`

Implements the `notify-upgrade` subcommand, which helps users migrate from legacy notification flags to the modern
[Shoutrrr](https://containrrr.dev/shoutrrr/) URL format.

When run, it converts any configured legacy notification options (email, Slack, MSTeams, Gotify) into equivalent
Shoutrrr URLs, writes them to a temporary file inside the container, and logs the `docker cp` command needed to
retrieve the file. The file is automatically deleted after 5 minutes or on container shutdown.

**Public Functions:**

---

#### `NewNotifyUpgradeCommand() *cobra.Command`

Factory function that creates and returns the `notify-upgrade` Cobra subcommand. Sets `runNotifyUpgrade` as the `Run`
handler.

---

## Internal Helpers

These unexported functions support the public surface but are worth noting for maintainers:

| Function | Description |
| -------- | ----------- |
| `logNotifyExit(err)` | Logs an error, closes the notifier, and exits with code 1 |
| `awaitDockerClient()` | Sleeps 1 second to allow the Docker client to initialize |
| `formatDuration(d)` | Formats a `time.Duration` into a human-readable string (e.g. `"1 hour, 30 minutes"`) |
| `writeStartupMessage(c, sched, filtering)` | Logs and optionally notifies about Watchtower's startup configuration |
| `runUpgradesOnSchedule(c, filter, filtering, lock)` | Starts the cron scheduler, runs update cycles, and blocks until a shutdown signal |
| `runUpdatesWithNotifications(filter)` | Runs a single update cycle, sends notifications, and returns scan metrics |
