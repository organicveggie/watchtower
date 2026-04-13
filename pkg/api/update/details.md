# `pkg/api/update` Package

This package provides the HTTP handler that exposes Watchtower's on-demand update trigger endpoint. It allows external
systems to initiate a container update scan via an HTTP request rather than waiting for the next scheduled poll. It is
registered by `cmd/root.go` when the `--http-api-update` flag is set, and is protected by bearer token authentication
via `pkg/api`.

---

## Files

### `update.go`

Defines the `Handler` type and its factory function. The update logic itself lives in `internal/actions`; this package
is solely responsible for receiving HTTP requests and dispatching them to the provided update function.

**Package-level Variables:**

| Variable | Type | Description |
|---|---|---|
| `lock` | `chan bool` | A shared channel used to ensure only one update cycle runs at a time. Accepts an externally provided lock from `cmd/root.go` (shared with the scheduler), or creates its own if none is provided. |

---

**Types:**

#### `Handler`

Represents the update trigger HTTP endpoint. Fields:

| Field | Type | Description |
|---|---|---|
| `fn` | `func(images []string)` | The update function to invoke when a request is received. Provided by the caller at construction time; typically `runUpdatesWithNotifications` from `cmd/root.go`. |
| `Path` | `string` | The URL path at which the handler is registered. Always `"/v1/update"`. |

---

**Public Functions:**

---

#### `New(updateFn func(images []string), updateLock chan bool) *Handler`

Factory function that creates and returns a new `Handler` instance. Accepts two arguments:

- `updateFn` — the function to call when an update is triggered. It receives a slice of image names to restrict the
  update to, or `nil` to update all matched containers.
- `updateLock` — a buffered boolean channel used as a mutex to prevent concurrent update cycles. If a non-nil lock is
  provided it is used directly (sharing it with the scheduler so that HTTP-triggered and scheduled updates are mutually
  exclusive). If `nil` is passed, a new single-slot buffered channel is created and pre-filled.

Returns a `Handler` with `Path` set to `"/v1/update"`.

---

#### `(handle *Handler) Handle(w http.ResponseWriter, r *http.Request)`

The `http.HandlerFunc` that processes incoming update requests. Its behaviour depends on whether specific images are
requested:

- **With images** (`?image=foo/bar,foo/baz`): Parses the `image` query parameter, splitting on commas to build a list of
  image names. Acquires the lock unconditionally (blocking until any in-progress update finishes) and then calls `fn`
  with the image list. This ensures that targeted updates are always executed, never silently dropped.

- **Without images**: Attempts to acquire the lock without blocking (using a `select`/`default`). If the lock is
  available, calls `fn` with a `nil` image list to update all matched containers. If another update is already running,
  logs a debug message and returns immediately without triggering a new cycle. This prevents unbounded queuing of
  full-scan updates.

In both cases the lock is released via a deferred send once `fn` returns. The request body is copied to `os.Stdout`
before dispatching.

---

## Test Coverage

This package has no dedicated test file. Its behaviour is exercised indirectly through the integration-level tests in
`pkg/api/metrics_test.go` (which verifies the API token middleware used to protect this handler) and through the
manual/end-to-end usage described in `docs/http-api-mode.md`.
