# `pkg/notifications/templates` Package

This package exposes the shared Go template function map used across Watchtower's notification template system. It is
the single source of truth for the custom functions available to users writing `--notification-template` values. It is
consumed by `pkg/notifications` (when parsing user-supplied and built-in templates) and by `pkg/notifications/preview`
(when rendering template previews).

---

## Files

### `funcs.go`

Declares the exported `Funcs` map and its one private helper.

---

**Package-level Variables:**

#### `Funcs template.FuncMap`

The shared `text/template` function map registered on every template parsed by Watchtower. Exposes the following
functions to template authors:

| Function | Signature | Description |
| -------- | --------- | ----------- |
| `ToUpper` | `func(string) string` | Converts a string to upper case. Delegates to `strings.ToUpper`. |
| `ToLower` | `func(string) string` | Converts a string to lower case. Delegates to `strings.ToLower`. |
| `Title` | `func(string) string` | Converts a string to title case using American English rules. Delegates to `golang.org/x/text/cases`. |
| `ToJSON` | `func(any) string` | Marshals any value to a pretty-printed JSON string (2-space indent). Returns a descriptive error string rather than panicking if marshalling fails. |

---

**Internal Helpers:**

| Function | Description |
| -------- | ----------- |
| `toJSON(v interface{}) string` | The implementation behind the `ToJSON` template function. Calls `json.MarshalIndent` with a 2-space indent. On error, returns a human-readable error string of the form `"failed to marshal JSON in notification template: <err>"` so that template rendering continues rather than failing silently or panicking. |

---

## Test Coverage

This package has no dedicated test file. The four functions in `Funcs` are exercised by the template tests in
`pkg/notifications/shoutrrr_test.go`, which verify `ToUpper`, `ToLower`, and `Title` against known inputs, and by the
`json.v1` built-in template in `pkg/notifications/common_templates.go` which uses `ToJSON`.
