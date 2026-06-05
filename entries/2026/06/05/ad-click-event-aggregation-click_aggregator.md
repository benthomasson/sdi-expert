# File: ad-click-event-aggregation/click_aggregator.py

**Date:** 2026-06-05
**Time:** 13:22

# `ad-click-event-aggregation/click_aggregator.py`

## Purpose

This file implements an **ad click event aggregation system** — a classic system design interview problem. It owns the responsibility of ingesting raw click events, deduplicating them, bucketing them into fixed-size time windows (tumbling windows), and producing aggregated counts per ad. This is the kind of component that sits between a raw event stream (Kafka, Kinesis) and a queryable analytics store in a production ad-tech pipeline.

## Key Components

### Data Classes

- **`ClickEvent`** — The input unit. Carries `event_id` (dedup key), `ad_id` (partition key), `user_id` (for unique-user counting), `timestamp` (event time, not processing time), and optional `metadata`.
- **`AggregationResult`** — The output unit. A materialized view of one window: ad, time range, total clicks, and unique user count. This is what downstream consumers (dashboards, billing) receive.
- **`_Window`** — Internal mutable accumulator. The underscore prefix signals it's not part of the public API. Tracks raw count, a set of user IDs, and a `WindowState` lifecycle.

### `WindowState` Enum

Three-state lifecycle: `OPEN → CLOSED → FINALIZED`. This is a one-way ratchet — windows never reopen. `OPEN` accepts all events, `CLOSED` accepts late events within the lateness allowance, `FINALIZED` rejects everything.

### `ClickAggregator`

The core class. Its constructor takes two parameters that control the fundamental tradeoff between latency and completeness:

- `window_size_seconds` (default 60) — the tumbling window width.
- `allowed_lateness_seconds` (default 300) — how long after a window closes it will still accept late arrivals.

Key methods:

| Method | Contract |
|--------|----------|
| `process_event(event)` | Idempotent per `event_id`. Returns `True` if accepted, `False` if deduped or rejected. Mutates window state. |
| `process_batch(events)` | Convenience wrapper. Returns a breakdown dict of accepted/deduped/rejected counts. |
| `advance_watermark(ts)` | Monotonic — ignores timestamps ≤ current watermark. Triggers window lifecycle transitions and dedup pruning. Returns newly finalized `AggregationResult`s. |
| `query(ad_id, start, end)` | Read-only range scan over windows for a single ad. |
| `query_top_ads(start, end, k)` | Cross-ad aggregation. Returns top-K ads by total click count. |

## Patterns

**Tumbling windows** — Non-overlapping, fixed-size time buckets. `_window_start` uses `math.floor(ts / size) * size` to snap any timestamp to its window boundary. This is deterministic and partition-safe — two nodes processing the same event will compute the same window.

**Event-time processing with watermarks** — The system separates event time (when the click happened) from processing time (when `process_event` is called). The watermark is an explicit "I believe all events before this time have arrived" signal. This is the same model as Apache Flink/Beam.

**Dedup via seen-event registry** — `seen_events` is a dict keyed by `event_id`. This is an at-least-once-to-exactly-once conversion layer. The registry is pruned on watermark advance to bound memory (events older than `2 * allowed_lateness` are evicted).

**Two-level dict for window lookup** — `windows[ad_id][window_start]` gives O(1) access to any window. The ad_id is the partition key; in a distributed version, each partition would hold its own subset.

## Dependencies

**Imports**: Only stdlib — `math`, `dataclasses`, `enum`. No external dependencies. This is intentional for an interview implementation: the design concepts are the focus, not framework integration.

**Imported by**: `test_click_aggregator.py` — the test suite is the sole consumer.

## Flow

1. **Ingestion**: `process_event` is called with a `ClickEvent`.
2. **Dedup gate**: If `event_id` already in `seen_events`, reject immediately.
3. **Window resolution**: Compute `window_start` from event timestamp, get-or-create the `_Window`.
4. **Lifecycle gate**: If `FINALIZED`, reject. If `CLOSED`, check if `watermark - window.end > allowed_lateness` — if so, finalize and reject; otherwise accept as a late event.
5. **Accumulate**: Add `event_id` to `seen_events`, increment `count`, add `user_id` to the window's user set.
6. **Watermark advance** (separate call): Scan all windows. Those past `watermark - allowed_lateness` get finalized and emitted as `AggregationResult`. Those past `watermark` (but not yet past lateness) get closed. Dedup entries older than `2 * allowed_lateness` are pruned.

The separation of event processing from watermark advance is critical — it means the caller controls when finalization happens, which maps to how stream processors work in practice.

## Invariants

- **Watermark monotonicity**: `advance_watermark` is a no-op if the new timestamp isn't strictly greater than the current watermark.
- **Window lifecycle is one-directional**: OPEN → CLOSED → FINALIZED. There's no path back.
- **Dedup is global across ads**: `seen_events` is keyed by `event_id` alone, not `(ad_id, event_id)`. An event ID that appears for two different ads will still be deduped.
- **Finalization is irreversible**: Once a window is `FINALIZED`, no further events can modify its counts. This is the "exactly once" guarantee for downstream consumers.
- **Late acceptance window**: A late event is accepted only if `watermark - window.end ≤ allowed_lateness`. This means the actual acceptance window extends `allowed_lateness` seconds beyond the window's end time.

## Error Handling

There is essentially none — and that's appropriate for this domain. The system uses **return-value signaling** (`True`/`False` from `process_event`) rather than exceptions. Invalid states (duplicate events, late arrivals) are expected in stream processing, not exceptional. The `stats` dict provides observability into rejection reasons without interrupting the processing pipeline.

One subtlety: `process_batch` infers the rejection reason by diffing stats counters before and after each call. This is fragile if `process_event`'s internal accounting ever changes — the batch method's classification logic is coupled to the ordering of checks in `process_event`.

---

## Topics to Explore

- [file] `ad-click-event-aggregation/test_click_aggregator.py` — See how watermark advance, late events, and dedup are tested; reveals edge cases the implementation handles
- [file] `ad-click-event-aggregation/plan.md` — The design rationale and interview-framing that drove the implementation choices
- [general] `tumbling-vs-sliding-vs-session-windows` — This implementation uses tumbling windows; understand when you'd pick sliding or session windows instead and what changes structurally
- [general] `distributed-aggregation-partitioning` — How this single-node design would partition by `ad_id` across workers, and what the merge/reconciliation layer looks like
- [function] `ad-click-event-aggregation/click_aggregator.py:advance_watermark` — The watermark advance is the most complex method; trace through the finalization and pruning logic with overlapping windows in different states

## Beliefs

- `dedup-is-global-not-per-ad` — The `seen_events` dedup registry keys on `event_id` alone; the same event ID arriving for different `ad_id`s will be deduplicated
- `watermark-drives-finalization` — Windows are never finalized by event processing alone; only `advance_watermark` transitions windows to FINALIZED and emits results
- `dedup-pruning-uses-2x-lateness` — The dedup registry evicts entries older than `2 * allowed_lateness` on watermark advance, meaning dedup coverage extends beyond the late-event acceptance window
- `batch-classification-coupled-to-process-event` — `process_batch` infers per-event rejection reasons by diffing global stats counters, creating an implicit coupling to the check ordering inside `process_event`
- `no-window-merging-or-retraction` — Once an `AggregationResult` is emitted from `advance_watermark`, there is no mechanism to update or retract it if a late event is subsequently accepted into that window (though finalized windows reject events, the closed-to-finalized transition in `process_event` can race with this)

