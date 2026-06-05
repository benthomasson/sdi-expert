# File: web-crawler/web_crawler.py

**Date:** 2026-06-05
**Time:** 13:26

# `web-crawler/web_crawler.py`

## Purpose

This file is a self-contained simulation of a web crawler, built to demonstrate the core data structures and algorithms you'd discuss in a system design interview. It doesn't hit the real internet — instead it crawls a `SimulatedWeb` (an in-memory page graph), which lets the code focus on the interesting parts: URL normalization, deduplication via Bloom filters, near-duplicate content detection via SimHash, politeness scheduling, robots.txt compliance, and BFS/DFS traversal strategies.

It owns the entire crawl lifecycle: seeding the frontier, fetching pages, extracting links, filtering duplicates, respecting robots rules, and collecting stats.

## Key Components

### `WebPage` (dataclass)
A node in the simulated web graph. Holds `url`, `content`, outbound `links`, and a `status_code`. The `links` field represents structured outbound edges, separate from links embedded in `content` (which are extracted via regex).

### `URLNormalizer`
Stateless canonicalization. `normalize()` lowercases scheme and host, strips default ports (80/443), removes trailing slashes, sorts query parameters, and drops fragments. This is the single definition of "same URL" used everywhere — the Bloom filter, the `SimulatedWeb` lookup, and the frontier all normalize before storing or comparing.

### `SimulatedWeb`
The fake internet. A dictionary from normalized URL → `WebPage`, plus a per-domain `robots.txt` store. `fetch()` and `get_robots_txt()` are the only two I/O operations the crawler uses, making it trivial to swap in real HTTP later.

### `BloomFilter`
A textbook probabilistic set for URL deduplication. Sized using the optimal formulas: bit array length `m = -n·ln(p) / (ln2)²`, hash count `k = (m/n)·ln2`. Uses double hashing from a single MD5 digest — splits the 128-bit output into two 64-bit halves (`h1`, `h2`) and derives `k` positions via `(h1 + i·h2) % m`. The bit array is a compact `bytearray` with manual bit-level get/set operations.

### `SimHash`
64-bit fingerprinting for content near-duplicate detection. `compute()` tokenizes text, hashes each token, and accumulates a 64-dimensional sign vector — positive dimensions become 1-bits in the fingerprint. `hamming_distance()` XORs two fingerprints and counts differing bits. `are_near_duplicates()` applies a configurable threshold (default 3 bits), meaning two pages are considered duplicates if their SimHash values differ in at most 3 of 64 bit positions.

### `RobotsParser`
Parses `robots.txt` into per-user-agent rule lists. `is_allowed()` implements longest-prefix-match semantics: among all matching rules, the one with the longest path wins. Also extracts `crawl-delay` directives. Falls back from a specific user-agent to `*` if no agent-specific rules exist.

### `URLFrontier`
A priority queue backed by `heapq` with politeness enforcement. The `strategy` parameter switches between BFS (FIFO via ascending sequence numbers) and DFS (LIFO via negated sequence numbers in the min-heap). `get_next()` skips URLs whose host was accessed too recently, deferring them back into the heap — this prevents hammering a single domain.

### `Crawler`
The orchestrator. Wires together all the above components and runs the main crawl loop. Configurable via `strategy`, `max_pages`, `max_depth`, `politeness_delay`, `duplicate_threshold`, and `user_agent`. Returns a `CrawlStats` dataclass summarizing what happened.

## Patterns

**Simulation over real I/O.** The entire web is in-memory, and the crawl loop uses a simulated clock (`sim_time`) rather than real `time.sleep()`. This makes the code fast, deterministic, and testable — the real wall-clock `time.time()` is only used to measure total crawl duration for stats.

**Strategy pattern via configuration.** BFS vs DFS is controlled by flipping the sign of the sequence counter in the heap, not by swapping out data structures. Clean and minimal.

**Layered deduplication.** Two independent filters: the Bloom filter catches exact-URL revisits (cheap, O(k) per check), and SimHash catches near-duplicate content (linear scan over seen hashes, but only reached after the Bloom filter passes). This mirrors real crawler architectures where you want fast URL-level dedup before expensive content-level dedup.

**Normalize-once convention.** URLs are normalized at the point of insertion (into the Bloom filter, frontier, and SimulatedWeb). Downstream code can assume URLs are already canonical.

## Dependencies

**Imports:** All stdlib — `hashlib` (MD5 for Bloom and SimHash hashing), `heapq` (priority queue), `math` (log for optimal Bloom sizing), `re` (link extraction and tokenization), `time` (wall-clock duration), `dataclasses`, `typing`, and `urllib.parse` (URL manipulation).

**Imported by:** `test_web_crawler.py` — the test suite is the only consumer.

No external dependencies. This is intentionally self-contained for interview demonstration purposes.

## Flow

1. **Seed.** `crawl()` normalizes each seed URL, adds it to the Bloom filter (marking it as "seen"), and pushes it into the frontier at depth 0.

2. **Main loop.** Runs while the frontier is non-empty and `pages_crawled < max_pages`.

3. **Dequeue.** `frontier.get_next(sim_time)` returns the highest-priority URL whose host isn't rate-limited. If all hosts are throttled, `sim_time` advances by `politeness_delay` and the loop retries.

4. **Robots check.** `_is_allowed()` lazily fetches and caches the `RobotsParser` for the URL's domain. If disallowed, the URL is skipped and counted.

5. **Fetch.** `SimulatedWeb.fetch()` returns the `WebPage` or `None` (page doesn't exist → silently skipped).

6. **Content dedup.** `_is_duplicate_content()` computes the SimHash and compares it against all previously seen hashes. If it's a near-duplicate, the page is skipped.

7. **Record.** The URL is appended to `_crawled_urls`, and the page's links (both from `page.links` and extracted from `page.content` via regex) are deduplicated and stored in `_page_graph`.

8. **Expand.** If `depth < max_depth`, each outbound link is normalized, checked against the Bloom filter, and if not seen, added to both the Bloom filter and the frontier at `depth + 1`.

9. **Stats.** After the loop, `crawl_duration` is set from wall-clock time and the `CrawlStats` object is returned.

## Invariants

- **Bloom before frontier.** A URL is added to the Bloom filter *before* it enters the frontier. This means the frontier never contains duplicate URLs (assuming no false negatives, which Bloom filters guarantee).
- **Normalize before compare.** Every URL passes through `URLNormalizer.normalize()` before being stored or looked up anywhere.
- **Politeness is per-host.** `URLFrontier.get_next()` enforces a minimum interval between consecutive fetches to the same hostname. The simulated clock ensures this is deterministic.
- **Depth monotonically increases.** Children are always enqueued at `depth + 1`. There's no mechanism to revisit a URL at a shallower depth.
- **SimHash list grows monotonically.** `_seen_hashes` only appends. A page that passes the content-dedup check has its hash permanently recorded.

## Error Handling

Minimal, by design. Missing pages (`fetch()` returns `None`) are silently skipped. Invalid `crawl-delay` values in robots.txt are caught with a bare `try/except ValueError` and ignored. There's no retry logic, no timeout handling, and no exception propagation — the crawl loop always runs to completion or until `max_pages` is hit. This is appropriate for a simulation but would need significant hardening for production use.

One subtle quirk: the `BloomFilter.add()` method calls `might_contain()` first, then compensates for the side effect on `_negative_check_count` by decrementing it. This coupling between `add` and `might_contain`'s bookkeeping is fragile — if the tracking logic in `might_contain` changes, `add` breaks silently.

## Topics to Explore

- [file] `web-crawler/test_web_crawler.py` — How each component is exercised in isolation and integration; reveals the expected contracts and edge cases
- [function] `web-crawler/web_crawler.py:BloomFilter._get_hashes` — Double hashing from a single MD5 digest; worth understanding why this approximates k independent hash functions
- [general] `simhash-vs-minhash` — SimHash detects near-duplicate *documents*; MinHash detects near-duplicate *sets* — understanding when to use which is a common interview topic
- [function] `web-crawler/web_crawler.py:URLFrontier.get_next` — The deferred-requeue mechanism for politeness; consider how this behaves under heavy load with many hosts vs. few hosts
- [general] `bloom-filter-false-positive-tradeoff` — The crawler uses the Bloom filter as a *gatekeeper* for the frontier; a false positive means a URL is never crawled — worth reasoning about acceptable FP rates in practice

## Beliefs

- `bloom-filter-prevents-frontier-duplicates` — Every URL is added to the Bloom filter before being pushed into the frontier, so the frontier never enqueues a URL that has already been seen (modulo false positives, which cause URLs to be skipped)
- `simhash-threshold-default-3` — Two pages are considered near-duplicates if their 64-bit SimHash fingerprints differ in 3 or fewer bits, configurable via `duplicate_threshold`
- `url-frontier-strategy-via-sequence-sign` — BFS vs DFS is implemented by flipping the sign of the sequence number in the min-heap, not by changing data structures
- `robots-longest-prefix-match` — `RobotsParser.is_allowed` resolves conflicting allow/disallow rules by selecting the rule with the longest matching path prefix
- `crawl-uses-simulated-clock` — The crawl loop advances a `sim_time` counter for politeness scheduling rather than calling `time.sleep()`, making the simulation deterministic and fast

