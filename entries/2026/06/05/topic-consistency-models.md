# Topic: Compare how different implementations handle consistency (eventual vs strong, CAP trade-offs made)

**Date:** 2026-06-05
**Time:** 13:30

# Consistency Models Across SDI Implementations

## The Spectrum: From Eventual to Strong

These implementations don't fall into a binary "eventual vs. strong" bucket — they sit on a spectrum, and several use *different consistency levels for different operations within the same system*. Here's how they break down.

---

## Strong Consistency Implementations

### Payment System — Invariant-Preserving, Single-Node Strong Consistency

The payment system makes the strongest consistency choice in the entire codebase: **double-entry bookkeeping as a hard invariant**. Every money movement produces exactly two ledger entries (a debit and a credit), and `verify_ledger_integrity()` checks that total debits equal total credits within floating-point tolerance of 0.001 (`payment_system.py`, lines 51-52, 83).

Balances are **never cached** — `get_balance()` does a full ledger scan on every call (line 44). This is a deliberate trade-off: correctness over performance. The system derives state from the append-only ledger rather than maintaining a mutable balance counter that could drift.

However, there's an important gap: the overdraft check in `process_payment` reads the balance and then processes, but this is **not atomic** (lines 85-86). In a concurrent system, this would be a race condition. The implementation chooses simplicity (single-threaded correctness) over distributed-safe strong consistency.

**CAP position:** Sacrifices availability and performance for consistency. Single-node design sidesteps partition tolerance entirely.

### Stock Exchange — Deterministic Ordering as Consistency

The matching engine enforces **price-time priority** (FIFO) using deques: bids sorted descending, asks sorted ascending, with strict time ordering within each price level via `deque.append`/`popleft` (`solution.py`, lines 54, 89). Trades always execute at the **resting order's price**, not the aggressor's (line 60).

This is strong consistency in a different sense — it's about *ordering guarantees*. The system ensures that no order can jump the queue, and filled orders are terminal (no modification possible). This models a real exchange's central limit order book, which is inherently a single-point-of-serialization design.

**CAP position:** Consistency and availability within a single partition. Real exchanges solve this by being a single strongly-consistent node with redundant failover, not a distributed system.

### Hotel Reservation — Optimistic Concurrency Control

The reservation system uses a **two-phase read-validate-write cycle** with version-based conflict detection (`hotel_reservation.py`, lines 40, 52-53). Each inventory record has a per-date version counter that monotonically increases (line 31). If the version changes between your read and your write, you get a `ConcurrencyError` (lines 86-87).

This is strong consistency achieved through **optimistic locking** — the system assumes conflicts are rare and detects them at commit time rather than preventing them with locks. The trade-off is that under high contention, callers must retry, which hurts availability.

**CAP position:** Chooses consistency over availability under contention. No partition tolerance (single-node).

---

## Tunable / Configurable Consistency

### Key-Value Store (Dynamo-style) — Quorum-Tunable Consistency

This is the most explicit about CAP trade-offs. The `N/W/R` quorum parameters directly control the consistency level (`key_value_store.py`, lines 36-46):

- **N** = number of replicas per key
- **W** = successful writes required before returning
- **R** = successful reads required before returning
- **W + R > N** guarantees read-write overlap (line 54), but this constraint is **assumed by callers, not enforced in code**

The system layers three consistency mechanisms:

1. **Vector clocks** for conflict detection — a partial order where `concurrent_with` means neither clock dominates, indicating a true conflict (lines 14-15)
2. **Read repair** — opportunistically pushes missing versions to stale replicas during reads (lines 45, 57)
3. **Anti-entropy** via Merkle tree comparison — all-pairs bidirectional sync for background convergence (lines 48-49)

Deletes use **tombstones** rather than physical removal (line 56) because a naive delete would be undone by anti-entropy from a replica that hasn't seen it yet.

**CAP position:** AP by default (available under partitions, eventually consistent). Can be tuned toward CP by increasing W and R, at the cost of availability.

### Distributed Message Queue — Consumer-Chosen Delivery Semantics

The message queue lets each consumer group choose its consistency level through delivery semantics (`solution.py`, lines 62-65):

| Mode | Mechanism | Consistency |
|------|-----------|-------------|
| At-most-once | Auto-commit in `poll()` before processing | Weakest — messages can be lost |
| At-least-once | Explicit `commit()` after processing | Duplicates possible, no loss |
| Exactly-once | Deduplication via `seen_message_ids` set | Strongest — requires consumer-side state |

The two-tier offset tracking (lines 36-37) — `current_offset` advances on every `poll()`, `committed_offset` advances on explicit `commit()` — is the mechanism that enables this choice. All three modes are enforced entirely within `poll()` and `commit()` (lines 164-165).

**CAP position:** The broker itself is AP (available, partition-tolerant, eventually consistent offsets). Exactly-once is pushed to the consumer, which must maintain its own dedup state.

---

## Eventual Consistency Implementations

### News Feed — Fan-Out Strategy as a Consistency/Latency Trade-off

The news feed system makes the consistency trade-off most visible through its three strategies (`news_feed.py`, lines 22, 55):

- **Fan-out-on-write (push):** Post IDs are pushed into followers' deques at write time (line 131). Feeds are immediately consistent for reads but writes are expensive (O(followers)).
- **Fan-out-on-read (pull):** Feeds are assembled at read time via heap merge of followed users' timelines (line 132). Always fresh but reads are expensive.
- **Hybrid:** Uses `celebrity_threshold` (default 1000 followers) to split — normal users get push, celebrities get pull (lines 57-58). The threshold is evaluated **at write time and not retroactively applied** (line 131).

Feed caches use `deque(maxlen=cache_size)` which **silently drops the oldest post IDs when full** (line 135). This is bounded eventual consistency — old posts can disappear from the cache without error.

**CAP position:** AP. Chooses availability and partition tolerance. The push strategy makes feeds "eventually consistent" with a very short window. The pull strategy is always consistent but slower.

### Google Drive — Version Vectors with Conflict Materialization

The sync system uses **per-device version vectors** (not simple counters) for conflict detection (`design_google_drive.py`, lines 18, 49). A conflict requires a *different* device to have written a version the requesting device hasn't seen (line 120).

When conflicts are detected, the system offers two resolution strategies (lines 49, 122-123):
- `latest_wins` — server version stands (no-op)
- `keep_both` — creates a conflict copy

Notably, `restore_version` doesn't roll back — it calls `update_file` with old content, creating a **new** version (line 43). This means the version history is append-only, which preserves consistency of the audit trail.

Soft deletes cascade via BFS traversal (line 121), and permanent deletion only happens in `empty_trash` after a time-based cutoff — a form of eventual garbage collection.

**CAP position:** AP with conflict detection. Available under partitions (devices work offline), with reconciliation on reconnect.

### Chat System — Dual Clock Ordering for Causal Consistency

The chat system uses two independent ordering mechanisms (`chat_system.py`, lines 18-19, 116-117):

1. **Sequence numbers** — per-conversation monotonic counter for total order *within* a conversation (line 88: dense, no gaps)
2. **Lamport timestamps** — global logical clock for *cross-conversation* causal ordering (lines 56-57)

This dual scheme means the system provides **causal consistency** (if you see message A before sending message B, everyone sees A before B) without requiring strong global ordering. Read cursors are monotonic — `mark_read` silently ignores attempts to set a lower sequence number (line 89).

Soft deletes preserve sequence number continuity by keeping the message in the list with `deleted=True` and content replaced with `'[deleted]'` (lines 62, 119-120). This prevents gaps that would break the dense ordering invariant.

**CAP position:** AP with causal consistency. The offline queue drains in FIFO order on reconnect (line 117), accepting that messages may arrive out of wall-clock order but preserving causal order.

### Ad Click Aggregation — Event-Time Consistency via Watermarks

The aggregation system separates **event time** from **processing time** and uses watermarks as an explicit "all events before this time have arrived" signal (`click_aggregator.py`, lines 45-46). This is eventual consistency with a *finalization boundary*:

- Windows have a one-directional lifecycle: `OPEN → CLOSED → FINALIZED` (line 68)
- Only `advance_watermark` can finalize windows — event processing alone never does (line 95)
- Once finalized, results are **immutable** — no retraction or update mechanism exists (line 98)
- Late events are accepted only within `allowed_lateness` seconds past the window end (lines 74-75)

Deduplication is **global, not per-ad** — the `seen_events` registry keys on `event_id` alone (line 96), converting at-least-once delivery into exactly-once aggregation.

**CAP position:** AP with bounded eventual consistency. The watermark mechanism trades latency (waiting for late events) for accuracy (not finalizing too early).

---

## Cross-Cutting Patterns

| Pattern | Implementations | Trade-off |
|---------|----------------|-----------|
| **Tombstones over physical deletes** | KV Store, Google Drive, Chat | Storage cost for consistency under replication |
| **Derived state over cached state** | Payment (balance), News Feed (hydration) | Latency for freshness |
| **Version/vector clocks** | KV Store, Google Drive, Chat | Metadata overhead for conflict detection |
| **Idempotency keys** | Payment, Message Queue, Ad Click | Memory/storage for exactly-once semantics |
| **Monotonic cursors** | Chat (read cursors), MQ (offsets) | Can't "unsee" — progress only moves forward |
| **Append-only with eventual GC** | Google Drive (versions), KV Store (tombstones), Ad Click (dedup registry) | Unbounded growth until pruning |

---

## Topics to Explore

- [function] `key-value-store/key_value_store.py:anti_entropy` — The Merkle-tree-based all-pairs sync is the most sophisticated consistency mechanism in the codebase; worth understanding how it interacts with read repair
- [general] `exactly-once-patterns` — Three implementations (Payment, Message Queue, Ad Click) solve exactly-once differently (idempotency keys, consumer dedup, event-id registry); comparing their failure modes reveals when each approach breaks down
- [function] `news-feed-system/news_feed.py:get_feed` — The three fan-out strategies are all implemented in this single method; trace how the hybrid path decides push vs. pull at the celebrity boundary
- [file] `design-google-drive/design_google_drive.py` — The version vector conflict detection is the only implementation that handles true multi-writer concurrent edits; compare its approach with the KV store's vector clocks
- [general] `consistency-under-failure` — None of these implementations model network partitions, node crashes, or partial failures; exploring how each would behave under partition would reveal which consistency guarantees actually hold

## Beliefs

- `kv-store-quorum-not-enforced` — The W+R>N consistency constraint in the key-value store is assumed by callers but never enforced in code, meaning misconfiguration silently degrades to eventual consistency
- `payment-balance-never-cached` — The payment system derives balances from a full ledger scan on every call to get_balance, never caching, trading performance for guaranteed consistency with the double-entry invariant
- `dmq-delivery-semantics-are-consumer-side` — All three delivery modes (at-most-once, at-least-once, exactly-once) in the message queue are enforced entirely within poll() and commit(), making consistency a consumer choice not a broker guarantee
- `news-feed-celebrity-threshold-evaluated-at-write-time` — The hybrid fan-out strategy evaluates celebrity_threshold when a post is created, not retroactively; a user crossing the threshold does not change the fan-out path for existing posts
- `no-implementation-models-network-partitions` — All ten SDI implementations operate as single-node in-memory systems; CAP trade-offs are structural choices visible in the API design, not runtime behaviors under actual network failure

