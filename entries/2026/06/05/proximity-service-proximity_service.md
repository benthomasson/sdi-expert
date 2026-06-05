# File: proximity-service/proximity_service.py

**Date:** 2026-06-05
**Time:** 13:58

# `proximity-service/proximity_service.py`

## Purpose

This file implements a **proximity service** — the backend for "find nearby businesses/places" queries (think Yelp, Google Maps "restaurants near me"). It owns **spatial indexing and nearest-neighbor search** over a set of Points of Interest (POIs), providing two alternative index strategies: **geohash** and **quadtree**. This is a classic system design interview problem: given millions of POIs and a user's coordinates, return the closest ones within a radius efficiently.

The file is self-contained — no external dependencies beyond stdlib — making it a reference implementation that demonstrates the core algorithms without infrastructure noise.

## Key Components

### `POI` (data class)
Simple value object holding `id`, `name`, `lat`, `lon`, `category`. No validation — it trusts callers to provide valid WGS-84 coordinates. This is the universal currency type that both indexes operate on.

### `haversine(lat1, lon1, lat2, lon2) -> float`
Computes great-circle distance in kilometers using the haversine formula. This is the **ground truth** distance function — both indexes use it as the final filter after their spatial pruning narrows candidates. Earth radius is hardcoded to 6371 km.

### `GeohashIndex`
A geohash-based spatial index that maps the 2D coordinate space to 1D strings using a space-filling curve.

**Storage**: Two dictionaries — `_pois` maps `id → (POI, geohash)` for O(1) removal, and `_index` maps `geohash → {id: POI}` for spatial lookup.

**Key methods**:
- `add(poi)` / `remove(poi_id)` — O(1) insert/delete. The dual-map design enables removal without scanning.
- `nearby(lat, lon, radius_km, limit=20)` — The main query. It dynamically selects a coarser geohash precision for larger radii (via `_PRECISION_TABLE`), finds the center cell plus its 8 neighbors, scans all POIs whose geohash **prefixes** match those cells, then filters by exact haversine distance.
- `encode(lat, lon, precision)` / `decode(geohash)` — The core geohash algorithm. Encode interleaves longitude and latitude bits (longitude first), grouping every 5 bits into a base-32 character. Decode reverses this to recover the cell's center and error bounds.
- `neighbors(geohash)` — Computes the 8 adjacent cells by decoding to center coordinates, stepping by the cell dimensions, and re-encoding. Handles latitude clamping and longitude wrapping.

**Precision table**: Maps search radius to geohash precision. A 5 km search uses precision 5 (cells ~5 km wide); a 1 km search uses precision 6 (~1.2 km cells). This ensures the center cell + neighbors always covers the search circle.

### `_QuadNode` (internal)
The recursive building block of the quadtree. Uses `__slots__` for memory efficiency. Each node either holds points (leaf) or has 4 children (internal). Children are ordered NW, NE, SW, SE.

**Key methods**:
- `insert(poi)` — Adds to leaf; if capacity exceeded (`max_points`) and depth allows, triggers `_subdivide()`.
- `_subdivide()` — Splits into 4 children along the midpoint, redistributes all current points into children, then clears the leaf's point list.
- `query_bbox(min_lat, min_lon, max_lat, max_lon, results)` — Recursive range query. Prunes branches whose bounds don't intersect the query box. Collects matching points from leaves.

### `Quadtree`
Public wrapper around `_QuadNode`. Default bounds cover the whole globe.

**Key method**: `query_range(lat, lon, radius_km)` — Converts the circular search area to a bounding box (approximating `dlat` and `dlon` from the radius), runs `query_bbox`, then filters candidates by exact haversine distance. The longitude approximation accounts for latitude-dependent distortion via `cos(lat)`.

## Patterns

1. **Two-phase search (coarse → fine)**: Both indexes share the same strategy — use spatial structure to get a rough candidate set cheaply, then filter with exact haversine. This is the standard pattern for spatial search and avoids computing haversine against every POI.

2. **Strategy pattern (implicit)**: `GeohashIndex` and `Quadtree` are interchangeable spatial indexes with similar APIs (`add`/`insert`, `nearby`/`query_range`). A production service would pick one based on access patterns — geohash for database-friendly prefix queries, quadtree for in-memory with uneven distribution.

3. **Adaptive precision**: `GeohashIndex.nearby` dynamically coarsens precision for larger radii. This is critical — a fixed-precision geohash would either miss results (too fine) or scan too many cells (too coarse).

4. **Recursive spatial decomposition**: The quadtree uses the classic recursive subdivision pattern with a capacity threshold (`max_points`) and depth limit (`max_depth`) to prevent infinite recursion on coincident points.

## Dependencies

**Imports**: `math` (trig for haversine, radians conversion, cos for longitude scaling) and `collections.defaultdict` (geohash index bucketing). No third-party dependencies.

**Imported by**: `test_proximity_service.py` — the test suite is the only consumer, confirming this is a standalone reference implementation.

## Flow

### GeohashIndex query flow
1. `nearby()` called with `(lat, lon, radius_km)`
2. `_precision_for_radius()` walks the precision table to find the coarsest precision that still covers the radius
3. `encode()` converts the query point to a geohash at that precision
4. `neighbors()` decodes the center cell, steps in 8 directions, re-encodes to get adjacent cells
5. For each of the 9 cells (center + 8 neighbors), scan `_index` for all geohashes matching the prefix
6. Compute haversine distance for each candidate, keep those within `radius_km`
7. Sort by distance, truncate to `limit`

### Quadtree query flow
1. `query_range()` called with `(lat, lon, radius_km)`
2. Radius converted to a lat/lon bounding box using `111 km/degree` approximation
3. `query_bbox()` recursively traverses the tree, pruning nodes whose bounds don't intersect the box
4. Leaf nodes check each point against the bounding box
5. Candidates filtered by exact haversine distance
6. Sort by distance, return all matches (no limit parameter)

## Invariants

- **Geohash encode/decode roundtrip**: `decode(encode(lat, lon, p))` returns coordinates within the error bounds of precision `p`. The error halves with each additional character.
- **Geohash longitude-first**: The encode algorithm starts with longitude bits. This is the standard geohash convention and must not be changed or neighbor computation breaks.
- **Every 5 bits = 1 base-32 character**: The geohash bit-grouping is fixed at 5. The `bit_count == 5` check in `encode` and the `range(4, -1, -1)` in `decode` must stay synchronized.
- **Quadtree subdivision triggers at `max_points + 1`**: A leaf splits only when `len(self.points) > self.max_points` (strictly greater), so a leaf can temporarily hold `max_points + 1` points during the insert that triggers subdivision.
- **Quadtree `_contains` uses inclusive bounds**: `<=` on both sides means a point on the boundary of multiple children gets inserted into the **first** matching child (NW → NE → SW → SE order). This is deterministic but means boundary points always go to the "earlier" quadrant.
- **`nearby()` prefix scan**: The geohash `nearby()` scans all entries in `_index` checking `gh[:precision] == cell` — this is O(n) over the number of distinct geohash buckets, not O(1). With many buckets this becomes a bottleneck; a production system would use a sorted structure or database prefix query.

## Error Handling

Essentially none — this is a reference implementation that trusts its inputs:
- No coordinate validation (lat outside [-90, 90] silently produces garbage geohashes)
- `remove()` returns `False` for missing IDs rather than raising
- `Quadtree.insert()` returns `False` if the point is outside bounds — silent rejection
- `_DECODE` will raise `KeyError` on invalid geohash characters (the only implicit error)
- Division by zero in longitude scaling is guarded with `max(cos(lat), 0.001)` in `query_range`

## Topics to Explore

- [file] `proximity-service/test_proximity_service.py` — How the two indexes are tested and whether edge cases (poles, antimeridian, coincident points) are covered
- [file] `nearby-friends/nearby_friends.py` — Related spatial problem (real-time nearby users) likely using different techniques — compare the index approach
- [general] `geohash-vs-quadtree-tradeoffs` — When to pick geohash (database-friendly, easy sharding by prefix) versus quadtree (adaptive to density, better for skewed distributions)
- [function] `proximity-service/proximity_service.py:GeohashIndex.nearby` — The O(n) prefix scan over `_index` is a design limitation worth understanding for interview discussions
- [file] `proximity-service/plan.md` — Design decisions and tradeoffs considered before implementation

## Beliefs

- `geohash-encode-longitude-first` — GeohashIndex.encode interleaves bits starting with longitude, matching the standard geohash specification; swapping the order would break neighbor computation and cross-index compatibility
- `both-indexes-use-haversine-filter` — Both GeohashIndex.nearby and Quadtree.query_range use the coarse-to-fine pattern: spatial structure narrows candidates, then haversine filters to exact distance
- `geohash-nearby-prefix-scan-is-linear` — GeohashIndex.nearby iterates all keys in `_index` to match prefixes, making it O(buckets) per query rather than O(1) — a deliberate simplification vs. a sorted/trie structure
- `quadtree-no-remove` — Quadtree supports insert and query but has no removal operation, unlike GeohashIndex which supports O(1) removal via its dual-map design
- `precision-table-descending-radius` — GeohashIndex._PRECISION_TABLE is ordered by descending radius threshold; _precision_for_radius returns the precision for the first threshold the radius exceeds, defaulting to maximum precision 8

