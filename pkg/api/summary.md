# `pkg/api` Packages — Feature Summary

## `pkg/api` — HTTP API Server

The central HTTP API server that manages handler registration, authentication, and server lifecycle.

- **Bearer token authentication**: All registered endpoints are automatically wrapped with `RequireToken` middleware,
  which validates the `Authorization: Bearer <token>` header on every incoming request and returns HTTP `401
  Unauthorized` if the token is absent or incorrect.
- **Handler registration**: Supports registering both `http.HandlerFunc` and `http.Handler` values at arbitrary paths on
  the default serve mux. Both registration methods apply token authentication automatically.
- **Conditional startup**: The server only starts if at least one handler has been registered. If `Start` is called with
  no handlers, it exits silently. If handlers are registered but no token is set, the process terminates with a fatal
  error.
- **Blocking and non-blocking modes**: `Start` can run the server in the current goroutine (blocking) or in a background
  goroutine, allowing the caller to continue with other work.
- **Fixed listen address**: The server listens on `:8080`. The address is hardcoded and not configurable.

---

## `pkg/api/metrics` — Prometheus Metrics Endpoint

A thin adapter that wires the Prometheus metrics registry to the HTTP API.

- **Metrics exposition**: Registers the standard Prometheus HTTP handler at `/v1/metrics`, serving all registered
  Watchtower metrics in the Prometheus text exposition format.
- **Integration with `pkg/metrics`**: Retrieves the singleton `Metrics` instance from `pkg/metrics` via
  `metrics.Default()`, ensuring the handler always reflects the current metric state.
- **Token-protected**: Intended to be registered via `api.RegisterHandler`, which wraps it with bearer token
  authentication before it is reachable by clients.

---

## `pkg/api/update` — On-Demand Update Trigger Endpoint

An HTTP handler that allows external systems to trigger a container update scan on demand.

- **On-demand update trigger**: Exposes a `GET /v1/update` endpoint that, when called, invokes the Watchtower update
  function immediately rather than waiting for the next scheduled poll.
- **Targeted image updates**: Accepts an optional `?image=` query parameter (comma-separated image names). When images
  are specified, the update is restricted to those images and the request always executes, blocking until any
  in-progress update finishes.
- **Concurrent update prevention**: For untargeted (full-scan) requests, uses a shared lock channel to ensure only one
  update cycle runs at a time. If an update is already in progress, the request is dropped silently rather than queued.
- **Shared lock with scheduler**: The lock channel can be shared with the scheduler (supplied by `cmd/root.go`), making
  HTTP-triggered and scheduled updates mutually exclusive so they never overlap.
- **Request body forwarding**: Copies the request body to `os.Stdout` before dispatching the update function.
