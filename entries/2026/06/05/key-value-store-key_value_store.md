# File: key-value-store/key_value_store.py

**Date:** 2026-06-05
**Time:** 13:16

# `key-value-store/key_value_store.py`

## Purpose

This file implements a **Dynamo-style distributed key-value store** as a single-process simulation. It models the core mechanisms from Amazon's Dynamo paper: consistent hashing with virtual nodes, quorum-based reads/writes (N/W/R), vector clocks for conflict detection, hinted handoff for availability during failures, Merkle trees for anti-entropy synchronization, and gossip-based failure detection. It's a teaching implementation — everything runs in-process with no networking — designed to demonstrate how these mechanisms compose into a coherent system.

## Key Components

### `VectorClock`
A logical clock mapping `node_id → counter`. Immutable-style API: `increment`, `merge`, and `prune` all return new instances. The partial order is defined by `dominates` (one clock is strictly ahead) and `concurrent_with` (neither dominates — a conflict). `prune(max_entries)` caps clock size by keeping only the highest-counter entries, which is Dynamo's approach to bounding clock growth at the cost of some precision.

### `VersionedValue`
A value tagged with its vector clock, wall-clock timestamp, and a tombstone flag. Tombstones are how deletes work in an eventually-consistent system — you can't just remove the key because replicas that haven't seen the delete would re-introduce it during anti-entropy.

### `MerkleTree`
A binary hash tree over sorted key-value pairs. Built from a `dict[str, str]` snapshot of a node's data. `find_differences` compares two trees — if root hashes match, there are zero differences; otherwise it falls back to a key-by-key comparison. The tree structure itself (stored in `_tree` keyed by `(depth, start)`) is used for construction but not for incremental diffing — the `find_differences` method is a brute-force fallback rather than a tree walk.

### `KVNode`
A single storage node. Holds a `store` mapping keys to **lists** of `VersionedValue` (multiple concurrent versions can coexist — this is the "siblings" concept from Dynamo). Key methods:

- **`local_put`**: Increments the vector clock, removes versions dominated by the new one, and appends if not itself dominated. This is the write path for coordinator-originated writes.
- **`local_put_raw`**: Accepts a pre-built `VersionedValue` without incrementing the clock. Used for replication, read-repair, and anti-entropy — operations that shouldn't advance causality.
- **`local_get`**: Returns a copy of all versions for a key (including concurrent ones).
- **`get_merkle_tree`**: Snapshots the node's data into a `MerkleTree` for anti-entropy comparison.
- **Gossip**: `heartbeat_tick` advances the local heartbeat counter; `receive_gossip` merges another node's heartbeat table using max-wins semantics.

### `HintedHandoff`
A buffer for writes destined for unavailable nodes. When a target node is down, the coordinator stores a hint. When the node recovers (`deliver_hints`), the buffered writes are replayed. This is a pure availability mechanism — it means the system can accept writes even when some replicas are down.

### `KVStore` (Coordinator)
The top-level coordinator that ties everything together. Constructor parameters `n`, `w`, `r` are the Dynamo quorum knobs:
- **N**: number of replicas for each key
- **W**: number of successful writes required before returning success
- **R**: number of successful reads required

Key methods:

- **`_get_preference_list`**: Walks the consistent hash ring clockwise from the key's hash position, collecting **distinct** physical nodes (skipping duplicate vnodes for the same node). Returns the ordered list of responsible nodes.
- **`put`**: Coordinator picks the first node in the preference list, increments the vector clock, replicates the write to N nodes. If a target is down, stores a hint and writes to the next available backup. Raises if fewer than W nodes acknowledge.
- **`get`**: Reads from R nodes, filters tombstones, removes dominated versions, deduplicates by vector clock, and performs **read repair** — pushing any versions that a responding node is missing back to it.
- **`delete`**: A put with `is_tombstone=True`. Same quorum logic.
- **`run_gossip_round`**: Each alive node ticks its heartbeat, then gossips to up to 2 random peers. Afterward, failure detection runs: nodes whose heartbeat age exceeds `suspect_timeout` become SUSPECT; beyond `down_timeout`, they become DOWN.
- **`run_anti_entropy`**: All-pairs Merkle tree comparison among alive nodes. For each differing key, versions are bidirectionally synced. This is the background consistency mechanism.
- **`deliver_hints`**: Replays buffered hints to a recovered node and clears them.

## Patterns

1. **Immutable value objects**: `VectorClock` operations return new instances, avoiding aliasing bugs in the distributed setting.
2. **Quorum intersection**: The W + R > N constraint (not enforced in code but assumed by callers) guarantees that reads and writes overlap on at least one node, ensuring consistency.
3. **Consistent hashing with vnodes**: 150 virtual nodes per physical node for balanced load distribution. The ring is rebuilt on `add_node`/`remove_node`.
4. **Tombstone-based deletes**: Deletes don't remove data; they write a tombstone marker that dominates the live value during reads.
5. **Read repair**: Opportunistic consistency fix during reads — stale replicas get updated as a side effect of the get path.
6. **Separation of coordinator vs. storage**: `KVStore` handles routing/quorum logic; `KVNode` handles local storage. `local_put` vs. `local_put_raw` cleanly separates "originating a write" from "accepting a replica write."

## Dependencies

**Imports**: Only stdlib — `hashlib` (consistent hashing + Merkle hashes), `random` (gossip target selection), `dataclasses`.

**Imported by**: `test_key_value_store.py` — the test suite drives all the scenarios (quorum failures, conflicts, anti-entropy, hinted handoff).

## Flow

### Write path
1. Client calls `KVStore.put(key, value, context)`
2. Coordinator computes preference list via consistent hash ring
3. First node in list is the coordinator — it increments the vector clock
4. Writes are sent to N target nodes via `local_put_raw`
5. If a target is DOWN, a hint is stored and the write goes to a backup node
6. If fewer than W nodes succeed, an exception is raised
7. Returns the new vector clock (client must pass this back as `context` on the next write to maintain causality)

### Read path
1. Client calls `KVStore.get(key)`
2. Coordinator reads from R nodes via `local_get`
3. All versions are collected, tombstones filtered, dominated versions removed
4. Deduplicated by vector clock — concurrent versions are returned as siblings
5. Read repair pushes missing versions to stale replicas
6. Returns list of `(value, vector_clock)` tuples — multiple entries means a conflict the client must resolve

### Failure detection
1. `run_gossip_round` advances heartbeats, gossips to random peers
2. Compares heartbeat timestamps against `suspect_timeout` and `down_timeout`
3. Nodes transition: ALIVE → SUSPECT → DOWN

## Invariants

- **`local_put` never stores dominated versions**: After a put, the key's version list contains only non-dominated entries. Two concurrent writes will both survive as siblings.
- **`local_put_raw` is idempotent for dominated writes**: If the incoming version is dominated by something already stored, it's silently dropped.
- **Preference list returns distinct nodes**: Despite 150 vnodes per node, `_get_preference_list` deduplicates by physical node ID.
- **Tombstones suppress live values**: `get` filters `is_tombstone=True` entries before returning — a deleted key returns an empty list.
- **Read repair only runs on deduped survivors**: It pushes only the final non-dominated, non-tombstone versions, so it doesn't re-introduce stale data.

## Error Handling

Errors are minimal and exception-based:
- **`put`**: Raises `Exception` if fewer than W nodes acknowledge the write (quorum failure).
- **`get`**: Raises `Exception` if fewer than R nodes are reachable.
- **`delete`**: Same quorum check as `put`.
- No retries, no timeouts on individual node operations (since it's a simulation with no real I/O).
- Silent degradation: if a gossip target is unreachable, it's simply not gossiped to. Hinted handoff writes to backup nodes don't count toward the W quorum — they're fire-and-forget.

## Topics to Explore

- [file] `key-value-store/test_key_value_store.py` — See how quorum failures, conflict resolution, anti-entropy, and hinted handoff are exercised in tests
- [function] `key-value-store/key_value_store.py:MerkleTree.find_differences` — The fallback to brute-force comparison means this doesn't get the O(log n) diffing benefit of a real Merkle tree; worth understanding the gap
- [file] `consistent-hashing/consistent_hashing.py` — Compare the standalone consistent hashing implementation with the ring built into KVStore
- [general] `dynamo-sloppy-quorum` — The backup-node write in `put` is a sloppy quorum (write goes to a node not in the preference list); trace how this interacts with `deliver_hints` to restore strict quorum membership
- [general] `vector-clock-pruning-tradeoffs` — `VectorClock.prune` discards low-counter entries, which can cause false concurrency detection; worth reasoning through when this matters

## Beliefs

- `kv-store-quorum-exception` — `put`, `get`, and `delete` raise `Exception` (not a custom type) when the quorum threshold (W or R) is not met
- `kv-node-stores-sibling-versions` — `KVNode.store` maps each key to a **list** of `VersionedValue`, not a single value; concurrent writes produce multiple siblings that the client must resolve
- `kv-delete-is-tombstone-write` — Deletes are implemented as a `put` of a `VersionedValue` with `is_tombstone=True` and `value=None`; no data is physically removed
- `kv-read-repair-on-get` — Every `get` performs read repair: versions missing from a responding node are pushed back to it via `local_put_raw`
- `kv-ring-150-vnodes` — Each physical node gets 150 virtual nodes on the consistent hash ring; the ring is fully rebuilt on every `add_node`/`remove_node` call

