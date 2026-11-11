# Seven Rules Audit: Four Fixes Across SDI Implementations

## Context

After building a belief network of ~420 observations across 25 system design implementations, we applied the [seven rules for data-intensive systems](/Users/ben/git/ddia-expert/entries/2026/06/06/seven-rules-for-data-intensive-systems.md) as an external audit framework. An Explore agent swept all implementations against each rule, surfacing 15 findings. We filtered to 4 actionable fixes — the rest were acceptable at pedagogical scale or already mitigated by existing design choices.

The four fixes address three of the seven rules:
- **Rule 1** (verify data on read): S3 ETag integrity
- **Rule 4** (protect shared mutable state): chat Lamport clock, nearby-friends grid
- **Rule 5** (build repair into the design): news feed cache rebuild

## Fix 1: Chat Lamport Clock Atomicity

**File:** `chat-system/chat_system.py`
**Rule:** Protect shared mutable state

The chat server maintains a global `lamport_clock` and per-conversation `next_sequence` counters. Four methods increment these: `send_message`, `send_group_message`, `add_member`, `remove_member`. Without synchronization, concurrent calls could produce duplicate sequence numbers or Lamport timestamps, violating the ordering guarantees the system depends on.

**Fix:** Added `threading.Lock` (`_clock_lock`). Each increment site captures `lamport_ts` as a local variable inside the lock:

```python
with self._clock_lock:
    self.lamport_clock += 1
    seq = conv.next_sequence
    conv.next_sequence += 1
    lamport_ts = self.lamport_clock
```

The lock scope is minimal — only the counter increments, not message construction or delivery. This preserves the system's existing concurrency model (delivery to inboxes doesn't need serialization) while making the ordering primitive safe.

**Belief added:** `chat-lamport-clock-is-thread-safe`

## Fix 2: Nearby Friends Concurrency Protection

**File:** `nearby-friends/nearby_friends.py`
**Rule:** Protect shared mutable state

`update_location` mutates three shared structures in sequence: `_locations` (current position), `_grid` (spatial index), and `_user_cell` (reverse index). `get_nearby_friends` reads from `_grid` to find candidates. Without synchronization, a concurrent read during a cell transition could miss a user who moved between cells — or worse, see them in both cells.

**Fix:** Added `threading.Lock` (`_lock`). In `update_location`, the lock wraps location storage, grid index update, and candidate snapshot:

```python
with self._lock:
    self._locations[user_id] = loc
    self._history[user_id].append(loc)
    new_cell = self._get_cell(lat, lon)
    # grid index update ...
    candidate_ids = set()
    friend_ids = self._friends.get(user_id, set())
    if self._sharing.get(user_id, True) and friend_ids:
        for cell in self._neighbor_cells(new_cell):
            candidate_ids.update(self._grid.get(cell, set()))
        candidate_ids &= friend_ids
```

The haversine distance computation and notification callbacks happen outside the lock — only the structural mutations and the candidate snapshot are serialized.

**Belief added:** `nearby-friends-grid-is-thread-safe`

## Fix 3: News Feed Cache Rebuild

**File:** `news-feed-system/news_feed.py`
**Rule:** Build repair into the design

The news feed uses an in-memory deque of post IDs as a feed cache. If this cache is lost (process restart, memory pressure, bug), there's no way to reconstruct it — the user sees an empty feed until new posts arrive. This is exactly the scenario Rule 5 warns about: systems that can only move forward but can't recover from state loss.

**Fix:** Added `rebuild_cache(user_id)` that reconstructs the feed from the social graph and post store:

```python
def rebuild_cache(self, user_id: str):
    self._feed_cache[user_id].clear()
    following = self.graph.get_following(user_id)
    all_posts = []
    for followee_id in following:
        if self.strategy == FeedStrategy.HYBRID:
            if self.graph.get_follower_count(followee_id) > self.celebrity_threshold:
                continue
        all_posts.extend(self.post_store.get_user_posts(followee_id, limit=self.cache_size))
    all_posts.sort(key=lambda p: p.created_at, reverse=True)
    for post in all_posts[:self.cache_size]:
        self._feed_cache[user_id].append(post.post_id)
```

The rebuild respects the hybrid strategy by skipping celebrity posts (they're pulled on read, not cached), matching the write-time fan-out policy.

**Belief added:** `news-feed-cache-is-rebuildable`

## Fix 4: S3 ETag Integrity Verification on Read

**File:** `s3-object-storage/s3_object_storage.py`
**Rule:** Verify data on read

The S3 implementation computes an MD5 ETag at write time and stores it alongside the data. But `get_object` returned data without ever re-checking the hash — a corrupted object would be silently served. This is the most basic form of Rule 1: if you have an integrity check, actually use it on the read path.

**Fix:** Added `_verify_integrity` called from `get_object` before returning:

```python
def _verify_integrity(self, version: ObjectVersion, key: str):
    if version.data and self._etag(version.data) != version.etag:
        raise ValueError(f"Data integrity check failed for '{key}' (version {version.version_id})")
```

Both the latest-version and specific-version code paths call this before `_version_to_dict`.

**Belief added:** `s3-etag-verified-on-read`

## Belief Network Changes

### New Premises (4)
| Belief | Summary |
|--------|---------|
| `chat-lamport-clock-is-thread-safe` | Clock/sequence increments atomic via Lock |
| `nearby-friends-grid-is-thread-safe` | Grid mutations and reads atomic via Lock |
| `s3-etag-verified-on-read` | ETag re-computed and verified on every read |
| `news-feed-cache-is-rebuildable` | Feed cache reconstructible from social graph + post store |

### New Derived (1)
| Belief | Depends On |
|--------|------------|
| `read-path-verification-is-improving` | `s3-etag-verified-on-read`, `news-feed-cache-is-rebuildable`, `read-path-responsibility-exceeds-verification` |

The derived belief captures that these fixes narrow the gap identified by `read-path-responsibility-exceeds-verification` — the read path now has verification (S3 ETag) and repair (feed rebuild) mechanisms that didn't exist before. But the gate remains active because other systems (KV read-repair, lazy evaluation, metadata-as-deletion) still lack comparable verification coverage.

### Gate Status
The remaining gate `read-path-responsibility-exceeds-verification` → `cost-shifting-is-fully-verifiable` stays open. Two of the read path's responsibilities now have verification, but the broader pattern — that read-path complexity outpaces test infrastructure — persists across the codebase.

## Pattern

All four fixes share a common shape: the code already had the right *structure* (Lamport clocks, spatial grid, ETag computation, post store as source of truth) but was missing the *operational mechanism* to make that structure safe under stress. The Lamport clock existed but wasn't atomic. The ETag was computed but never checked. The post store held all data but couldn't regenerate a lost cache. The grid indexed users but wasn't protected from concurrent mutation.

This matches the belief network's deepest finding: the codebase achieves correctness by construction (structural invariants built into the data model) but sometimes stops short of correctness under operation (runtime mechanisms needed to maintain those invariants when things go wrong or happen concurrently).
