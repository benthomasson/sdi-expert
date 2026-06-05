# File: google-maps/map_routing.py

**Date:** 2026-06-05
**Time:** 13:49

# `google-maps/map_routing.py` — Map Routing Service

## Purpose

This file implements a **map routing service** — the core pathfinding engine behind a Google Maps-style system design. It owns the responsibility of modeling a road network as a weighted graph and computing routes through it, including shortest-path queries, alternative routes, ETA estimation, turn-by-turn directions, and basic map tile serving. It's a self-contained single-file implementation designed to demonstrate the key algorithmic and architectural concepts you'd discuss in a system design interview for Google Maps.

## Key Components

### Data Models

Four `@dataclass` types define the domain:

- **`Node`** — An intersection or waypoint, identified by `node_id` with geographic coordinates (`lat`, `lon`) and an optional `name` for geocoding.
- **`Edge`** — A road segment connecting two nodes. Carries `distance_km`, `speed_limit_kmh` (default 50), and a `one_way` flag. Note: `Edge` is only used as input to `add_edge` — internally the service stores edge data as plain dicts.
- **`RouteStep`** — A single instruction in turn-by-turn directions (e.g., "Turn right onto Main St").
- **`Route`** — The complete result: a list of `RouteStep`s, aggregate distance/duration, and the raw node path.

### Geo Utilities

- **`_haversine(lat1, lon1, lat2, lon2)`** — Computes great-circle distance in km between two points. Used as the A* heuristic and is critical for heuristic admissibility — haversine never overestimates road distance since roads can't be shorter than the straight-line distance.
- **`_bearing(lat1, lon1, lat2, lon2)`** — Computes compass bearing in degrees (0–360) between two points. Used to determine turn directions at intersections.
- **`_turn_instruction(bearing_change)`** — Maps a bearing delta to one of three instructions: "Continue straight" (±30°), "Turn right" (>30°), or "Turn left" (<-30°). The threshold of 30° is a simplification — real systems use finer buckets (slight left, sharp right, U-turn).

### `MapService`

The main class. Three internal data structures:

| Field | Type | Role |
|-------|------|------|
| `self.nodes` | `dict[str, Node]` | Node lookup by ID |
| `self.adj` | `dict[str, dict[str, dict]]` | Adjacency list — `adj[u][v]` is edge data dict |
| `self.geocode_map` | `dict[str, tuple]` | Name-to-coordinates index (lowercased) |

**Graph construction:**
- `add_node(node)` — Registers a node, initializes its adjacency entry, and indexes its name for geocoding.
- `add_edge(edge)` — Adds the edge to the adjacency list. If not `one_way`, adds both directions with identical data — the same dict object is shared, which means both directions always have the same speed limit and distance.

**Pathfinding:**
- `_get_weight(edge_data, optimize)` — Dual-mode weight function. For `"time"`, returns travel time in minutes (`distance / speed * 60`). For anything else (including `"distance"`), returns raw km.
- `_heuristic(node_id, goal_id, optimize)` — A* heuristic. For distance optimization, it's just haversine. For time optimization, it divides haversine distance by the **global maximum speed limit** — this ensures admissibility but can be a weak heuristic on networks with heterogeneous speed limits.
- `_find_path(start_id, end_id, algorithm, optimize, blocked_edges)` — The core search. Implements both Dijkstra (`algorithm != "astar"`) and A* (default) via a priority queue. The `blocked_edges` parameter supports Yen's algorithm by excluding specific directed edges. Returns `(path, cost)` or `(None, None)`.
- `shortest_path(start_id, end_id, ...)` — Public API wrapping `_find_path` + `_build_route`.

**Alternative routes:**
- `alternative_routes(start_id, end_id, k=3)` — Implements **Yen's K-shortest paths algorithm**. Iteratively finds the next-shortest path by blocking edges from previously-found paths at each spur node, then selecting the shortest candidate from a heap. Always optimizes for distance (hardcoded).

**Route construction:**
- `_build_route(path, optimize)` — Converts a raw node-ID path into a `Route` with turn-by-turn `RouteStep`s. The first step always says "Head onto {road}". Subsequent steps compute bearing changes between consecutive triplets of nodes to determine turns. If the road name hasn't changed and the bearing is straight, it coalesces into a "Continue on" instruction.

**Supporting features:**
- `eta(start_id, end_id)` — Convenience method: finds the time-optimized route and returns duration in minutes.
- `get_directions(start_id, end_id)` — Returns just the `RouteStep` list from the distance-optimized route.
- `get_tile(lat, lon, tile_size=0.01)` — Returns all nodes/edges within a rectangular tile. The tile is axis-aligned and defined by flooring the input coordinates to `tile_size` increments. This models the concept of map tile serving for rendering.
- `geocode(name)` — Case-insensitive name lookup returning `{lat, lon, node_id}`.

## Patterns

1. **Graph as adjacency dict** — Rather than a formal graph class, the road network is a nested dict (`adj[u][v] = {road_name, distance_km, speed_limit_kmh}`). This is the standard Python idiom for weighted directed graphs and keeps edge lookup O(1).

2. **Strategy pattern via `optimize` parameter** — The weight function and heuristic both branch on `optimize`, allowing the same algorithm to serve both shortest-distance and fastest-time queries without code duplication.

3. **Yen's algorithm for K-shortest paths** — A well-known algorithm choice for alternative routes. The implementation blocks edges (not nodes) from previously found paths, which is the edge-disjoint variant.

4. **Input types vs. internal representation** — `Edge` dataclass is used only at the API boundary (`add_edge`). Internally, edges are plain dicts. This avoids coupling internal traversal to the input schema.

5. **Dual-mode A*/Dijkstra** — The heuristic is conditionally zero, which collapses A* to Dijkstra. One code path, two algorithms.

## Dependencies

**Imports:** Only standard library — `heapq` for the priority queue, `math` for trig/geo calculations, `dataclasses` for the data models. No external dependencies.

**Imported by:** `test_map_routing.py` — the test suite. No other modules in the repo depend on this.

## Flow

A typical usage sequence:

1. **Build graph**: Call `add_node()` for each intersection, then `add_edge()` for each road segment.
2. **Query a route**: Call `shortest_path("A", "B", optimize="time")`.
   - `_find_path` runs A* using the priority queue, expanding nodes by lowest `g(n) + h(n)`.
   - On reaching the goal, it reconstructs the path via the `prev` dict.
   - `_build_route` walks the path, looks up each edge, computes bearings between consecutive node triplets, and generates `RouteStep` instructions.
3. **Alternative routes**: `alternative_routes("A", "B", k=3)` iterates: for each prefix of the best path, it blocks the edge that was taken and re-runs A* from the spur node. Candidate paths are kept in a min-heap sorted by total distance.

## Invariants

- **Heuristic admissibility**: `_heuristic` never overestimates true cost. For distance, haversine ≤ road distance (triangle inequality on a sphere). For time, haversine / max_speed ≤ actual travel time (using the fastest possible speed as the divisor).
- **Bidirectional edges by default**: Unless `one_way=True`, every road is traversable in both directions with identical cost.
- **Geocode keys are lowercased**: All name lookups are case-insensitive. The last node added with a given name wins (no duplicate detection).
- **`_find_path` returns `(None, None)` on failure**: Both unreachable destinations and missing node IDs produce the same sentinel. Callers must null-check.
- **`_build_route` rounds totals to 10 decimal places**: `round(total_dist, 10)` — this avoids floating-point drift in cumulative sums while preserving precision.

## Error Handling

Error handling is minimal and follows the "return None" convention:

- **Missing nodes**: `_find_path` returns `(None, None)` if start or end node isn't in `self.nodes`. No exception raised.
- **No path exists**: If the priority queue is exhausted without reaching the goal, `(None, None)` is returned.
- **`shortest_path`** returns `None` if pathfinding fails. Callers (like `eta`) must check: `route.total_duration_min if route else None`.
- **`get_directions`** returns `[]` on failure.
- **No input validation**: There's no check for duplicate nodes, negative distances, zero speed limits (which would cause division-by-zero in `_get_weight`), or self-loop edges. The code trusts its callers.
- **`geocode`** returns `None` for unknown names.

There are no exceptions raised anywhere in this module — all failures are communicated via `None` or empty collections.

## Topics to Explore

- [file] `google-maps/test_map_routing.py` — See how the road network is constructed in tests and what edge cases are covered (disconnected graphs, one-way streets, alternative routes)
- [function] `google-maps/map_routing.py:alternative_routes` — Yen's K-shortest paths has subtle correctness requirements around edge blocking and spur node selection; worth verifying against the reference algorithm
- [general] `a-star-heuristic-tightness` — The time-optimized heuristic uses global max speed, which can be very loose on mixed-speed networks; explore how contraction hierarchies or ALT heuristics improve this
- [file] `google-maps/plan.md` — The design plan likely discusses system-level concerns (tile serving, caching, partitioning) that this implementation simplifies away
- [general] `yen-vs-penalty-method` — Alternative route algorithms: Yen's finds K-shortest, but real map apps use penalty-based or plateau methods to find *meaningfully different* routes, not just slightly different ones

## Beliefs

- `astar-heuristic-admissible` — The A* heuristic is admissible: distance mode uses haversine (always ≤ road distance), time mode uses haversine divided by the global maximum speed limit (always ≤ actual travel time)
- `find-path-returns-none-pair-on-failure` — `_find_path` returns `(None, None)` for both missing nodes and unreachable destinations; it never raises exceptions
- `bidirectional-edges-share-same-dict` — When `one_way=False`, both directions of an edge reference the same data dict, so modifying one direction's data mutates the other
- `alternative-routes-always-optimizes-distance` — `alternative_routes` hardcodes `optimize="distance"` for both the initial path and all spur paths, ignoring any time-based optimization
- `geocode-last-writer-wins` — If multiple nodes share the same name, only the last one added is retained in the geocode index; earlier entries are silently overwritten

