# Three Findings Accepted at Pedagogical Scope

## Context

The seven-rules audit of SDI implementations surfaced 15 findings total. We fixed the 4 most actionable ones (see [seven-rules-audit-fixes](seven-rules-audit-fixes.md)). Three findings were real issues but not worth fixing because the fix would either obscure the teaching point, change the API contract, or require infrastructure the codebase deliberately omits. This entry documents each finding, why it's real, why we left it, and what a production system would do differently.

## Finding 5: Payment Lock Scope Too Broad

**File:** `payment-system/payment_system.py:101-148`
**Rule violated:** Rule 4 (protect shared mutable state — but only what needs protecting)

The `process_payment` lock wraps the entire payment flow: balance check, payment creation, external processor call with retry, and ledger entry creation. The external processor call (`_call_processor_with_retry`) happens inside `with self._lock:`, meaning all other payments are serialized behind a potentially slow network call.

```python
with self._lock:
    if self.get_balance(payer_id) < amount:
        # ... fail path ...
    payment = Payment(...)
    payment.status = "PROCESSING"
    result = self._call_processor_with_retry(amount, currency, payer_id, payee_id)  # blocks everyone
    if result["status"] == "success":
        # ... ledger entries ...
```

**Why it's real:** In production, holding a global lock across an external call is a throughput killer. The correct pattern is: check balance under lock, reserve the amount (decrement available balance), release lock, call processor, then under lock again commit or rollback the reservation.

**Why we left it:** We added this lock specifically to fix the TOCTOU race in `process_payment` (the `payment-balance-check-not-atomic` belief we retracted). Splitting the lock into check-reserve / call / commit-or-rollback would triple the complexity of the critical section and obscure the TOCTOU fix that's the teaching point. The existing beliefs (`payment-balance-check-not-atomic` OUT, `financial-concurrency-is-comprehensively-safe` IN) correctly reflect that the race condition is fixed. Adding lock-scope optimization would shift the lesson from "make concurrent balance checks atomic" to "implement a two-phase reservation protocol" — a different topic.

**Production approach:** Reservation-based locking. Debit a hold amount under a short lock, release, call processor, then under lock either convert the hold to a real debit or release it. This is what `wallet.py` already demonstrates with its sorted two-tier locking strategy.

## Finding 6: Message Queue exactly_once Checkpoint Not Enforced

**File:** `distributed-message-queue/solution.py:193-199`
**Rule violated:** Rule 1 (verify on the read path)

The `exactly_once` delivery mode tracks seen message IDs in an in-memory set (`seen_message_ids`) and skips duplicates during `poll()`. But there's no durable checkpoint — if the consumer crashes between `poll()` and `commit()`, it will re-poll from the last committed offset and re-process messages whose IDs are no longer in the seen set.

```python
if group.delivery == "exactly_once" and msg.message_id in group.seen_message_ids:
    group.current_offset[p_idx] = msg.offset + 1
    continue
result.append(msg)
group.current_offset[p_idx] = msg.offset + 1
if group.delivery == "exactly_once":
    group.seen_message_ids.add(msg.message_id)
```

**Why it's real:** True exactly-once requires either transactional offset commits (Kafka's approach: atomically commit offsets and processing side-effects) or idempotent processing (the consumer's downstream operation is itself idempotent). An in-memory set survives neither process restarts nor rebalances.

**Why we left it:** The belief `message-delivery-guarantees-are-consumer-side` captures the design intent: delivery semantics are enforced at the consumer layer through poll/commit mechanics. The broker is deliberately stateless with respect to delivery guarantees. Making the broker enforce durable exactly-once checkpoints would change the API contract — the two-tier offset tracking (`current_offset` vs `committed_offset`) is the teaching point, showing how the gap between "delivered" and "confirmed" creates the window where delivery semantics are decided. Adding transactional commits would make this a Kafka implementation guide rather than a teaching artifact for understanding delivery guarantee tradeoffs.

The related belief `delivery-guarantees-follow-write-read-cost-asymmetry` also captures why this is by design: just as KV reads absorb convergence costs, message consumers absorb delivery semantics through their own behavior.

**Production approach:** Kafka's transactional API: `beginTransaction()`, process messages, `sendOffsetsToTransaction()`, `commitTransaction()`. The offset commit and processing side-effects are atomically committed, making exactly-once a broker+consumer cooperative guarantee rather than a consumer-only one.

## Finding 7: Click Aggregator Dedup Not Durable

**File:** `ad-click-event-aggregation/click_aggregator.py:55,76,97,124-126`
**Rule violated:** Rule 5 (build repair into the design)

The click aggregator's dedup registry is an in-memory dict (`seen_events`) mapping `(event_id, ad_id)` pairs to timestamps. It's pruned on watermark advance (entries older than `2 * allowed_lateness` are evicted). After a process restart, the entire registry is lost and previously-seen events will be re-counted.

```python
self.seen_events: dict[str, float] = {}
# ...
self.seen_events = {eid: ts for eid, ts in self.seen_events.items() if ts >= cutoff}
```

**Why it's real:** In a production click aggregation pipeline (Flink, Spark Streaming), dedup state is checkpointed to durable storage. A restart recovers the checkpoint and continues without re-counting. Without this, any restart within the `2 * allowed_lateness` window produces inflated click counts — exactly the kind of billing-impacting bug that ad systems must prevent.

**Why we left it:** All 25 SDI implementations are in-process simulations with no persistence layer. They use `dict`, `deque`, and `list` where production systems would use Redis, Kafka, or RocksDB. Adding durable checkpointing to the click aggregator alone would require either a file-based checkpoint (introducing I/O into an otherwise pure-computation module) or a mock persistence layer (adding infrastructure that no other module uses). The belief `sdi-modules-are-standalone-learning-artifacts` captures this constraint: each module is independently runnable with only stdlib, no shared code, and in-process simulation.

The existing belief chain already models the correctness correctly: `dedup-and-finalization-are-coordinated` establishes that dedup retention outlives finalization windows (the 2× lateness factor), and `forward-only-stream-processing-is-exactly-once` captures that the pipeline achieves exactly-once when running — the gap is specifically durability across restarts, which is infrastructure rather than algorithm.

**Production approach:** Flink's checkpointing: periodic snapshots of dedup state to a distributed filesystem (S3, HDFS). On restart, restore the latest checkpoint and replay from the corresponding Kafka offset. The dedup registry, watermark position, and window state are all atomically checkpointed together.

## Pattern

All three findings share a common shape: the algorithm is correct but the infrastructure is absent. The payment lock scope is algorithmically right (it prevents the race) but infrastructurally naive (it serializes across external calls). The exactly-once dedup is algorithmically right (it prevents re-processing) but infrastructurally volatile (it doesn't survive restarts). The click dedup is algorithmically right (it prevents re-counting with a 2× lateness buffer) but infrastructurally volatile (same).

This is the expected tradeoff for pedagogical implementations. The belief network captures it through `sdi-modules-are-standalone-learning-artifacts` and `brute-force-acceptable-at-pedagogical-scale`: these modules prioritize implementation clarity over operational robustness, which is the right choice when the goal is teaching the algorithm rather than building the infrastructure.

The interesting observation is that the seven-rules audit cleanly separates the algorithmic issues (which we fixed) from the infrastructural issues (which we left). Rules 1 and 4 exposed real algorithmic gaps — missing verification, missing synchronization — that cost nothing in clarity to fix. Rules 4 (lock scope) and 5 (durability) exposed infrastructural gaps where the fix would require adding a new architectural layer that would obscure the algorithm being taught.
