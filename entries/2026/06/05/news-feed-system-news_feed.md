# File: news-feed-system/news_feed.py

**Date:** 2026-06-05
**Time:** 13:23

# `news-feed-system/news_feed.py`

## Purpose

This file implements the core of a **News Feed System** — the kind you'd design in a system design interview (think Twitter/Facebook feed). It owns the entire feed lifecycle: social graph management, post creation, feed generation, and ranking. Its central design question is the **fan-out strategy**: how and when posts get distributed to followers' feeds.

## Key Components

### Data Models

- **`Post`** — The primary content unit. Carries engagement counters (`likes_count`, `comments_count`) and a `liked_by` set for idempotent like tracking. `created_at` is a float timestamp used both for ordering and as a cursor for pagination.
- **`Comment`** — Attached to a post by `post_id`. Lightweight — only increments the parent post's `comments_count`.
- **`FeedItem`** — A `(Post, score)` pair. The score is computed at read time and determines feed ordering.

### Enums

- **`FeedStrategy`** — The three classic fan-out strategies: `FAN_OUT_ON_WRITE` (push), `FAN_OUT_ON_READ` (pull), `HYBRID`.
- **`RankingMode`** — `CHRONOLOGICAL` (score = timestamp) or `RELEVANCE` (timestamp + weighted engagement).

### `SocialGraph`

Manages bidirectional follow relationships with two mirrored dicts (`_followers`, `_following`). All operations are O(1) via sets. Returns *copies* from `get_followers`/`get_following` to prevent external mutation.

### `PostStore`

Owns post persistence and retrieval. Maintains two indices:
- `_posts`: primary key lookup by `post_id`
- `_user_posts`: per-author ordered list for timeline queries

`get_user_posts` sorts on every call (no pre-sorted index), which is fine for an interview implementation but worth noting. `like_post` is idempotent — returns `False` if the user already liked.

### `NewsFeedService`

The orchestrator. Configurable at construction time with strategy, ranking mode, celebrity threshold, and cache size.

**`_feed_cache`**: A `dict[str, deque[str]]` mapping user IDs to bounded deques of post IDs. The `maxlen` on the deque enforces the cache size limit — oldest entries are silently evicted when the deque is full. This is only used by push and hybrid strategies.

**Key methods:**

| Method | Contract |
|--------|----------|
| `create_post` | Creates post, conditionally fans out based on strategy |
| `get_feed` | Retrieves, ranks, paginates feed for a user |
| `follow` | Updates graph + backfills cache for push strategies |
| `unfollow` | Updates graph + purges author's posts from cache |
| `_score` | Computes ranking score: either raw timestamp or timestamp + log-engagement |

## Patterns

**Strategy pattern** — The `FeedStrategy` enum selects between three distinct feed-generation paths (`_get_feed_push`, `_get_feed_pull`, `_get_feed_hybrid`). The strategy is chosen at construction and affects both write path (`create_post`) and read path (`get_feed`).

**Celebrity/hotspot optimization** — The hybrid strategy uses `celebrity_threshold` (default 1000 followers) to split users into two tiers. Non-celebrities get push fan-out (write-time distribution). Celebrities skip fan-out; their posts are pulled at read time. This mirrors the real-world Twitter/Instagram approach to avoid O(millions) writes when a celebrity posts.

**K-way merge** — `_get_feed_pull` uses `heapq.merge` to merge pre-sorted per-user timelines into a single stream without materializing everything first. The key is negated (`-p.created_at`) to get newest-first ordering from the min-heap.

**Cursor-based pagination** — `get_feed` supports both offset pagination (`page`/`page_size`) and cursor-based filtering (`cursor` as a timestamp). The cursor filter runs before ranking, so it works correctly with both chronological and relevance modes.

## Dependencies

**Imports**: All stdlib — `heapq` (k-way merge), `uuid` (post/comment IDs), `collections.defaultdict`/`deque` (feed cache), `dataclasses`, `enum`, `math.log` (engagement scoring).

**Imported by**: `test_news_feed.py` — the test suite exercises all three strategies and both ranking modes.

No external dependencies. This is a self-contained in-memory simulation.

## Flow

### Write Path (Post Creation)

```
create_post(author, content, ...)
  → PostStore.create_post()  (stores post, indexes by author)
  → strategy check:
      PUSH:   _fan_out_write → appendleft to every follower's deque
      HYBRID: if follower_count ≤ threshold → _fan_out_write
              else → no-op (pulled at read time)
      PULL:   no-op
```

### Read Path (Feed Generation)

```
get_feed(user, page, cursor)
  → strategy dispatch:
      PUSH:   hydrate post IDs from deque cache
      PULL:   heapq.merge across all followed users' timelines
      HYBRID: push posts + pull from celebrity follows, deduplicate
  → cursor filter (created_at < cursor)
  → score each post (_score)
  → sort by score descending
  → paginate [start:start+page_size]
```

### Follow/Unfollow

Follow backfills the new follower's cache with the followee's existing posts (for push/hybrid with non-celebrities). Unfollow purges the unfollowed user's posts from the cache via `_remove_author_from_cache`, which does a full linear scan and rebuild of the deque.

## Invariants

1. **Idempotent likes** — `like_post` checks `user_id in post.liked_by` before incrementing. Returns `False` on duplicate.
2. **Bounded cache** — `deque(maxlen=cache_size)` guarantees the feed cache never exceeds the configured size per user.
3. **Celebrity threshold determines fan-out path** — In hybrid mode, a user with `follower_count > celebrity_threshold` is *never* pushed; they're always pulled. There's no migration if someone crosses the threshold after posts are already cached.
4. **Feed cache stores IDs, not objects** — Posts are hydrated at read time, so engagement counts are always current.
5. **Comments require an existing post** — `add_comment` raises `ValueError` if the post doesn't exist. This is the only explicit validation/error in the module.

## Error Handling

Minimal, by design:

- `add_comment` raises `ValueError` for missing posts — the **only** exception in the module.
- `like_post` silently returns `False` for missing posts or duplicate likes.
- `get_post` returns `None` for missing posts; callers (like `_get_feed_push`) silently skip `None` results.
- No validation on user IDs, timestamps, or content. The social graph operations are all no-ops on non-existent relationships (using `set.discard`).

## Topics to Explore

- [file] `news-feed-system/test_news_feed.py` — See how the three strategies are tested and what edge cases the test suite covers (e.g., celebrity threshold crossover, pagination, unfollow cache invalidation)
- [function] `news-feed-system/news_feed.py:_get_feed_hybrid` — The most complex read path; worth tracing how deduplication interacts with the push/pull split and whether post ordering is preserved correctly
- [general] `fan-out-threshold-migration` — The celebrity threshold is static: if a user crosses it mid-lifecycle, already-pushed posts stay in caches while new ones switch to pull. Worth exploring whether this causes feed gaps or duplicates
- [function] `news-feed-system/news_feed.py:_score` — The relevance formula uses `log(1 + likes + 2*comments)` scaled by `RELEVANCE_WEIGHT=50.0`; explore how this balances recency vs. engagement and whether the weight makes older viral posts dominate
- [file] `chat-system/chat_system.py` — Another social-graph-based system in the repo; compare how it handles fan-out for real-time message delivery vs. feed generation

## Beliefs

- `news-feed-fan-out-write-is-push` — Fan-out-on-write pushes post IDs (not full posts) into followers' deques at write time, deferring hydration to read time so engagement metrics are fresh
- `news-feed-hybrid-splits-on-follower-count` — Hybrid strategy uses `celebrity_threshold` (default 1000) to decide push vs. pull per-author; the threshold is evaluated at write time and not retroactively applied to cached posts
- `news-feed-pull-uses-heap-merge` — Fan-out-on-read merges all followed users' timelines via `heapq.merge` with negated timestamps, avoiding full materialization
- `news-feed-only-exception-is-missing-post-comment` — The only raised exception in the module is `ValueError` from `PostStore.add_comment` when the target post doesn't exist; all other error cases return sentinel values or silently skip
- `news-feed-cache-is-bounded-deque` — Feed caches use `deque(maxlen=cache_size)` which silently drops the oldest post IDs when full, providing implicit eviction without explicit cache management

