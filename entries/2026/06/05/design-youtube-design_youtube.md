# File: design-youtube/design_youtube.py

**Date:** 2026-06-05
**Time:** 13:45

# `design-youtube/design_youtube.py`

## Purpose

This file implements a simulation of YouTube's core backend subsystems as a system design interview exercise. It doesn't build a real video platform — it models the *architectural concepts* you'd discuss in an interview: video upload pipelines, adaptive streaming, approximate counting at scale, and multi-strategy recommendation. Each component is self-contained and testable in-process without any external infrastructure.

## Key Components

### Data Models

**`Video`** — The central entity. Tracks metadata (title, tags, uploader), lifecycle status via `VideoStatus`, engagement counters (views, likes, dislikes), and post-processing artifacts (`manifest`, `thumbnails`). Status transitions are `UPLOADING → PROCESSING → READY | FAILED`.

**`TranscodedVariant`** — Represents a single encoding of a video at a specific resolution/bitrate. Created during transcoding, consumed by `StreamingManifest`.

**`StreamingManifest`** — Models an HLS/DASH-style adaptive bitrate manifest. `select_quality(bandwidth_kbps)` walks variants from highest to lowest bitrate and returns the best one the client can sustain. Returns `None` if no variant fits — the caller must handle that.

### Processing Pipeline

**`ProcessingDAG`** — A general-purpose DAG executor. Stages are registered with named dependencies, then executed in topological order (Kahn's algorithm). Two key behaviors:
- Cycle detection runs *before* execution and raises `ValueError`.
- If a stage fails (exception or simulated random failure), all downstream dependents are `SKIPPED` — failure cascades through the graph.

**`VideoUploadPipeline`** — Wires up a specific 5-stage DAG for video processing:

```
validate → transcode ─┐
validate → thumbnail ──┤→ finalize
validate → metadata ───┘
```

`transcode`, `thumbnail`, and `metadata` are independent after validation — they'd run in parallel in a real system. The `finalize` stage fans in, assembling the manifest and thumbnails onto the `Video` object.

The pipeline handlers are plain functions (`handle_validate`, `handle_transcode`, etc.) that read/write a shared `ctx` dict. This is essentially a poor-man's blackboard pattern.

### Approximate Counters

**`MorrisCounter`** — Implements Morris's approximate counting algorithm (Morris+ variant). Uses 32 independent counters, each incremented with probability `1/2^c`. The estimate `2^c - 1` is averaged across all counters for better accuracy. This is relevant at YouTube scale where exact counting of billions of views is expensive.

**`HyperLogLogCounter`** — Estimates cardinality (distinct count) using HyperLogLog with configurable precision (default `p=14`, so `m=16384` registers). Includes the small-range correction from the original Flajolet paper. Used here to approximate unique viewer counts.

**`ViewCounter`** — Facade that combines all three counting strategies per video:
- Exact count (baseline/ground truth for testing)
- Morris approximate count (total views)
- HLL unique viewer estimate
- Watch percentage tracking for engagement metrics

### Storage & Search

**`VideoStore`** — In-memory CRUD store. `search()` does case-insensitive substring matching on title and tags, filtering to `READY` or `UPLOADING` status. `upload()` generates a UUID and returns the `Video` object in `UPLOADING` state — the caller is responsible for running the pipeline separately.

### Recommendations

**`RecommendationEngine`** — Three strategies, combinable via weighted scoring:

1. **Content-based** (`recommend_by_content`): Jaccard similarity on tag sets. Simple and deterministic.
2. **Popularity** (`recommend_popular`): Sorts by exact view count.
3. **Collaborative filtering** (`recommend_collaborative`): Co-occurrence — "users who watched video X also watched Y." Counts how many co-watchers viewed each unseen video.

**`get_feed`** merges all three using reciprocal rank fusion: each strategy contributes `weight * 1/(rank+1)` to a video's score. Already-watched videos are excluded. Default weights favor collaborative (0.5) over popular (0.3) over content (0.2).

## Patterns

- **DAG-based pipeline**: Processing stages form a dependency graph. This is the standard pattern for media processing pipelines (similar to Apache Airflow's task model). The separation between `ProcessingDAG` (generic executor) and `VideoUploadPipeline` (specific wiring) keeps the engine reusable.
- **Context-passing via dict**: Pipeline stages communicate through a mutable `ctx` dict rather than return values. This avoids coupling stages to each other's types but sacrifices type safety.
- **Probabilistic data structures**: Morris counters and HLL demonstrate how YouTube handles counting at scale — trading precision for memory/throughput. The `ViewCounter` layering exact + approximate is a common interview talking point.
- **Strategy pattern in recommendations**: Three independent recommendation algorithms behind a unified `get_feed` that blends them with configurable weights.
- **Fault injection**: `failure_rate` on `ProcessingStage` enables testing failure cascades without mocking — the pipeline can simulate real-world transcoding failures.

## Dependencies

**Imports**: All stdlib — `hashlib` (HLL hashing), `math` (HLL correction), `random` (Morris counters, fault injection), `uuid` (video IDs), `collections.defaultdict`, `dataclasses`, `enum`, `typing`.

**Imported by**: `test_design_youtube.py` — the test suite exercises all components.

No external dependencies. No cross-module imports within the repo.

## Flow

A typical lifecycle:

1. `VideoStore.upload()` creates a `Video` in `UPLOADING` status.
2. `VideoUploadPipeline.process()` sets status to `PROCESSING`, builds the DAG, executes it.
3. DAG runs: validate → (transcode | thumbnail | metadata) → finalize.
4. `handle_finalize` builds the `StreamingManifest` and sets status to `READY`.
5. Views are recorded via `ViewCounter.record_view()`, which updates exact counts, Morris counters, HLL, and watch percentages.
6. `RecommendationEngine.get_feed()` blends strategies to produce a ranked list for a user.

## Invariants

- **Format validation**: `handle_validate` rejects formats not in `VALID_FORMATS` (`mp4`, `avi`, `mkv`, `mov`, `webm`) and durations exceeding 43200s (12 hours).
- **DAG acyclicity**: `_detect_cycle()` enforces no cycles before execution. This is checked on every `execute()` call.
- **Failure cascade**: A failed stage causes all transitive dependents to be `SKIPPED`. The pipeline sets `VideoStatus.FAILED` if `finalize` doesn't complete.
- **Watch percentage clamping**: `record_view` clamps to `[0.0, 100.0]`.
- **Deterministic quality selection**: `select_quality` always returns the highest-bitrate variant that fits, or `None`. No randomness.

## Error Handling

- **Pipeline stage failures** are caught by `ProcessingDAG.execute()` — the bare `except Exception` swallows the error, marks the stage `FAILED`, and lets execution continue for independent branches. This means stage errors never propagate to the caller; you only see them in the returned status dict.
- **Validation errors** (`handle_validate`) raise `ValueError`, which the DAG catches.
- **`VideoStore.get()`** returns `None` for missing videos rather than raising.
- **`VideoStore.delete()`** returns a boolean success flag.
- **`StreamingManifest.select_quality()`** returns `None` when no variant fits the bandwidth — callers must check.

## Topics to Explore

- [file] `design-youtube/test_design_youtube.py` — See how the pipeline, counters, and recommendation engine are exercised; reveals edge cases the implementation handles
- [function] `design_youtube.py:ProcessingDAG.execute` — Trace how failure cascades work through the DAG; consider what happens when `thumbnail` fails but `transcode` succeeds
- [general] `morris-counter-accuracy` — Compare Morris counter estimates against exact counts at various scales to understand the precision/memory tradeoff
- [general] `hll-small-range-correction` — The HLL implementation includes the linear counting correction for small cardinalities; worth understanding when it activates and why
- [function] `design_youtube.py:RecommendationEngine.get_feed` — Explore how reciprocal rank fusion blends strategies and how weight tuning affects feed composition

## Beliefs

- `dag-failure-cascade` — When a ProcessingDAG stage fails, all transitively dependent stages are marked SKIPPED; independent branches continue executing
- `pipeline-status-contract` — VideoUploadPipeline sets video status to FAILED if and only if the finalize stage does not reach COMPLETED
- `hll-default-precision` — HyperLogLogCounter defaults to precision 14 (16384 registers), matching the standard HLL recommendation for ~1% error rate
- `recommendation-excludes-watched` — get_feed excludes videos the user has already watched from popular and content-based scores, but collaborative filtering excludes them in its own co-occurrence logic
- `search-status-filter` — VideoStore.search returns only videos with status READY or UPLOADING, excluding PROCESSING and FAILED videos

