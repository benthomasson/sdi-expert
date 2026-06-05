# Proposed Beliefs

Edit each entry: change `[ACCEPT/REJECT]` to `[ACCEPT]` or `[REJECT]`.
Then run: `code-expert accept-beliefs`

---

**Generated:** 2026-06-05
**Source:** 31 entries from entries/
**Model:** claude

### [ACCEPT] dedup-is-global-not-per-ad
The `seen_events` dedup registry keys on `event_id` alone; the same event ID arriving for different `ad_id`s will be deduplicated.
- Source: entries/2026/06/05/ad-click-event-aggregation-click_aggregator.md

### [ACCEPT] watermark-drives-finalization
Windows are never finalized by event processing alone; only `advance_watermark` transitions windows to FINALIZED and emits `AggregationResult`s.
- Source: entries/2026/06/05/ad-click-event-aggregation-click_aggregator.md

### [ACCEPT] dedup-pruning-uses-2x-lateness
The dedup registry evicts entries older than `2 * allowed_lateness` on watermark advance, meaning dedup coverage extends beyond the late-event acceptance window.
- Source: entries/2026/06/05/ad-click-event-aggregation-click_aggregator.md

### [ACCEPT] batch-classification-coupled-to-process-event
`process_batch` infers per-event rejection reasons by diffing global stats counters, creating an implicit coupling to the check ordering inside `process_event`.
- Source: entries/2026/06/05/ad-click-event-aggregation-click_aggregator.md

### [ACCEPT] no-window-merging-or-retraction
Once an `AggregationResult` is emitted from `advance_watermark`, there is no mechanism to update or retract it — finalized windows reject all further events.
- Source: entries/2026/06/05/ad-click-event-aggregation-click_aggregator.md

### [ACCEPT] window-lifecycle-one-directional
Window state follows a one-way ratchet: OPEN → CLOSED → FINALIZED. Windows never reopen; OPEN accepts all events, CLOSED accepts late events within the lateness allowance, FINALIZED rejects everything.
- Source: entries/2026/06/05/ad-click-event-aggregation-click_aggregator.md

### [ACCEPT] click-aggregator-return-value-signaling
`process_event` returns `True`/`False` to signal acceptance or rejection — invalid states (duplicates, late arrivals) are expected in stream processing, not exceptional, so no exceptions are raised.
- Source: entries/2026/06/05/ad-click-event-aggregation-click_aggregator.md

### [ACCEPT] chat-dm-conversation-dedup
DM conversations use a deterministic sorted-pair ID (`dm:{min}:{max}`) guaranteeing exactly one conversation per user pair regardless of who messages first.
- Source: entries/2026/06/05/chat-system-chat_system.md

### [ACCEPT] chat-dual-ordering-scheme
Messages carry both per-conversation sequence numbers (for pagination and read cursors) and global Lamport timestamps (for causal ordering), serving different purposes by design.
- Source: entries/2026/06/05/chat-system-chat_system.md

### [ACCEPT] chat-offline-queue-flush-on-connect
When a user connects, their `offline_queue` is drained into `inbox` in FIFO order, preserving message arrival ordering.
- Source: entries/2026/06/05/chat-system-chat_system.md

### [ACCEPT] chat-read-cursors-monotonic
`mark_read` only advances the read cursor; it silently ignores attempts to set a lower sequence number, preventing regression.
- Source: entries/2026/06/05/chat-system-chat_system.md

### [ACCEPT] chat-soft-delete-preserves-sequence
Deleted messages remain in the conversation list with `deleted=True` and content replaced with `"[deleted]"`, preserving sequence number continuity for pagination.
- Source: entries/2026/06/05/chat-system-chat_system.md

### [ACCEPT] chat-contacts-always-bidirectional
`send_message` adds both directions to the contacts graph as a side effect — contacts are always symmetric.
- Source: entries/2026/06/05/chat-system-chat_system.md

### [ACCEPT] chat-group-size-cap-500
`add_member` raises `ValueError` if the group already has 500 or more members, enforcing a hard cap on group size.
- Source: entries/2026/06/05/chat-system-chat_system.md

### [ACCEPT] chat-fanout-on-write
`_deliver` routes messages at send time based on recipient presence: online/away users get messages in `inbox`, offline users in `offline_queue` — this is fan-out-on-write, not fan-out-on-read.
- Source: entries/2026/06/05/chat-system-chat_system.md

### [ACCEPT] ch-vnode-naming-scheme
Virtual nodes are keyed as `"{node_id}#{i}"` for `i` in `range(num_virtual_nodes)`, so the hash distribution depends entirely on the hash function's behavior on these strings.
- Source: entries/2026/06/05/consistent-hashing-consistent_hashing.md

### [ACCEPT] ch-default-hash-32bit
The default hash function truncates MD5 to 32 bits (first 4 bytes of the digest), mapping all positions to the `[0, 2^32)` ring space.
- Source: entries/2026/06/05/consistent-hashing-consistent_hashing.md

### [ACCEPT] ch-add-remove-idempotent
`add_node` on an existing node and `remove_node` on a missing node are both no-ops returning empty results, making the ring safe against duplicate operations.
- Source: entries/2026/06/05/consistent-hashing-consistent_hashing.md

### [ACCEPT] ch-get-nodes-deduplicates-physical
`get_nodes` walks clockwise and skips virtual nodes belonging to already-collected physical nodes, guaranteeing distinct physical nodes for replication.
- Source: entries/2026/06/05/consistent-hashing-consistent_hashing.md

### [ACCEPT] ch-stats-measures-arc-ownership
`get_stats` computes load standard deviation from ring arc length ownership per physical node, not from actual key counts — it's a theoretical balance metric.
- Source: entries/2026/06/05/consistent-hashing-consistent_hashing.md

### [ACCEPT] ch-triple-bookkeeping
The ring maintains three synchronized structures (`_sorted_positions`, `_position_to_node`, `_node_positions`) as the price of O(log n) lookups via bisect — all three must stay consistent on every add/remove.
- Source: entries/2026/06/05/consistent-hashing-consistent_hashing.md

### [ACCEPT] ch-migration-tracking-optional
Both `add_node` and `remove_node` accept an optional `keys` list and report which keys moved — the ring itself is stateless w.r.t. data and only computes migration impact when given a key set.
- Source: entries/2026/06/05/consistent-hashing-consistent_hashing.md

### [ACCEPT] gdrive-permission-inheritance
Permission checks walk up the folder tree via `parent_folder_id` pointers; access granted on any ancestor grants access to all descendants.
- Source: entries/2026/06/05/design-google-drive-design_google_drive.md

### [ACCEPT] gdrive-quota-reservation
Chunked uploads reserve full quota at init time and release-then-reaccount at completion, preventing over-commitment from multiple concurrent uploads.
- Source: entries/2026/06/05/design-google-drive-design_google_drive.md

### [ACCEPT] gdrive-version-vector-conflict
Conflict detection uses per-device version vectors, not simple version counters; a conflict requires a *different* device to have written a version the requesting device hasn't seen.
- Source: entries/2026/06/05/design-google-drive-design_google_drive.md

### [ACCEPT] gdrive-soft-delete-cascade
Deleting a folder soft-deletes all descendants via BFS traversal; permanent deletion only occurs in `empty_trash` after a time-based cutoff.
- Source: entries/2026/06/05/design-google-drive-design_google_drive.md

### [ACCEPT] gdrive-restore-creates-new-version
`restore_version` delegates to `update_file`, so restoring to an old version creates a new version entry rather than rolling back the version counter — preserving the audit trail.
- Source: entries/2026/06/05/design-google-drive-design_google_drive.md

### [ACCEPT] gdrive-owner-bypasses-permission
`_check_permission` returns immediately if `meta.owner_id == user_id`, bypassing the full permission inheritance walk.
- Source: entries/2026/06/05/design-google-drive-design_google_drive.md

### [ACCEPT] gdrive-version-list-bounded
`update_file` prunes the version history to `max_versions` (default 100), keeping only the most recent entries.
- Source: entries/2026/06/05/design-google-drive-design_google_drive.md

### [ACCEPT] dag-failure-cascade
When a `ProcessingDAG` stage fails, all transitively dependent stages are marked SKIPPED; independent branches continue executing.
- Source: entries/2026/06/05/design-youtube-design_youtube.md

### [ACCEPT] pipeline-status-contract
`VideoUploadPipeline` sets video status to FAILED if and only if the `finalize` stage does not reach COMPLETED.
- Source: entries/2026/06/05/design-youtube-design_youtube.md

### [ACCEPT] hll-default-precision
`HyperLogLogCounter` defaults to precision 14 (16384 registers), matching the standard HLL recommendation for ~1% error rate.
- Source: entries/2026/06/05/design-youtube-design_youtube.md

### [ACCEPT] recommendation-excludes-watched
`get_feed` excludes videos the user has already watched from popular and content-based scores; collaborative filtering excludes them in its own co-occurrence logic.
- Source: entries/2026/06/05/design-youtube-design_youtube.md

### [ACCEPT] search-status-filter
`VideoStore.search` returns only videos with status READY or UPLOADING, excluding PROCESSING and FAILED videos.
- Source: entries/2026/06/05/design-youtube-design_youtube.md

### [ACCEPT] youtube-pipeline-dag-structure
The video processing pipeline wires a 5-stage DAG: validate → (transcode | thumbnail | metadata) → finalize, where the three middle stages are independent after validation.
- Source: entries/2026/06/05/design-youtube-design_youtube.md

### [ACCEPT] youtube-ctx-dict-blackboard
Pipeline stages communicate through a mutable `ctx` dict rather than return values — a blackboard pattern that avoids inter-stage type coupling but sacrifices type safety.
- Source: entries/2026/06/05/design-youtube-design_youtube.md

### [ACCEPT] youtube-reciprocal-rank-fusion
`get_feed` blends recommendation strategies using reciprocal rank fusion: each strategy contributes `weight * 1/(rank+1)` to a video's score, with default weights favoring collaborative (0.5) over popular (0.3) over content-based (0.2).
- Source: entries/2026/06/05/design-youtube-design_youtube.md

### [ACCEPT] morris-counter-32-estimators
`MorrisCounter` uses 32 independent counters averaged together for improved accuracy — each incremented probabilistically with probability `1/2^c`.
- Source: entries/2026/06/05/design-youtube-design_youtube.md


### [ACCEPT] wallet-lock-ordering-prevents-deadlock
Transfers acquire wallet locks sorted by `wallet_id`, preventing deadlock when two concurrent transfers involve the same pair of wallets in opposite directions.
- Source: entries/2026/06/05/digital-wallet-wallet.md

### [ACCEPT] wallet-two-tier-locking
Per-wallet locks protect balance mutations; a separate `_tx_lock` protects the shared transactions list. The wallet lock is always acquired before `_tx_lock`, never the reverse.
- Source: entries/2026/06/05/digital-wallet-wallet.md

### [ACCEPT] daily-limit-only-counts-outflows
Daily spending is computed from `withdrawal` and `transfer_out` transactions only; deposits and incoming transfers are uncapped.
- Source: entries/2026/06/05/digital-wallet-wallet.md

### [ACCEPT] verify-integrity-ignores-transfers
`verify_integrity` sums only `deposit` and `withdrawal` amounts against total balances, because `transfer_in`/`transfer_out` pairs are zero-sum and cancel out.
- Source: entries/2026/06/05/digital-wallet-wallet.md

### [ACCEPT] wallet-creation-silently-overwrites
`create_wallet` does not check for an existing `wallet_id`; calling it twice with the same ID replaces the wallet and loses the original balance.
- Source: entries/2026/06/05/digital-wallet-wallet.md

### [ACCEPT] wallet-frozen-check-inside-lock
Frozen status is checked inside the wallet lock, eliminating TOCTOU races between freeze/unfreeze and money movement operations.
- Source: entries/2026/06/05/digital-wallet-wallet.md

### [ACCEPT] wallet-no-exceptions-caught
The wallet service has no `try`/`except` blocks; all seven custom exceptions propagate to the caller unconditionally.
- Source: entries/2026/06/05/digital-wallet-wallet.md

### [ACCEPT] wallet-currency-immutable-after-creation
Currency is immutable after wallet creation; `transfer` checks currency match before acquiring locks, which is safe because the field never changes.
- Source: entries/2026/06/05/digital-wallet-wallet.md

---

### [ACCEPT] email-service-single-folder-per-user
A message can only exist in one folder per user; `move_to_folder` removes from the source before adding to the target.
- Source: entries/2026/06/05/distributed-email-service-email_service.md

### [ACCEPT] email-service-thread-id-is-first-msg
Thread IDs are the message ID of the thread's first message, not a separately generated identifier.
- Source: entries/2026/06/05/distributed-email-service-email_service.md

### [ACCEPT] email-service-get-email-marks-read
`get_email` always side-effects a `mark_read`; there is no read-without-marking retrieval path.
- Source: entries/2026/06/05/distributed-email-service-email_service.md

### [ACCEPT] email-service-two-phase-delete
First `delete()` call moves to trash; second call on a trashed message permanently removes it.
- Source: entries/2026/06/05/distributed-email-service-email_service.md

### [ACCEPT] email-service-bcc-stored-in-record
BCC recipients are stored in the email dict alongside to/cc, which would leak BCC information to other recipients in a real system.
- Source: entries/2026/06/05/distributed-email-service-email_service.md

### [ACCEPT] email-service-lazy-user-init
`_init_user()` is called at the start of most operations to bootstrap default folders, so accounts don't require explicit `create_account()` before use.
- Source: entries/2026/06/05/distributed-email-service-email_service.md

### [ACCEPT] email-service-input-storage-decoupling
`Email` dataclass objects are the send-time input representation; stored emails are plain dicts created by `_store_email`, decoupling the send API from the query format.
- Source: entries/2026/06/05/distributed-email-service-email_service.md

---

### [ACCEPT] dmq-partition-offset-is-logical
Partition offsets are logical (not physical array indices); `base_offset` adjusts after trimming so consumers using stored offsets see consistent numbering across retention evictions.
- Source: entries/2026/06/05/distributed-message-queue-solution.md

### [ACCEPT] dmq-delivery-semantics-in-poll
The three delivery modes (at_least_once, at_most_once, exactly_once) are enforced entirely within `poll()` and `commit()`, not in the publish path.
- Source: entries/2026/06/05/distributed-message-queue-solution.md

### [ACCEPT] dmq-one-topic-per-consumer-group
Each ConsumerGroup is permanently bound to a single topic at creation; there is no mechanism for multi-topic subscription.
- Source: entries/2026/06/05/distributed-message-queue-solution.md

### [ACCEPT] dmq-dlq-is-regular-topic
Dead-letter queues are standard topics with `__dlq_` prefix, created lazily on first failure and consumable through the same poll/commit API.
- Source: entries/2026/06/05/distributed-message-queue-solution.md

### [ACCEPT] dmq-retention-trimmed-on-publish
Retention is enforced per-publish: every `publish()` call trims the target partition. Messages are never evicted between publishes.
- Source: entries/2026/06/05/distributed-message-queue-solution.md

### [ACCEPT] dmq-rebalance-modular-assignment
Consumer group rebalancing uses simple modular assignment: partition `p` goes to `consumers[p % len(consumers)]`, producing deterministic assignments for the same consumer list.
- Source: entries/2026/06/05/distributed-message-queue-solution.md

### [ACCEPT] dmq-two-tier-offset-tracking
`current_offset` advances on `poll()` and `committed_offset` advances on explicit `commit()`; the gap between them is the uncommitted window of delivered-but-unconfirmed messages.
- Source: entries/2026/06/05/distributed-message-queue-solution.md

### [ACCEPT] dmq-reused-by-stock-exchange
The message queue is imported by `stock-exchange/test_exchange.py` as an event bus for order matching, demonstrating cross-module reuse across SDI implementations.
- Source: entries/2026/06/05/distributed-message-queue-solution.md

---

### [ACCEPT] maps-astar-heuristic-admissible
The A* heuristic is admissible: distance mode uses haversine (always ≤ road distance), time mode divides haversine by the global maximum speed limit (always ≤ actual travel time).
- Source: entries/2026/06/05/google-maps-map_routing.md

### [ACCEPT] maps-dual-mode-astar-dijkstra
`_find_path` implements both A* and Dijkstra via a single code path; setting `algorithm != "astar"` zeroes the heuristic, collapsing A* to Dijkstra.
- Source: entries/2026/06/05/google-maps-map_routing.md

### [ACCEPT] maps-yen-k-shortest-for-alternatives
`alternative_routes` implements Yen's K-shortest paths algorithm, blocking edges from prior paths at each spur node; it always optimizes for distance (hardcoded).
- Source: entries/2026/06/05/google-maps-map_routing.md

### [ACCEPT] maps-no-exceptions-none-returns
`map_routing.py` raises no exceptions; all failures (missing nodes, unreachable destinations, unknown geocode names) return `None` or empty collections.
- Source: entries/2026/06/05/google-maps-map_routing.md

### [ACCEPT] maps-bidirectional-edges-share-data
Non-one-way edges are added in both directions sharing the same dict object, meaning both directions always have identical distance and speed limit values.
- Source: entries/2026/06/05/google-maps-map_routing.md

### [ACCEPT] maps-time-heuristic-uses-global-max-speed
The time-optimized A* heuristic divides haversine distance by the global maximum speed limit, which is admissible but can be a weak heuristic on heterogeneous-speed networks.
- Source: entries/2026/06/05/google-maps-map_routing.md

---

### [ACCEPT] hotel-occ-two-phase-reserve
`reserve()` implements optimistic concurrency via a read-then-verify-version pattern, but both phases execute synchronously with no actual interleaving, making the version check demonstrative rather than functional.
- Source: entries/2026/06/05/hotel-reservation-system-hotel_reservation.md

### [ACCEPT] hotel-pricing-stacks-multipliers
Dynamic occupancy pricing and seasonal pricing multiply independently on the base price; a 90% occupied room during peak season costs `base * seasonal_mult * 1.5`.
- Source: entries/2026/06/05/hotel-reservation-system-hotel_reservation.md

### [ACCEPT] hotel-checkout-exclusive
Date ranges are half-open `[check_in, check_out)`: a booking from March 15 to March 16 occupies inventory on March 15 only.
- Source: entries/2026/06/05/hotel-reservation-system-hotel_reservation.md

### [ACCEPT] hotel-idempotency-ignores-params
The idempotency key check returns the cached reservation without validating that guest name, dates, or room type match the original request.
- Source: entries/2026/06/05/hotel-reservation-system-hotel_reservation.md

### [ACCEPT] hotel-cancel-no-underflow-guard
`cancel()` decrements `booked` without checking for `booked >= 0`, so a double-cancel bug (if the status check were bypassed) could produce negative inventory counts.
- Source: entries/2026/06/05/hotel-reservation-system-hotel_reservation.md

### [ACCEPT] hotel-seasonal-pricing-first-match
If seasonal pricing date ranges overlap, the first appended rule wins due to a `break` after the first match in `_get_price`.
- Source: entries/2026/06/05/hotel-reservation-system-hotel_reservation.md

### [ACCEPT] hotel-search-availability-is-bottleneck-date
`search()` reports availability as the minimum available rooms across all nights in the requested range; the most-booked date determines what's bookable.
- Source: entries/2026/06/05/hotel-reservation-system-hotel_reservation.md


### [ACCEPT] kv-store-quorum-exception
`put`, `get`, and `delete` raise generic `Exception` (not a custom type) when the quorum threshold (W or R) is not met
- Source: entries/2026/06/05/key-value-store-key_value_store.md

### [ACCEPT] kv-node-stores-sibling-versions
`KVNode.store` maps each key to a list of `VersionedValue`; concurrent writes produce multiple siblings that the client must resolve
- Source: entries/2026/06/05/key-value-store-key_value_store.md

### [ACCEPT] kv-delete-is-tombstone-write
Deletes are implemented as a `put` of a `VersionedValue` with `is_tombstone=True` and `value=None`; no data is physically removed
- Source: entries/2026/06/05/key-value-store-key_value_store.md

### [ACCEPT] kv-read-repair-on-get
Every `get` performs read repair — pushing non-dominated, non-tombstone versions back to stale replicas as a side effect of the read path
- Source: entries/2026/06/05/key-value-store-key_value_store.md

### [ACCEPT] kv-vector-clock-immutable
`VectorClock` operations (`increment`, `merge`, `prune`) return new instances rather than mutating in place, avoiding aliasing bugs across replicas
- Source: entries/2026/06/05/key-value-store-key_value_store.md

### [ACCEPT] kv-merkle-tree-brute-force-diff
`MerkleTree.find_differences` checks root hashes for fast equality but falls back to key-by-key comparison on mismatch, not O(log n) tree walking
- Source: entries/2026/06/05/key-value-store-key_value_store.md

### [ACCEPT] kv-150-vnodes-per-node
Consistent hash ring uses 150 virtual nodes per physical node; `_get_preference_list` deduplicates by physical node ID when walking the ring
- Source: entries/2026/06/05/key-value-store-key_value_store.md

### [ACCEPT] kv-local-put-vs-raw-separation
`local_put` increments the vector clock (for coordinator-originated writes); `local_put_raw` accepts pre-built `VersionedValue` without advancing causality (for replication, read-repair, anti-entropy)
- Source: entries/2026/06/05/key-value-store-key_value_store.md

### [ACCEPT] kv-hinted-handoff-not-counted-in-quorum
Hinted handoff writes to backup nodes are fire-and-forget and do not count toward the W quorum acknowledgment count
- Source: entries/2026/06/05/key-value-store-key_value_store.md

### [ACCEPT] metrics-series-sorted-invariant
Every time series in `_data` is maintained in sorted timestamp order; `ingest` (bisect insertion), `downsample` (re-sort), and `apply_retention` (bisect truncation) all preserve this invariant
- Source: entries/2026/06/05/metrics-monitoring-and-alerting-metrics.md

### [ACCEPT] metrics-alert-state-machine-four-states
Alert evaluation follows a four-state machine (OK → PENDING → ALERTING → RESOLVED) where PENDING requires `duration_seconds` to elapse before transitioning to ALERTING; PENDING resets to OK (not RESOLVED) if the condition clears early
- Source: entries/2026/06/05/metrics-monitoring-and-alerting-metrics.md

### [ACCEPT] metrics-tag-matching-is-subset
`_matching_keys` performs subset matching: a query with `tags_filter={"a": 1}` matches any series whose tags include `a=1`, regardless of additional tags present
- Source: entries/2026/06/05/metrics-monitoring-and-alerting-metrics.md

### [ACCEPT] metrics-condition-uses-window-average
Threshold alert conditions (`gt`, `lt`, `gte`, `lte`) compare against the average of all points in the lookback window (sized `max(duration_seconds, 60)`), not the most recent value
- Source: entries/2026/06/05/metrics-monitoring-and-alerting-metrics.md

### [ACCEPT] metrics-downsample-two-tier
Downsampling uses two age-based tiers: 1–7 days old → 5-minute buckets, >7 days old → 1-hour buckets, both using averaging which loses min/max/percentile fidelity
- Source: entries/2026/06/05/metrics-monitoring-and-alerting-metrics.md

### [ACCEPT] metrics-series-keyed-by-frozen-tags
Series are stored keyed by `(metric_name, frozenset(tags.items()))`, so each unique tag combination creates a distinct time series — the standard cardinality model
- Source: entries/2026/06/05/metrics-monitoring-and-alerting-metrics.md

### [ACCEPT] nearby-friends-grid-cell-size-tied-to-threshold
Grid cell size is `distance_threshold_km / 111.0` degrees (111 km ≈ 1° latitude), meaning changing the distance threshold automatically rescales the spatial index
- Source: entries/2026/06/05/nearby-friends-nearby_friends.md

### [ACCEPT] nearby-friends-friendship-always-symmetric
`add_friendship` and `remove_friendship` always update both directions; there is no one-way follow relationship
- Source: entries/2026/06/05/nearby-friends-nearby_friends.md

### [ACCEPT] nearby-friends-update-location-skips-grid-filtering
`update_location` iterates all friends and computes haversine for each (O(friends)), while `get_nearby_friends` uses the grid index to narrow candidates before distance checks
- Source: entries/2026/06/05/nearby-friends-nearby_friends.md

### [ACCEPT] nearby-friends-staleness-enforced-on-both-paths
Both notification (`update_location`) and query (`get_nearby_friends`) paths reject friend locations older than `location_ttl_seconds`
- Source: entries/2026/06/05/nearby-friends-nearby_friends.md

### [ACCEPT] nearby-friends-bidirectional-visibility
For user A to see friend B as nearby, both A and B must have location sharing enabled and fresh locations; checked on both update and query paths
- Source: entries/2026/06/05/nearby-friends-nearby_friends.md

### [ACCEPT] nearby-friends-history-bounded-100
Per-user location history uses `deque(maxlen=100)`, silently dropping oldest entries to prevent unbounded memory growth
- Source: entries/2026/06/05/nearby-friends-nearby_friends.md

### [ACCEPT] news-feed-fan-out-write-pushes-ids
Fan-out-on-write pushes post IDs (not full post objects) into followers' deques at write time, deferring hydration to read time so engagement metrics are always current
- Source: entries/2026/06/05/news-feed-system-news_feed.md

### [ACCEPT] news-feed-hybrid-splits-on-follower-count
Hybrid strategy uses `celebrity_threshold` (default 1000) to decide push vs. pull per-author; the threshold is evaluated at write time and not retroactively applied to already-cached posts
- Source: entries/2026/06/05/news-feed-system-news_feed.md

### [ACCEPT] news-feed-pull-uses-heap-merge
Fan-out-on-read merges all followed users' timelines via `heapq.merge` with negated timestamps, avoiding full materialization of all posts
- Source: entries/2026/06/05/news-feed-system-news_feed.md

### [ACCEPT] news-feed-only-exception-is-missing-post-comment
The only raised exception in the module is `ValueError` from `PostStore.add_comment` when the target post doesn't exist; all other error cases return sentinel values or silently skip
- Source: entries/2026/06/05/news-feed-system-news_feed.md

### [ACCEPT] news-feed-cache-is-bounded-deque
Feed caches use `deque(maxlen=cache_size)` which silently drops the oldest post IDs when full, providing implicit eviction without explicit cache management
- Source: entries/2026/06/05/news-feed-system-news_feed.md

### [ACCEPT] news-feed-unfollow-purges-via-linear-scan
`_remove_author_from_cache` does a full linear scan and rebuild of the follower's deque to purge the unfollowed author's posts
- Source: entries/2026/06/05/news-feed-system-news_feed.md

### [ACCEPT] notif-rate-limit-on-success-only
Rate limit budget (`RateLimiter.record`) is consumed only after successful delivery, not on failed send attempts — failed attempts don't eat rate limit quota
- Source: entries/2026/06/05/notification-system-notification_system.md

### [ACCEPT] notif-priority-heap-stable
The priority queue uses a three-element key `(priority, timestamp, seq)` where `seq` is a monotonic counter, ensuring stable FIFO ordering within the same priority level
- Source: entries/2026/06/05/notification-system-notification_system.md

### [ACCEPT] notif-exponential-backoff-with-jitter
Delivery retry uses exponential backoff (`2^(retry-1)`) multiplied by a random jitter factor in [0.5, 1.5] to prevent thundering herd on provider recovery
- Source: entries/2026/06/05/notification-system-notification_system.md

### [ACCEPT] notif-two-phase-rate-limit
Rate limits are checked both at `send()` time (fast rejection before queuing) and again at `process_queue()` time (because budget may have been consumed between enqueue and delivery)
- Source: entries/2026/06/05/notification-system-notification_system.md

### [ACCEPT] notif-template-rendering-strict
`TemplateRegistry.render` raises `KeyError` for missing templates or missing context variables — fail-fast with no silent defaults, but uncaught in `process_queue` so bad templates crash queue processing
- Source: entries/2026/06/05/notification-system-notification_system.md

### [ACCEPT] notif-quiet-hours-midnight-crossing
`_is_quiet_hours` handles overnight spans (start > end, e.g., 22–8) with `hour >= start or hour < end`, correctly wrapping across midnight
- Source: entries/2026/06/05/notification-system-notification_system.md

### [ACCEPT] notif-group-key-unused
`_pending_groups` dict and `_group_window` field are initialized in `NotificationService.__init__` but never used — appears to be a planned notification batching/digest feature that was never implemented
- Source: entries/2026/06/05/notification-system-notification_system.md

### [ACCEPT] notif-caller-controls-time
`process_queue` takes `current_time` as an explicit parameter rather than reading the system clock, making the notification system fully deterministic and testable
- Source: entries/2026/06/05/notification-system-notification_system.md


### [ACCEPT] payment-idempotency-is-key-based
Idempotency is enforced by mapping client-provided keys to payment IDs in `_idempotency`; a repeated key returns the original `Payment` without re-processing
- Source: entries/2026/06/05/payment-system-payment_system.md

### [ACCEPT] balance-derived-from-ledger
Account balances are computed by scanning the full ledger on every call to `get_balance`, not cached; the ledger is the single source of truth
- Source: entries/2026/06/05/payment-system-payment_system.md

### [ACCEPT] double-entry-invariant
Every money movement (initial funding, payment, refund) creates exactly two ledger entries — one debit and one credit — and `verify_ledger_integrity` audits that total debits equal total credits
- Source: entries/2026/06/05/payment-system-payment_system.md

### [ACCEPT] webhook-errors-swallowed
Webhook callback exceptions are caught with bare `except Exception: pass`, making observer failures invisible but preventing them from disrupting payment processing
- Source: entries/2026/06/05/payment-system-payment_system.md

### [ACCEPT] retry-converts-timeout-to-failure
`_call_processor_with_retry` retries only on `"timeout"` status with exponential backoff (0.1s × 2^attempt); after `_max_retries` exhausted timeouts, it returns `{"status": "failure"}` — the caller never sees a timeout as a final result
- Source: entries/2026/06/05/payment-system-payment_system.md

### [ACCEPT] payment-error-strategy-split
Programming errors (unknown account, duplicate account, invalid refund state) raise `ValueError`; business-rule failures (insufficient balance, currency mismatch, processor failure) return a `Payment` with `status="FAILED"`
- Source: entries/2026/06/05/payment-system-payment_system.md

### [ACCEPT] payment-processor-injectable
The external payment processor is injected via `set_processor()` as a callable returning a status dict, decoupling processing logic from payment orchestration
- Source: entries/2026/06/05/payment-system-payment_system.md

### [ACCEPT] payment-balance-check-not-atomic
`process_payment` checks balance before processing but the check and ledger write are not atomic — a race condition under concurrency that real systems solve with optimistic locking or serializable transactions
- Source: entries/2026/06/05/payment-system-payment_system.md

### [ACCEPT] geohash-encode-longitude-first
GeohashIndex.encode interleaves bits starting with longitude, matching the standard geohash specification; swapping the order would break neighbor computation and cross-index compatibility
- Source: entries/2026/06/05/proximity-service-proximity_service.md

### [ACCEPT] both-indexes-use-haversine-filter
Both GeohashIndex.nearby and Quadtree.query_range use the coarse-to-fine pattern: spatial structure narrows candidates, then haversine filters to exact distance
- Source: entries/2026/06/05/proximity-service-proximity_service.md

### [ACCEPT] geohash-nearby-prefix-scan-is-linear
GeohashIndex.nearby iterates all keys in `_index` to match prefixes, making it O(buckets) per query rather than O(1) — a deliberate simplification vs. a sorted/trie structure
- Source: entries/2026/06/05/proximity-service-proximity_service.md

### [ACCEPT] quadtree-no-remove
Quadtree supports insert and query but has no removal operation, unlike GeohashIndex which supports O(1) removal via its dual-map design
- Source: entries/2026/06/05/proximity-service-proximity_service.md

### [ACCEPT] precision-table-descending-radius
GeohashIndex._PRECISION_TABLE is ordered by descending radius threshold; _precision_for_radius returns the precision for the first threshold the radius exceeds, defaulting to maximum precision 8
- Source: entries/2026/06/05/proximity-service-proximity_service.md

### [ACCEPT] proximity-no-coordinate-validation
Neither GeohashIndex nor Quadtree validates input coordinates; latitude outside [-90, 90] silently produces garbage geohashes, and Quadtree.insert silently returns False for out-of-bounds points
- Source: entries/2026/06/05/proximity-service-proximity_service.md

### [ACCEPT] quadtree-boundary-first-child-wins
Quadtree `_contains` uses inclusive bounds (`<=`), so a point on the boundary of multiple children is inserted into the first matching child in NW → NE → SW → SE order
- Source: entries/2026/06/05/proximity-service-proximity_service.md

### [ACCEPT] rate-limiter-four-algorithms
The rate limiter provides exactly four algorithms: token bucket, fixed window counter, sliding window log, and sliding window counter, all interchangeable via the `RateLimiter` ABC
- Source: entries/2026/06/05/rate-limiter-rate_limiter.md

### [ACCEPT] rate-limiter-time-injectable
Every rate limiter method that depends on wall-clock time accepts an optional `current_time` parameter defaulting to `time.time()`, making all algorithms deterministically testable
- Source: entries/2026/06/05/rate-limiter-rate_limiter.md

### [ACCEPT] rate-limiter-per-client-state
All rate limiting state is tracked per `client_id` string; there is no global (cross-client) rate limiting
- Source: entries/2026/06/05/rate-limiter-rate_limiter.md

### [ACCEPT] rate-limiter-middleware-path-routing
`HTTPRateLimitMiddleware` supports per-path rate limiting via `path_rules` dict, falling back to `default_limiter` for unmatched paths
- Source: entries/2026/06/05/rate-limiter-rate_limiter.md

### [ACCEPT] rate-limiter-no-persistence
All rate limiter state is in-memory with no persistence or distribution mechanism; the implementation is single-process only
- Source: entries/2026/06/05/rate-limiter-rate_limiter.md

### [ACCEPT] fixed-window-single-key-gc
FixedWindowCounterLimiter replaces its entire counter dict with just the current window's entry on every allowed request, effectively garbage-collecting all old windows immediately
- Source: entries/2026/06/05/rate-limiter-rate_limiter.md

### [ACCEPT] sliding-window-counter-weighted-approximation
SlidingWindowCounterLimiter approximates a sliding window by keeping current and previous fixed-window counters, weighting the previous window's count by `(1 - elapsed_fraction)` as time progresses
- Source: entries/2026/06/05/rate-limiter-rate_limiter.md

### [REJECT] verify-py-is-empty
Too ephemeral — an empty file is not a codebase invariant; it will either get filled in or deleted, and a belief about it adds no useful guidance
- Source: entries/2026/06/05/rate-limiter-verify.md

### [REJECT] verify-py-is-unique
Trivial structural observation that provides no actionable guidance and will become stale as soon as another module adds a verify.py
- Source: entries/2026/06/05/rate-limiter-verify.md

### [ACCEPT] leaderboard-negated-score-ordering
Scores are stored negated in `_entries` so that `SortedList`'s ascending order yields descending-score ranking; the tiebreaker is timestamp-ascending (earlier timestamp = higher rank)
- Source: entries/2026/06/05/realtime-gaming-leaderboard-leaderboard.md

### [ACCEPT] leaderboard-dual-index-consistency
Every mutation in `Leaderboard` must update both `_entries` (SortedList) and `_players` (dict) atomically; a desync between them is a correctness bug that surfaces as `ValueError` on the next operation
- Source: entries/2026/06/05/realtime-gaming-leaderboard-leaderboard.md

### [ACCEPT] leaderboard-rank-is-one-based
All public-facing rank values are 1-based; internal `SortedList.index()` returns 0-based and every call site adds 1
- Source: entries/2026/06/05/realtime-gaming-leaderboard-leaderboard.md

### [ACCEPT] leaderboard-range-query-is-O-m-log-n
`range_by_score` calls `_entries.index(entry)` per matched result to compute rank, making it O(m log n) rather than the O(m + log n) achievable by tracking the start index and incrementing
- Source: entries/2026/06/05/realtime-gaming-leaderboard-leaderboard.md

### [ACCEPT] leaderboard-sortedcontainers-dependency
`sortedcontainers.SortedList` is the sole external dependency and the load-bearing data structure; it provides O(log n) add/remove/index that makes the entire leaderboard design viable
- Source: entries/2026/06/05/realtime-gaming-leaderboard-leaderboard.md

### [ACCEPT] leaderboard-update-by-remove-reinsert
`update_score` removes the old entry from the sorted list and inserts a new one rather than mutating in place, which is the correct approach since in-place mutation would violate sort order
- Source: entries/2026/06/05/realtime-gaming-leaderboard-leaderboard.md


### [ACCEPT] sdi-repo-is-25-independent-modules
Each of the 25 system design implementations is fully self-contained in its own directory with no shared libraries, no common base classes, and no cross-module imports.
- Source: entries/2026/06/05/scan-sdi-implementations.md

### [ACCEPT] sdi-repo-lifecycle-convention
Every module follows a consistent lifecycle: `plan.md` → `plan_review.md` → Python implementation → pytest-based tests.
- Source: entries/2026/06/05/scan-sdi-implementations.md

### [ACCEPT] sdi-implementations-are-in-process-simulations
All implementations simulate distributed behavior in-process with no real network or storage dependencies — they are pedagogical reference implementations, not production systems.
- Source: entries/2026/06/05/scan-sdi-implementations.md

### [ACCEPT] run-tests-is-repo-convention
Multiple modules use identical `run_tests.py` wrapper scripts that derive the test file path via `__file__.replace("run_tests.py", ...)` and invoke `pytest.main()` programmatically.
- Source: entries/2026/06/05/search-autocomplete-run_tests.md

### [REJECT] run-tests-relies-on-own-filename
Too narrow — this is an implementation detail of a utility script, not an architectural invariant a developer needs to know to work in the codebase effectively.
- Source: entries/2026/06/05/search-autocomplete-run_tests.md

### [REJECT] run-tests-propagates-exit-code
Trivial — `sys.exit(pytest.main(...))` is standard boilerplate, not a codebase-specific insight.
- Source: entries/2026/06/05/search-autocomplete-run_tests.md

### [REJECT] run-tests-has-no-error-handling
Not actionable — describing the absence of error handling in a 6-line wrapper script is trivia, not an invariant.
- Source: entries/2026/06/05/search-autocomplete-run_tests.md

### [ACCEPT] autocomplete-cache-consistency
Every trie mutation (insert, increment, delete) immediately rebuilds `top_k_cache` for all ancestor nodes via `_update_caches_on_path`; caches are never stale between operations.
- Source: entries/2026/06/05/search-autocomplete-search_autocomplete.md

### [ACCEPT] autocomplete-decay-is-read-time
Time decay is computed lazily at query time in `search_prefix` using `raw_freq * decay_factor^hours_elapsed`; raw frequencies stored in the trie are never modified by decay.
- Source: entries/2026/06/05/search-autocomplete-search_autocomplete.md

### [ACCEPT] autocomplete-blocklist-is-substring
The blocklist filter removes results where any blocklisted term appears as a **substring** of the query, not just exact matches.
- Source: entries/2026/06/05/search-autocomplete-search_autocomplete.md

### [ACCEPT] autocomplete-delete-is-soft
Deleting a query zeroes its frequency and unsets `is_end` but does not remove trie nodes from the tree structure — deleted queries leave structural residue.
- Source: entries/2026/06/05/search-autocomplete-search_autocomplete.md

### [ACCEPT] autocomplete-normalize-at-boundary
All public methods in `AutocompleteTrie` lowercase and truncate queries to 200 characters before any trie operation; internal methods assume normalized input.
- Source: entries/2026/06/05/search-autocomplete-search_autocomplete.md

### [ACCEPT] autocomplete-service-overfetches-for-blocklist
`AutocompleteService` requests `k + len(blocklist) * 2` results from the trie to compensate for results that will be removed by blocklist filtering.
- Source: entries/2026/06/05/search-autocomplete-search_autocomplete.md

### [ACCEPT] autocomplete-trie-k-floor-is-10
The `AutocompleteService` constructor forces `k=max(k, 10)` for the internal trie, even if the service default is smaller, to provide headroom for blocklist filtering.
- Source: entries/2026/06/05/search-autocomplete-search_autocomplete.md

### [ACCEPT] autocomplete-fuzzy-is-last-char-only
`fuzzy_suggest()` only tries single-character edits (substitution, deletion, insertion) on the **last character** of the prefix — it is not a full edit-distance search.
- Source: entries/2026/06/05/search-autocomplete-search_autocomplete.md

### [ACCEPT] stock-exchange-fifo-matching
Orders at the same price level are matched in FIFO order, enforced by `deque` append/popleft semantics — this is the price-time priority algorithm.
- Source: entries/2026/06/05/stock-exchange-solution.md

### [ACCEPT] stock-exchange-resting-price-execution
Trades always execute at the resting (maker) order's price, not the incoming (taker) order's price — standard exchange behavior.
- Source: entries/2026/06/05/stock-exchange-solution.md

### [ACCEPT] stock-exchange-market-orders-never-rest
Market orders that can't be fully filled have their remainder cancelled; they are never added to the order book.
- Source: entries/2026/06/05/stock-exchange-solution.md

### [ACCEPT] stock-exchange-trade-counter-global
`Trade._counter` is a class-level counter that monotonically increases across all symbols and never resets, making trade IDs globally sequential but test-order-dependent.
- Source: entries/2026/06/05/stock-exchange-solution.md

### [ACCEPT] stock-exchange-no-input-validation
The matching engine performs no validation on order fields (quantity, price, side); callers are trusted to provide valid inputs.
- Source: entries/2026/06/05/stock-exchange-solution.md

### [ACCEPT] stock-exchange-price-sort-is-full-resort
`_add_to_book` re-sorts the entire price list on every insertion (O(n log n)) rather than using `bisect.insort` — acceptable for interview scope but not production-grade.
- Source: entries/2026/06/05/stock-exchange-solution.md

### [ACCEPT] stock-exchange-aggressive-then-rest
`place_order` first attempts to match the incoming order against the opposite side (`_match_order`), then adds any unfilled remainder to the book — the standard aggressor/resting model.
- Source: entries/2026/06/05/stock-exchange-solution.md

### [ACCEPT] s3-unversioned-put-replaces
In unversioned buckets, `put_object` replaces the entire version list with a single-element list, making previous data unrecoverable.
- Source: entries/2026/06/05/s3-object-storage-s3_object_storage.md

### [ACCEPT] s3-delete-marker-hides-not-removes
Deleting a versioned object appends a delete marker; all prior versions remain accessible by explicit `version_id`.
- Source: entries/2026/06/05/s3-object-storage-s3_object_storage.md

### [ACCEPT] s3-multipart-sorts-by-part-number
`complete_multipart` assembles parts by sorting on part number, so upload order is irrelevant.
- Source: entries/2026/06/05/s3-object-storage-s3_object_storage.md

### [ACCEPT] s3-presigned-secret-is-class-level
`_SECRET` is generated once per class load (`uuid4().hex`), not per instance, so all `ObjectStorage` instances in the same process share the same HMAC signing key.
- Source: entries/2026/06/05/s3-object-storage-s3_object_storage.md

### [ACCEPT] s3-policy-default-allow
`check_bucket_policy` returns `True` (allow) when no policies exist or when no policy matches — opposite of AWS IAM's default-deny.
- Source: entries/2026/06/05/s3-object-storage-s3_object_storage.md

### [ACCEPT] s3-version-list-append-only
Each object key maps to a `list[ObjectVersion]` where the latest version is always `versions[-1]`; versioned puts append, unversioned puts replace the entire list.
- Source: entries/2026/06/05/s3-object-storage-s3_object_storage.md

### [ACCEPT] s3-get-returns-none-for-missing
`get_object` and `head_object` return `None` for missing objects rather than raising, matching S3's HTTP 404 semantics.
- Source: entries/2026/06/05/s3-object-storage-s3_object_storage.md

### [ACCEPT] s3-bucket-delete-requires-empty
Bucket deletion requires the bucket to be logically empty: no live objects, and in versioned buckets, no non-marker versions at all.
- Source: entries/2026/06/05/s3-object-storage-s3_object_storage.md

### [ACCEPT] sdi-implementations-use-only-stdlib
All implementations use only Python standard library imports (no external dependencies), keeping each module self-contained and dependency-free.
- Source: entries/2026/06/05/scan-sdi-implementations.md


### [ACCEPT] payment-balance-never-cached
`PaymentSystem.get_balance()` derives balance by scanning the full ledger on every call; no running total or balance cache is maintained.
- Source: entries/2026/06/05/topic-consistency-models.md

### [ACCEPT] payment-overdraft-not-atomic
The overdraft check in `process_payment` reads balance and then processes without atomicity — a TOCTOU race in any concurrent context.
- Source: entries/2026/06/05/topic-consistency-models.md

### [ACCEPT] kv-store-quorum-overlap-not-enforced
The key-value store's W+R>N read-write overlap guarantee is assumed by callers but never validated in code.
- Source: entries/2026/06/05/topic-consistency-models.md

### [ACCEPT] kv-store-deletes-use-tombstones
KV store uses tombstones for deletes because a naive physical delete would be resurrected by anti-entropy from a replica that hasn't seen the deletion.
- Source: entries/2026/06/05/topic-consistency-models.md

### [ACCEPT] message-queue-two-tier-offsets
The message queue separates `current_offset` (advances on every `poll()`) from `committed_offset` (advances on explicit `commit()`), enabling consumer-chosen delivery semantics.
- Source: entries/2026/06/05/topic-consistency-models.md

### [ACCEPT] news-feed-celebrity-threshold-at-write-time
Fan-out strategy selection via `celebrity_threshold` is evaluated at write time and not retroactively applied to existing posts.
- Source: entries/2026/06/05/topic-consistency-models.md

### [ACCEPT] news-feed-cache-silently-drops-oldest
News feed caches use `deque(maxlen=cache_size)` which silently drops the oldest post IDs when full — no error or notification on eviction.
- Source: entries/2026/06/05/topic-consistency-models.md

### [ACCEPT] chat-dual-ordering-sequence-and-lamport
Chat system uses per-conversation monotonic sequence numbers for total order within a conversation and Lamport timestamps for cross-conversation causal ordering.
- Source: entries/2026/06/05/topic-consistency-models.md

### [ACCEPT] chat-soft-delete-preserves-sequence
Chat soft deletes keep the message in the list with `deleted=True` and content replaced with `'[deleted]'` to preserve dense sequence number continuity.
- Source: entries/2026/06/05/topic-consistency-models.md

### [ACCEPT] ad-click-dedup-global-not-per-ad
Ad click deduplication keys on `event_id` alone (global registry), not per-ad, converting at-least-once delivery into exactly-once aggregation.
- Source: entries/2026/06/05/topic-consistency-models.md

### [ACCEPT] ad-click-window-lifecycle-one-directional
Ad click aggregation windows follow a one-directional lifecycle: OPEN → CLOSED → FINALIZED, with only `advance_watermark` able to finalize — event processing alone never does.
- Source: entries/2026/06/05/topic-consistency-models.md

### [ACCEPT] hotel-reservation-optimistic-locking
Hotel reservation uses optimistic concurrency control with per-date version counters; version mismatch between read and write raises `ConcurrencyError`.
- Source: entries/2026/06/05/topic-consistency-models.md

### [ACCEPT] google-drive-restore-is-append-only
`restore_version` in Google Drive doesn't roll back — it calls `update_file` with old content, creating a new version and preserving append-only history.
- Source: entries/2026/06/05/topic-consistency-models.md

### [ACCEPT] plans-are-prescriptive
Plans specify exact method signatures, data models, and assertion examples, leaving implementations with near-zero design freedom at the API level.
- Source: entries/2026/06/05/topic-plan-to-implementation-fidelity.md

### [ACCEPT] no-divergence-annotations
The rate limiter, payment system, and chat system contain zero TODO/FIXME/HACK comments; deviations from plans are undocumented in the code itself.
- Source: entries/2026/06/05/topic-plan-to-implementation-fidelity.md

### [ACCEPT] current-time-fallback-inconsistent
Rate limiter defaults `current_time` to `time.time()` while chat system defaults to `0.0`, creating different behavior when the parameter is omitted.
- Source: entries/2026/06/05/topic-plan-to-implementation-fidelity.md

### [REJECT] payment-balance-is-O-n
Duplicate of `payment-balance-never-cached` which captures the same claim more precisely.
- Source: entries/2026/06/05/topic-plan-to-implementation-fidelity.md

### [ACCEPT] plan-review-documents-known-gaps
The `plan_review.md` pattern explicitly catalogues TOCTOU, blocking sleep, and permanent-failure-on-idempotency-key as accepted divergences rather than bugs.
- Source: entries/2026/06/05/topic-plan-to-implementation-fidelity.md

### [ACCEPT] snowflake-sequence-max-4096-per-ms
SnowflakeGenerator allows at most 4096 IDs per millisecond per worker (12-bit sequence 0–4095), then spin-waits for the next millisecond.
- Source: entries/2026/06/05/unique-id-generator-unique_id_generator.md

### [ACCEPT] snowflake-rejects-backward-clock
SnowflakeGenerator raises RuntimeError on clock regression, while FlakeIDGenerator and ULIDGenerator silently handle it (spin-wait and random increment respectively).
- Source: entries/2026/06/05/unique-id-generator-unique_id_generator.md

### [ACCEPT] ulid-monotonic-within-millisecond
ULIDGenerator maintains sort order within the same millisecond by incrementing the random component rather than generating new random bits.
- Source: entries/2026/06/05/unique-id-generator-unique_id_generator.md

### [ACCEPT] all-stateful-generators-thread-safe
Every generator with mutable state (Snowflake, Ticket, Flake, ULID, Coordinator) protects it with a `threading.Lock`.
- Source: entries/2026/06/05/unique-id-generator-unique_id_generator.md

### [ACCEPT] ticket-server-first-value-equals-offset
TicketServerGenerator initializes its counter to `offset - step` so the first `generate()` call returns exactly `offset`.
- Source: entries/2026/06/05/unique-id-generator-unique_id_generator.md

### [ACCEPT] url-shortener-two-strategies
URLShortener supports two short code generation strategies: "counter" (monotonic base62) and "hash" (truncated SHA-256 with collision retry), selected at construction time.
- Source: entries/2026/06/05/url-shortener-url_shortener.md

### [ACCEPT] url-shortener-expiration-lazy
Expired URLs are never eagerly removed from storage; expiration is checked at read time in `redirect` and `list_urls` — no background reaper.
- Source: entries/2026/06/05/url-shortener-url_shortener.md

### [ACCEPT] url-shortener-click-history-bounded
Click history per URL is capped at 1000 entries; older events are silently dropped on each redirect.
- Source: entries/2026/06/05/url-shortener-url_shortener.md

### [ACCEPT] url-shortener-rate-limit-sliding-window
Rate limiting uses a per-creator sliding window of 60 seconds, pruned eagerly on each `_check_rate_limit` call.
- Source: entries/2026/06/05/url-shortener-url_shortener.md

### [ACCEPT] url-shortener-hash-collision-loop-unbounded
The hash strategy's collision resolution loop has no max-attempts guard — it will loop indefinitely if the keyspace is near-saturated.
- Source: entries/2026/06/05/url-shortener-url_shortener.md

### [REJECT] url-shortener-no-dedup-counter
Too narrow — this is implied by the counter strategy description in `url-shortener-two-strategies` (monotonic counter produces distinct codes by definition).
- Source: entries/2026/06/05/url-shortener-url_shortener.md


### [ACCEPT] bloom-filter-prevents-frontier-duplicates
Every URL is added to the Bloom filter before being pushed into the frontier, so the frontier never enqueues a URL that has already been seen (modulo false positives, which cause URLs to be permanently skipped).
- Source: entries/2026/06/05/web-crawler-web_crawler.md

### [ACCEPT] simhash-threshold-default-3
Two pages are considered near-duplicates if their 64-bit SimHash fingerprints differ in 3 or fewer bits, configurable via the `duplicate_threshold` parameter.
- Source: entries/2026/06/05/web-crawler-web_crawler.md

### [ACCEPT] url-frontier-strategy-via-sequence-sign
BFS vs DFS is implemented by flipping the sign of the sequence number in the heapq min-heap — ascending for BFS (FIFO), negated for DFS (LIFO) — not by swapping data structures.
- Source: entries/2026/06/05/web-crawler-web_crawler.md

### [ACCEPT] robots-longest-prefix-match
`RobotsParser.is_allowed` resolves conflicting allow/disallow rules by selecting the rule with the longest matching path prefix, falling back from agent-specific rules to `*`.
- Source: entries/2026/06/05/web-crawler-web_crawler.md

### [ACCEPT] crawl-uses-simulated-clock
The crawl loop advances a `sim_time` counter for politeness scheduling rather than calling `time.sleep()`, making the simulation deterministic; real `time.time()` is only used to measure total crawl duration for stats.
- Source: entries/2026/06/05/web-crawler-web_crawler.md

### [ACCEPT] crawler-normalize-once-convention
URLs are normalized via `URLNormalizer.normalize()` at the point of insertion into the Bloom filter, frontier, and SimulatedWeb; downstream code assumes URLs are already canonical.
- Source: entries/2026/06/05/web-crawler-web_crawler.md

### [ACCEPT] crawler-layered-dedup-bloom-then-simhash
Deduplication is two-tiered: the Bloom filter rejects exact-URL revisits first (O(k) per check), then SimHash rejects near-duplicate content (linear scan over seen hashes) — content dedup only runs for URLs that pass the Bloom filter.
- Source: entries/2026/06/05/web-crawler-web_crawler.md

### [ACCEPT] bloom-filter-add-couples-with-might-contain
`BloomFilter.add()` internally calls `might_contain()` and then compensates for its side effect on `_negative_check_count` by decrementing — if `might_contain`'s bookkeeping logic changes, `add` breaks silently.
- Source: entries/2026/06/05/web-crawler-web_crawler.md


