# File: digital-wallet/wallet.py

**Date:** 2026-06-05
**Time:** 13:46

# `digital-wallet/wallet.py` — Digital Wallet Service

## Purpose

This file implements an in-memory digital wallet system for a system design interview context. It owns the core domain: wallet lifecycle (create/freeze/unfreeze), money movement (deposit/withdraw/transfer), daily spending limits, and a complete audit trail. It's a single-process simulation — no database, no network — but it models the concurrency and correctness guarantees you'd need in a real distributed wallet service.

## Key Components

### Data Classes

**`Wallet`** — Represents a single wallet. Holds `balance`, `currency`, a `daily_limit`, a `frozen` flag, and a `version` counter for optimistic concurrency. Each wallet carries its own `threading.Lock` for fine-grained locking.

**`Transaction`** — Immutable record of a single money movement. The `tx_type` discriminates between `deposit`, `withdrawal`, `transfer_out`, and `transfer_in`. Stores `balance_after` (the wallet's balance post-operation), `counterparty` (for transfers), and auto-generates a `timestamp` if not provided.

**`AuditEntry`** — Captures before/after balance snapshots with a human-readable `reason`. Separate from `Transaction` — audit entries track administrative actions (freeze/unfreeze/create) in addition to money movements.

### `WalletService`

The service layer. All operations go through this class. Key methods:

| Method | Contract |
|--------|----------|
| `create_wallet` | Registers a new wallet, logs audit entry. No uniqueness check — will silently overwrite. |
| `deposit` | Adds funds. Checks frozen status. No daily limit check on deposits. |
| `withdraw` | Removes funds. Checks frozen, sufficient balance, and daily spending limit. |
| `transfer` | Atomic debit+credit across two wallets. Enforces currency match, frozen checks on both, balance check on sender, daily limit on sender. |
| `verify_integrity` | Conservation-of-money check: `deposits - withdrawals == sum(balances)`. Transfers are excluded because they're zero-sum. |

## Patterns

**Fine-grained locking with ordered acquisition.** Each `Wallet` has its own lock. `transfer` acquires both locks sorted by `wallet_id` to prevent deadlocks — this is the classic lock-ordering pattern. Single-wallet operations (`deposit`, `withdraw`) only grab one lock.

**Two-tier locking.** The per-wallet lock protects balance mutations. A separate `_tx_lock` protects the shared `transactions` list. The wallet lock is held while `_tx_lock` is acquired (nested), which is safe because `_tx_lock` is never held when acquiring a wallet lock.

**Defensive rounding.** Every amount is `round(amount, 2)` before use, and balances are rounded after arithmetic. This avoids floating-point drift (e.g., `0.1 + 0.2 != 0.3`). A production system would use `Decimal`, but for an interview simulation this is pragmatic.

**Audit as a cross-cutting concern.** `_record_audit` is called from every state-changing operation. The audit trail is append-only and captures both financial and administrative events.

**Timestamp synchronization on transfers.** Both the debit and credit transactions get the same `time.time()` value, ensuring they're correlated in the audit trail.

## Dependencies

**Imports:** All stdlib — `threading` for concurrency, `uuid` for transaction IDs, `time` for timestamps, `dataclasses` for models, `datetime` for daily-limit date bucketing.

**Imported by:** `test_wallet.py` — the test suite is the only consumer, which is typical for these self-contained SDI implementations.

## Flow

A typical transfer:

1. Validate amount is positive
2. Look up both wallets (raise `WalletNotFound` if missing)
3. Check currency match
4. Acquire both wallet locks in `wallet_id` order
5. Check neither wallet is frozen
6. Check sender has sufficient balance
7. Compute sender's daily spending; check adding this transfer won't exceed the limit
8. Mutate both balances, increment both versions
9. Create two `Transaction` records (debit + credit) with synchronized timestamps
10. Append both to `self.transactions` under `_tx_lock`
11. Append two `AuditEntry` records
12. Return the `(debit, credit)` tuple

## Invariants

- **Conservation of money**: `verify_integrity` asserts `total_deposits - total_withdrawals == sum(all_balances)`. Transfers don't create or destroy money — they produce paired `transfer_out`/`transfer_in` entries that cancel out.
- **Daily limit applies only to outflows**: Deposits are uncapped. Only `withdrawal` and `transfer_out` count toward daily spending.
- **Lock ordering**: Wallets are always locked in `wallet_id` sort order during transfers. Violating this would risk deadlock.
- **Frozen wallets block all money movement**: Checked inside the lock, so there's no TOCTOU gap between checking and mutating.
- **Amounts must be positive**: Enforced by `_validate_amount` at the top of every money-movement method.

## Error Handling

Seven custom exceptions model distinct failure modes. They're raised (never caught) — the caller is expected to handle them. The error hierarchy is flat (all extend `Exception` directly). Key semantics:

- `InsufficientFunds` and `DailyLimitExceeded` are checked *inside* the lock, after all preconditions, so no partial mutation occurs on failure.
- `WalletFrozen` is also checked inside the lock — no race between freeze and operation.
- `CurrencyMismatch` is checked *before* acquiring locks in `transfer`, which is fine since currency is immutable after creation.
- No error is ever swallowed. There are no try/except blocks anywhere.

One notable gap: `create_wallet` doesn't check for duplicate `wallet_id` — it silently overwrites.

## Topics to Explore

- [file] `digital-wallet/test_wallet.py` — See how concurrency, daily limits, and transfer atomicity are verified in tests
- [function] `digital-wallet/wallet.py:verify_integrity` — Understand why transfers are excluded from the conservation check and what the 0.01 epsilon covers
- [file] `payment-system/payment_system.py` — Compare with the payment system implementation for a different take on financial transactions
- [general] `float-vs-decimal-in-financial-systems` — Why production wallets use `Decimal` or integer-cents instead of `float`, and when the rounding strategy here breaks down
- [general] `distributed-wallet-consistency` — How this single-process locking model would change with multiple service instances (two-phase commit, saga pattern, event sourcing)

## Beliefs

- `wallet-lock-ordering-prevents-deadlock` — Transfers acquire wallet locks sorted by `wallet_id`, preventing deadlock when two concurrent transfers involve the same pair of wallets in opposite directions
- `daily-limit-only-counts-outflows` — Daily spending is computed from `withdrawal` and `transfer_out` transactions only; deposits and incoming transfers are uncapped
- `verify-integrity-ignores-transfers` — `verify_integrity` sums only `deposit` and `withdrawal` amounts against total balances, because `transfer_in`/`transfer_out` pairs are zero-sum and cancel out
- `wallet-creation-silently-overwrites` — `create_wallet` does not check for an existing `wallet_id`; calling it twice with the same ID replaces the wallet and loses the original balance
- `audit-trail-covers-admin-and-financial-events` — The audit trail records both money movements (deposit/withdraw/transfer) and administrative actions (create/freeze/unfreeze), making it a superset of the transaction log

