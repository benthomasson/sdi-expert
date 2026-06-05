# File: nearby-friends/nearby_friends.py

**Date:** 2026-06-05
**Time:** 13:53

## Purpose

This file implements a **Nearby Friends service** — the kind of feature you'd find in Facebook Messenger or Snapchat's Snap Map. It answers the question: "which of my friends are physically near me right now?" The service owns three responsibilities:

1. **Location tracking** — ingesting GPS updates and maintaining current + historical positions per user
2. **Spatial querying** — efficiently finding which friends are within a configurable distance threshold
3. **Real-time notification** — pushing alerts to subscribers when a friend moves within range

This is a single-process, in-memory simulation of what would be a distributed system in production (Redis + pub/sub + geohash sharding).

## Key Components

### Data Classes

**`User`** — Identity wrapper. Just `user_id` + `name`. No behavior — it's a record.

**`LocationUpdate`** — A timestamped GPS fix: `(user_id, lat, lon, timestamp)`. These are both the "current location" and the elements of per-user history deques.

### `_haversine(lat1, lon1, lat2, lon2) → float`

Great-circle distance between two points in **kilometers**, using Earth radius of 6371 km. This is the distance function used everywhere — no approximations or flat-earth shortcuts. It's module-level (not a method) because it's pure math with no state dependency.

### `NearbyFriendsService`

The core class. Constructor takes two tuning knobs:

| Parameter | Default | Role |
|-----------|---------|------|
| `distance_threshold_km` | 5.0 | Max distance to qualify as "nearby" |
| `location_ttl_seconds` | 600 | Locations older than this are treated as stale/absent |

**Internal state (all `dict`/`set`-based):**

- `_users` — user registry
- `_friends` — symmetric adjacency list (friendship is always bidirectional)
- `_locations` — latest `LocationUpdate` per user
- `_sharing` — per-user toggle for location sharing (defaults to `True` on registration)
- `_history` — bounded `deque(maxlen=100)` per user
- `_subscriptions` — callback registry for push notifications
- `_grid` / `_user_cell` — the spatial index (see Patterns)

## Patterns

### Grid-Based Spatial Index

The most interesting design choice. Instead of brute-forcing all friends on every query, the service maintains a **uniform grid** that partitions lat/lon space into cells.

Cell size is derived directly from the distance threshold:

```python
self._cell_size = distance_threshold_km / 111.0
```

The magic number `111.0` is the approximate km-per-degree of latitude. So for 5 km threshold, cells are ~0.045° wide. This means two users within 5 km of each other are guaranteed to be in the **same cell or adjacent cells** (the 3×3 neighborhood). `get_nearby_friends` exploits this: it only checks users in the 9 neighboring cells rather than scanning all users.

`_get_cell` maps `(lat, lon) → (row, col)` via floor division. `_neighbor_cells` returns the 3×3 Moore neighborhood. The grid is maintained incrementally in `update_location` — if a user's cell changes, the old cell's set is updated and the new one gets the user added.

**Limitation:** Cell size is uniform and calibrated for the equator. At high latitudes, longitude degrees shrink (cos(lat) factor), so the grid over-partitions east-west — queries still return correct results (haversine is exact), but spatial filtering becomes less effective.

### Symmetric Friendship

`add_friendship` and `remove_friendship` always update both directions. There's no concept of a one-way follow — this models mutual friendship (like Facebook, not Twitter).

### Push via Callback Subscription

`subscribe(user_id, callback)` registers a function that gets called as `callback(updater_user_id, distance_km)` whenever a friend moves within range. This is the pub/sub pattern — in production this would be a WebSocket push or a message queue publish.

### TTL-Based Staleness

Locations expire after `location_ttl_seconds`. Both `update_location` (notification path) and `get_nearby_friends` (query path) enforce this — a friend with a stale location is invisible even if they'd be within range.

## Dependencies

**Imports:** Standard library only — `math` (haversine trig), `time` (default timestamps), `collections.defaultdict` (adjacency lists, grid), `collections.deque` (bounded history).

**Imported by:** Two test files — `test_nearby_friends.py` and `test_nearby_friends_qa.py`. No other module depends on this; it's a self-contained system design exercise.

## Flow

### Location Update Flow (`update_location`)

1. Create `LocationUpdate`, store as current location, append to history deque
2. Compute new grid cell; if changed from previous cell, update `_grid` index
3. If user has sharing disabled, return empty list (no notifications)
4. Iterate over user's friend list:
   - Skip friends with sharing disabled
   - Skip friends with no location or stale location
   - Compute haversine distance
   - If within threshold: add to notified list, fire subscriber callback if registered
5. Return list of notified friend IDs

### Nearby Query Flow (`get_nearby_friends`)

1. Check requesting user has a fresh, shared location — bail early if not
2. Look up user's grid cell, expand to 3×3 neighborhood
3. Collect all user IDs in those 9 cells
4. Intersect with friend set (`candidate_ids & friend_ids`) — this is the key optimization
5. For each candidate: check sharing, freshness, haversine distance
6. Build result dicts with name, coordinates, distance, timestamp
7. Sort by distance ascending, return

### Asymmetry Between the Two Paths

`update_location` iterates over **all friends** and checks distance (no grid filtering). `get_nearby_friends` uses the grid index. This is intentional — `update_location` is triggered per-user and the friend list is typically small, while `get_nearby_friends` needs to avoid scanning the entire user base.

## Invariants

- **Friendship symmetry:** If A is in B's friend set, B is in A's friend set. Enforced by `add_friendship`/`remove_friendship` always touching both directions.
- **Grid consistency:** `_user_cell[uid]` always matches `_get_cell(loc.lat, loc.lon)` for the user's current location. Maintained by `update_location`.
- **History bounded at 100:** The `deque(maxlen=100)` silently drops oldest entries — no unbounded memory growth.
- **Sharing defaults to True:** `add_user` sets `_sharing[uid] = True`. The `.get(uid, True)` fallback in query methods also defaults to True for unknown users.
- **Bidirectional visibility:** For user A to see friend B as nearby, **both** A and B must have sharing enabled and fresh locations. This is checked in both `update_location` and `get_nearby_friends`.

## Error Handling

Essentially none — and that's deliberate for a system design exercise. There's no input validation on lat/lon ranges, no checks for duplicate user IDs, no handling of `subscribe` for unregistered users. Methods return empty lists on edge cases (no location, no friends, sharing off) rather than raising. `_history` and `_friends` use `defaultdict`/pre-initialization so KeyError doesn't arise. The code assumes callers follow the happy path: register users first, then add friendships, then update locations.

## Topics to Explore

- [file] `nearby-friends/test_nearby_friends.py` — Unit tests reveal the expected API usage patterns and edge cases the service was designed to handle
- [file] `nearby-friends/test_nearby_friends_qa.py` — QA-level tests likely cover integration scenarios, boundary conditions around TTL and distance thresholds
- [file] `proximity-service/proximity_service.py` — Related geospatial system in the same repo; compare how it handles spatial indexing (likely geohash-based vs. grid-based here)
- [general] `grid-vs-geohash-tradeoffs` — This implementation uses a uniform grid; geohashes (used by Redis GEO) handle the latitude-distortion problem and support variable precision — worth comparing approaches
- [function] `nearby-friends/nearby_friends.py:update_location` — The notification path doesn't use the grid index while the query path does; explore whether this is intentional or a missed optimization

## Beliefs

- `nearby-friends-grid-cell-size-tied-to-threshold` — Grid cell size is `distance_threshold_km / 111.0` degrees, meaning changing the distance threshold automatically rescales the spatial index
- `nearby-friends-friendship-always-symmetric` — `add_friendship` and `remove_friendship` always update both directions; there is no one-way relationship
- `nearby-friends-update-location-skips-grid-filtering` — `update_location` iterates all friends and computes haversine for each, while `get_nearby_friends` uses the grid index to narrow candidates before distance checks
- `nearby-friends-staleness-enforced-on-both-paths` — Both notification (update_location) and query (get_nearby_friends) paths reject friend locations older than `location_ttl_seconds`
- `nearby-friends-no-input-validation` — No validation on lat/lon ranges, duplicate users, or unregistered user operations; all edge cases return empty results rather than raising exceptions

