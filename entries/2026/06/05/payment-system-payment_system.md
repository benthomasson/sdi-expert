# File: payment-system/payment_system.py

**Date:** 2026-06-05
**Time:** 13:18

# `payment-system/payment_system.py`

## Purpose

This file implements a **payment processing system** — one of the canonical system design interview problems. It owns the entire payment lifecycle: account creation, payment processing with external processor integration, refunds (full and partial), and balance tracking. The implementation demonstrates three core concepts that come up in payment system interviews: **idempotency** (preventing duplicate charges), **double-entry bookkeeping** (every money movement has a matching debit and credit), and **retry with exponential backoff** (handling transient processor failures).

This is a single-process in-memory simulation — there's no database or network — but the abstractions are faithful to how a real payment system is structured.

## Key Components

### Data Classes

- **`Payment`** — Represents a payment through its lifecycle. Tracks status (`CREATED` → `PROCESSING` → `COMPLETED`/`FAILED`, with `REFUND_PENDING` → `REFUNDED` for refunds), the `idempotency_key` for dedup, and `refunded_amount` for tracking partial refunds against the original amount.

- **`LedgerEntry`** — A single debit or credit against an account. Immutable record — entries are appended, never modified. The `reference_id` links back to the payment or operation that caused it.

- **`SYSTEM_ACCOUNT`** (`"__SYSTEM__"`) — The contra account used for initial balance funding. When you seed an account with money, the system debits `__SYSTEM__` and credits the account, keeping the books balanced from the start.

### `PaymentSystem` Class

The main orchestrator. Internal state is organized into five dictionaries/lists:

| Field | Type | Role |
|-------|------|------|
| `_accounts` | `dict[str, str]` | account_id → currency mapping |
| `_payments` | `dict[str, Payment]` | payment_id → Payment lookup |
| `_ledger` | `list[LedgerEntry]` | append-only transaction log |
| `_idempotency` | `dict[str, str]` | idempotency_key → payment_id dedup map |
| `_webhooks` | `dict[str, list[callable]]` | event → callback list |

**Key methods:**

- **`create_account(account_id, currency, initial_balance)`** — Registers an account. If `initial_balance > 0`, immediately creates a balanced debit/credit pair against `SYSTEM_ACCOUNT`.

- **`process_payment(amount, currency, payer_id, payee_id, idempotency_key)`** — The core method. Runs a validation pipeline (idempotency → account existence → currency match → balance sufficiency), then delegates to the external processor with retry, and records ledger entries on success. Returns the `Payment` object regardless of outcome.

- **`refund(payment_id, amount=None)`** — Full or partial refund. Creates reverse ledger entries (debit payee, credit payer). Tracks cumulative `refunded_amount` to prevent over-refunding.

- **`get_balance(account_id)`** — Derives balance by scanning the entire ledger. No cached balance — always computed from the source of truth.

- **`verify_ledger_integrity()`** — Checks the double-entry invariant: total debits == total credits (within floating-point tolerance of 0.001).

## Patterns

**Idempotency via key mapping.** The `_idempotency` dict maps client-provided keys to payment IDs. If a key is seen again, the existing payment is returned immediately — no re-processing. This is the standard pattern for payment APIs where network failures can cause clients to retry.

**Double-entry bookkeeping.** Every money movement produces exactly two ledger entries: a debit on one account and a matching credit on another. This is enforced in `create_account`, `process_payment`, and `refund`. The `verify_ledger_integrity` method audits this invariant.

**Strategy pattern for the processor.** The external payment processor is injected via `set_processor()` — a callable that returns a status dict. This decouples processing logic from the payment orchestration and makes testing easy (inject a fake that returns `"timeout"` or `"failure"`).

**Exponential backoff retry.** `_call_processor_with_retry` retries up to `_max_retries` times on `"timeout"`, with delays of 0.1s, 0.2s, 0.4s (base × 2^attempt). Non-timeout results (success or failure) return immediately.

**Webhook / observer pattern.** Events (`payment.created`, `payment.completed`, `payment.failed`, `payment.refunded`) fire registered callbacks. Exceptions in callbacks are silently swallowed — webhooks are fire-and-forget.

## Dependencies

**Imports:** Only stdlib — `time` (timestamps, retry delays), `uuid` (ID generation), `dataclasses` (data modeling). No external dependencies.

**Imported by:** `test_payment_system.py` — the test suite is the sole consumer.

## Flow

A typical successful payment flows through `process_payment` like this:

1. **Idempotency check** — return cached payment if key exists
2. **Validation** — account existence, currency compatibility, sufficient balance
3. **Payment creation** — status `CREATED`, registered in `_payments` and `_idempotency`
4. **Processing** — status transitions to `PROCESSING`, external processor called with retry
5. **Settlement** — on success, two ledger entries (debit payer, credit payee), status `COMPLETED`
6. **Notification** — webhook fired at each state transition

A refund reverses the ledger entries — debit the payee's account, credit the payer's — and transitions through `REFUND_PENDING` → `REFUNDED`.

Balance is always derived, never cached: `get_balance` does a full ledger scan, summing credits and subtracting debits for the given account.

## Invariants

- **Ledger balance invariant:** Total debits == total credits across the entire ledger. Verified by `verify_ledger_integrity()`.
- **Idempotency:** A given `idempotency_key` always maps to the same payment. Once set, never overwritten.
- **No overdraft:** `process_payment` checks `get_balance(payer_id) < amount` before processing. However, this is not atomic — in a concurrent system this would be a race condition.
- **Refund cap:** `refunded_amount + refund_amount` must not exceed `payment.amount`.
- **Refund precondition:** Only payments in `COMPLETED` or `REFUNDED` status can be refunded.
- **Currency homogeneity:** Payer and payee must both have accounts denominated in the payment's currency.
- **Account uniqueness:** `create_account` raises on duplicate `account_id`.

## Error Handling

Two error strategies are used:

1. **Raise `ValueError`** — for programming errors / invalid requests: unknown account, duplicate account, invalid refund state, refund exceeding payment amount. These are caller bugs.

2. **Return `FAILED` payment** — for business-rule failures: insufficient balance, currency mismatch, processor failure/timeout. The caller gets back a `Payment` with `status="FAILED"` and can inspect it.

Webhook callbacks are wrapped in a bare `except Exception: pass` — a deliberate choice to prevent observer failures from breaking the payment flow. This means webhook errors are invisible, which is a tradeoff worth noting.

The retry logic in `_call_processor_with_retry` converts exhausted retries (all timeouts) into `{"status": "failure"}` — the caller never sees `"timeout"` as a final result.

## Topics to Explore

- [file] `payment-system/test_payment_system.py` — See how idempotency, retry, refund edge cases, and ledger integrity are tested
- [file] `payment-system/plan.md` — The design rationale and interview-style requirements that drove this implementation
- [function] `payment-system/payment_system.py:_call_processor_with_retry` — The retry/backoff strategy and how it interacts with the injectable processor
- [general] `concurrent-payment-safety` — The balance check and ledger write are not atomic; explore what breaks under concurrency and how real systems solve it (optimistic locking, serializable transactions)
- [file] `digital-wallet/wallet.py` — Compare with the digital wallet implementation which likely tackles similar double-entry and balance concerns from a different angle

## Beliefs

- `payment-idempotency-is-key-based` — Idempotency is enforced by mapping client-provided keys to payment IDs in `_idempotency`; a repeated key returns the original `Payment` without re-processing
- `balance-derived-from-ledger` — Account balances are computed by scanning the full ledger on every call to `get_balance`, not cached; the ledger is the single source of truth
- `double-entry-invariant` — Every money movement (initial funding, payment, refund) creates exactly two ledger entries — one debit and one credit — and `verify_ledger_integrity` audits that total debits equal total credits
- `webhook-errors-swallowed` — Webhook callback exceptions are caught with bare `except Exception: pass`, making observer failures invisible but preventing them from disrupting payment processing
- `retry-converts-timeout-to-failure` — `_call_processor_with_retry` retries only on `"timeout"` status with exponential backoff; after `_max_retries` exhausted timeouts, it returns `{"status": "failure"}` — the caller never sees a timeout as a final result

