# `pkg/api/metrics` Package

This package provides the HTTP handler that exposes Watchtower's Prometheus metrics over the API. It acts as a thin
adapter between the `pkg/metrics` package (which owns the metric state and registration) and the HTTP layer provided by
`pkg/api`, wiring the Prometheus default registry's HTTP handler to the `/v1/metrics` endpoint.

---

## Files

### `metrics.go`

Defines the `Handler` type and its factory function. The actual metric collection and processing logic lives in
`pkg/metrics`; this package is solely responsible for serving those metrics over HTTP.

**Types:**

#### `Handler`

Represents the metrics HTTP endpoint. Fields:

| Field | Type | Description |
| ----- | ---- | ----------- |
| `Path` | `string` | The URL path at which the handler is registered. Always `"/v1/metrics"`. |
| `Handle` | `http.HandlerFunc` | The Prometheus HTTP handler, sourced from `promhttp.Handler()`. Serves the current state of all registered Prometheus metrics in the standard text exposition format. |
| `Metrics` | `*metrics.Metrics` | A reference to the shared metrics instance from `pkg/metrics`, obtained via `metrics.Default()`. |

**Public Functions:**

---

#### `New() *Handler`

Factory function that creates and returns a new `Handler` instance. On each call it:

1. Retrieves (or initialises) the singleton `Metrics` instance from `pkg/metrics` via `metrics.Default()`.
2. Obtains the standard Prometheus HTTP handler via `promhttp.Handler()`.
3. Returns a `Handler` with `Path` set to `"/v1/metrics"` and `Handle` set to the Prometheus handler's `ServeHTTP`
   method.

The returned handler is intended to be registered with the API server via `api.RegisterHandler`, which wraps it with
bearer token authentication before it is reachable by clients.

---

## Test Coverage

`metrics_test.go` contains a single Ginkgo spec that exercises the full metrics request/response cycle via `pkg/api`'s
`RequireToken` middleware:

| Test | Description |
| ---- | ----------- |
| `should serve metrics` | Verifies the end-to-end behaviour of the metrics endpoint. Confirms the initial state has `watchtower_containers_updated` at `0`, then registers a scan with known values (`Scanned: 4`, `Updated: 3`, `Failed: 1`) and asserts the response body reflects those values. Also registers three skipped scans (`nil` metrics) and verifies that `watchtower_scans_total` increments to `4` and `watchtower_scans_skipped` increments to `3`. |

The test uses `httptest` to issue requests directly against the handler without starting a real HTTP server, and parses
the Prometheus text format response into a key-value map for assertion.
