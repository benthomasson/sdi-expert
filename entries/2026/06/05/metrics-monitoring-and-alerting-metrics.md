# File: metrics-monitoring-and-alerting/metrics.py

**Date:** 2026-06-05
**Time:** 13:52

# `metrics-monitoring-and-alerting/metrics.py`

## Purpose

This file implements an in-memory time-series metrics monitoring and alerting system — the kind of thing you'd build if asked "Design a metrics monitoring system like Datadog or Prometheus" in a system design interview. It owns the entire pipeline: ingestion of metric data points, time-windowed querying with aggregation, alert rule evaluation with state machines, and data lifecycle management (downsampling + retention).

## Key Components

### Data Classes

- **`DataPoint`** — The ingestion primitive. A metric name, float value, timestamp, and optional tags dict. Tags enable multi-dimensional filtering (e.g., `{"host": "web-01", "region": "us-east"}`).
- **`AlertRule`** — Defines a threshold-based alerting condition. Supports comparisons (`gt`, `lt`, `gte`, `lte`) and `rate_change` (percentage change over a window). `duration_seconds` controls how long a condition must hold before firing.
- **`Alert`** — A snapshot of an alert state transition, emitted when `evaluate_alerts` detects a change.

### `MetricsService`

The core class. Single-node, in-memory — no persistence or distribution, consistent with the SDI interview scope of demonstrating architecture rather than production readiness.

**Storage model:** `_data` is a dict keyed by `(metric_name, frozenset(tags.items()))`. Values are sorted lists of `(timestamp, value)` tuples. The `frozenset` key means each unique tag combination gets its own series — this is the standard time-series cardinality model (identical to how Prometheus stores series by label set).

**Ingestion:**
- `ingest()` uses `bisect.bisect_right` to insert in sorted order — O(log n) search but O(n) insertion due to list shifting. Adequate for interview demonstration; a production system would use an append-only buffer with periodic sorting.
- `ingest_batch()` is a simple loop over `ingest()` — no bulk optimization.

**Querying:**
- `query()` supports time-range slicing, bucketed aggregation (sum, avg, min, max, count, percentiles), and `group_by` for tag-based grouping.
- `_slice()` uses bisect for O(log n) range lookups on sorted series.
- `_aggregate()` handles percentile computation via linear interpolation between floor/ceil indices — the standard interpolation method.
- `_bucketize()` partitions points into fixed-width time buckets.

**Alerting:**
- `add_alert_rule()` / `remove_alert_rule()` manage the rule registry.
- `evaluate_alerts()` is the poll-based evaluation loop. It checks every rule, drives the state machine, and returns a list of state transitions.
- `on_alert()` registers callbacks invoked on state changes.

**Data Lifecycle:**
- `downsample()` compacts old data: 1–7 days old → 5-minute buckets, >7 days old → 1-hour buckets.
- `apply_retention()` drops data older than `retention_seconds` (default 30 days). Cleans up empty series from `_data`.

## Patterns

**Sorted-list time series with bisect.** Every series is kept sorted by timestamp, enabling O(log n) range queries via `bisect_left`/`bisect_right`. This is the in-memory analog of a time-series database's sorted index. The tuple comparison `(timestamp,)` works because Python compares tuples element-by-element.

**Tag-based series multiplexing.** Using `frozenset(tags.items())` as part of the key gives each unique label set its own series, then `_matching_keys` does a linear scan with subset matching for queries. This mirrors how Prometheus/Datadog handle label cardinality — each unique label combination is a distinct time series.

**Alert state machine.** Four states: `OK → PENDING → ALERTING → RESOLVED → (back to OK or PENDING)`. The `PENDING` state implements the `duration_seconds` "for" clause — the condition must persist for the specified duration before firing. This prevents noisy alerts from transient spikes.

**Observer pattern for alert callbacks.** `_callbacks` list with `on_alert()` registration. Simple pub-sub for notification delivery.

## Dependencies

**Imports:** Only stdlib — `bisect` for sorted-list operations, `math` for percentile floor/ceil, `dataclasses` for data modeling. No external dependencies.

**Imported by:** `test_metrics.py` and `test_smoke.py` — the test suites.

## Flow

### Ingestion Path
`DataPoint` → `ingest()` → compute `(metric, frozenset(tags))` key → `bisect.bisect_right` to find insert position → `list.insert` in sorted order.

### Query Path
`query()` → `_matching_keys()` scans `_data` for matching metric + tag subset → `_slice()` each matching series → merge + sort across series → `_bucketize()` into time windows → `_aggregate()` each bucket → return list of `{timestamp, value}` dicts.

### Alert Evaluation Path
`evaluate_alerts(current_time)` → for each rule: `_check_condition()` gathers data in the lookback window → computes avg (or rate for `rate_change`) → compare to threshold → drive state machine → emit `Alert` objects for transitions → invoke callbacks.

### Data Lifecycle Path
`downsample(current_time)` → for each series, apply age-based bucketing rules → replace raw points with averaged buckets.
`apply_retention(current_time)` → bisect to find cutoff index → truncate series → delete empty keys.

## Invariants

1. **Series are always sorted by timestamp.** `ingest()` maintains this via `bisect_right` insertion. `downsample()` and `apply_retention()` preserve it (re-sort after downsample, bisect-based truncation for retention).

2. **Alert state transitions follow the state machine.** `OK → PENDING → ALERTING` is the only firing path. `PENDING` resets to `OK` (not `RESOLVED`) if the condition clears before `duration_seconds` elapses. Only `ALERTING → RESOLVED` produces a `RESOLVED` alert.

3. **Tag matching is a subset check.** `_matching_keys` requires all `tags_filter` key-value pairs to be present in the series tags, but extra tags in the series are fine. This means `tags_filter={"host": "web-01"}` matches series with tags `{"host": "web-01", "region": "us-east"}`.

4. **`_check_condition` uses the average of all points in the lookback window** for threshold comparisons (`gt`/`lt`/etc.), not the latest point. The window is `max(duration_seconds, 60)`.

5. **Downsampling is lossy.** It replaces raw points with bucket averages — min/max/percentile accuracy is lost for downsampled data.

## Error Handling

Essentially none — this is interview-demonstration code. No validation on ingestion (negative timestamps, NaN values), no bounds checking on aggregation types, no protection against tag cardinality explosion. `_aggregate` silently falls back to `avg` for unrecognized aggregation names. `_check_condition` returns `(False, 0)` for empty data or unrecognized conditions, silently suppressing evaluation.

## Topics to Explore

- [file] `metrics-monitoring-and-alerting/test_metrics.py` — How the alert state machine transitions are exercised, and edge cases around downsampling and retention
- [function] `metrics-monitoring-and-alerting/metrics.py:evaluate_alerts` — The alert state machine is the most complex logic; trace the PENDING→ALERTING transition timing and the RESOLVED→re-fire path
- [general] `pull-vs-push-alerting` — This system uses poll-based evaluation (caller invokes `evaluate_alerts`); compare with push-based evaluation triggered on ingest, and the tradeoffs for latency vs. computational cost
- [general] `time-series-storage-tradeoffs` — The sorted-list model has O(n) insertion; explore LSM-tree or columnar approaches (like Gorilla compression) used by real TSDB systems
- [file] `metrics-monitoring-and-alerting/plan.md` — The design rationale and requirements that shaped this implementation

## Beliefs

- `metrics-series-sorted-invariant` — Every series in `_data` is maintained in sorted timestamp order; `ingest`, `downsample`, and `apply_retention` all preserve this invariant
- `metrics-alert-state-machine-four-states` — Alert evaluation follows a four-state machine (OK → PENDING → ALERTING → RESOLVED) where PENDING requires `duration_seconds` to elapse before transitioning to ALERTING
- `metrics-tag-matching-is-subset` — `_matching_keys` performs subset matching on tags: a query with `tags_filter={"a": 1}` matches any series whose tags include `a=1`, regardless of other tags present
- `metrics-condition-uses-window-average` — Threshold alert conditions (`gt`, `lt`, `gte`, `lte`) compare against the average of all points in the lookback window, not the most recent value
- `metrics-downsample-two-tier` — Downsampling uses two age-based tiers: data 1–7 days old is compacted to 5-minute buckets, data older than 7 days is compacted to 1-hour buckets, both using averaging which loses min/max/percentile fidelity

