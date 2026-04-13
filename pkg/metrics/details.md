# `pkg/metrics` Package

This package owns all Prometheus metric definitions and their registration for Watchtower. It defines the gauges and
counters exposed at the `/v1/metrics` endpoint, provides a singleton accessor for the shared metrics instance, and
exposes a `RegisterScan` function that records the result of each update session. It is consumed by `pkg/api/metrics`
(for HTTP exposition) and by `cmd/root.go` (via `runUpdatesWithNotifications`, which calls `RegisterScan` after every
cycle).

---

## Files

### `metrics.go`

Defines the `Metric` and `Metrics` types and all functions for initialising and updating them.

---

**Types:**

#### `Metric`

A plain data struct holding the counts from a single scan.

| Field | Type | Description |
| ----- | ---- | ----------- |
| `Scanned` | `int` | Number of containers inspected during the scan. |
| `Updated` | `int` | Number of containers updated (includes stale containers for backwards compatibility). |
| `Failed` | `int` | Number of containers that failed to update. |

---

#### `Metrics`

Holds the Prometheus metrics and the channel used to process scan results asynchronously.

| Field | Type | Metric name | Kind | Description |
| ----- | ---- | ----------- | ---- | ----------- |
| `channel` | `chan *Metric` | — | — | Buffered channel (capacity 10) used to deliver `Metric` values to the background `HandleUpdate` goroutine. |
| `scanned` | `prometheus.Gauge` | `watchtower_containers_scanned` | Gauge | Number of containers scanned during the **last** scan. Set (not incremented) on each scan. |
| `updated` | `prometheus.Gauge` | `watchtower_containers_updated` | Gauge | Number of containers updated during the **last** scan. Set on each scan. |
| `failed` | `prometheus.Gauge` | `watchtower_containers_failed` | Gauge | Number of containers that failed to update during the **last** scan. Set on each scan. |
| `total` | `prometheus.Counter` | `watchtower_scans_total` | Counter | Total number of scans since Watchtower started, including skipped ones. |
| `skipped` | `prometheus.Counter` | `watchtower_scans_skipped` | Counter | Total number of skipped scans since Watchtower started. |

---

**Package-level Variables:**

| Variable | Type | Description |
| -------- | ---- | ----------- |
| `metrics` | `*Metrics` | The singleton `Metrics` instance. Initialised once by the first call to `Default()`. |

---

**Public Functions:**

---

#### `NewMetric(report types.Report) *Metric`

Constructs a `Metric` from a `types.Report`. `Updated` is set to `len(report.Updated()) + len(report.Stale())` — stale
containers are folded in for backwards compatibility.

---

#### `Default() *Metrics`

Returns the singleton `Metrics` instance, creating it if it has not yet been initialised. On first call, constructs all
five Prometheus metrics using `promauto` (which registers them automatically with the default registry) and starts a
background goroutine running `HandleUpdate`. Subsequent calls return the already-initialised instance.

---

#### `RegisterScan(metric *Metric)`

Package-level convenience function. Calls `Default()` to obtain the singleton and then calls `Register` to enqueue the
metric.

---

**Methods:**

---

#### `(metrics *Metrics) Register(metric *Metric)`

Sends `metric` to the internal channel. Blocks if the channel buffer (capacity 10) is full. The actual Prometheus
updates happen asynchronously in the `HandleUpdate` goroutine.

---

#### `(metrics *Metrics) QueueIsEmpty() bool`

Returns `true` when no metrics are waiting in the channel. Used in tests to wait for the background goroutine to finish
processing.

---

#### `(metrics *Metrics) HandleUpdate(channel <-chan *Metric)`

Runs as a goroutine started by `Default()`. Processes each `*Metric` received from the channel:

- **`nil` metric** (skipped scan): Increments `watchtower_scans_total` and `watchtower_scans_skipped` by 1. Resets the
  three gauges (`scanned`, `updated`, `failed`) to 0.
- **Non-nil metric**: Increments `watchtower_scans_total` by 1. Sets `watchtower_containers_scanned`,
  `watchtower_containers_updated`, and `watchtower_containers_failed` to the values from the metric.

Because the per-scan fields are Gauges rather than Counters, they always reflect the **most recent** scan, not a running
total.

---

## Test Coverage

There is no dedicated `metrics_test.go` file in this package.
