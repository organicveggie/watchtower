# `internal/flags` Package

This package centralises all command-line flag and environment variable handling for Watchtower. It is responsible for
declaring every flag the application accepts, binding those flags to their corresponding environment variables via
[Viper](https://github.com/spf13/viper), resolving secrets stored in files, processing flag aliases, and configuring the
global logger. It is consumed almost exclusively by `cmd/root.go` and `cmd/notify-upgrade.go`.

---

## Files

### `flags.go`

Declares and registers all flags, provides helpers for reading their values, and implements several cross-cutting
concerns such as secret file resolution, alias expansion, and logging setup.

**Constants:**

| Constant | Value | Description |
| -------- | ----- | ----------- |
| `DockerAPIMinVersion` | `"1.25"` | The minimum Docker API version required by Watchtower. Used as the default value for the `--api-version` flag. |

---

**Public Functions:**

---

#### `RegisterDockerFlags(rootCmd *cobra.Command)`

Registers the three flags that are passed directly to the Docker API client:

| Flag | Env Var | Description |
| ---- | ------- | ----------- |
| `--host, -H` | `DOCKER_HOST` | Docker daemon socket to connect to. |
| `--tlsverify, -v` | `DOCKER_TLS_VERIFY` | Enable TLS and verify the remote daemon's certificate. |
| `--api-version, -a` | `DOCKER_API_VERSION` | Docker API version for the client to use. |

---

#### `RegisterSystemFlags(rootCmd *cobra.Command)`

Registers all flags that control Watchtower's runtime behaviour. These cover scheduling, container selection, update
strategy, and HTTP API configuration. The full set of flags registered is:

| Flag | Env Var | Description |
| ---- | ------- | ----------- |
| `--interval, -i` | `WATCHTOWER_POLL_INTERVAL` | How often (in seconds) to check for image updates. |
| `--schedule, -s` | `WATCHTOWER_SCHEDULE` | Cron expression defining the update schedule. Mutually exclusive with `--interval`. |
| `--stop-timeout, -t` | `WATCHTOWER_TIMEOUT` | Duration to wait before forcefully stopping a container. |
| `--no-pull` | `WATCHTOWER_NO_PULL` | Skip pulling new images; only compare local image cache. |
| `--no-restart` | `WATCHTOWER_NO_RESTART` | Do not restart containers after updating. |
| `--no-startup-message` | `WATCHTOWER_NO_STARTUP_MESSAGE` | Suppress the startup notification. |
| `--cleanup, -c` | `WATCHTOWER_CLEANUP` | Remove old images after a successful update. |
| `--remove-volumes` | `WATCHTOWER_REMOVE_VOLUMES` | Remove anonymous volumes before recreating a container. |
| `--label-enable, -e` | `WATCHTOWER_LABEL_ENABLE` | Only watch containers with the enable label set to `true`. |
| `--disable-containers, -x` | `WATCHTOWER_DISABLE_CONTAINERS` | Comma/space-separated list of container names to always exclude. |
| `--log-format, -l` | `WATCHTOWER_LOG_FORMAT` | Console log format: `Auto`, `LogFmt`, `Pretty`, or `JSON`. |
| `--debug, -d` | `WATCHTOWER_DEBUG` | Alias for `--log-level debug`. |
| `--trace` | `WATCHTOWER_TRACE` | Alias for `--log-level trace`. Exposes credentials in logs. |
| `--monitor-only, -m` | `WATCHTOWER_MONITOR_ONLY` | Check for updates and send notifications, but do not restart containers. |
| `--run-once, -R` | `WATCHTOWER_RUN_ONCE` | Perform a single update cycle and exit. |
| `--include-restarting` | `WATCHTOWER_INCLUDE_RESTARTING` | Include containers that are in a restarting state. |
| `--include-stopped, -S` | `WATCHTOWER_INCLUDE_STOPPED` | Include created and exited containers. |
| `--revive-stopped` | `WATCHTOWER_REVIVE_STOPPED` | Start stopped containers that have had their image updated. |
| `--enable-lifecycle-hooks` | `WATCHTOWER_LIFECYCLE_HOOKS` | Enable pre/post-check and pre/post-update lifecycle hook execution. |
| `--rolling-restart` | `WATCHTOWER_ROLLING_RESTART` | Restart containers one at a time instead of all at once. |
| `--http-api-update` | `WATCHTOWER_HTTP_API_UPDATE` | Enable the HTTP API update trigger endpoint. |
| `--http-api-metrics` | `WATCHTOWER_HTTP_API_METRICS` | Enable the Prometheus metrics endpoint. |
| `--http-api-token` | `WATCHTOWER_HTTP_API_TOKEN` | Bearer token required for HTTP API requests. |
| `--http-api-periodic-polls` | `WATCHTOWER_HTTP_API_PERIODIC_POLLS` | Continue running scheduled updates even when the HTTP API is enabled. |
| `--no-color` | `NO_COLOR` | Disable ANSI colour codes in log output. |
| `--scope` | `WATCHTOWER_SCOPE` | Limit this instance to containers tagged with a specific scope label. |
| `--porcelain, -P` | `WATCHTOWER_PORCELAIN` | Write machine-readable session results to stdout. Supported value: `v1`. |
| `--log-level` | `WATCHTOWER_LOG_LEVEL` | Maximum log level written to stderr (`panic` → `trace`). |
| `--health-check` | _(none)_ | Perform a health check and exit. Intended for use in Docker `HEALTHCHECK` only. |
| `--label-take-precedence` | `WATCHTOWER_LABEL_TAKE_PRECEDENCE` | Let per-container labels override global arguments. |

---

#### `RegisterNotificationFlags(rootCmd *cobra.Command)`

Registers all flags related to sending notifications. Covers the modern Shoutrrr URL-based approach as well as the
legacy per-service flags (email, Slack, MSTeams, Gotify) that are retained for backwards compatibility.

| Flag | Env Var | Description |
| ---- | ------- | ----------- |
| `--notifications, -n` | `WATCHTOWER_NOTIFICATIONS` | Notification types to enable (`email`, `slack`, `msteams`, `gotify`, `shoutrrr`). |
| `--notifications-level` | `WATCHTOWER_NOTIFICATIONS_LEVEL` | Minimum log level that triggers a notification. |
| `--notifications-delay` | `WATCHTOWER_NOTIFICATIONS_DELAY` | Seconds to wait before sending a notification batch. |
| `--notifications-hostname` | `WATCHTOWER_NOTIFICATIONS_HOSTNAME` | Override the hostname shown in notification titles. |
| `--notification-template` | `WATCHTOWER_NOTIFICATION_TEMPLATE` | Go `text/template` string used to format notification messages. |
| `--notification-url` | `WATCHTOWER_NOTIFICATION_URL` | One or more Shoutrrr service URLs. Can reference a file. |
| `--notification-report` | `WATCHTOWER_NOTIFICATION_REPORT` | Use the session report struct as template data instead of log entries. |
| `--notification-title-tag` | `WATCHTOWER_NOTIFICATION_TITLE_TAG` | Prefix tag added to notification titles. |
| `--notification-skip-title` | `WATCHTOWER_NOTIFICATION_SKIP_TITLE` | Omit the title parameter from notifications entirely. |
| `--notification-log-stdout` | `WATCHTOWER_NOTIFICATION_LOG_STDOUT` | Write `logger://` output to stdout instead of the log stream. |
| `--warn-on-head-failure` | `WATCHTOWER_WARN_ON_HEAD_FAILURE` | When to warn about HEAD request failures: `always`, `auto`, or `never`. |
| `--notification-email-from` | `WATCHTOWER_NOTIFICATION_EMAIL_FROM` | Sender address for email notifications. |
| `--notification-email-to` | `WATCHTOWER_NOTIFICATION_EMAIL_TO` | Recipient address for email notifications. |
| `--notification-email-delay` | `WATCHTOWER_NOTIFICATION_EMAIL_DELAY` | Per-email delay in seconds (legacy). |
| `--notification-email-server` | `WATCHTOWER_NOTIFICATION_EMAIL_SERVER` | SMTP server hostname. |
| `--notification-email-server-port` | `WATCHTOWER_NOTIFICATION_EMAIL_SERVER_PORT` | SMTP server port (default `25`). |
| `--notification-email-server-tls-skip-verify` | `WATCHTOWER_NOTIFICATION_EMAIL_SERVER_TLS_SKIP_VERIFY` | Disable TLS certificate verification for SMTP. For testing only. |
| `--notification-email-server-user` | `WATCHTOWER_NOTIFICATION_EMAIL_SERVER_USER` | SMTP authentication username. |
| `--notification-email-server-password` | `WATCHTOWER_NOTIFICATION_EMAIL_SERVER_PASSWORD` | SMTP authentication password. Can reference a file. |
| `--notification-email-subjecttag` | `WATCHTOWER_NOTIFICATION_EMAIL_SUBJECTTAG` | Subject line prefix for email notifications. |
| `--notification-slack-hook-url` | `WATCHTOWER_NOTIFICATION_SLACK_HOOK_URL` | Slack (or Discord) incoming webhook URL. Can reference a file. |
| `--notification-slack-identifier` | `WATCHTOWER_NOTIFICATION_SLACK_IDENTIFIER` | Display name for Watchtower in Slack messages (default `watchtower`). |
| `--notification-slack-channel` | `WATCHTOWER_NOTIFICATION_SLACK_CHANNEL` | Override the webhook's default Slack channel. |
| `--notification-slack-icon-emoji` | `WATCHTOWER_NOTIFICATION_SLACK_ICON_EMOJI` | An emoji code string to use in place of the default icon. |
| `--notification-slack-icon-url` | `WATCHTOWER_NOTIFICATION_SLACK_ICON_URL` | An icon image URL string to use in place of the default icon. |
| `--notification-msteams-hook` | `WATCHTOWER_NOTIFICATION_MSTEAMS_HOOK_URL` | Microsoft Teams incoming webhook URL. Can reference a file. |
| `--notification-msteams-data` | `WATCHTOWER_NOTIFICATION_MSTEAMS_USE_LOG_DATA` | Include structured log fields as Teams message facts. |
| `--notification-gotify-url` | `WATCHTOWER_NOTIFICATION_GOTIFY_URL` | Gotify server URL. |
| `--notification-gotify-token` | `WATCHTOWER_NOTIFICATION_GOTIFY_TOKEN` | Gotify application token. Can reference a file. |
| `--notification-gotify-tls-skip-verify` | `WATCHTOWER_NOTIFICATION_GOTIFY_TLS_SKIP_VERIFY` | Disable TLS certificate verification for Gotify. For testing only. |

---

#### `SetDefaults()`

Initialises Viper with `AutomaticEnv()` and sets default values for all environment variables that have one. Must be
called before any flag registration. Key defaults include:

| Variable | Default |
| -------- | ------- |
| `DOCKER_HOST` | `unix:///var/run/docker.sock` |
| `DOCKER_API_VERSION` | `DockerAPIMinVersion` (`"1.25"`) |
| `WATCHTOWER_POLL_INTERVAL` | `86400` (24 hours) |
| `WATCHTOWER_TIMEOUT` | `10s` |
| `WATCHTOWER_NOTIFICATIONS` | `[]` (empty slice) |
| `WATCHTOWER_NOTIFICATIONS_LEVEL` | `"info"` |
| `WATCHTOWER_NOTIFICATION_EMAIL_SERVER_PORT` | `25` |
| `WATCHTOWER_NOTIFICATION_EMAIL_SUBJECTTAG` | `""` (empty string) |
| `WATCHTOWER_NOTIFICATION_SLACK_IDENTIFIER` | `"watchtower"` |
| `WATCHTOWER_LOG_LEVEL` | `"info"` |
| `WATCHTOWER_LOG_FORMAT` | `"auto"` |

---

#### `EnvConfig(cmd *cobra.Command) error`

Reads the Docker connection flags (`--host`, `--tlsverify`, `--api-version`) from the parsed command and writes them to
the corresponding `DOCKER_*` environment variables. This is necessary because the Docker SDK client reads its
configuration from the environment rather than accepting values directly.

Returns an error if any flag cannot be read or if `os.Setenv` fails.

---

#### `ReadFlags(cmd *cobra.Command) (cleanup bool, noRestart bool, monitorOnly bool, timeout time.Duration)`

Reads the four most commonly used runtime flags from the persistent flag set and returns them as typed Go values. Calls
`log.Fatal` if any flag is missing, as these flags are always expected to be registered before `ReadFlags` is called.

---

#### `GetSecretsFromFiles(rootCmd *cobra.Command)`

Iterates over a fixed list of sensitive flags and, for each one, checks whether the flag's current value is a path to an
existing file rather than a literal secret. If it is, the flag's value is replaced with the contents of that file
(whitespace trimmed). For slice-valued flags (e.g. `--notification-url`), each entry in the slice is checked and
expanded independently, and blank lines in the file are discarded.

The flags checked are: `notification-email-server-password`, `notification-slack-hook-url`, `notification-msteams-hook`,
`notification-gotify-token`, `notification-url`, and `http-api-token`.

Calls `log.Fatalf` if a referenced file cannot be read.

---

#### `ProcessFlagAliases(flags *pflag.FlagSet)`

Expands higher-level convenience flags into their underlying equivalents. Should be called after flag parsing and before
the main run logic. Performs the following transformations:

- **`--porcelain v1`** — Appends `logger://` to `--notification-url`, sets `--notification-log-stdout`,
  `--notification-report`, and `--notification-template` to the appropriate porcelain template, if those flags have not
  already been set explicitly.
- **`--interval` / `--schedule`** — Enforces mutual exclusion (calls `log.Fatal` if both are set), then converts
  `--interval N` into the equivalent cron expression `@every Ns` and writes it to `--schedule`.
- **`--debug`** — Sets `--log-level` to `debug`.
- **`--trace`** — Sets `--log-level` to `trace`.

---

#### `SetupLogging(f *pflag.FlagSet) error`

Reads `--log-format` and `--log-level` from the flag set and applies them to the global Logrus logger. Supported formats
and their effects:

| Format | Behaviour |
| ------ | --------- |
| `auto` | `TextFormatter` with colour support driven by the terminal and `NO_COLOR`/`CLICOLOR` environment variables. |
| `json` | `JSONFormatter`. |
| `logfmt` | `TextFormatter` with colours disabled and full timestamps enabled. |
| `pretty` | `TextFormatter` with colours forced on (unless `--no-color`) and timestamps shown as elapsed time. |

Returns an error if the format string or level string is not recognised.

---

## Internal Helpers

| Function | Description |
| -------- | ----------- |
| `envString(key)` | Binds a Viper key to its environment variable and returns the current string value. |
| `envStringSlice(key)` | Binds a Viper key to its environment variable and returns the current string slice value. |
| `envInt(key)` | Binds a Viper key to its environment variable and returns the current integer value. |
| `envBool(key)` | Binds a Viper key to its environment variable and returns the current boolean value. |
| `envDuration(key)` | Binds a Viper key to its environment variable and returns the current `time.Duration` value. |
| `setEnvOptStr(env, opt)` | Sets an environment variable to `opt` if `opt` is non-empty and differs from the current value. |
| `setEnvOptBool(env, opt)` | Sets an environment variable to `"1"` if `opt` is `true`. |
| `getSecretFromFile(flags, secret)` | Core implementation used by `GetSecretsFromFiles`. Handles both scalar and slice-valued flags. |
| `isFile(s)` | Returns `true` if `s` refers to a path that exists on disk. Skips strings containing `:` at a position other than index 1 (to avoid misidentifying URLs as file paths while still supporting Windows-style `C:\` paths). |
| `flagIsEnabled(flags, name)` | Returns the boolean value of a named flag, calling `log.Fatalf` if the flag does not exist. |
| `appendFlagValue(flags, name, values...)` | Appends one or more values to a slice-typed flag. Returns an error if the flag does not exist or is not a slice. |
| `setFlagIfDefault(flags, name, value)` | Sets a flag's value only if it has not been explicitly changed by the user. |

---

## Test Coverage

`flags_test.go` covers the following scenarios:

| Test | Description |
| ---- | ----------- |
| `TestEnvConfig_Defaults` | Verifies that `EnvConfig` writes the default `DOCKER_HOST` and clears `DOCKER_TLS_VERIFY` when no flags are set. |
| `TestEnvConfig_Custom` | Verifies that custom `--host`, `--tlsverify`, and `--api-version` values are correctly propagated to environment variables. |
| `TestGetSecretsFromFilesWithString` | Verifies that a plain string value is left unchanged by `GetSecretsFromFiles`. |
| `TestGetSecretsFromFilesWithFile` | Verifies that a flag value pointing to a file path is replaced with the file's contents. |
| `TestGetSliceSecretsFromFiles` | Verifies that slice flags expand file references and merge them with inline values, discarding blank lines. |
| `TestHTTPAPIPeriodicPollsFlag` | Verifies that `--http-api-periodic-polls` is correctly parsed as `true`. |
| `TestIsFile` | Verifies that URLs are not treated as files and that the running binary path is correctly identified as a file. |
| `TestProcessFlagAliases` | Verifies that `--porcelain v1`, `--interval`, and `--trace` are correctly expanded into their underlying flags. |
| `TestProcessFlagAliasesLogLevelFromEnvironment` | Verifies that `WATCHTOWER_DEBUG=true` in the environment causes the log level to be set to `debug`. |
| `TestLogFormatFlag` | Verifies that each supported `--log-format` value (`Auto`, `JSON`, `pretty`, `logfmt`) configures the correct Logrus formatter, and that an invalid value returns an error. |
| `TestLogLevelFlag` | Verifies that an unrecognised `--log-level` value returns an error from `SetupLogging`. |
| `TestProcessFlagAliasesSchedAndInterval` | Verifies that providing both `--schedule` and `--interval` causes a fatal error. |
| `TestProcessFlagAliasesScheduleFromEnvironment` | Verifies that `WATCHTOWER_SCHEDULE` in the environment is respected by `ProcessFlagAliases`. |
| `TestProcessFlagAliasesInvalidPorcelaineVersion` | Verifies that an unsupported `--porcelain` version value causes a fatal error. |
| `TestFlagsArePrecentInDocumentation` | Verifies that every registered flag and environment variable appears in at least one of the documentation Markdown files under `docs/`. |
