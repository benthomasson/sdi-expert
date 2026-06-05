# File: rate-limiter/rate_limiter.py

**Date:** 2026-06-05
**Time:** 13:19

## Purpose

This file implements the **rate limiter** system design problem — a core infrastructure component that controls how many requests a client can make within a time window. It owns the complete rate-limiting domain: four distinct algorithms, per-client tracking, HTTP middleware integration, and a factory for algorithm selection.

This is a teaching implementation — all state is in-memory (no Redis/shared storage), making it suitable for single-process use and demonstrating algorithm mechanics clearly.

## Key Components

### `RateLimiter` (abstract base)

The strategy interface. Every algorithm must implement `allow_request(client_id, current_time) -> bool`. The base class provides two opt-in query methods (`get_remaining`, `get_retry_after`) that return zero defaults, and a `_record` helper that tracks per-client allow/deny counts in `total_allowed` / `total_denied` dictionaries. These counters are bookkeeping only — they don't affect limiting decisions.

The `current_time` parameter is injectable everywhere, which makes the entire system deterministically testable without mocking `time.time()`.

### `TokenBucketLimiter`

Classic token bucket. Each client gets a bucket of `bucket_size` tokens that refills at `refill_rate` tokens/second. A request costs 1 token. The bucket state is lazily initialized on first request and refilled on each access by computing `elapsed * refill_rate`, capped at `bucket_size`.

State is stored as `[tokens_float, last_refill_time]` — a two-element list used mutably. The float token count allows sub-token precision, so `get_remaining` truncates via `int()` while `allow_request` checks `>= 1.0`.

`get_retry_after` calculates exactly how long until one token is available: `(1.0 - current_tokens) / refill_rate`.

### `FixedWindowCounterLimiter`

Divides time into aligned windows of `window_size_seconds`. Each window has an independent counter. When a request arrives, it computes the window key via integer division (`int(t // window_size)`) and checks if the count is below `max_requests`.

Memory management: on every allowed request, it replaces the entire counter dict with just the current window's entry (`{wk: count + 1}`). This is a simple but effective garbage collection — old windows are discarded immediately.

`get_retry_after` returns the time until the current window ends.

### `SlidingWindowLogLimiter`

The most precise algorithm. Stores every request timestamp in a `deque` per client. On each access, `_prune` removes timestamps older than the window. A request is allowed if `len(log) < max_requests`.

`get_retry_after` is the most interesting method here: it returns how long until the *oldest* entry in the log expires (`log[0] + window_size - current_time`), which is exactly when one slot opens up.

Trade-off: perfect accuracy, but O(n) memory per client where n = max_requests.

### `SlidingWindowCounterLimiter`

A hybrid that approximates the sliding window using two fixed windows. It keeps counters for the current and previous windows, then computes a weighted count:

```
weighted = prev_count * (1 - elapsed_fraction) + current_count
```

where `elapsed_fraction = (current_time - window_start) / window_size`. As you move through a window, the previous window's contribution linearly decays from 100% to 0%.

Memory: retains at most 2 window keys per client (current and previous), actively pruning older ones.

### `HTTPRateLimitMiddleware`

Bridges rate limiters to HTTP semantics. Takes a `default_limiter` and optional `path_rules` dict mapping URL paths to specific limiters. Returns `200` with `X-RateLimit-Remaining` on success, `429` with both `X-RateLimit-Remaining` and `X-RateLimit-Retry-After` on rejection.

The request/response format is dict-based (not tied to any HTTP framework), keeping the implementation portable.

### `RateLimiterFactory`

Simple static registry mapping algorithm name strings to classes. `create("token_bucket", bucket_size=10, refill_rate=1.0)` instantiates the right class with kwargs forwarded.

## Patterns

- **Strategy pattern**: `RateLimiter` is the strategy interface; the four algorithms are interchangeable strategies. The middleware and factory are both strategy-agnostic.
- **Template method**: `_record` is a shared hook called by all subclass `allow_request` implementations.
- **Factory pattern**: `RateLimiterFactory.create` decouples algorithm selection from instantiation.
- **Time injection**: Every method accepts `current_time` with a `time.time()` default, enabling deterministic testing without patching.
- **Lazy initialization**: Token bucket creates state on first request; fixed/sliding window counters use `defaultdict` for implicit initialization.

## Dependencies

**Imports**: Only stdlib — `time`, `abc.ABC`/`abstractmethod`, `collections.defaultdict`/`deque`. No external dependencies.

**Imported by**: `test_rate_limiter.py` — the test suite exercises all algorithms, the middleware, and the factory.

## Flow

A typical request flow through the middleware:

1. `HTTPRateLimitMiddleware.handle_request(request)` extracts `client_id`, `path`, `timestamp`
2. Selects the limiter: checks `path_rules[path]`, falls back to `default_limiter`
3. Calls `limiter.allow_request(client_id, timestamp)` — the algorithm decides and records the result
4. Calls `limiter.get_remaining(...)` unconditionally for the response header
5. If denied, also calls `limiter.get_retry_after(...)` for the retry header
6. Returns a status/headers dict

Within each algorithm, the flow is: **normalize time → update/refill state → check against limit → record → return**.

## Invariants

- **Token count is capped**: `_get_bucket` ensures `tokens <= bucket_size` after refill — burst capacity is bounded.
- **Token cost is fixed at 1.0**: There's no variable-cost API; every request consumes exactly one token.
- **Sliding window log stores only in-window timestamps**: `_prune` runs before every read, so `len(log)` is always an accurate count.
- **Sliding window counter keeps exactly 2 windows**: Older keys are actively deleted on every allowed request.
- **Fixed window counter keeps exactly 1 window**: The entire dict is replaced on allow.
- **`_record` is always called**: Every `allow_request` path (both true and false branches) calls `_record`, so `total_allowed[id] + total_denied[id]` equals total requests for that client.

## Error Handling

Minimal — this is an in-memory, single-process implementation:

- `RateLimiterFactory.create` raises `ValueError` for unknown algorithm names. This is the only explicit error.
- No validation on constructor args (negative `bucket_size`, zero `refill_rate`, etc.) — callers are trusted.
- No validation on request dicts — missing keys will raise `KeyError` from dict access, not a custom error.
- Division by zero is possible if `refill_rate=0` in `get_retry_after`. Not guarded.

## Topics to Explore

- [file] `rate-limiter/test_rate_limiter.py` — See how each algorithm is exercised, especially edge cases around window boundaries and token refill timing
- [file] `rate-limiter/plan.md` — Understand the design decisions and trade-offs considered before implementation
- [general] `sliding-window-counter-accuracy` — How the weighted approximation diverges from exact sliding window log counts, and when the approximation matters in production
- [general] `distributed-rate-limiting` — How this single-node design would change with Redis-backed counters, race conditions under concurrent access, and Lua scripting for atomicity
- [function] `rate-limiter/rate_limiter.py:SlidingWindowLogLimiter.get_retry_after` — The retry-after calculation for the log-based approach is the most nuanced; worth tracing through with concrete numbers

## Beliefs

- `rate-limiter-four-algorithms` — The implementation provides exactly four rate limiting algorithms: token bucket, fixed window counter, sliding window log, and sliding window counter, all interchangeable via the `RateLimiter` ABC
- `rate-limiter-time-injectable` — Every method that depends on wall-clock time accepts an optional `current_time` parameter, defaulting to `time.time()`, making all algorithms deterministically testable
- `rate-limiter-per-client-state` — All rate limiting state is tracked per `client_id` string; there is no global (cross-client) rate limiting
- `rate-limiter-middleware-path-routing` — `HTTPRateLimitMiddleware` supports per-path rate limiting via `path_rules` dict, falling back to `default_limiter` for unmatched paths
- `rate-limiter-no-persistence` — All state is in-memory with no persistence or distribution mechanism; the implementation is single-process only

