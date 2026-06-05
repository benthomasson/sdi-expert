# File: hotel-reservation-system/hotel_reservation.py

**Date:** 2026-06-05
**Time:** 13:24

## Purpose

This file is a self-contained implementation of a **hotel reservation system** designed for system design interview preparation. It owns the full booking lifecycle: hotel/room catalog management, availability search with dynamic pricing, reservation creation with optimistic concurrency control, and cancellation with tiered refund policies. It's a single-file teaching implementation — no database, no network — that demonstrates the core data modeling and concurrency concepts you'd discuss in an SDI.

## Key Components

### Data Classes

- **`Hotel`** — Minimal hotel entity: ID, name, city. City is the only searchable attribute.
- **`RoomType`** — Represents a category of room within a hotel (e.g., "Standard", "Deluxe"). Keyed by `(hotel_id, type_id)`. `total_rooms` is the capacity ceiling for availability calculations.
- **`Reservation`** — A confirmed booking. Tracks the date range, computed total price, and status (`"CONFIRMED"` or `"CANCELLED"`). Uses a UUID for the reservation ID.

### Custom Exceptions

- **`AvailabilityError`** — Raised when no rooms remain for a requested date.
- **`ConcurrencyError`** — Raised when optimistic locking detects a version mismatch (another booking modified inventory between read and write).

### `HotelReservationSystem`

The main class. All state lives in dictionaries:

| Field | Key | Value | Purpose |
|-------|-----|-------|---------|
| `hotels` | `hotel_id` | `Hotel` | Hotel catalog |
| `room_types` | `(hotel_id, type_id)` | `RoomType` | Room catalog |
| `inventory` | `(hotel_id, type_id, date)` | `{"booked": int, "version": int}` | Per-date availability tracking |
| `reservations` | `reservation_id` | `Reservation` | Booking records |
| `idempotency_keys` | `key` | `reservation_id` | Dedup map for retried requests |
| `seasonal_pricing` | `(hotel_id, type_id)` | `[(start, end, multiplier)]` | Price overrides by date range |

**Key methods:**

- **`search(check_in, check_out, city?, room_type?, max_price?)`** — Scans all room types, filters by city/type/price, returns availability and average nightly price. Availability is the *minimum* across all nights in the range (bottleneck date determines what's bookable).

- **`reserve(hotel_id, room_type_id, guest_name, check_in, check_out, idempotency_key?)`** — Two-phase booking with optimistic concurrency: reads inventory versions, verifies they haven't changed, then commits. Returns a `Reservation`.

- **`cancel(reservation_id, cancel_time?)`** — Releases inventory, computes a tiered refund (100% if 24h+ before check-in, 50% if 0–24h, 0% after check-in), sets status to `CANCELLED`.

- **`_get_price(hotel_id, type_id, date)`** — Computes the price for a single night by stacking two multipliers on the base price: seasonal pricing (first matching range wins) and dynamic occupancy-based surcharges (1.1x at 50%+, 1.3x at 70%+, 1.5x at 90%+).

### `_date_range(check_in, check_out)`

Module-level helper. Converts a `[check_in, check_out)` half-open date range into a list of date strings. Check-out date is excluded — matching hotel industry convention where you don't occupy the room on checkout day.

## Patterns

**Optimistic Concurrency Control (OCC):** The `reserve` method implements a read-validate-write cycle. It snapshots inventory versions in Phase 1, then in Phase 2 re-reads and checks they haven't changed before committing. In a real system, Phase 2 would be atomic (e.g., a SQL `UPDATE ... WHERE version = ?`). Here, because it's single-threaded in-process, the version check is demonstrative — it shows the *shape* of the pattern without actual thread contention.

**Idempotency Keys:** `reserve` accepts an optional `idempotency_key`. If a key has been seen before, it returns the existing reservation without double-booking. This models the real-world pattern for safely retrying payment/booking requests.

**Dynamic Pricing:** Price is a function of state (occupancy) rather than a static lookup. This means the price you see at search time can differ from what you pay at booking time if someone else books between your search and reserve — a realistic race condition in hotel systems.

**Lazy Inventory Initialization:** `_get_inventory` creates `{"booked": 0, "version": 0}` on first access for any `(hotel, type, date)` tuple. No need to pre-populate dates.

## Dependencies

**Imports:** Standard library only — `dataclasses`, `datetime`, `uuid`. No external dependencies.

**Imported by:** `test_hotel_reservation.py`, which likely contains a more thorough test suite beyond the inline `test_all()`.

## Flow

A typical booking flow:

1. **Setup:** `add_hotel` + `add_room_type` to populate the catalog.
2. **Search:** Client calls `search("2024-03-15", "2024-03-17", city="NYC")`. The system iterates all room types, filters by city, computes per-date availability (min across the range), computes average nightly price (which includes occupancy-based surcharges), and returns matching results.
3. **Reserve:** Client calls `reserve("h1", "std", "Alice", "2024-03-15", "2024-03-17")`. The system reads and snapshots versions for March 15 and 16, verifies no version drift, computes total price, increments `booked` and `version` for both dates, creates a `Reservation`, and returns it.
4. **Cancel:** Client calls `cancel(reservation_id, cancel_time="2024-03-01")`. The system decrements `booked` for each date, bumps versions, computes refund based on time-to-check-in, and marks the reservation `CANCELLED`.

## Invariants

- **`booked <= total_rooms`** for every `(hotel, type, date)` — enforced by the availability check in `reserve`. Cancellation can drive `booked` below 0 if there's a bug, but no guard prevents that.
- **Check-out is exclusive** — `_date_range` generates `[check_in, check_out)`. A 1-night stay from March 15 to March 16 occupies only March 15.
- **Seasonal pricing uses first-match** — if date ranges overlap in `seasonal_pricing`, the first appended rule wins (`break` after first match in `_get_price`).
- **Version monotonically increases** — every `reserve` or `cancel` bumps the version for affected dates. Versions never decrease.
- **Idempotency is key-scoped, not content-scoped** — replaying the same `idempotency_key` returns the original reservation regardless of whether the other parameters (guest name, dates) differ.
- **Cancellation is terminal** — a cancelled reservation cannot be re-confirmed; attempting to cancel again raises `ValueError`.

## Error Handling

- **`AvailabilityError`** — Raised by `reserve` when any date in the range has zero rooms left. Includes the specific date that's full.
- **`ConcurrencyError`** — Raised by `reserve` on version mismatch. In practice, this is hard to trigger in the single-threaded implementation since Phase 1 and Phase 2 run back-to-back with no interleaving.
- **`ValueError`** — Raised for invalid inputs: unknown room type, unknown reservation ID, or double-cancellation.
- **No rollback on partial failure:** If the version check fails on the second date after the first passes, no inventory was modified yet (versions are only checked, not written, in Phase 2). The commit loop is all-or-nothing only because it runs after all checks pass. However, if `_get_price` or `uuid.uuid4()` somehow threw mid-commit, inventory would be left in an inconsistent state — there's no transaction boundary.

## Topics to Explore

- [file] `hotel-reservation-system/test_hotel_reservation.py` — The external test suite likely covers edge cases beyond the inline `test_all()`, possibly concurrency simulation with threading
- [function] `hotel-reservation-system/hotel_reservation.py:reserve` — The two-phase OCC implementation is the architectural centerpiece; worth tracing how it would behave under real concurrency (e.g., with threading or async)
- [general] `occ-vs-pessimistic-locking` — Compare this optimistic approach to pessimistic locking (SELECT FOR UPDATE); understand when each is preferred in real hotel/booking systems
- [general] `inventory-overbooking-risk` — The gap between Phase 1 (read) and Phase 2 (verify) is zero in single-threaded code, but in a distributed system this is where overbooking bugs live — worth studying how databases close this gap
- [file] `digital-wallet/wallet.py` — Another system in this repo that likely deals with concurrency and idempotency in financial transactions; compare the patterns

## Beliefs

- `hotel-occ-two-phase-reserve` — `reserve()` implements optimistic concurrency via a read-then-verify-version pattern, but both phases execute synchronously with no actual interleaving, making the version check demonstrative rather than functional
- `hotel-pricing-stacks-multipliers` — Dynamic occupancy pricing and seasonal pricing multiply independently on the base price; a 90% occupied room during peak season costs `base * seasonal_mult * 1.5`
- `hotel-checkout-exclusive` — Date ranges are half-open `[check_in, check_out)`: a booking from March 15 to March 16 occupies inventory on March 15 only
- `hotel-idempotency-ignores-params` — The idempotency key check returns the cached reservation without validating that guest name, dates, or room type match the original request
- `hotel-cancel-no-underflow-guard` — `cancel()` decrements `booked` without checking for `booked >= 0`, so a double-cancel bug (if the status check were bypassed) could produce negative inventory counts

