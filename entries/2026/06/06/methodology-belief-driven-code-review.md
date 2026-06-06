# Belief-Driven Code Review: Finding Bugs Through Derived Reasoning

**Date:** 2026-06-06
**Scope:** Methodology entry — how multi-pass belief extraction and derivation surfaced real architectural issues across 25 SDI implementations

## The Problem

Code review at scale faces a fundamental limitation: reviewers look at code one file at a time and assess it locally. Cross-cutting architectural properties — "is deletion safe across all modules?", "does every write path prevent regression?" — require holding dozens of files in your head simultaneously. Humans can't do this reliably. We tried a different approach: record atomic observations about each file, derive higher-level beliefs from those observations, and let the derivation process surface contradictions.

## The Method

### Pass 1: Observation Extraction

We scanned all 25 modules in `/Users/ben/git/sdi-implementations` using `code-expert scan` and `code-expert explore`, producing 31 entry files that explained every non-test Python file. From these entries, `code-expert propose-beliefs` extracted 199 premise beliefs — atomic, verifiable observations about individual files.

Each premise records one fact about one file. Examples:

```
payment-balance-check-not-atomic: `process_payment` checks balance before
processing but the check and ledger write are not atomic — a race condition
under concurrency.

kv-store-quorum-overlap-not-enforced: The key-value store's W+R>N
read-write overlap guarantee is assumed by callers but never validated
in code.

dedup-is-global-not-per-ad: The `seen_events` dedup registry keys on
`event_id` alone; the same event ID arriving for different `ad_id`s
will be deduplicated.
```

At this stage, each observation stands alone. `payment-balance-check-not-atomic` is just a note about one function in one file. It doesn't tell you whether it matters.

### Pass 2: Derivation

`reasons derive --exhaust` combines premises into derived beliefs using SL (support-list) justifications. Each derived belief names its antecedents and, critically, its **unless** clauses — premises that, if true, would invalidate the conclusion.

This is where the method produces value that local code review cannot. The derivation engine asked: "Can wallet transfers be safe under concurrency?" and produced:

```
wallet-transfers-are-safe-under-concurrency: Wallet concurrent transfers
are deadlock-free via sorted lock acquisition with frozen-check inside
the lock — but this safety guarantee assumes stable wallet identity,
which silent creation-overwrite could violate mid-transfer.
  Depends on: wallet-deadlock-free-concurrent-transfers
  Unless: wallet-creation-silently-overwrites
```

The system noticed that `create_wallet` doesn't check for an existing wallet ID. A reviewer looking at the transfer code alone would see correct locking. A reviewer looking at `create_wallet` alone would see a simple factory method. The bug only becomes visible when you reason about the interaction — sorted lock acquisition assumes wallet identity is stable, but `create_wallet` can replace a wallet mid-transfer.

After 12 rounds, derivation produced 160 derived beliefs at depths up to 9. Many of the deepest beliefs had unless clauses pointing at premise-level observations.

### Pass 3: Gate Analysis

`reasons list-gated` identified 15 premises that appeared as unless clauses on derived beliefs — concrete code-level facts that were blocking higher-level architectural conclusions. These 15 "blocker" beliefs gated 21 derived beliefs.

The gated beliefs weren't arbitrary. They clustered around the architecture's strongest claims:

| Blocker (code fact) | Gated belief (architectural claim) |
|---|---|
| `payment-balance-check-not-atomic` | `financial-correctness-is-end-to-end` |
| `kv-store-quorum-overlap-not-enforced` | `kv-reads-eventually-converge` |
| `dedup-is-global-not-per-ad` | `forward-only-stream-processing-is-exactly-once` |
| `wallet-creation-silently-overwrites` | `wallet-transfers-are-safe-under-concurrency` |
| `hotel-cancel-no-underflow-guard` | `writes-always-produce-valid-forward-progress` |
| `proximity-no-coordinate-validation` | `perimeter-defense-ensures-data-quality` |
| `stock-exchange-no-input-validation` | `stock-exchange-matching-produces-valid-trades` |
| `notif-group-key-unused` | `notification-system-is-delivery-complete` |

Each row is a concrete code deficiency preventing a general architectural property from holding. The derivation engine found these by trying to prove the architectural property and discovering which premises blocked the proof.

### Pass 4: Fixing the Code

With 15 specific, prioritized issues, we applied targeted fixes:

**Payment atomicity** (`payment-system/payment_system.py`): Added `threading.Lock` wrapping the balance check through ledger write. The TOCTOU window between reading the balance and writing the ledger entry was the exact gap the derivation identified.

```python
self._lock = threading.Lock()
# ...
with self._lock:
    balance = self.get_balance(from_account, currency)
    if balance < amount:
        # ... reject
    self._ledger.append(debit_entry)
    self._ledger.append(credit_entry)
```

**KV quorum validation** (`key-value-store/key_value_store.py`): Added `W+R>N` enforcement at construction time. The KV store's consistency model depends on quorum overlap, but nothing prevented constructing a store with `w=1, r=1, n=3` — which silently provides no consistency guarantee.

```python
if w + r <= n:
    raise ValueError(f"Quorum overlap required: W({w}) + R({r}) must be > N({n})")
```

This broke three existing tests that used `w=1` or `r=1` — the tests were passing despite testing a configuration that provides no consistency.

**Ad click dedup** (`ad-click-event-aggregation/click_aggregator.py`): Changed the dedup key from `event.event_id` to `(event.event_id, event.ad_id)`. The global dedup meant that if the same event ID appeared for two different ads (legitimate in a system where event IDs come from click infrastructure, not per-ad), the second ad's click was silently dropped.

**Notification grouping** (`notification-system/notification_system.py`): The `_pending_groups` dict and `_group_window` field were initialized but never used. Rather than dead code, these were an unfinished planned feature (plan item 9: notification batching). We implemented the feature — notifications with the same `(user_id, group_key)` within the time window collapse into a single summary.

### Pass 5: Belief Retraction and Cascade

After each fix, we retracted the corresponding blocker premise. The reasons system propagated the retraction:

```
$ reasons retract payment-balance-check-not-atomic
Retracted payment-balance-check-not-atomic
  Went IN (4):
    [+] financial-concurrency-is-comprehensively-safe
    [+] ledger-interpretation-is-consistently-safe
    [+] financial-correctness-is-end-to-end
    [+] all-monetary-operations-maintain-double-entry-invariant
```

One retraction unblocked four architectural conclusions. The system now proves what it previously could not: that financial operations maintain their invariants under concurrency.

Not every blocker required a code fix. `dmq-reused-by-stock-exchange` claimed the stock exchange imported the message queue module, creating a cross-module dependency. Inspection showed this was factually wrong — no such import exists. Retracting the false premise unblocked `modules-are-independently-correct` and `pedagogical-tradeoffs-are-safe-within-module-boundaries`.

### Pass 6: Re-derivation and Review

With blockers removed, `reasons derive --exhaust --sample` produced 118 new derived beliefs (depth up to 11). `reasons review-beliefs` then evaluated them for quality, flagging 24 overgeneralizations that `reasons research` softened. Two beliefs were abandoned as structurally flawed:

- `monotonically-expanding-verified-state`: committed a category error — "state lattice only grows" (data domain) does not imply "correctness never regresses" (semantic domain).
- `nearby-friends-visibility-scales`: had an inverted logical structure where its unless clause contradicted its own claim text.

## What This Found That Local Review Would Not

The key insight is that **derivation surfaces cross-cutting issues that file-level review misses**. Consider the chain that led to finding the payment atomicity bug:

1. Observation: `double-entry-invariant` — every payment creates balanced debit-credit pairs.
2. Observation: `balance-derived-from-ledger` — balances are computed from full ledger scans.
3. Derivation: `payment-double-entry-guarantees-balance-integrity` — the double-entry design guarantees correct balances... *unless* `payment-balance-check-not-atomic`.
4. Derivation: `financial-correctness-combines-locking-with-structural-asymmetry` — financial systems combine locking with structural asymmetry for end-to-end correctness... *unless* `payment-balance-check-not-atomic`.
5. Derivation: `financial-correctness-is-end-to-end` — financial domains achieve simultaneous safety, lifecycle correctness, and full traceability... *unless* `payment-balance-check-not-atomic`.

A reviewer looking at `process_payment` might notice the race condition. But the derivation engine showed *why it matters* — it's the single fact preventing the entire financial subsystem from being provably correct. That prioritization is what turns a list of code smells into an ordered repair plan.

## Results

| Metric | Value |
|---|---|
| Premises extracted | 199 |
| Derived beliefs (round 1) | 160 |
| Blockers identified via gates | 15 |
| Code fixes applied | 13 |
| Beliefs retracted (false premises) | 2 |
| Derived beliefs unblocked | 21 |
| New derivations (round 2) | 118 |
| Overgeneralizations softened | 24 |
| Structurally flawed beliefs abandoned | 2 |
| Final belief count | 477 (420 IN, 57 OUT) |
| Max derivation depth | 11 |
| Remaining active gates | 1 (test coverage gap, not a code bug) |

## When This Works

This method is effective when:
- The codebase has many independent modules with similar architectural patterns (cross-cutting derivation has material to work with).
- Correctness properties span multiple files or modules (local review is insufficient).
- You want to prioritize which issues matter most (gate analysis ranks by architectural impact).

It is less useful for:
- Single-file bugs with purely local effects.
- Performance issues (the belief network reasons about correctness, not speed).
- Codebases where every module is architecturally unique (derivation needs recurring patterns).
