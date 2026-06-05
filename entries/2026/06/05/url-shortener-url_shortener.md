# File: url-shortener/url_shortener.py

**Date:** 2026-06-05
**Time:** 14:05

# `url-shortener/url_shortener.py`

## Purpose

This file is a self-contained, in-memory URL shortening service — the kind you'd sketch on a whiteboard in a system design interview and then implement as a working prototype. It owns the full lifecycle of a shortened URL: creation (with two generation strategies), redirection with click tracking, analytics aggregation, expiration, and deletion. Everything lives in a single process with no external dependencies — the storage is a Python dict, not a database.

## Key Components

### `Base62`
A stateless utility for encoding integers into URL-safe strings using `[0-9a-zA-Z]`. The `encode`/`decode` pair is bijective — every positive integer maps to a unique string and back. This is the alphabet that turns a monotonic counter or hash digest into a short code like `0000001` or `a3Bf9x2`.

### `URLEntry`
A dataclass holding everything known about a single shortened URL: the mapping itself (`short_code` → `long_url`), metadata (creation time, expiration, creator), and analytics state (`click_count`, `click_history`). The `click_history` list is bounded to the last 1000 events — an implicit design decision that trades completeness for memory.

### `AnalyticsReport`
A read-only view computed on demand from a `URLEntry`. Aggregates clicks per day, recent clicks (last 10), and referrer frequency. This is a projection, not stored state — it's recomputed on every `get_analytics` call.

### `URLShortener`
The service itself. Constructor parameters control the domain prefix, default TTL, short code length, generation strategy (`"counter"` vs `"hash"`), and rate limit threshold. The two strategies reflect a real design tradeoff:

- **Counter** (`_generate_counter_code`): Monotonically incrementing integer → base62. Guaranteed unique on first attempt. Predictable (sequential), which leaks information about creation order and total URL count.
- **Hash** (`_generate_hash_code`): SHA-256 of the long URL, truncated to `code_length` base62 characters. Deterministic for a given URL (same input → same hash), but collisions are possible since we're truncating a 256-bit hash to ~41 bits (7 base62 chars). Handles collisions by appending a null-byte-separated attempt counter and rehashing.

## Patterns

**Strategy pattern** for code generation — the `strategy` field selects between counter and hash at construction time, with dispatch in `shorten()` via a simple if/else. No formal interface; it's a lightweight variant.

**Injectable time** — every method that touches timestamps accepts an optional `current_time` parameter. This avoids mocking `time.time()` in tests and makes the service fully deterministic when you control the clock. This is a common pattern in SDI implementations where you need to test expiration and rate limiting.

**Sliding window rate limiting** — `_check_rate_limit` keeps a list of timestamps per creator, prunes entries older than 60 seconds, and rejects if the window is full. This is a fixed-window approximation (the window slides on each call, but the pruning is eager, not lazy).

**Bounded history** — `click_history` is capped at 1000 entries with tail retention (`[-1000:]`). This prevents unbounded memory growth per URL but means early click data is silently dropped.

## Dependencies

**Imports**: All stdlib — `hashlib` for SHA-256 in hash strategy, `time` for wall-clock defaults, `collections.defaultdict` for sparse counters, `dataclasses` for structured data, `datetime` for UTC day formatting in analytics, `urllib.parse` for URL validation.

**Imported by**: `test_url_shortener.py` — the test suite is the only consumer. This module has no downstream dependents in the repo.

## Flow

A typical lifecycle:

1. **`shorten(long_url)`** — Validates the URL (scheme + domain), checks rate limits if a `creator_id` is provided, generates a short code via the configured strategy, wraps it in a `URLEntry`, stores it in `_urls[code]`, and returns the full short URL string (`domain/code`).

2. **`redirect(short_code)`** — Looks up the code in `_urls`, checks expiration against the current time, increments the click counter, appends a click event (with timestamp and optional referrer), trims history if it exceeds 1000 entries, and returns the original long URL. Returns `None` for missing or expired entries — no exception, no distinction between "never existed" and "expired."

3. **`get_analytics(short_code)`** — Iterates the click history to compute per-day counts and referrer frequencies. Returns `None` for unknown codes.

## Invariants

- **URL validation**: Only `http` and `https` schemes are accepted. The domain must contain a dot (rejects `localhost`, bare hostnames).
- **Custom alias constraints**: Must be 4–16 characters, alphanumeric only. Uniqueness is enforced — duplicate aliases raise `ValueError`.
- **Rate limit**: At most `rate_limit` (default 60) shortening operations per creator per 60-second sliding window.
- **Counter monotonicity**: The counter only increments. Codes from the counter strategy are zero-padded to `code_length` characters, so `_counter=1` → `"0000001"`.
- **Hash collision resolution**: The hash strategy will loop indefinitely until it finds an unused code. In practice, with 62^7 ≈ 3.5 trillion possible codes, this terminates quickly unless the keyspace is near-saturated.
- **Expiration is checked on read, not enforced eagerly**: Expired entries stay in `_urls` until accessed via `redirect` or filtered out by `list_urls`. No background reaper.

## Error Handling

All validation errors raise `ValueError` with descriptive messages — invalid URLs, bad custom aliases, rate limit violations, duplicate aliases. These are the only exceptions the module produces.

`redirect` and `get_analytics` return `None` for missing/expired entries instead of raising — the caller is expected to check. `delete` returns a boolean. There's no error type hierarchy or custom exceptions; `ValueError` is the single error channel.

Notably, there's no error handling for the hash collision loop — if the keyspace were exhausted, `_generate_hash_code` would loop forever. This is fine for an SDI prototype but would need a max-attempts guard in production.

## Topics to Explore

- [file] `url-shortener/test_url_shortener.py` — See how expiration, rate limiting, and both strategies are exercised in tests
- [function] `url-shortener/url_shortener.py:_generate_hash_code` — Understand the collision resolution loop and why SHA-256 truncation makes collisions plausible at ~3.5T codes
- [file] `url-shortener/plan.md` — Design decisions and tradeoffs documented before implementation
- [general] `counter-vs-hash-tradeoffs` — When predictability (counter) vs determinism (hash) matters in URL shortening — hash gives you deduplication for free, counter gives you guaranteed O(1) generation
- [file] `unique-id-generator/unique_id_generator.py` — Compare ID generation strategies — the unique ID generator likely tackles similar encoding/distribution problems in a distributed context

## Beliefs

- `url-shortener-two-strategies` — URLShortener supports two short code generation strategies: "counter" (monotonic base62) and "hash" (truncated SHA-256 with collision retry), selected at construction time
- `url-shortener-expiration-lazy` — Expired URLs are never eagerly removed from storage; expiration is checked at read time in `redirect` and `list_urls`
- `url-shortener-click-history-bounded` — Click history per URL is capped at 1000 entries; older events are silently dropped on each redirect
- `url-shortener-rate-limit-sliding-window` — Rate limiting uses a per-creator sliding window of 60 seconds, pruned eagerly on each `_check_rate_limit` call
- `url-shortener-no-dedup-counter` — The counter strategy does not deduplicate: shortening the same long URL twice produces two different short codes

