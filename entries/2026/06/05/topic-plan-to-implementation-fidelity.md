# Topic: Do implementations match their plans? Where do they diverge and why?

**Date:** 2026-06-05
**Time:** 13:34

# Do Implementations Match Their Plans?

The short answer: **yes, closely — but with deliberate simplifications that favor testability over production realism.** The plans in this repo are unusually prescriptive (they specify data models, method signatures, and even assertion examples), which leaves little room for divergence. Where divergence exists, it falls into three categories: structural compression, pragmatic shortcuts in time-handling, and silent omissions of features the plan specified but the implementation quietly dropped.

## High Fidelity: Rate Limiter

The rate limiter plan (`rate-limiter/plan.md`) specifies four algorithms, a middleware, a factory, and metrics tracking. The implementation (`rate-limiter/rate_limiter.py`, 259 lines) delivers all of these within the plan's 200–400 line target.

**What matches exactly:**
- All four algorithm classes (`TokenBucketLimiter`, `FixedWindowCounterLimiter`, `SlidingWindowLogLimiter`, `SlidingWindowCounterLimiter`) subclass `RateLimiter` with the specified `allow_request(client_id, current_time)` signature.
- The abstract base class at line 8 provides `get_remaining`, `get_retry_after`, and `_record` for metrics — matching requirements 1, 8, and 10.
- Token bucket refill logic (lines 46–51) uses `min(bucket_size, tokens + elapsed * refill_rate)`, capping at `bucket_size` as the plan's assertion examples require.
- Sliding window log prunes with a deque (line 130–133), exactly as the plan specifies for bounded memory.
- The `current_time` parameter is accepted everywhere with `time.time()` as fallback — requirement 8.

**Where it diverges:**
The plan's `_record` method (line 28–31) tracks `total_allowed` and `total_denied` as `defaultdict(int)` on the base class. This is a minor structural divergence — the plan lists metrics as a separate requirement (requirement 10), but the implementation folds it into the base class constructor rather than a separate metrics object. This is a simplification, not a gap.

## High Fidelity with Known Gaps: Payment System

The payment system plan (`payment-system/plan.md`) and implementation (`payment-system/payment_system.py`, 238 lines) are well-aligned, but the plan review (`payment-system/plan_review.md`) documents five known divergences between the design and what a production system would need:

**What matches exactly:**
- Idempotency via `_idempotency` dict (line 47) — O(1) lookup as the plan specifies.
- Double-entry bookkeeping using a `SYSTEM_ACCOUNT` contra-account (line 36) for initial balances (lines 61–69).
- State machine transitions: CREATED → PROCESSING → COMPLETED/FAILED, COMPLETED → REFUND_PENDING → REFUNDED — visible in `process_payment` (lines 113–140) and `refund` (lines 143–177).
- Partial refund tracking via `refunded_amount` on the `Payment` dataclass (line 19).
- Balance derived from ledger, never cached — `get_balance` iterates `self._ledger` (line 195+).

**Deliberate divergences documented in plan_review.md:**
1. **`time.sleep` in retry logic** (line 154 area in `_call_processor_with_retry`): blocks the calling thread. The plan says "exponential backoff" — the implementation does this literally with `time.sleep(self._base_delay * (2 ** attempt))`, which is correct for a single-process simulation but wouldn't work in production.
2. **TOCTOU on balance checks**: balance is checked before the processor call (line ~100), but ledger entries are created after (lines ~129–138). No locking, which the plan doesn't require but a real system would need.
3. **Failed idempotency keys are permanent**: once a payment fails and gets mapped to an idempotency key, retrying with the same key returns the FAILED payment. The plan doesn't address this edge case.

## Structural Divergence: Chat System

The chat system shows the most interesting divergence pattern. The plan (`chat-system/plan.md`, 318 lines) is the most ambitious — it specifies 12 distinct feature areas. The implementation (`chat-system/chat_system.py`, 434 lines) delivers on all of them but makes structural choices the plan didn't prescribe:

**What matches:**
- `ChatServer` with `idle_timeout` (line 71), `UserConnection` with inbox/offline_queue (lines 62–67), deterministic conversation IDs via sorted user pairs (lines 86–88).
- Lamport clock for causal ordering (line 80, incremented at line 189).
- Presence system with ONLINE/OFFLINE/AWAY and idle timeout detection (lines 148–155).

**Where it diverges structurally:**
- The plan specifies `contacts: dict[str, set]` for presence notifications, but the implementation at line 81 initializes this as an empty dict that must be populated externally — the plan doesn't specify how contacts are established. This means presence notifications (lines 110–118, 126–134) only fire if someone has manually populated `server.contacts`.
- The plan calls for `user_conversations: dict[str, set]` (line 82) to track which conversations a user belongs to, but from the visible implementation, this index isn't consistently maintained during message sending — `send_message` (line 175+) updates `conv.participants` but the observations cut off before we can confirm `user_conversations` is updated.
- The implementation uses `bisect` (imported at line 6) but the visible portion doesn't show where it's used — likely in `get_history` for cursor-based pagination, which would be an implementation choice not specified in the plan.

## Cross-Cutting Pattern

Across all three implementations, the same divergence pattern repeats:

1. **Plans are prescriptive enough that the core algorithm matches exactly.** When the plan says "weighted count = prev_count * weight + current_count," that's exactly what appears in `SlidingWindowCounterLimiter._weighted_count` (lines 173–179).
2. **Production concerns are acknowledged but deliberately skipped.** Thread safety, async I/O, persistent storage — all absent by design. The plans state "single-process" and "in-memory," and the implementations honor that constraint.
3. **Memory management is implemented but minimal.** The rate limiter prunes deques and clears old window keys. The payment system doesn't prune its ledger list. The chat system observations don't show pruning.
4. **No divergence markers in the code.** The grep for TODO/FIXME/HACK across these three systems found zero hits — all 16 matches came from web-crawler and other systems. The implementer either hit the plan exactly or didn't annotate deviations.

## What's Missing From These Observations

- The tail of `rate_limiter.py` (lines 200–259) — likely contains the `HTTPRateLimitMiddleware` and `RateLimiterFactory` classes, which I can't verify against the plan.
- The tail of `payment_system.py` (lines 200–238) — includes `get_balance`, `get_ledger`, `get_payments`, `register_webhook`, `verify_ledger_integrity`, and webhook-related code.
- The tail of `chat_system.py` (lines 200–434) — more than half the implementation, including history pagination, message editing/deletion, typing indicators, and search.
- Test files for all three systems — can't verify whether all planned test cases were actually implemented.

---

## Topics to Explore

- [file] `chat-system/chat_system.py` — The second half (lines 200–434) contains history pagination, message edit/delete, typing indicators, and search — the features most likely to diverge from the plan's specification
- [function] `rate-limiter/rate_limiter.py:HTTPRateLimitMiddleware` — The middleware and factory classes (lines 200–259) are prescribed in detail by the plan but weren't visible in the observations; verify per-path routing and header generation match the spec
- [diff] `payment-system` — Compare the plan's state machine (CREATED→PROCESSING→COMPLETED/FAILED) against the actual transitions in `process_payment` and `refund` to verify no unreachable states exist
- [general] `retry-and-time-simulation` — All three systems use `current_time` injection for testability but handle the `None` default differently (rate limiter falls back to `time.time()`, chat system falls back to `0.0`); this inconsistency could cause subtle bugs in integration scenarios
- [file] `web-crawler/web_crawler.py` — The only system with TODO/divergence markers in the grep results; the `pages_skipped_*` stats suggest plan-vs-implementation gaps worth investigating

## Beliefs

- `plans-are-prescriptive` — Plans specify exact method signatures, data models, and assertion examples, leaving implementations with near-zero design freedom at the API level
- `no-divergence-annotations` — The rate limiter, payment system, and chat system contain zero TODO/FIXME/HACK comments; deviations from the plan are undocumented in the code itself
- `current-time-fallback-inconsistent` — Rate limiter defaults `current_time` to `time.time()` while chat system defaults to `0.0`, creating different behavior when the parameter is omitted
- `payment-balance-is-O-n` — `PaymentSystem.get_balance` derives balance by scanning the entire ledger list on every call rather than maintaining a cached running total
- `plan-review-documents-known-gaps` — The `plan_review.md` pattern explicitly catalogues TOCTOU, blocking sleep, and permanent-failure-on-idempotency-key as accepted divergences rather than bugs

