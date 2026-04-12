# `pkg/metrics` Package

This package owns all Prometheus metric definitions and their registration for Watchtower. It defines the counters exposed at the `/v1/metrics` endpoint, provides a singleton accessor for the shared metrics instance, and exposes a `RegisterScan` function that records the result of each update session. It is consumed by `pkg/api/metrics` (for HTTP exposition) and by `cmd/root.go` (via `runUpdatesWithNotifications`, which calls `RegisterScan` after every cycle).

---

## Files

### `metrics.go`

Defines the `Metrics` struct, its three Prometheus counters, and all functions for initialising and updating them.

---

**Types:**

#### `Metrics`

Holds the three Prometheus counter vectors that track update session outcomes. All counters use the label `"status"` to distinguish between different result categories.

| Field | Type | Metric name | Labels | Description |
|---|---|---|---|---|
| `scanned` | `*prometheus.CounterVec` | `watchtower_containers_scanned` | — | Total number of containers inspected across all update sessions. |
| `updated` | `*prometheus.CounterVec` | `watchtower_containers_updated` | — | Total number of containers successfully updated across all sessions. |
| `failed` | `*prometheus.CounterVec` | `watchtower_containers_failed` | — | Total number of containers that failed to update across all sessions. |
| `total` | `*prometheus.CounterVec` | `watchtower_scans_total` | — | Total number of update sessions run, including skipped (no-op) ones. |
| `skipped` | `*prometheus.CounterVec` | `watchtower_scans_skipped` | — | Total number of update sessions that were skipped (i.e. `nil` report). |

---

**Package-level Variables:**

| Variable | Type | Description |
|---|---|---|
| `registered` | `*Metrics` | The singleton `Metrics` instance. Initialised once by the first call to `Default()`. |

---

**Public Functions:**

---

#### `Default() *Metrics`

Returns the singleton `Metrics` instance, creating and registering it if it has not yet been initialised. On first call, constructs all five `prometheus.CounterVec` instances and registers them with the default Prometheus registry. Calls `log.Fatal` if any counter cannot be registered. Subsequent calls return the already-initialised instance without re-registering.

---

#### `(m *Metrics) RegisterScan(report *types.Report)`

Records the outcome of a single update session into the Prometheus counters. Behaviour depends on whether `report` is nil:

- **`report` is `nil`** (skipped scan): Increments `watchtower_scans_total` and `watchtower_scans_skipped` by 1. The container-level counters are not touched.
- **`report` is non-nil**: Increments `watchtower_scans_total` by 1, `watchtower_containers_scanned` by `report.Scanned()`, `watchtower_containers_updated` by `report.Updated()`, and `watchtower_containers_failed` by `report.Failed()`.

All counter increments use the `Add` method with a `float64` cast of the integer report values.

---

## Test Coverage

`metrics_test.go` contains a Ginkgo spec suite for the `Metrics` type:

| Test | Description |
|---|---|
| `Registered scan should set scan metrics` | Verifies that after calling `RegisterScan` with a report of `Scanned: 3, Updated: 2, Failed: 1`, the counters `watchtower_containers_scanned`, `watchtower_containers_updated`, and `watchtower_containers_failed` reflect those values when gathered from the Prometheus registry. |
| `Skipped scan should only increment total and skipped` | Verifies that calling `RegisterScan(nil)` increments `watchtower_scans_total` and `watchtower_scans_skipped` by 1 each, while leaving the container-level counters at 0. |
| `Multiple scans should accumulate` | Verifies that successive calls to `RegisterScan` (one real report followed by two `nil` reports) correctly accumulate: `watchtower_scans_total` reaches 3, `watchtower_scans_skipped` reaches 2, and the container counters reflect only the one real scan. |

Each test creates a fresh `Metrics` instance directly (bypassing `Default()`) and uses `prometheus/testutil` to gather and compare counter values.
