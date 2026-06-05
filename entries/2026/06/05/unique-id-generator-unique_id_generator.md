# File: unique-id-generator/unique_id_generator.py

**Date:** 2026-06-05
**Time:** 14:04

# `unique-id-generator/unique_id_generator.py`

## Purpose

This file implements a **unique ID generation system** — one of the classic system design interview topics. It owns the responsibility of producing globally unique identifiers using five distinct strategies, each modeling a real-world approach with different tradeoffs in sortability, size, coordination requirements, and throughput. It also provides a coordinator that distributes generation across multiple backends round-robin, simulating a multi-node deployment.

## Key Components

### Constants

- **`DEFAULT_EPOCH_MS`** — Custom epoch set to 2024-01-01T00:00:00Z. Used by Snowflake to maximize the 41-bit timestamp range by measuring time relative to a recent date rather than Unix epoch (1970). This gives ~69 years of headroom from 2024.
- **`CROCKFORD_ALPHABET`** — The 32-character alphabet for Crockford Base32 encoding, which omits visually ambiguous characters (I, L, O, U). Used exclusively by `ULIDGenerator`.

### Abstract Base

**`IDGenerator`** — Defines the contract: every generator must implement `generate() -> int | str`. Provides a default `generate_batch(n)` that calls `generate()` in a loop. The return type is a union because some strategies produce integers (Snowflake, Ticket) and others produce strings (UUID, Flake, ULID).

### Generators

| Class | Output | Bits | Sortable? | Coordination |
|-------|--------|------|-----------|-------------|
| `UUIDGenerator` | string (v4) | 128 | No | None |
| `SnowflakeGenerator` | int | 64 | Yes (time) | datacenter_id + worker_id |
| `TicketServerGenerator` | int | unbounded | Yes (order) | step + offset |
| `FlakeIDGenerator` | hex string | 128 | Yes (time) | worker_id |
| `ULIDGenerator` | Base32 string | 128 | Yes (time) | None |

**`UUIDGenerator`** — Thin wrapper around `uuid.uuid4()`. No state, no coordination, no ordering. The simplest strategy but produces non-sortable, large IDs.

**`SnowflakeGenerator`** — The centerpiece. Packs a 64-bit ID from four fields: 41-bit timestamp delta, 5-bit datacenter, 5-bit worker, 12-bit sequence. The sequence counter allows up to 4096 IDs per millisecond per worker before the generator spin-waits for the next millisecond via `_wait_next_ms`. Accepts an injectable `clock_fn` for testing.

**`TicketServerGenerator`** — Models a database auto-increment approach. Configured with `step` and `offset` so multiple ticket servers can interleave (e.g., server A: 1, 3, 5; server B: 2, 4, 6). The constructor initializes `_counter = offset - step` so the first `generate()` returns exactly `offset`.

**`FlakeIDGenerator`** — Produces 128-bit IDs as 32-character hex strings. Bit layout: 64-bit timestamp + 48-bit worker + 16-bit sequence. The wider fields compared to Snowflake give more worker capacity (48 bits vs 10) and the same spin-wait overflow behavior for sequences.

**`ULIDGenerator`** — Produces 26-character Crockford Base32 strings. 48-bit millisecond timestamp + 80-bit random component. Enforces **monotonicity within the same millisecond** by incrementing the random part rather than generating fresh randomness — this is a key ULID spec requirement that preserves sort order.

### Coordinator

**`IDGeneratorCoordinator`** — Takes a list of generators and dispatches `generate()` calls round-robin. Models the system design pattern of distributing load across multiple ID-generation nodes. The modulo index wraps naturally; no explicit reset needed.

## Patterns

1. **Strategy Pattern** — `IDGenerator` ABC with five concrete implementations. The coordinator accepts any `IDGenerator`, enabling mix-and-match.
2. **Dependency Injection** — `clock_fn` parameter on time-dependent generators (`Snowflake`, `Flake`, `ULID`) allows deterministic testing without mocking `time.time`.
3. **Thread Safety** — Every stateful generator uses `threading.Lock()` around its mutable state (`_sequence`, `_counter`, `_last_timestamp_ms`). The coordinator also locks its round-robin index.
4. **Spin-Wait for Overflow** — When the per-millisecond sequence counter overflows, Snowflake and Flake generators busy-wait until the clock advances. This trades latency for correctness (no duplicate IDs).

## Dependencies

**Imports**: All stdlib — `uuid`, `time`, `random`, `threading`, `abc`, `datetime`, `typing`. No external dependencies. `Optional` is imported but unused.

**Imported by**: `test_unique_id_generator.py` — the test suite is the only consumer. This is a self-contained teaching implementation.

## Flow

A typical Snowflake generation:

1. Acquire lock
2. Read current time via `clock_fn`, convert to milliseconds
3. **Guard**: reject if clock is before epoch or moved backward (raises `RuntimeError`)
4. If same millisecond as last call → increment sequence; if sequence overflows 4095 → spin-wait to next ms
5. If new millisecond → reset sequence to 0
6. Store timestamp, bitwise-pack all fields into a single 64-bit int
7. Release lock, return ID

For ULID, the flow differs at step 4: instead of a sequence counter, it increments the random component to maintain monotonicity within the same millisecond.

## Invariants

- **Snowflake datacenter_id and worker_id** must be in [0, 31] — validated in `__init__`, enforced by 5-bit masks in `generate()`.
- **Snowflake IDs are positive 63-bit integers** — the sign bit is always 0 (validated by `validate()`).
- **No backward clock** — Snowflake raises `RuntimeError` if the clock moves backward. Flake and ULID silently handle it (Flake spin-waits, ULID increments random).
- **Monotonicity** — Within a single generator instance and millisecond, IDs are strictly increasing for Snowflake (sequence increments), Flake (sequence increments), and ULID (random part increments).
- **Ticket server first value** — Always equals `offset`, guaranteed by `_counter = offset - step` initialization.
- **ULID length** — Always exactly 26 characters: 10 (timestamp) + 16 (random).

## Error Handling

Error handling is minimal and fail-fast:

- **`SnowflakeGenerator.__init__`** — Raises `ValueError` for out-of-range datacenter/worker IDs. This is the only input validation.
- **`SnowflakeGenerator.generate`** — Raises `RuntimeError` for two clock anomalies: clock before epoch, and clock moving backward. These are unrecoverable — the caller must handle them.
- **All other generators** — No explicit error handling. `FlakeIDGenerator` and `ULIDGenerator` silently absorb backward clock movement by spin-waiting or incrementing. `TicketServerGenerator` and `UUIDGenerator` cannot fail under normal conditions.
- **No error wrapping or logging** — Exceptions propagate directly to the caller.

## Topics to Explore

- [file] `unique-id-generator/test_unique_id_generator.py` — See how clock injection is used to test Snowflake edge cases (overflow, backward clock) and how monotonicity is verified across generators
- [function] `unique-id-generator/unique_id_generator.py:SnowflakeGenerator.parse` — Round-trip decomposition of Snowflake IDs back to components; useful for understanding the bit-packing layout
- [general] `snowflake-vs-ulid-tradeoffs` — Compare the 64-bit compact sortable ID (Snowflake) against the 128-bit lexicographically sortable ID (ULID) in terms of database indexing, storage, and coordination requirements
- [file] `consistent-hashing/consistent_hashing.py` — Another distributed systems primitive in the repo; interesting to compare how worker/node identity is handled across different system designs
- [general] `ticket-server-multi-node` — How step/offset interleaving achieves uniqueness across N ticket servers without coordination, and the failure modes when a server goes down and comes back with stale state

## Beliefs

- `snowflake-sequence-max-4096-per-ms` — SnowflakeGenerator allows at most 4096 IDs per millisecond per worker (12-bit sequence 0-4095), then spin-waits for the next millisecond
- `snowflake-rejects-backward-clock` — SnowflakeGenerator raises RuntimeError on clock regression, while FlakeIDGenerator and ULIDGenerator silently handle it
- `ulid-monotonic-within-millisecond` — ULIDGenerator maintains sort order within the same millisecond by incrementing the random component rather than generating new random bits
- `all-stateful-generators-thread-safe` — Every generator with mutable state (Snowflake, Ticket, Flake, ULID, Coordinator) protects it with a threading.Lock
- `ticket-server-first-value-equals-offset` — TicketServerGenerator initializes its counter to `offset - step` so the first generate() call returns exactly `offset`

