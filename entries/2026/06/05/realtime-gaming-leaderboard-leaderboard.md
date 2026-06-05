# File: realtime-gaming-leaderboard/leaderboard.py

**Date:** 2026-06-05
**Time:** 14:00

# `realtime-gaming-leaderboard/leaderboard.py`

## Purpose

This file implements a real-time gaming leaderboard — the kind you'd design in a system design interview when asked "How would you build a leaderboard that supports millions of players with instant rank lookups?" It owns all core leaderboard operations: score updates, rank lookups, top-K/bottom-K queries, neighborhood queries ("around me"), score range queries, percentile computation, and per-player score history.

The implementation is a **single-node, in-memory** solution. It doesn't address distribution, persistence, or replication — those are architectural concerns above this layer. What it does demonstrate is the right data structure choice and API surface for the problem.

## Key Components

### `Leaderboard`

The core class. Maintains three parallel data structures that must stay in sync:

| Field | Type | Role |
|-------|------|------|
| `_entries` | `SortedList` of `(-score, timestamp, player_id)` | Ordered sequence for rank operations |
| `_players` | `dict[str, (score, timestamp)]` | O(1) lookup of current score/timestamp by player ID |
| `_history` | `defaultdict[str, deque]` | Bounded score-change log per player |

**Key methods and their contracts:**

- **`update_score(player_id, score, timestamp=None) -> dict`** — Sets (not increments) a player's score. If the player already exists, removes their old entry from `_entries` first. Returns `{rank, previous_score, new_score}`. This is the single mutation path — `increment_score` delegates to it.

- **`increment_score(player_id, delta, timestamp=None) -> dict`** — Additive wrapper around `update_score`. Defaults to 0.0 if the player doesn't exist yet. Returns `{rank, new_score}`.

- **`get_rank(player_id) -> int | None`** — 1-based rank via `SortedList.index()`. Returns `None` for unknown players.

- **`top_k(k) / bottom_k(k)`** — Slice the sorted list from the front/back. O(k).

- **`around_me(player_id, count)`** — Returns `count` players above and below (plus the player), giving the "neighborhood" view common in gaming UIs.

- **`range_by_score(min_score, max_score)`** — Uses `SortedList.irange()` with negated bounds to find all players in a score window. Note: this calls `_entries.index(entry)` per result to compute rank, making it O(m log n) where m is the result count.

- **`percentile(player_id) -> float | None`** — `(total - rank) / total * 100`. The rank-1 player in a 100-player board gets 99.0, not 100.0.

- **`get_history(player_id, limit)`** — Returns the last `limit` entries from the bounded deque.

- **`remove_player(player_id) -> bool`** — Removes from all three data structures. Returns `False` if player didn't exist.

### `LeaderboardManager`

Thin registry of named `Leaderboard` instances. Supports the multi-board pattern (daily, weekly, seasonal leaderboards). No cross-board operations.

## Patterns

### Negated-score trick

The central design choice. `SortedList` sorts ascending, but leaderboards rank descending. Storing `(-score, timestamp, player_id)` makes the highest score sort first. The timestamp as the second tuple element is a tiebreaker — equal scores are ordered by who got there first (lower timestamp = higher rank). The player_id as the third element ensures uniqueness even with identical score+timestamp.

### Dual-index pattern

`_entries` gives O(log n) ordered operations; `_players` gives O(1) point lookups. Every mutation must update both. This is the classic tradeoff: you pay double the write cost to get fast reads on both access patterns.

### Update-by-remove-and-reinsert

`update_score` doesn't mutate entries in place. It removes the old entry from the sorted list, then inserts a new one. This is the correct approach with sorted containers — in-place mutation would violate sort order.

### Bounded history via `deque(maxlen=...)`

Score history is capped by `history_size` (default 10) using `deque`'s built-in eviction. No manual size management needed.

## Dependencies

**Imports:**
- `sortedcontainers.SortedList` — The load-bearing dependency. Provides a B-tree-backed sorted list with O(log n) add/remove/index. This is the data structure that makes the whole design work; without it you'd need a skip list, balanced BST, or Redis sorted set.
- `collections.defaultdict` / `deque` — Standard library, used for history tracking.
- `time` — Default timestamps when caller doesn't provide one.

**Imported by:**
- `test_leaderboard.py` — The external test suite (separate from the inline `__main__` tests).

## Flow

A typical write path through `update_score`:

1. Check if player exists in `_players` dict (O(1))
2. If yes, look up old `(score, ts)` and remove `(-old_score, old_ts, player_id)` from `_entries` (O(log n))
3. Insert new `(-score, timestamp, player_id)` into `_entries` (O(log n))
4. Update `_players[player_id]` to `(score, timestamp)` (O(1))
5. Append to `_history[player_id]` (O(1), deque handles eviction)
6. Compute rank via `_entries.index(entry)` (O(log n))
7. Return result dict

A typical read path through `get_rank`:

1. Look up `(score, ts)` from `_players` (O(1))
2. Reconstruct the exact tuple `(-score, ts, player_id)`
3. Call `_entries.index(tuple)` (O(log n))
4. Add 1 for 1-based ranking

## Invariants

- **Three-structure consistency**: `_entries`, `_players`, and `_history` must agree. Every player in `_players` has exactly one corresponding entry in `_entries`. `remove_player` and `reset` clean all three.
- **Negation invariant**: Scores in `_entries` are always negated. Every method that reads from `_entries` must negate back. `_entry_to_dict` handles this with `-entry[0]`.
- **Timestamp uniqueness assumption**: The tiebreaking scheme assumes no two updates for different players share the exact same `(score, timestamp)` pair. If they did, the player_id string comparison would still disambiguate, but rank ordering among tied players would be lexicographic by ID rather than chronological.
- **1-based ranking**: All public rank values are 1-based (`index + 1`). Internal operations use 0-based indices.
- **History bounded**: The deque maxlen is set at construction time and cannot be changed per-player.

## Error Handling

Minimal, by design. This is a data-structure-level implementation, not a service layer.

- **Missing players**: Methods return `None` (for `get_rank`, `get_score`, `percentile`) or empty list (for `around_me`) or `False` (for `remove_player`). No exceptions.
- **No input validation**: Negative scores, empty player IDs, or nonsensical ranges are not rejected. The caller is trusted.
- **`_entries.remove()` will raise `ValueError`** if the entry doesn't exist — but this only happens if the three-structure invariant is violated, which is a bug, not a user error.
- **`range_by_score`**: If `min_score > max_score`, the negated irange bounds flip and it returns an empty list silently.

## Topics to Explore

- [file] `realtime-gaming-leaderboard/test_leaderboard.py` — The external test suite; check whether it covers edge cases the inline tests miss (empty boards, single-player boards, concurrent-timestamp ties)
- [function] `realtime-gaming-leaderboard/leaderboard.py:range_by_score` — The irange bounds logic with negated scores is subtle and potentially has edge cases at boundary values; worth verifying with float edge cases
- [general] `sorted-set-vs-sortedlist` — How this compares to a Redis ZSET-based approach: SortedList gives the same O(log n) guarantees but is single-process; understanding when you'd swap this for Redis is the key scaling question
- [file] `realtime-gaming-leaderboard/plan.md` — The design decisions and tradeoffs documented before implementation; likely discusses distribution strategies this single-node implementation doesn't cover
- [general] `percentile-at-boundaries` — The percentile formula returns 0.0 for a single-player board regardless of rank; edge behavior worth examining if percentile accuracy matters at low player counts

## Beliefs

- `leaderboard-negated-score-ordering` — Scores are stored negated in `_entries` so that `SortedList`'s ascending order yields descending-score ranking; the tiebreaker is timestamp-ascending (earlier timestamp = higher rank)
- `leaderboard-dual-index-consistency` — Every mutation in `Leaderboard` must update both `_entries` (sorted list) and `_players` (dict) atomically; a desync between them is a correctness bug that will surface as `ValueError` on the next operation
- `leaderboard-rank-is-one-based` — All public-facing rank values are 1-based; internal `SortedList.index()` returns 0-based and every call site adds 1
- `leaderboard-range-query-is-O-m-log-n` — `range_by_score` calls `_entries.index(entry)` per matched result to compute rank, making it O(m log n) rather than the O(m + log n) achievable by tracking the start index and incrementing

