# File: stock-exchange/solution.py

**Date:** 2026-06-05
**Time:** 13:20

# `stock-exchange/solution.py` — Stock Exchange Matching Engine

## Purpose

This file implements a **limit order book (LOB) matching engine** — the core component of a stock exchange that accepts buy/sell orders and matches them into trades. It's a system design interview implementation demonstrating how exchanges achieve price-time priority matching, the fundamental algorithm behind every modern stock exchange.

The file owns three responsibilities:
1. **Order lifecycle management** — tracking orders from NEW through PARTIALLY_FILLED to FILLED or CANCELLED
2. **Price-time priority matching** — executing trades at the best available price, with earlier orders at the same price level filled first
3. **Multi-symbol routing** — the `Exchange` class dispatches orders to per-symbol `OrderBook` instances

## Key Components

### `Order`
A mutable order record. Notable contract details:
- `remaining` is a computed property (`quantity - filled_quantity`), not stored — so it's always consistent with fill state.
- `status` transitions: `NEW` → `PARTIALLY_FILLED` → `FILLED`, or `NEW`/`PARTIALLY_FILLED` → `CANCELLED`. There's no explicit state machine enforcing this; `_update_status` and `cancel_order` handle transitions directly.
- `timestamp` is set via `time.time()` at construction. This gives FIFO ordering within a price level since orders at the same price are appended to a `deque`.

### `Trade`
An immutable record of a fill. Uses a class-level `_counter` for sequential trade IDs (`t1`, `t2`, ...). The `create` classmethod is the only intended constructor — it auto-generates the ID and timestamp.

**Warning for tests:** `_counter` is class-level and never resets. Tests that assert on specific trade IDs (e.g., `t1`) will break if test ordering changes or fixtures don't reset the counter.

### `OrderBook`
The core data structure — one per symbol. Internal state:

| Field | Type | Purpose |
|-------|------|---------|
| `_bids` | `dict[float, deque[Order]]` | Buy orders grouped by price |
| `_asks` | `dict[float, deque[Order]]` | Sell orders grouped by price |
| `_bid_prices` | `list[float]` | Sorted descending — `[0]` is best bid |
| `_ask_prices` | `list[float]` | Sorted ascending — `[0]` is best ask |
| `_orders` | `dict[str, Order]` | All orders by ID (including filled/cancelled) |
| `_trades` | `list[Trade]` | Full trade history |

Key methods:
- **`place_order(order)`** — The main entry point. Matches aggressively first, then rests any remaining quantity (limit orders) or cancels it (market orders).
- **`_match_order(order)`** — Walks the opposite side of the book, consuming resting orders at each price level until the incoming order is filled or no more matchable prices exist.
- **`cancel_order(order_id)`** — Removes a resting order from the book. Returns `False` for already-filled or already-cancelled orders.
- **`get_book_depth(levels)`** — Returns L2 market data: aggregated quantity per price level, top N levels each side.
- **`get_bbo()`** — Best bid and offer with spread.

### `Exchange`
A thin routing layer. Lazily creates `OrderBook` instances on first access to a symbol. This is the public API surface — callers interact with `Exchange`, not `OrderBook` directly (though the example code does grab the book for assertions).

## Patterns

**Price-time priority (FIFO).** Within each price level, orders are stored in a `deque` and consumed from the left (`popleft`). New orders append to the right. This gives strict time priority without needing to sort by timestamp.

**Aggressive-then-rest.** `place_order` first tries to match the incoming order against the opposite side (`_match_order`), then adds any unfilled remainder to the book. This is the standard exchange pattern — incoming orders are "aggressors" and existing orders are "resting."

**Sort-on-insert for price levels.** `_add_to_book` appends a price then re-sorts the entire price list. This is O(n log n) per insertion — fine for an interview implementation but a real exchange would use `bisect.insort` or a sorted container. The sorted lists keep `_bid_prices[0]` as best bid and `_ask_prices[0]` as best ask, making BBO lookups O(1).

**Passive matching price.** Trades always execute at the **resting order's price**, not the aggressor's. This is correct exchange behavior — if you place a buy at $151 and the best ask is $150, you get filled at $150.

## Dependencies

**Imports:** Only stdlib — `time` for timestamps and `deque` for FIFO queues at each price level. No external dependencies.

**Imported by:**
- `stock-exchange/test_exchange.py` — the primary test suite
- `distributed-message-queue/test_solution.py` — interesting cross-reference; the message queue tests apparently import from here, likely reusing the `Order`/`Trade` classes or testing integration scenarios

## Flow

A typical order lifecycle:

1. Caller creates an `Order` (status: `NEW`)
2. Calls `exchange.place_order(order)` → routes to `OrderBook.place_order`
3. `place_order` registers the order in `_orders`, then calls `_match_order`
4. `_match_order` walks the opposite side:
   - **Buy order**: iterates `_ask_prices` ascending (cheapest first). At each price level, consumes resting sells from the deque front until the buy is filled or the level is exhausted.
   - **Sell order**: iterates `_bid_prices` descending (most expensive first). Same consumption logic.
   - For limit orders, matching stops when the best available price crosses the limit price.
   - Each fill creates a `Trade`, updates both orders' `filled_quantity`, and calls `_update_status`.
5. Back in `place_order`: if the order has remaining quantity:
   - **Market order**: cancelled (no price to rest at)
   - **Limit order**: added to the book via `_add_to_book`
6. All generated trades are appended to `_trades` and returned to the caller.

## Invariants

- **Price-time priority**: bids sorted descending, asks sorted ascending. Best prices are always at index `[0]`. Within a price level, FIFO ordering via `deque`.
- **Resting price execution**: trades execute at the resting order's price, never the aggressor's.
- **Market orders never rest**: if a market order can't be fully filled, the remainder is cancelled — it never enters the book.
- **Empty level cleanup**: when the last order at a price level is consumed, the price is removed from both the dict and the sorted list. This keeps `_bid_prices[0]` / `_ask_prices[0]` always valid as BBO.
- **Order ID uniqueness assumed**: `_orders` is keyed by `order_id` with no duplicate check — callers must ensure unique IDs.
- **Filled orders are terminal**: `cancel_order` returns `False` for filled orders. No mechanism to modify a filled order.

## Error Handling

Minimal — consistent with an interview implementation:

- **`cancel_order`** returns `False` (not an exception) for orders that don't exist or are already terminal. This is a design choice — real exchanges use error codes.
- **`_remove_from_book`** swallows `ValueError` if the order isn't found in the deque. This is defensive against double-removal.
- **No validation** on order fields — no checks for negative quantities, zero prices, or invalid sides. The code trusts its callers.
- **No exception propagation** — nothing in this file raises. Errors are silent (return `False` / `None`) rather than loud.

## Topics to Explore

- [file] `stock-exchange/test_exchange.py` — How the matching engine is tested, especially edge cases like partial fills, self-trade prevention, and multi-level sweeps
- [file] `stock-exchange/plan.md` — The original system design plan, which likely covers scalability considerations (sharding by symbol, sequencer architecture) that this single-threaded implementation doesn't address
- [function] `solution.py:OrderBook._match_order` — The matching algorithm is the heart of the system; trace through a multi-level sweep scenario (market buy consuming multiple ask levels) to understand the nested loop structure
- [general] `price-time-priority-alternatives` — Real exchanges sometimes use pro-rata allocation or size-priority at certain price levels; understanding why FIFO is the default and when alternatives apply
- [file] `distributed-message-queue/test_solution.py` — Why does the message queue test suite import from the stock exchange? This cross-dependency is unusual and worth investigating

## Beliefs

- `stock-exchange-fifo-matching` — Orders at the same price level are matched in FIFO order, enforced by `deque` append/popleft semantics in `OrderBook._match_order`
- `stock-exchange-resting-price-execution` — Trades always execute at the resting (maker) order's price, not the incoming (taker) order's price
- `stock-exchange-market-orders-never-rest` — Market orders that can't be fully filled have their remainder cancelled; they are never added to the order book
- `stock-exchange-trade-counter-global` — `Trade._counter` is a class-level counter that monotonically increases across all symbols and never resets, making trade IDs globally sequential but test-order-dependent
- `stock-exchange-no-input-validation` — The matching engine performs no validation on order fields (quantity, price, side); callers are trusted to provide valid inputs

