# `pkg/notifications/preview` Package

This package provides a single function for rendering notification template previews. Given a template string and a description of the desired synthetic session data, it renders the template and returns the result as a string. It is consumed by the `tplprev` command-line tool (both its native and WebAssembly variants) to allow users to validate and iterate on custom `--notification-template` values without running a real Watchtower update cycle.

---

## Files

### `tplprev.go`

Contains a single public function that drives the full preview render cycle.

**Public Functions:**

---

#### `Render(input string, states []data.State, loglevels []data.LogLevel) (string, error)`

Parses and executes a Go `text/template` string against a synthetic session data object. The function:

1. Calls `data.New()` to create a fresh `previewData` instance with a fixed random seed, ensuring deterministic output across repeated calls with the same inputs.
2. Parses `input` as a Go template, applying the shared template function map from `pkg/notifications/templates`. Returns an error wrapping the parse failure if the template is syntactically invalid.
3. Calls `data.AddFromState(state)` for each entry in `states`, populating the synthetic report with one container per state value (e.g. `UpdatedState`, `FailedState`).
4. Calls `data.AddLogEntry(level)` for each entry in `loglevels`, populating the synthetic log entry list with entries at the specified log levels.
5. Executes the parsed template against the populated `previewData` into a `strings.Builder`. Returns an error wrapping the execution failure if the template cannot be rendered.
6. Returns the rendered string.

The function has no side effects and does not interact with the Docker daemon or any external service. All randomness is seeded deterministically so that the same `states` and `loglevels` inputs always produce the same output.

---

## Test Coverage

This package has no dedicated test file. Its behaviour is exercised indirectly by the `tplprev` tool's native and WebAssembly entry points, and by the expected output strings declared in `pkg/notifications/preview/data/preview_strings.go`.
