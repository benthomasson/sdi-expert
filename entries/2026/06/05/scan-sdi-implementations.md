# Scan: sdi-implementations

**Date:** 2026-06-05
**Time:** 13:14

This is a fresh knowledge base — no entries or beliefs yet. I'll produce the reconnaissance from the repository structure. Each directory is a standalone system design interview problem with a consistent pattern: `plan.md` → `plan_review.md` → implementation → tests.

---

## Architecture Sketch

This is a **collection of 25 independent system design interview implementations**, each self-contained in its own directory. Every module follows the same lifecycle: a design plan (`plan.md`), a plan review (`plan_review.md`), a Python implementation, and pytest-based tests. There are no shared libraries, no common base classes, and no cross-module imports — each implementation is a standalone demonstration of a specific distributed systems concept. The implementations simulate distributed behavior in-process (no real network/storage dependencies), making them pedagogical reference implementations rather than production systems.

## Module Map

**Infrastructure Primitives** — foundational building blocks used inside larger systems:
- `consistent-hashing` — hash ring, virtual nodes, rebalancing
- `key-value-store` — storage engine, replication, partitioning
- `rate-limiter` — token bucket / sliding window algorithms
- `unique-id-generator` — snowflake / UUID generation
- `distributed-message-queue` — pub/sub, ordering, delivery guarantees

**Application Systems** — full product-level designs:
- `chat-system` — real-time messaging
- `news-feed-system` — fan-out, ranking, timeline
- `notification-system` — multi-channel delivery
- `hotel-reservation-system` — booking, inventory, concurrency
- `payment-system` / `digital-wallet` — transactions, idempotency
- `stock-exchange` — order matching, low-latency

**Search & Discovery**:
- `search-autocomplete` — trie/prefix, ranking
- `proximity-service` / `nearby-friends` — geospatial indexing, real-time location
- `google-maps` — routing, graph algorithms

**Storage & Media**:
- `design-google-drive` — file sync, versioning, conflict resolution
- `design-youtube` — video pipeline, CDN, metadata
- `s3-object-storage` — object storage, multipart upload
- `url-shortener` — encoding, redirection

**Data Processing & Monitoring**:
- `ad-click-event-aggregation` — stream processing, aggregation windows
- `metrics-monitoring-and-alerting` — time series, alerting rules
- `web-crawler` — crawl frontier, politeness, dedup
- `distributed-email-service` — queuing, delivery, bounce handling

**Gaming**:
- `realtime-gaming-leaderboard` — sorted sets, real-time ranking

## Critical Files

The most important files to understand first, chosen for maximal coverage of distinct patterns:

1. **`consistent-hashing/consistent_hashing.py`** — The purest infrastructure primitive. Likely used conceptually by several other implementations. Hash rings and virtual nodes are a foundational SDI concept.

2. **`key-value-store/key_value_store.py`** — Storage engine internals (LSM tree? hash index?), replication, consistency. Core to understanding how data is modeled across all other systems.

3. **`distributed-message-queue/solution.py`** — Messaging backbone. Ordering guarantees, consumer groups, delivery semantics. Conceptual dependency for chat, notifications, event aggregation.

4. **`payment-system/payment_system.py`** — Highest correctness bar. Idempotency, distributed transactions, exactly-once semantics. Will reveal how the codebase handles the hardest consistency problems.

5. **`rate-limiter/rate_limiter.py`** — Algorithm-heavy (token bucket, sliding window, fixed window). Cross-cutting concern for any API system.

6. **`chat-system/chat_system.py`** — Real-time + persistence. WebSocket modeling, message ordering, presence. Different architectural flavor from request/response systems.

7. **`news-feed-system/news_feed.py`** — Fan-out-on-write vs fan-out-on-read. A classic SDI trade-off problem.

8. **`stock-exchange/solution.py`** — Low-latency matching engine. Order book data structures, price-time priority. Very different from typical CRUD systems.

9. **`ad-click-event-aggregation/click_aggregator.py`** — Stream processing patterns. Windowed aggregation, exactly-once counting. Different from request/response.

10. **`design-google-drive/design_google_drive.py`** — File sync and conflict resolution. Operational transform or CRDT patterns potentially.

11. **`hotel-reservation-system/hotel_reservation.py`** — Inventory management with concurrency. Overbooking prevention, pessimistic/optimistic locking patterns.

12. **`web-crawler/web_crawler.py`** — Distributed crawl scheduling. Politeness, URL frontier, deduplication. Producer-consumer at scale.

## Exploration Strategy

The order below maximizes understanding transfer — each file teaches patterns used by later ones:

**Phase 1: Primitives** (understand the building blocks)
1. `consistent-hashing` — partitioning foundation
2. `key-value-store` — storage foundation
3. `unique-id-generator` — ID generation patterns
4. `rate-limiter` — algorithm patterns

**Phase 2: Messaging & Events** (understand async patterns)
5. `distributed-message-queue` — pub/sub foundation
6. `ad-click-event-aggregation` — stream processing on top of messaging
7. `notification-system` — multi-channel delivery

**Phase 3: Application Systems** (full system designs)
8. `payment-system` — strongest consistency requirements
9. `hotel-reservation-system` — concurrency + inventory
10. `stock-exchange` — performance-critical matching
11. `chat-system` — real-time + persistence

**Phase 4: Search, Storage & Discovery**
12. `news-feed-system` — fan-out trade-offs
13. `design-google-drive` — sync + conflict resolution
14. `web-crawler` — distributed scheduling
15. `search-autocomplete` — trie/ranking

**Phase 5: Remaining systems** (round out coverage)
16–25: proximity-service, nearby-friends, google-maps, design-youtube, s3-object-storage, url-shortener, digital-wallet, distributed-email-service, metrics-monitoring, realtime-gaming-leaderboard

---

## Topics to Explore

- [file] `consistent-hashing/consistent_hashing.py` — Hash ring implementation; foundational partitioning primitive used conceptually across many other designs
- [file] `key-value-store/key_value_store.py` — Storage engine internals (replication, consistency model, partitioning); reveals codebase's approach to data modeling
- [file] `distributed-message-queue/solution.py` — Message ordering, consumer groups, delivery guarantees; backbone pattern for async systems
- [file] `payment-system/payment_system.py` — Idempotency and transaction patterns; highest correctness bar in the collection
- [file] `rate-limiter/rate_limiter.py` — Algorithm implementations (token bucket, sliding window); cross-cutting API protection concern
- [file] `stock-exchange/solution.py` — Order matching engine; reveals low-latency data structure choices (order book, price-time priority)
- [file] `chat-system/chat_system.py` — Real-time messaging architecture; different flavor from request/response systems
- [file] `ad-click-event-aggregation/click_aggregator.py` — Stream processing and windowed aggregation; event-driven architecture patterns
- [file] `news-feed-system/news_feed.py` — Fan-out-on-write vs read trade-off; classic SDI design decision
- [file] `hotel-reservation-system/hotel_reservation.py` — Inventory concurrency control; pessimistic vs optimistic locking patterns
- [file] `design-google-drive/design_google_drive.py` — File sync and conflict resolution; potentially CRDT or OT patterns
- [file] `web-crawler/web_crawler.py` — Distributed crawl frontier; URL scheduling, politeness, deduplication
- [general] `cross-cutting-patterns` — Identify shared design patterns across implementations (idempotency, retry, circuit breaker, sharding strategies)
- [general] `consistency-models` — Compare how different implementations handle consistency (eventual vs strong, CAP trade-offs made)
- [general] `plan-to-implementation-fidelity` — Do implementations match their plans? Where do they diverge and why?

