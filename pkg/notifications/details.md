# `pkg/notifications` Package

This package implements Watchtower's entire notification system. It defines the data model used by templates, the core
Shoutrrr-based notifier that implements the `types.Notifier` and `logrus.Hook` interfaces, four legacy per-service
adapters that convert old-style flags into Shoutrrr URLs, the built-in template library, and the public constructor and
helper functions consumed by `cmd/root.go`. It is the primary integration point between Watchtower's update cycle and
any external notification service.

---

## Files

### `model.go`

Defines the data types passed into Go notification templates.

**Types:**

#### `StaticData`

The portion of the template data that is fixed for the lifetime of a notifier instance. Set once during initialisation
from flags and environment.

| Field | Type | Description |
|---|---|---|
| `Title` | `string` | The notification title, composed from the hostname and optional tag prefix. Empty if `--notification-skip-title` is set. |
| `Host` | `string` | The hostname of the machine running Watchtower. |

---

#### `Data`

The full template data model passed to every notification template at render time. Embeds `StaticData` and adds
per-session fields.

| Field | Type | Description |
|---|---|---|
| `StaticData` | _(embedded)_ | Static title and host fields. |
| `Entries` | `[]*log.Entry` | Logrus log entries collected during the session. Used by legacy templates. |
| `Report` | `t.Report` | The structured session report. Used by report templates. `nil` for legacy template mode. |

---

### `common_templates.go`

Declares the built-in named templates available to users via `--notification-template`. A user may specify one of these
names instead of a raw template string.

**Package-level Variables:**

#### `commonTemplates map[string]string`

| Key | Description |
|---|---|
| `default-legacy` | The default template for legacy (log-entry) mode. Renders each log entry's message on its own line. |
| `default` | The default template for report mode. Renders a summary line (`N Scanned, N Updated, N Failed`) and per-container lines for updated, fresh, skipped, and failed containers. Only sends a notification if at least one container was updated or failed. Falls back to log-entry rendering if no report is available. |
| `porcelain.v1.summary-no-log` | The machine-readable template used by `--porcelain v1`. Renders one line per container in the format `<name> (<image>): <state>`, with an error suffix for failed/skipped containers. Outputs `"no containers matched filter"` if the report is empty. |
| `json.v1` | Renders the full `Data` struct as pretty-printed JSON using the `ToJSON` template function. |

---

### `notifier.go`

Provides the public-facing constructor and helper functions consumed by `cmd/root.go` and `cmd/notify-upgrade.go`.

**Package-level Constants:**

| Constant | Value | Description |
|---|---|---|
| `ColorHex` | `"#406170"` | The default notification accent colour as a CSS hex string. Used by MSTeams and Slack notifiers. |
| `ColorInt` | `0x406170` | The same colour as an integer. Used by the Discord variant of the Slack notifier. |

---

**Public Functions:**

---

#### `NewNotifier(c *cobra.Command) ty.Notifier`

The primary constructor for the notification system. Reads all notification-related flags from `c`, calls
`GetTemplateData` to build the static data, calls `AppendLegacyUrls` to convert any legacy notifier flags into Shoutrrr
URLs, and delegates to `createNotifier` to build and return the configured `shoutrrrTypeNotifier`. Calls `log.Fatalf` if
the log level string is invalid.

---

#### `AppendLegacyUrls(urls []string, cmd *cobra.Command) ([]string, time.Duration)`

Reads the `--notifications` flag and, for each legacy type (`email`, `slack`, `msteams`, `gotify`), constructs the
corresponding legacy notifier and calls its `GetURL` method to produce a Shoutrrr URL. Appends the resulting URLs to
`urls`. Also reads any per-notifier delay (currently only `email` implements `ty.DelayNotifier`) and passes it to
`GetDelay`. Returns the extended URL slice and the resolved notification delay. Calls `log.Fatal` if an unknown
notification type is specified.

---

#### `GetDelay(c *cobra.Command, legacyDelay time.Duration) time.Duration`

Resolves the notification delay. If a legacy per-notifier delay is non-zero, it takes precedence. Otherwise reads
`--notifications-delay` from the flag set. Returns zero if neither is set.

---

#### `GetTitle(hostname string, tag string) string`

Constructs the notification title string. If `tag` is non-empty, prepends `[tag]`. If `hostname` is non-empty, appends
`" on <hostname>"`. The base string is always `"Watchtower updates"`. Returns `"Watchtower updates"` if both are empty.

---

#### `GetTemplateData(c *cobra.Command) StaticData`

Builds the `StaticData` instance from flags. Reads `--notifications-hostname` (falling back to `os.Hostname()`).
Assembles the title using `GetTitle` with the resolved hostname and tag, unless `--notification-skip-title` is set, in
which case `Title` is left empty. Also checks the legacy `--notification-email-subjecttag` flag as a fallback tag
source.

---

### `shoutrrr.go`

The core notification engine. Implements `types.Notifier` and `logrus.Hook` via `shoutrrrTypeNotifier`, manages the send
goroutine, and handles template rendering.

**Package-level Variables:**

| Variable | Description |
|---|---|
| `LocalLog` | A Logrus logger with the field `notify: "no"` set. Used for internal log messages that should not trigger further notifications, avoiding recursive loops. |

---

**Types:**

#### `shoutrrrTypeNotifier` _(unexported)_

The production implementation of `ty.Notifier` and `logrus.Hook`. Manages a buffered `messages` channel and a dedicated
send goroutine.

| Field | Type | Description |
|---|---|---|
| `Urls` | `[]string` | The Shoutrrr service URLs to send to. |
| `Router` | `router` | The Shoutrrr sender. Abstracted behind the `router` interface for testability. |
| `entries` | `[]*log.Entry` | Log entries accumulated during the current notification batch. `nil` outside of a batch (entries are sent immediately when `nil`). |
| `logLevel` | `log.Level` | The maximum log level that triggers a notification. |
| `template` | `*template.Template` | The parsed notification template. |
| `messages` | `chan string` | Buffered channel carrying rendered messages to the send goroutine. |
| `done` | `chan bool` | Signals that the send goroutine has finished, used by `Close`. |
| `legacyTemplate` | `bool` | If true, passes `data.Entries` as the template root rather than the full `Data` struct. |
| `params` | `*types.Params` | Shoutrrr send parameters, including the notification title. |
| `data` | `StaticData` | The static title and host fields for this notifier instance. |
| `receiving` | `bool` | Guards against `AddLogHook` being called more than once. |
| `delay` | `time.Duration` | How long to sleep before dispatching each message. |

---

**Public Methods on `shoutrrrTypeNotifier`:**

---

##### `GetScheme(url string) string`

Extracts and returns the scheme portion of a Shoutrrr URL (the part before the first `:`). Returns `"invalid"` if no
colon is found or it is the first character.

---

##### `GetNames() []string`

Returns a slice of scheme names (one per configured URL), derived by calling `GetScheme` on each entry in `Urls`. Used
by `NewNotifier` and tests to verify which services are registered.

---

##### `GetURLs() []string`

Returns the raw Shoutrrr URL slice. Used by `cmd/notify-upgrade.go` to write the converted URLs to the output file.

---

##### `AddLogHook()`

Registers the notifier as a Logrus hook and starts the send goroutine. Guarded by `receiving` so it is safe to call
multiple times; subsequent calls are no-ops. After registration, every log entry at or below `logLevel` will be passed
to `Fire`.

---

##### `StartNotification()`

Begins accumulating log entries into the `entries` buffer. Called at the start of each update session. Entries received
via `Fire` are buffered rather than sent immediately until `SendNotification` is called.

---

##### `SendNotification(report t.Report)`

Flushes the buffered entries and the session report through `sendEntries`, then clears the buffer. This renders the
template against the accumulated data and queues the result for sending.

---

##### `Close()`

Closes the `messages` channel (preventing further sends), then blocks on `done` until the send goroutine has finished
dispatching all queued messages.

---

##### `Levels() []log.Level`

Returns the log levels that trigger this hook, from `PanicLevel` up to and including `logLevel`. Satisfies the
`logrus.Hook` interface.

---

##### `Fire(entry *log.Entry) error`

Called by Logrus for each log entry at an eligible level. Entries tagged with `notify: "no"` (i.e. from `LocalLog`) are
silently ignored to prevent recursive loops. If currently inside a batch (`entries != nil`), the entry is appended to
the buffer. Otherwise, it is sent immediately via `sendEntries`. Always returns `nil`.

---

**Internal Helpers in `shoutrrr.go`:**

| Function | Description |
|---|---|
| `createNotifier(urls, level, tplString, legacy, data, stdout, delay)` | Parses the template, initialises the Shoutrrr router (directing output to stdout or the trace log level), sets the title param, and returns a fully configured `shoutrrrTypeNotifier`. Logs an error and falls back to the default template if `tplString` is invalid. |
| `sendNotifications(n)` | The send goroutine. Reads from `n.messages`, sleeps for `n.delay`, sends via `n.Router`, and logs per-service errors using `LocalLog`. Signals `n.done` when the channel is closed. |
| `buildMessage(data)` | Executes the parsed template against `data` (or `data.Entries` for legacy mode) and returns the rendered string. |
| `sendEntries(entries, report)` | Builds the message from entries and report. If the rendered result is empty, skips sending (and logs a debug message). Otherwise queues the message on `n.messages`. |
| `getShoutrrrTemplate(tplString, legacy)` | Looks up `tplString` in `commonTemplates` (treating it as a named template if found), then parses it. Falls back to `default` or `default-legacy` if the string is empty or parsing fails. |

---

### `email.go`

Legacy email notifier adapter. Converts `--notification-email-*` flags into a Shoutrrr SMTP URL.

**Type:** `emailTypeNotifier` — stores all SMTP connection parameters and implements both `ty.ConvertibleNotifier` and
`ty.DelayNotifier`.

**Internal Functions:**

| Function | Description |
|---|---|
| `newEmailNotifier(c)` | Reads all `--notification-email-*` flags and returns a configured `emailTypeNotifier`. |
| `(e) GetURL(c)` | Builds a `shoutrrr/smtp.Config` from the stored parameters, enabling `STARTTLS` unless `tlsSkipVerify` is set, and returns the Shoutrrr URL string. |
| `(e) GetDelay()` | Returns `e.delay`, satisfying `ty.DelayNotifier`. The delay is read from `--notification-email-delay`. |

---

### `slack.go`

Legacy Slack (and Discord) notifier adapter. Converts `--notification-slack-*` flags into a Shoutrrr Slack or Discord
URL.

**Type:** `slackTypeNotifier` — stores the webhook URL, username, channel, and icon options.

**Internal Functions:**

| Function | Description |
|---|---|
| `newSlackNotifier(c)` | Reads all `--notification-slack-*` flags and returns a configured `slackTypeNotifier`. |
| `(s) GetURL(c)` | Inspects the hook URL. If it points to `discord.com` or `discordapp.com`, produces a Shoutrrr Discord URL. Otherwise strips the Slack webhook prefix, builds a `shoutrrr/slack.Config` with the username, colour, and icon, and returns the Shoutrrr URL string. |

---

### `msteams.go`

Legacy Microsoft Teams notifier adapter. Converts `--notification-msteams-hook` into a Shoutrrr Teams URL.

**Type:** `msTeamsTypeNotifier` — stores the webhook URL and the `data` flag.

**Internal Functions:**

| Function | Description |
|---|---|
| `newMsTeamsNotifier(cmd)` | Reads `--notification-msteams-hook` and `--notification-msteams-data` flags. Calls `log.Fatal` if the hook URL is empty. |
| `(n) GetURL(c)` | Parses the raw webhook URL, calls `shoutrrrTeams.ConfigFromWebhookURL` to extract the token components, sets `ColorHex`, and returns the Shoutrrr Teams URL string. |

---

### `gotify.go`

Legacy Gotify notifier adapter. Converts `--notification-gotify-*` flags into a Shoutrrr Gotify URL.

**Type:** `gotifyTypeNotifier` — stores the Gotify API URL, application token, and TLS skip-verify setting.

**Internal Functions:**

| Function | Description |
|---|---|
| `newGotifyNotifier(c)` | Reads all `--notification-gotify-*` flags via `getGotifyURL` and `getGotifyToken`. |
| `getGotifyURL(flags)` | Validates that the URL is non-empty and has an `http://` or `https://` scheme. Warns if using plain HTTP. |
| `getGotifyToken(flags)` | Validates that the token is non-empty. |
| `(n) GetURL(c)` | Builds a `shoutrrr/gotify.Config` from the stored parameters, setting `DisableTLS` if the scheme is `http`, and returns the Shoutrrr Gotify URL string. |

---

## Test Coverage

| File | Description |
|---|---|
| `notifications_suite_test.go` | Bootstraps the Ginkgo test suite for the `notifications_test` package. Sets `CharactersAroundMismatchToInclude` to 20 for more context in diff output. |
| `shoutrrr_test.go` | Internal package tests (`package notifications`) covering `getShoutrrrTemplate` (named template lookup, invalid template fallback), `AddLogHook` (idempotency), legacy template rendering (`default-legacy`, custom templates, `ToUpper`/`ToLower`/`Title` functions, invalid template fallback), report template rendering (`default`, all container states, empty report, `porcelain.v1.summary-no-log`), `Title` and `Host` template fields, notification batching (empty batch suppression, non-empty batch delivery), title param omission when `Title` is empty, and blocking router behaviour (slow send not lost, send completes after unblock). |
| `notifier_test.go` | External package tests (`package notifications_test`) covering `NewNotifier` (empty shoutrrr type produces no names), `GetTemplateData` (custom hostname, no resolvable hostname, title tag, legacy email subject tag, skip-title flag), legacy URL conversion for each service type (email full config, email default fields, Slack with URL/username/channel/icon emoji, Gotify, MSTeams), and notification delay resolution (no delay, legacy delay, flag delay). |
