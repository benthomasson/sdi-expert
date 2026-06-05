# File: distributed-message-queue/solution.py

**Date:** 2026-06-05
**Time:** 13:17

# `distributed-message-queue/solution.py`

## Purpose

This file implements an **in-memory Kafka-like distributed message queue** — one of the system design interview implementations in the `sdi-implementations` repo. It models the core abstractions of a partitioned, consumer-group-based message broker: topics with configurable partitions, key-based and round-robin routing, consumer group rebalancing, offset tracking with multiple delivery semantics, and a dead-letter queue for poison messages.

It's a teaching implementation — no networking, no persistence, no replication — but it faithfully models the **logical contracts** that a real distributed message queue like Kafka enforces.

## Key Components

### Data Classes

- **`Message`** — The producer-facing message type. Carries an optional `key` (used for partition routing), a `value` (arbitrary object), and optional `headers`. This is what you hand to `publish()`.

- **`StoredMessage`** — The broker-internal envelope. Extends `Message` with `topic`, `partition`, `offset`, `timestamp`, and `message_id`. This is what consumers receive from `poll()`. The `message_id` (a UUID) is critical for exactly-once deduplication.

### `Partition`

Models a single append-only log segment. Key contract:

- **`base_offset`** tracks the logical offset of the first message still in memory. After `trim()`, earlier offsets are gone but the numbering stays consistent — `get()` translates logical offsets to physical indices via `offset - self.base_offset`.
- **`next_offset`** is always `base_offset + len(messages)` — the offset the next appended message will receive.
- **`trim(retention_count)`** evicts the oldest messages when the partition exceeds `retention_count`, advancing `base_offset` accordingly. This is called on every `publish()`.

### `ConsumerGroup`

Tracks the full consumer-side state for a group subscribed to one topic:

- **`consumers`** — ordered list of consumer IDs (insertion order).
- **`assignments`** — which partitions each consumer owns, computed by `rebalance()`.
- **`committed_offset` / `current_offset`** — two-tier offset tracking. `current_offset` advances on every `poll()`. `committed_offset` advances on explicit `commit()` (or auto-commit for at-most-once). The gap between them is the "uncommitted" window — messages that have been delivered but not confirmed.
- **`seen_message_ids`** — deduplication set for exactly-once delivery.
- **`failure_counts`** — per-message negative-ack counter for DLQ routing.

**`rebalance()`** uses simple modular assignment: partition `p` goes to `consumers[p % len(consumers)]`. This is a simplified version of Kafka's range/round-robin assignors.

### `MessageQueue`

The top-level broker. Owns all topics, partitions, consumer groups, and routing state.

| Method | Contract |
|--------|----------|
| `create_topic` / `delete_topic` | Topic lifecycle. `delete_topic` cascades to remove all consumer groups bound to that topic. |
| `publish` | Routes message to partition (key-hash or round-robin), appends, trims for retention, returns the `StoredMessage`. |
| `create_consumer_group` | Binds a group to a topic with a delivery semantic. |
| `add_consumer` / `remove_consumer` | Membership changes trigger `rebalance()`. |
| `poll` | Fetches up to `max_messages` from the consumer's assigned partitions, respecting delivery semantics. |
| `commit` | Advances `committed_offset` to match `current_offset` for a consumer's partitions. |
| `seek` | Random-access offset reset. Supports sentinel values: `-1` = beginning, `-2` = end. |
| `acknowledge` | Negative-ack path. Tracks failure counts per message; after `max_failures` (default 3), routes to DLQ. |
| `get_topic_info` / `get_consumer_lag` | Observability: partition metadata and per-partition lag (distance between committed offset and partition head). |

## Patterns

**Kafka's logical model, faithfully miniaturized.** The code mirrors Kafka's core abstractions almost 1:1: topics → partitions → offsets, consumer groups with rebalancing, committed vs. current offsets, key-based partitioning, round-robin for null keys, retention-based trimming.

**Two-tier offset tracking.** `current_offset` and `committed_offset` are separate dictionaries, enabling the three delivery semantics:
- **At-most-once**: auto-commit in `poll()` before processing — if the consumer crashes, messages are skipped.
- **At-least-once** (default): consumer must call `commit()` explicitly — if it crashes before committing, messages are re-delivered.
- **Exactly-once**: deduplication via `seen_message_ids` set — already-seen messages are skipped during `poll()`.

**Dead-letter queue as a regular topic.** `_send_to_dlq` creates a topic named `__dlq_{topic}` on first use and publishes the failed message into it. This means DLQ messages are consumable with the same API — no special-case code needed.

**Round-robin via counter.** `_rr_counters` is a per-topic monotonic counter for null-key messages, giving even distribution across partitions without randomness.

## Dependencies

**Imports**: Only stdlib — `time` (timestamps), `uuid` (message IDs), `defaultdict` (failure counters), `dataclass` (message types). No external dependencies.

**Imported by**:
- `distributed-message-queue/test_solution.py` — the primary test suite.
- `stock-exchange/test_exchange.py` — the stock exchange implementation reuses this message queue as its order-matching event bus, which demonstrates the queue's role as a reusable infrastructure component across multiple SDI solutions.

## Flow

### Publish path

```
publish(topic, Message)
  → key-hash or round-robin → select partition index
  → create StoredMessage (offset = partition.next_offset, timestamp = now, message_id = uuid)
  → partition.append()
  → partition.trim(retention_count)
  → return StoredMessage
```

### Consume path

```
poll(group_id, consumer_id, max_messages)
  → look up assigned partitions for this consumer
  → for each assigned partition:
      → partition.get(current_offset, remaining_capacity)
      → for each message:
          → [exactly_once] skip if message_id in seen_message_ids
          → append to result, advance current_offset
          → [exactly_once] add message_id to seen_message_ids
      → stop if max_messages reached
  → [at_most_once] auto-commit offsets
  → return messages
```

### Rebalance path

```
add_consumer / remove_consumer
  → update consumers list
  → rebalance(): for each partition p, assign to consumers[p % len(consumers)]
  → return new assignments
```

### Failure path

```
acknowledge(group_id, message, success=False)
  → increment failure_counts[message_id]
  → if count >= max_failures:
      → create __dlq_{topic} if needed
      → publish message to DLQ
      → clear failure count
```

## Invariants

1. **Offset monotonicity**: `base_offset` only increases (via `trim()`), `next_offset` only increases (via `append()`). A partition's offset space is never reused.

2. **`current_offset >= committed_offset`** per partition within a group — `poll()` advances `current_offset`, `commit()` catches `committed_offset` up. They never cross.

3. **Partition assignment is deterministic**: given the same consumer list and partition count, `rebalance()` always produces the same assignment. Partition `p` always goes to `consumers[p % len(consumers)]`.

4. **Retention is enforced per-publish**: every `publish()` call trims the target partition. Messages are never evicted between publishes.

5. **One topic per consumer group**: a `ConsumerGroup` is bound to exactly one topic at creation time. There's no multi-topic subscription.

6. **DLQ topics use the `__dlq_` prefix convention** and are created lazily on first failure.

## Error Handling

All validation uses `ValueError` with descriptive messages:
- Duplicate topic/group creation raises immediately.
- Operations on nonexistent topics/groups raise immediately.
- Invalid partition index in `seek()` raises.

Notably, `poll()` for an unassigned consumer silently returns `[]` rather than raising — this is intentional, since a consumer may temporarily have no assignments during rebalancing.

The `acknowledge()` path never raises — it silently counts failures and routes to DLQ. If DLQ topic creation or publish fails, that would propagate up (but in practice can't fail since the broker controls both).

## Topics to Explore

- [file] `distributed-message-queue/test_solution.py` — How the delivery semantics (at-least-once, at-most-once, exactly-once), rebalancing, and DLQ are tested; reveals edge cases the implementation handles
- [function] `stock-exchange/solution.py:publish` — How the stock exchange reuses this message queue as an event bus for order matching, showing cross-module integration
- [general] `consumer-rebalance-strategies` — Kafka's range vs. round-robin vs. sticky assignors compared to the simple modular assignment used here
- [file] `distributed-message-queue/plan.md` — The design decisions and tradeoffs documented before implementation; context for why certain simplifications were made
- [general] `exactly-once-semantics-tradeoffs` — How the unbounded `seen_message_ids` set compares to Kafka's idempotent producer + transactional consumer approach, and the memory implications

## Beliefs

- `dmq-partition-offset-is-logical` — Partition offsets are logical (not physical array indices); `base_offset` adjusts after trimming so consumers using stored offsets see consistent numbering across retention evictions
- `dmq-delivery-semantics-in-poll` — The three delivery modes (at_least_once, at_most_once, exactly_once) are enforced entirely within `poll()` and `commit()`, not in the publish path
- `dmq-one-topic-per-consumer-group` — Each ConsumerGroup is permanently bound to a single topic at creation; there is no mechanism for multi-topic subscription
- `dmq-dlq-is-regular-topic` — Dead-letter queues are standard topics with a `__dlq_` prefix, consumable through the same poll/commit API as any other topic
- `dmq-rebalance-on-every-membership-change` — Both `add_consumer` and `remove_consumer` trigger a full rebalance; there is no incremental or sticky assignment

