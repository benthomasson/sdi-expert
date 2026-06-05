# File: search-autocomplete/search_autocomplete.py

**Date:** 2026-06-05
**Time:** 14:02

## Purpose

This file implements a **search autocomplete system** — the kind of typeahead you see in Google Search. It's a system design interview implementation that demonstrates how to serve prefix-based query suggestions ranked by frequency, with time-decay so stale queries lose relevance over time.

The file owns three responsibilities:
1. **Storage & retrieval** of queries via a trie with per-node top-k caches (`AutocompleteTrie`)
2. **Batched ingestion** of raw search queries (`QueryCollector`)
3. **Service-layer concerns** like blocklisting offensive terms and fuzzy matching typos (`AutocompleteService`)

## Key Components

### `TrieNode`

A dataclass representing a single node in the trie. Each node carries:
- `children` — character-keyed map to child nodes
- `is_end` / `frequency` — marks whether this node terminates a complete query and its search count
- `last_updated` — timestamp for time-decay calculations
- `top_k_cache` — **precomputed** list of the top-k most frequent completions reachable from this node. This is the key optimization: it turns a prefix lookup from O(subtree) into O(1).

### `AutocompleteTrie`

The core data structure. Constructor takes `k` (cache size, default 10) and `decay_factor` (hourly decay multiplier, default 0.99).

**Critical methods:**

- **`_walk(query)`** — Traverses the trie character-by-character, returning the full path of nodes visited and the terminal node (or `None`). Used by nearly every other method as the primitive traversal.

- **`_update_caches_on_path(path, query)`** — The cache maintenance routine. After any mutation (insert, increment, delete), this walks the path **bottom-up** and rebuilds each node's `top_k_cache` by merging all children's caches plus the node's own frequency. Sorting is by descending frequency, then lexicographic for ties.

- **`insert(query, frequency, timestamp)`** — Sets a query to an exact frequency. Lowercases and truncates to 200 chars. Tracks `_size` (distinct query count).

- **`increment(query, amount, timestamp)`** — Adds to existing frequency (or inserts at `amount` if new). This is the method `QueryCollector` calls — it models "another user searched for X."

- **`delete(query)`** — Soft-deletes by setting `is_end=False`, `frequency=0`. The trie nodes remain (no pruning), but the query disappears from all caches via `_update_caches_on_path`.

- **`search_prefix(prefix, k, current_time)`** — The read path. Without `current_time`, returns raw cached results directly. With `current_time`, applies exponential decay: `raw_freq * decay_factor^hours_elapsed`. This means the decay is **applied at read time**, not stored — raw frequencies stay clean.

- **`serialize()` / `deserialize()`** — JSON-compatible round-trip. `deserialize` rebuilds all caches bottom-up via `_rebuild_all`, which mirrors the logic in `_update_caches_on_path`.

### `QueryCollector`

A write-buffer that accumulates raw `(query, timestamp)` records and flushes them as aggregated increments. On `flush()`, it groups by lowercased query, sums counts, takes the max timestamp, then calls `trie.increment` per unique query. This models the real-world pattern of batching analytics events before updating the trie.

### `AutocompleteService`

The top-level API that composes the trie and collector. Adds:

- **Blocklist filtering** — `_filter` removes any result where a blocklisted term appears as a substring (not just exact match). When querying, it over-fetches by `len(blocklist) * 2` to compensate for filtered results.

- **`suggest()` / `suggest_with_scores()`** — Clean interface returning just query strings or `(query, score)` tuples.

- **`fuzzy_suggest()`** — Fallback when exact prefix yields nothing. Tries single-character edits on the last character: substitution, deletion, and insertion. Collects results from all variant prefixes, deduplicates, and returns top-k. This is intentionally limited to one edit distance on the last char — a pragmatic approximation, not full Levenshtein.

## Patterns

**Top-k cache per node** — This is the defining design choice. Rather than DFS-ing the subtree on every query, each node maintains a precomputed list of the best completions below it. The tradeoff: writes are more expensive (every insert/increment updates caches along the entire path from root to leaf), but reads are O(prefix_length) regardless of subtree size. This matches the real-world read-heavy workload of autocomplete.

**Write-time cache maintenance, read-time decay** — Frequencies are stored raw; decay is computed lazily during `search_prefix` only when `current_time` is provided. This avoids periodic background jobs to age out all entries and keeps the stored state clean.

**Normalization at the boundary** — Every public method lowercases and truncates queries to 200 chars immediately on entry. Internal methods can assume normalized input.

**Soft deletion** — `delete` zeroes out the node but doesn't prune the trie path. Simpler but means deleted queries leave structural residue.

## Dependencies

**Imports:** Only stdlib — `dataclasses` for `TrieNode`, `time` for timestamps. No external dependencies.

**Imported by:** `test_search_autocomplete.py` — the test suite exercises the trie, collector, and service.

## Flow

A typical lifecycle:

1. `AutocompleteService` is created with a k, decay factor, and optional blocklist.
2. Queries arrive via `record_query()`, which buffers in `QueryCollector` then immediately flushes (so it's effectively unbatched in this implementation — each `record_query` calls `flush()`).
3. `flush()` aggregates the buffer and calls `trie.increment()` for each unique query.
4. `increment()` walks the trie, creating nodes as needed, bumps the frequency, then calls `_update_caches_on_path` to propagate the change bottom-up through every ancestor's cache.
5. On a read, `suggest("fo")` calls `search_prefix`, walks to the node for "fo", and returns its `top_k_cache` (optionally with decay applied).
6. The service filters blocklisted terms and returns the final list.

## Invariants

- **Cache consistency**: After every mutation, `_update_caches_on_path` is called, so `top_k_cache` is always consistent with the current subtree state. There is no lazy invalidation or dirty flag.
- **Normalization**: All queries are lowercased and truncated to 200 characters before storage. The trie never contains mixed-case or oversized keys.
- **Size tracking**: `_size` counts distinct queries (nodes where `is_end=True`). It's incremented on first insert, decremented on delete — never from raw node counts.
- **Sorting**: Caches are sorted by `(-frequency, query)` — highest frequency first, ties broken lexicographically. This order is maintained at every cache rebuild.
- **The trie stores k=10 candidates internally** even if the service default is k=5 — the service constructor does `k=max(k, 10)` for the trie, giving headroom for blocklist filtering.

## Error Handling

Minimal — this is a data structure implementation, not a network service. Key behaviors:

- `delete()` returns `False` if the query doesn't exist; it doesn't raise.
- `get_frequency()` returns `0` for nonexistent queries.
- `search_prefix()` returns `[]` for unmatched prefixes.
- Negative decay hours (future timestamps) are clamped to 0 in `search_prefix`.
- No validation on `frequency` or `amount` being positive — negative values would silently corrupt state.
- `_walk` returns `(path, None)` on miss rather than raising, so callers must null-check.

## Topics to Explore

- [file] `search-autocomplete/test_search_autocomplete.py` — See what edge cases are tested (decay behavior, blocklist substring matching, fuzzy search, serialization round-trips)
- [function] `search-autocomplete/search_autocomplete.py:_update_caches_on_path` — The heart of write-path performance; trace through a multi-level insert to see how caches propagate bottom-up
- [general] `trie-pruning-on-delete` — Current delete is soft (nodes remain); explore whether structural pruning would improve memory for high-churn workloads
- [function] `search-autocomplete/search_autocomplete.py:fuzzy_suggest` — Limited to last-char edits; compare against full edit-distance approaches and understand the latency tradeoff
- [file] `search-autocomplete/plan.md` — Design rationale and interview-context decisions that shaped this implementation

## Beliefs

- `autocomplete-cache-consistency` — Every mutation (insert, increment, delete) immediately rebuilds `top_k_cache` for all ancestor nodes on the path; caches are never stale between operations
- `autocomplete-decay-is-read-time` — Time decay is computed at query time in `search_prefix`, not stored; raw frequencies in the trie are never modified by decay
- `autocomplete-blocklist-is-substring` — The blocklist filter removes results where any blocklisted term appears as a substring of the query, not just exact matches
- `autocomplete-delete-is-soft` — Deleting a query zeroes its frequency and unsets `is_end` but does not remove trie nodes from the tree structure
- `autocomplete-normalize-at-boundary` — All public methods lowercase and truncate queries to 200 characters before any trie operation

