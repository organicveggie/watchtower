# `pkg/api` Package

This package provides the HTTP API server for Watchtower. It is responsible for managing handler registration, enforcing
bearer token authentication on all endpoints, and starting the HTTP server. It does not implement any specific endpoint
itself — those live in the subpackages `pkg/api/update` and `pkg/api/metrics`. It is instantiated and started by
`cmd/root.go` when either `--http-api-update` or `--http-api-metrics` is set.

---

## Files

### `api.go`

Defines the `API` type and all of its methods. Uses the standard library `net/http` package with the default serve mux.

**Constants:**

| Constant | Value | Description |
|---|---|---|
| `tokenMissingMsg` | `"api token is empty or has not been set. exiting"` | The fatal log message emitted if `Start` is called with no token set but at least one handler registered. |

---

**Types:**

#### `API`

The central API server instance. Fields:

| Field | Type | Description |
|---|---|---|
| `Token` | `string` | The bearer token that all incoming requests must present in their `Authorization` header. Set at construction time via `New`. |
| `hasHandlers` | `bool` | Internal flag set to `true` when at least one handler or function has been registered. Used by `Start` to decide whether to actually launch the HTTP server. |

---

**Public Functions:**

---

#### `New(token string) *API`

Factory function that creates and returns a new `API` instance with the provided bearer token and `hasHandlers`
initialised to `false`.

---

#### `(api *API) RequireToken(fn http.HandlerFunc) http.HandlerFunc`

Middleware that wraps an `http.HandlerFunc` with bearer token authentication. Returns a new `http.HandlerFunc` that:

1. Reads the `Authorization` header from the incoming request.
2. Compares it against the expected value `"Bearer <token>"`.
3. Returns HTTP `401 Unauthorized` immediately if the header is absent or does not match.
4. Calls through to the wrapped handler `fn` if the token is valid.

This is applied automatically to every handler registered via `RegisterFunc` or `RegisterHandler`.

---

#### `(api *API) RegisterFunc(path string, fn http.HandlerFunc)`

Registers an `http.HandlerFunc` on the default serve mux at the given path, wrapped with `RequireToken`. Also sets
`hasHandlers` to `true` so that `Start` knows to launch the server.

---

#### `(api *API) RegisterHandler(path string, handler http.Handler)`

Registers an `http.Handler` on the default serve mux at the given path, wrapping its `ServeHTTP` method with
`RequireToken`. Also sets `hasHandlers` to `true`. Used by `pkg/api/metrics` to register the Prometheus handler, which
implements `http.Handler` rather than `http.HandlerFunc`.

---

#### `(api *API) Start(block bool) error`

Starts the HTTP server listening on `:8080`. Behaviour:

- If no handlers have been registered (`hasHandlers` is `false`), logs a debug message and returns `nil` without
  starting the server.
- If `Token` is empty, calls `log.Fatal` with `tokenMissingMsg` to prevent the server starting without authentication.
- If `block` is `true`, runs the server in the current goroutine (blocking until the process exits).
- If `block` is `false`, runs the server in a new goroutine, allowing the caller to continue.

Returns `nil` in all non-fatal cases. The underlying `http.ListenAndServe` call is wrapped in `log.Fatal`, so any server
error will terminate the process.

---

## Internal Helpers

| Function | Description |
|---|---|
| `runHTTPServer()` | Calls `http.ListenAndServe(":8080", nil)` and passes any error to `log.Fatal`. The listen address is currently hardcoded and not configurable. |

---

## Test Coverage

`api_test.go` contains a Ginkgo spec focused on the `RequireToken` middleware:

| Test | Description |
|---|---|
| `should return 401 Unauthorized when token is not provided` | Verifies that a request with no `Authorization` header receives a `401` response. |
| `should return 401 Unauthorized when token is invalid` | Verifies that a request with an incorrect bearer token receives a `401` response. |
| `should return 200 OK when token is valid` | Verifies that a request with the correct bearer token is passed through to the wrapped handler and receives a `200` response. |
