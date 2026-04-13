# `pkg/notifications` Packages — Feature Summary

## `pkg/notifications` — Notification System

The complete notification system for Watchtower, bridging the update cycle with external notification services via
Shoutrrr.

### Template Data Model (`model.go`)

- **`StaticData`**: Fixed per-notifier-instance fields — a composed notification title (hostname + optional tag prefix)
  and the machine hostname. The title is omitted entirely when `--notification-skip-title` is set.
- **`Data`**: The full template data model passed to every template at render time. Embeds `StaticData` and adds
  per-session fields: a slice of Logrus log entries (for legacy templates) and a structured session `Report` (for
  report-mode templates).

### Built-in Templates (`common_templates.go`)

Four named templates selectable via `--notification-template`:

| Name | Description |
|---|---|
| `default-legacy` | Renders each log entry's message on its own line. Used in legacy (log-entry) mode. |
| `default` | Renders a summary line (`N Scanned, N Updated, N Failed`) and per-container lines for updated, fresh, skipped, and failed containers. Suppresses the notification entirely if nothing was updated or failed. Falls back to log-entry rendering if no report is available. |
| `porcelain.v1.summary-no-log` | Machine-readable output used by `--porcelain v1`. One line per container in the format `<name> (<image>): <state>`, with an error suffix for failed/skipped containers. Outputs `"no containers matched filter"` for an empty report. |
| `json.v1` | Pretty-printed JSON of the full `Data` struct via the `ToJSON` template function. |

### Notifier Construction (`notifier.go`)

- **`NewNotifier`**: Primary constructor. Reads all notification flags, builds static template data, converts legacy
  service flags to Shoutrrr URLs, and returns a fully configured notifier.
- **`AppendLegacyUrls`**: Translates `--notifications` entries for each legacy service type (`email`, `slack`,
  `msteams`, `gotify`) into Shoutrrr URLs by constructing the appropriate adapter and calling its `GetURL` method.
- **`GetTemplateData`**: Builds the `StaticData` instance from flags. Resolves the hostname (with
  `--notifications-hostname` override), composes the title via `GetTitle`, and checks for legacy email subject tag as a
  fallback tag source.
- **`GetTitle`**: Composes a notification title from a base string (`"Watchtower updates"`), an optional tag prefix, and
  an optional hostname suffix.
- **`GetDelay`**: Resolves the notification send delay, giving precedence to the legacy per-notifier delay (currently
  only email) over the `--notifications-delay` flag.
- **Brand colours**: Exports `ColorHex` (`"#406170"`) and `ColorInt` (`0x406170`) for use by Slack and MSTeams notifier
  adapters.

### Core Notifier Engine (`shoutrrr.go`)

- **Shoutrrr integration**: Sends rendered notifications to one or more services via Shoutrrr service URLs. Supports any
  service Shoutrrr supports.
- **Logrus hook**: Implements `logrus.Hook`, automatically receiving log entries at or below the configured level.
  Entries tagged with `notify: "no"` are silently ignored to prevent recursive notification loops.
- **Batched sending**: `StartNotification` begins accumulating log entries; `SendNotification` flushes the batch
  together with the session report through the template and queues the rendered message for dispatch. Outside a batch,
  entries are sent immediately.
- **Async dispatch**: A dedicated send goroutine reads from a buffered `messages` channel, sleeps for the configured
  delay, and dispatches via the Shoutrrr router. `Close` drains the channel and waits for the goroutine to finish.
- **Empty message suppression**: If template rendering produces an empty string, the notification is silently dropped.
- **Template selection**: Looks up the template string in the named template registry first; falls back to the
  appropriate default template (`default` or `default-legacy`) if the string is empty or invalid.
- **Legacy template mode**: When `--notification-report` is not set, executes the template against the raw log entry
  slice rather than the full `Data` struct.
- **Stdout routing**: When `--notification-log-stdout` is set (used by `--porcelain v1`), directs Shoutrrr output to
  stdout instead of the trace log level.

### Legacy Service Adapters

Each adapter reads its CLI flags and converts them to a Shoutrrr URL:

- **Email (`email.go`)**: Converts `--notification-email-*` flags into a Shoutrrr SMTP URL. Enables `STARTTLS` by
  default unless `--notification-email-server-tls-skip-verify` is set. Implements `ty.DelayNotifier` to supply a
  per-send delay.
- **Slack (`slack.go`)**: Converts `--notification-slack-*` flags into a Shoutrrr Slack URL. Automatically detects
  Discord webhook URLs (via `discord.com`/`discordapp.com` hostname) and produces a Shoutrrr Discord URL instead.
- **Microsoft Teams (`msteams.go`)**: Converts `--notification-msteams-hook` into a Shoutrrr Teams URL using
  `ConfigFromWebhookURL` to extract token components. Applies `ColorHex` as the accent colour.
- **Gotify (`gotify.go`)**: Converts `--notification-gotify-*` flags into a Shoutrrr Gotify URL. Sets `DisableTLS`
  automatically when the URL scheme is `http`. Validates that both the URL and token are non-empty before constructing
  the config.

---

## `pkg/notifications/preview` — Template Preview Renderer

Allows users to validate and iterate on custom `--notification-template` values without running a real update cycle.

- **Template rendering**: `Render` accepts a template string, a list of container states, and a list of log levels. It
  parses the template (with the shared Watchtower function map applied), populates a synthetic data object, executes the
  template, and returns the rendered string.
- **Deterministic output**: The random source is seeded to a fixed value, so identical inputs always produce identical
  output.
- **Template validation**: Returns a descriptive error on parse or execution failure.

---

## `pkg/notifications/preview/data` — Synthetic Preview Data Generator

Generates realistic-looking session state for template preview rendering. Has no runtime role.

- **Synthetic container generation**: Produces container entries with random hex IDs, pooled names and image references,
  and randomly selected error messages for failed/skipped states. Entries are routed into the appropriate report bucket
  (scanned, updated, failed, skipped, stale, fresh).
- **Synthetic log entry generation**: Produces log entries at specified levels with randomly selected messages.
  Error-level messages draw from a separate pool. Timestamps advance monotonically.
- **`LogLevel` type**: String-based enum with seven levels (Trace through Panic). `LevelsFromString` parses compact
  single-character level strings.
- **`State` type**: String-based enum with six container outcome states. `StatesFromString` parses compact
  single-character state strings (`c`, `u`, `e`, `k`, `t`, `f`).
- **Static template data**: Each data instance exposes fixed `Title` and `Host` fields available to templates.
- **String pools**: Pre-populated arrays of 40 container names, 39 organisation names, 42 error messages, 20 skip-reason
  messages, 13 informational log messages, and 5 error log messages.
