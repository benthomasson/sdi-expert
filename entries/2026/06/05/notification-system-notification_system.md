# File: notification-system/notification_system.py

**Date:** 2026-06-05
**Time:** 13:55

## Purpose

This file implements a **notification delivery system** — the kind you'd design in a system design interview for services like Facebook notifications, Uber alerts, or any platform that sends push/SMS/email to users. It owns the full lifecycle: accepting notification requests, queuing them by priority, enforcing rate limits and user preferences, rendering templates, delivering through channel-specific providers, and retrying on failure.

It's a single-process simulation — no actual network calls, no real message brokers — but it faithfully models the architectural concerns: priority queuing, per-user per-channel rate limiting, exponential backoff with jitter, quiet hours, opt-out enforcement, and delivery status tracking.

## Key Components

### Enums

- **`Priority(IntEnum)`** — Four levels (CRITICAL=0 through LOW=3). Uses `IntEnum` so values are directly comparable and work as heap sort keys — lower numeric value = higher priority.
- **`Channel(Enum)`** — Three delivery channels: PUSH, SMS, EMAIL.
- **`DeliveryStatus(Enum)`** — Six-state lifecycle: PENDING → QUEUED → SENDING → DELIVERED/FAILED/RATE_LIMITED.

### Data Classes

- **`Notification`** — The core domain object. Carries routing info (`user_id`, `channel`, `priority`), content (`template_name`/`template_context` or `raw_content`), delivery state (`status`, `retry_count`, `status_history`), scheduling (`deliver_at`), and grouping (`group_key`). The `status_history` field records every state transition as a `(status, timestamp)` tuple, giving full audit trail.

- **`UserPreferences`** — Per-user delivery settings: opted-out channels, quiet hours (as hour-of-day range), and channel priority ordering. The `preferred_channels` field has a default ordering of PUSH > EMAIL > SMS.

### Delivery Channels

- **`DeliveryChannel`** — Base class with configurable `failure_rate` (0.0–1.0) for simulating unreliable delivery. Uses `random.random() >= failure_rate` to decide success. Maintains a `_sent_log` for test assertions.
- **`PushChannel`**, **`SMSChannel`**, **`EmailChannel`** — Channel-specific subclasses. `SMSChannel` is the interesting one: it truncates messages to 160 characters, modeling the real SMS constraint. The others just tag their log entries with the channel name.

### TemplateRegistry

Stores templates keyed by `(name, channel)` pairs — the same notification type can have different templates per channel (e.g., a short push vs. a rich email). Rendering uses regex substitution on `{{var}}` placeholders. Raises `KeyError` for missing templates or missing context variables — fail-fast, no silent defaults.

### RateLimiter

Sliding-window rate limiter keyed by `(user_id, channel)`. The `_limits` dict maps each channel to `(max_count, window_size_seconds)`. Default limits: 3 pushes/minute, 1 SMS/minute, 5 emails/hour.

- **`check()`** — Evicts expired timestamps from the deque, returns whether the count is under the limit. Read-only — doesn't record the attempt.
- **`record()`** — Appends the timestamp only after successful delivery. This is the correct pattern: you don't consume rate limit budget on failed attempts.

### NotificationService

The orchestrator. Key methods:

- **`send()`** — Entry point. Validates opt-out and rate limits eagerly (before queuing), then pushes onto the priority heap. Returns the notification ID immediately — this is an async-style API even though processing is synchronous.

- **`process_queue()`** — Drains the heap in priority order. For each notification: checks terminal states (skip), future scheduling (defer), quiet hours (defer), rate limits (reject), then attempts delivery. On failure, implements **exponential backoff with jitter**: `delay = 2^(retry-1) * uniform(0.5, 1.5)`. Deferred items are re-pushed after processing. Returns the count of successful deliveries.

- **`send_batch()`** — Thin wrapper that calls `send()` in a loop. No transactional semantics.

## Patterns

**Priority queue with tie-breaking** — The heap stores `(priority, timestamp, seq, notification)` tuples. `priority` gives CRITICAL-first ordering, `timestamp` gives FIFO within the same priority, and `seq` (a monotonic counter) breaks ties when timestamps collide. This three-level key is the standard pattern for stable priority queues with `heapq`.

**Sliding window rate limiting** — Uses a `deque` per (user, channel) pair, evicting entries older than the window on each `check()`. This is the classic approach from the rate limiter SDI chapter — O(1) amortized eviction, O(1) check.

**Exponential backoff with jitter** — On delivery failure, retry delay doubles each attempt (`2^(n-1)`) multiplied by a random factor in [0.5, 1.5]. This prevents thundering herd on provider recovery.

**Two-phase rate limit checking** — Rate limits are checked both at `send()` time (fast rejection) and again at `process_queue()` time (because time may have passed and new sends may have consumed budget between enqueue and delivery).

**Status machine with audit trail** — Every status transition goes through `_update_status()`, which both sets the current status and appends to `status_history`. This gives a complete delivery timeline per notification.

**Template-per-channel** — Templates are keyed by `(name, channel)`, so the same logical notification type (e.g., "order_confirmation") can render differently for push vs. email. This models how real notification systems adapt content to channel constraints.

## Dependencies

**Imports**: All stdlib — `heapq` for priority queue, `collections.deque` for sliding window, `re` for template rendering, `random` for failure simulation and jitter, `datetime` for quiet-hours calculation, `uuid` (imported but unused — IDs are passed in externally).

**Imported by**: `test_notification_system.py` — the test suite is the only consumer, as expected for a standalone SDI implementation.

## Flow

1. **Setup**: Create `DeliveryChannel` instances, wire them into `NotificationService`, register templates, set user preferences.
2. **Enqueue**: Call `send()` — notification gets opt-out check, rate limit check, then pushed to heap with `QUEUED` status.
3. **Process**: Call `process_queue(current_time)` — pops from heap in priority order, checks scheduling/quiet-hours/rate-limits, renders content (template or raw), calls the channel's `send()`, handles success (record rate limit, update status) or failure (calculate backoff, re-enqueue or mark `FAILED`).
4. **Query**: Call `get_status()`, `get_user_history()`, or `get_stats()` to inspect results.

The caller controls time — `current_time` is always passed explicitly, never read from the system clock. This makes the system fully deterministic and testable (modulo `random` for failure simulation).

## Invariants

- **A notification's status only moves forward** through the lifecycle, with one exception: `QUEUED` can reappear after `SENDING` on retry. The state machine is: PENDING → QUEUED → SENDING → {DELIVERED | FAILED | back to QUEUED on retry}.
- **Rate limit budget is consumed only on successful delivery** — `record()` is called after `channel.send()` returns `True`. Failed attempts don't eat rate limit quota.
- **Retries are bounded** — `retry_count` is compared against `max_retries` (default 3). After exhaustion, the notification is terminal `FAILED`.
- **Quiet hours span midnight correctly** — `_is_quiet_hours` handles the overnight case (`start > end`, e.g., 22–8) with `hour >= start or hour < end`.
- **Heap ordering is stable** — the `_seq` counter ensures notifications with identical `(priority, timestamp)` are processed in insertion order.
- **Template rendering is strict** — missing templates or missing variables raise `KeyError`, never silently produce partial output.

## Error Handling

- **Missing template or variable**: `TemplateRegistry.render()` raises `KeyError` with a descriptive message. This will propagate up through `process_queue()` uncaught — a bug in template setup will crash queue processing rather than deliver garbage.
- **Missing channel**: If `self.channels` doesn't have the notification's channel, the notification is marked `FAILED` — a silent failure, not an exception.
- **Missing notification ID**: `get_status()` raises `KeyError` for unknown IDs.
- **Delivery failures**: Handled via retry with backoff, eventually marked `FAILED` — no exceptions raised to the caller.
- **Opt-out notifications**: Marked `RATE_LIMITED` (arguably a misnomer — opt-out is semantically different from rate limiting) and stored, but not counted in `_stats["total_rate_limited"]`. This is an inconsistency: rate-limited notifications at `send()` time *are* counted, but opt-outs are not.

## Topics to Explore

- [file] `notification-system/test_notification_system.py` — See how the deterministic time model and configurable failure rates are exercised in tests
- [function] `notification-system/notification_system.py:process_queue` — The retry backoff logic and deferred-item re-enqueue pattern are the most complex control flow; trace through a multi-retry scenario
- [general] `group-key-batching` — The `_pending_groups` dict and `_group_window` field are initialized but never used — this appears to be a planned notification batching/digest feature that was never implemented
- [file] `rate-limiter/rate_limiter.py` — Compare the standalone rate limiter SDI implementation against the embedded `RateLimiter` class here to see how the same concept adapts to different contexts
- [general] `quiet-hours-timezone` — Quiet hours use `datetime.UTC` exclusively; explore how a real system would handle per-user timezones

## Beliefs

- `notif-rate-limit-on-success-only` — Rate limit budget (`RateLimiter.record`) is consumed only after successful delivery, not on send attempts or failures
- `notif-priority-heap-stable` — The priority queue uses a three-element key `(priority, timestamp, seq)` ensuring CRITICAL-first, FIFO within priority, and stable ordering on timestamp ties
- `notif-group-batching-unimplemented` — `_pending_groups` and `_group_window` are initialized in `NotificationService.__init__` but never read or written anywhere else — dead code for an unfinished feature
- `notif-opt-out-misclassified-as-rate-limited` — Opt-out rejections set status to `RATE_LIMITED` rather than a distinct status, and unlike actual rate-limited notifications, they don't increment `_stats["total_rate_limited"]`
- `notif-template-render-crash-propagates` — A missing template or variable in `TemplateRegistry.render` raises `KeyError` that propagates uncaught through `process_queue`, halting queue processing mid-batch

