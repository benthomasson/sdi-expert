# File: consistent-hashing/consistent_hashing.py

**Date:** 2026-06-05
**Time:** 13:15

## Purpose

This file implements a **consistent hashing ring** — the foundational data structure used in distributed systems to map keys to nodes with minimal redistribution when nodes join or leave. It's a standalone, interview-ready implementation that demonstrates the core algorithm: hash keys and nodes onto a circular `[0, 2^32)` space, walk clockwise to find the responsible node, and use virtual nodes to smooth out load distribution.

The file owns two responsibilities: the ring logic itself (`ConsistentHashRing`) and a text-based visualization (`HashRingVisualizer`) for inspecting how nodes partition the ring.

## Key Components

### `default_hash(key: str) -> int`

The hash function used unless overridden. Truncates MD5 to 32 bits by taking the first 4 bytes of the digest. This maps every key to the `[0, 2^32)` ring space. MD5 is chosen for uniform distribution, not cryptographic security — and truncating to 32 bits is fine for a hash ring since collisions at the virtual-node level are handled gracefully (duplicates are skipped in `add_node`).

### `ConsistentHashRing`

The core class. Internal state is maintained in three synchronized structures:

| Field | Type | Role |
|-------|------|------|
| `_sorted_positions` | `list[int]` | Sorted ring positions for binary search |
| `_position_to_node` | `dict[int, str]` | Maps each ring position to its physical node ID |
| `_node_positions` | `dict[str, list[int]]` | Maps each physical node to all its virtual node positions |

This triple-bookkeeping is the price of O(log n) lookups — `_sorted_positions` enables `bisect`, `_position_to_node` resolves which node owns a position, and `_node_positions` enables O(v) node removal where v is the virtual node count.

**Constructor** accepts `num_virtual_nodes` (default 150) and an optional custom hash function. 150 vnodes is a practical sweet spot — high enough for reasonable load balance across ~10 nodes, low enough that add/remove stays fast.

**`add_node(node_id, keys=None)`** — Generates `num_virtual_nodes` positions using the scheme `"{node_id}#{i}"`, inserts each into the sorted list via `bisect.insort`, and updates both lookup dicts. Skips positions that collide with existing ones (`if pos in self._position_to_node`). If a `keys` list is provided, returns which keys migrated to the new node — useful for orchestrating data transfer during scaling events.

**`remove_node(node_id, keys=None)`** — Identifies keys currently owned by the departing node *before* removing it, then strips all its positions from the ring, and finally reports where those keys land now. The ordering matters: it captures affected keys while the old node is still present, then removes it, then resolves new owners.

**`get_node(key)`** — The primary lookup. Binary-searches for the key's hash in `_sorted_positions`, wraps around to index 0 if past the last position (the ring is circular), and returns the owning physical node.

**`get_nodes(key, n)`** — Clockwise walk for replication. Starting from the key's position, walks the ring collecting *distinct physical nodes* (skipping virtual nodes belonging to already-seen physical nodes) until it has `n` or exhausts the ring. Caps `n` at the total number of physical nodes.

**`get_distribution(keys)`** — Counts how many of the provided keys map to each node. Useful for verifying balance.

**`get_stats()`** — Computes load standard deviation by measuring each physical node's *ring ownership* — the sum of arc lengths (gaps) that each node's virtual nodes are responsible for. This is a theoretical metric independent of actual key distribution.

### `HashRingVisualizer`

Static utility that renders a text bar of width `width` showing which node owns each segment of the ring. Assigns single-character labels (A, B, C...) to nodes. Useful for debugging and understanding how virtual nodes interleave on the ring.

## Patterns

**Sorted array + binary search** — Rather than a balanced BST or skip list, the ring uses a plain sorted list with `bisect.bisect_left` for O(log n) lookups and `bisect.insort` for O(n) insertion. This is appropriate because node additions/removals are infrequent compared to lookups, and the sorted list has excellent cache locality.

**Virtual nodes for load balancing** — Each physical node gets `num_virtual_nodes` positions spread across the ring, keyed as `"{node_id}#{i}"`. This prevents hotspots that occur when a small number of physical nodes cluster in one region of the hash space.

**Migration tracking via optional `keys` parameter** — Both `add_node` and `remove_node` accept an optional keys list and report which keys moved. This is a clean separation: the ring doesn't track keys itself (it's stateless w.r.t. data), but can compute migration impact when given a key set.

**Dependency injection for hash function** — The constructor accepts a custom `hash_fn`, enabling tests to use deterministic or adversarial hash functions.

## Dependencies

**Imports:**
- `bisect` — Binary search and sorted insertion into the position list
- `hashlib` — MD5 for the default hash function
- `math` — `sqrt` for load standard deviation in `get_stats`
- `typing.Callable` — Type annotation for the pluggable hash function

**Imported by:**
- `consistent-hashing/test_consistent_hashing.py` — Unit tests

No other modules in the repo import this; it's a self-contained implementation.

## Flow

**Key lookup (`get_node`):**
1. Hash the key → 32-bit integer `h`
2. `bisect_left` finds the insertion point in `_sorted_positions`
3. If past the end, wrap to index 0 (ring semantics)
4. Index into `_position_to_node` via the position at that index

**Node addition (`add_node`):**
1. Guard: if node already exists, return early
2. Snapshot current ownership of provided keys (if any)
3. Generate `num_virtual_nodes` positions, skip collisions, `insort` each
4. Record positions in `_node_positions`
5. Diff ownership: return keys whose owner changed to the new node

**Node removal (`remove_node`):**
1. Guard: if node doesn't exist, return early
2. Identify keys currently owned by this node (before removal)
3. For each position: delete from `_position_to_node`, binary-search + pop from `_sorted_positions`
4. Delete from `_node_positions`
5. Re-resolve affected keys to their new owners

## Invariants

- **`_sorted_positions` is always sorted** — maintained by exclusive use of `bisect.insort` for insertion and index-based `pop` for removal.
- **No duplicate positions** — `add_node` skips positions that collide with existing ones. This means a node may end up with fewer than `num_virtual_nodes` positions if collisions occur (unlikely with a 32-bit space but handled).
- **The three data structures are always consistent** — every position in `_sorted_positions` has an entry in `_position_to_node`, and every node in `_node_positions` has its positions in `_sorted_positions`.
- **Ring wrap-around** — `_get_position` wraps index to 0 when the hash exceeds the largest position, enforcing circularity.
- **`get_nodes` returns at most `min(n, num_physical_nodes)` nodes** — you can't replicate to more nodes than exist.

## Error Handling

Minimal and appropriate for a data-structure library:

- **`get_node` and `get_nodes`** raise `ValueError("No nodes in the ring")` when called on an empty ring. This is the only explicit error.
- **`add_node` on a duplicate** silently returns `[]` — idempotent.
- **`remove_node` on a missing node** silently returns `{}` — idempotent.
- **Hash collisions** are silently skipped in `add_node` (the `if pos in self._position_to_node: continue` guard).
- **No input validation** on key types or node IDs — the hash function will raise if given something that can't `.encode()`.

## Topics to Explore

- [file] `consistent-hashing/test_consistent_hashing.py` — See how balance, migration, and replication are verified; the test patterns reveal the intended contract more precisely than the implementation alone
- [file] `consistent-hashing/plan.md` — Understand the design decisions and trade-offs considered before implementation (e.g., why 150 vnodes, why MD5)
- [general] `ring-ownership-vs-key-distribution` — `get_stats` measures theoretical ring arc ownership while `get_distribution` measures empirical key counts; understanding when these diverge matters for capacity planning
- [function] `consistent-hashing/consistent_hashing.py:get_nodes` — The clockwise walk for replication is where virtual-node-aware deduplication happens; worth tracing with a small ring to understand edge cases (e.g., fewer physical nodes than requested replicas)
- [general] `bounded-loads-extension` — Google's "consistent hashing with bounded loads" adds a capacity cap per node to prevent hotspots even when the ring is balanced; a natural next step for this implementation

## Beliefs

- `ch-vnode-naming-scheme` — Virtual nodes are keyed as `"{node_id}#{i}"` for `i` in `range(num_virtual_nodes)`, so the hash distribution depends entirely on the hash function's behavior on these strings
- `ch-default-hash-32bit` — The default hash function truncates MD5 to 32 bits (first 4 bytes), mapping all positions to the `[0, 2^32)` ring space
- `ch-add-remove-idempotent` — `add_node` on an existing node and `remove_node` on a missing node are both no-ops returning empty results, making the ring safe against duplicate operations
- `ch-get-nodes-deduplicates-physical` — `get_nodes` walks clockwise and skips virtual nodes belonging to already-collected physical nodes, guaranteeing distinct physical nodes for replication
- `ch-stats-measures-arc-ownership` — `get_stats` computes load standard deviation from ring arc length ownership per physical node, not from actual key counts — it's a theoretical balance metric

