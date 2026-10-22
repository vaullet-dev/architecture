# ADR-004: Atomic Balance Reservations in the Ledger

## Status

**Accepted** (2026-08-27)
**Revised** (2026-08-27) — bucketed balances, to support independently sellable rewards modules
**Revised** (2026-09-02) — caller-supplied hold lifetimes (ADR-012's two-phase flow); `DEBT` excluded
from funding; the journal stated as single-sided

Supersedes [ADR-001](001-distributed-locking-for-balance-consistency.md).

**Amended by [ADR-016](016-transactional-outbox-for-event-producers.md)** (2026-10-22): buckets are
grouped into wallets as built in wallet-ledger-service, and an account's rows here are created when
its first wallet is opened — which Auth Service's outbox relay does for every account it creates. See
*Balance buckets*.

## Context

[ADR-001](001-distributed-locking-for-balance-consistency.md) decided to prevent overdrafts using Redis distributed locks held by Transaction Service around a balance check. Reviewing that decision before implementation, we found it does not prevent the failure it was written to prevent.

### The Flaw in ADR-001

ADR-001's critical section reads the balance from Vaullet, decides, publishes an event, and releases the lock:

```java
lock.tryLock(...);
BigDecimal balance = vaulletService.getBalance(userId);  // READ
if (balance.compareTo(amount) < 0) reject();
publishEvent("transaction.created", txn);                // WRITE happens later, elsewhere
lock.unlock();
```

Vaullet creates the ledger entry **asynchronously**, when it consumes `transaction.created` (see `arch.md`, Vaullet event consumption). The balance mutation therefore happens *after* the lock is released, in a different service, over a Kafka hop.

The lock serializes the read. Nothing serializes the write. The state the lock is meant to protect is modified outside the critical section, so the lock protects nothing.

### Failure Trace

Account holds $100. Two withdrawals of $60, arriving 50ms apart — not even concurrent:

| t | Request A | Request B | Ledger state |
|---|-----------|-----------|--------------|
| 0ms | acquires lock | — | balance $100 |
| 1ms | reads balance: **$100** | — | balance $100 |
| 2ms | $100 ≥ $60 → **approve** | — | balance $100 |
| 3ms | publishes `transaction.created` | — | balance $100 |
| 4ms | releases lock | — | balance $100 |
| 50ms | — | acquires lock (free) | balance $100 |
| 51ms | — | reads balance: **$100** | balance $100 |
| 52ms | — | $100 ≥ $60 → **approve** | balance $100 |
| ~200ms | Vaullet consumes A | | balance $40 |
| ~250ms | | Vaullet consumes B | balance **-$20** |

```mermaid
sequenceDiagram
    autonumber
    participant A as Withdrawal A
    participant B as Withdrawal B
    participant L as Redis lock
    participant K as Kafka
    participant V as Vaullet (ledger)

    Note over A,V: ADR-001 — the lock serializes the read, nothing serializes the write
    A->>L: acquire lock(account)
    A->>V: GET balance
    V-->>A: 100.00
    Note right of A: 100 >= 60 → approve
    A->>K: publish transaction.created
    A->>L: release lock
    Note over L: 50ms later — the lock is free,<br/>but the balance has not moved yet
    B->>L: acquire lock(account)
    B->>V: GET balance
    V-->>B: 100.00
    Note right of B: 100 >= 60 → approve
    B->>K: publish transaction.created
    K-->>V: consume A
    Note over V: balance 40.00
    K-->>V: consume B
    Note over V: balance -20.00 — overdraft
```

The lock is released at step 6 and the balance does not change until step 13. Every approval decision
in that window reads stale state, and the window is a Kafka round-trip wide — orders of magnitude
longer than the lock was ever held.

This is the exact scenario in ADR-001's opening paragraph. Any lock hold time shorter than the Kafka round-trip leaves the window open, and the whole point of a short lock TTL was to keep hold times low — the mitigation and the bug pull in the same direction.

Two secondary problems in ADR-001, noted for completeness:

- **TTL vs latency budget.** A 500ms lock TTL inside a stated 200–500ms approval budget means the lock expires mid-operation at p95, while the operation continues believing it holds it.
- **Redlock vs Redis Cluster.** Redlock requires N *independent* Redis masters. ADR-001 specifies a single Redis Cluster (3 masters + 3 replicas), where Redisson's `RLock` is a single-master lock replicated asynchronously — a lock that can be silently lost on failover. The named algorithm and the named deployment are incompatible.

### Constraints (unchanged from ADR-001)

- Zero tolerance for overdrafts
- >1000 TPS system-wide, sub-200ms p95 approval latency
- One tenant per deployment (see ADR-005) — isolation is at the cluster boundary, so balances are scoped by `account_id` alone
- The ledger must express non-cash money (bonus, loyalty, referral) with distinct withdrawal rules, while remaining correct when no rewards module is deployed
- One currency per deployment, supplied by the operator as configuration
- Prefer proven, operationally simple mechanisms

## Decision

We will make **check-and-reserve a single atomic operation inside Vaullet**, and remove distributed locking from the balance path entirely.

Available balance becomes `posted_balance − held_total`. Money is *held* synchronously at approval time and *settled* asynchronously at completion. A hold is a real row, committed in the same database transaction as the check that authorized it.

### Balance buckets

A single `posted_balance` scalar cannot express "the account holds $100, but $60 is bonus money with wagering still outstanding and cannot be withdrawn." Since Bonus is the flagship module — and Loyalty and Referral sell independently of it — the ledger models money as **typed buckets**.

`CASH` always exists. Rewards modules create additional buckets by emitting grant events. With no rewards module deployed, every account has exactly one `CASH` bucket and the model degrades to a plain balance.

Buckets are per *grant*, not per type: a user can hold three separate `BONUS` buckets with different wagering requirements and expiry dates, because that is what the terms of three separate promotions actually mean.

**The ledger is authoritative about amounts and dumb about policy.** It stores `wagering_remaining` but never decides how a wager contributes to it — that is the Bonus module's rule, applied by emitting events. This keeps promotional logic out of the service that must stay correct when Bonus isn't deployed.

> **Amended 2026-10-22 by [ADR-016](016-transactional-outbox-for-event-producers.md): wallets, and
> who opens them** — as built in wallet-ledger-service 0.1.0 (#10).
>
> - **Buckets are grouped into wallets.** One `REAL` wallet per account holds `CASH` (and `DEBT`, once
>   ADR-009 creates one); one `BONUS` wallet holds the promotional buckets. A wallet groups buckets
>   and does not replace them: allocation by spend priority, the single `account_balances` lock and
>   the journal are as above. What a wallet adds is a status — `ACTIVE`, `FROZEN`, `CLOSED` — and
>   reservations skip the buckets of a wallet that is not `ACTIVE`.
> - **Opening a wallet opens the account.** `POST /v1/wallets` (`ledger:write`, one wallet per type
>   per account, so a repeat returns the existing one) creates the `account_balances` row with the
>   account's first wallet, and a `REAL` wallet starts with its `CASH` bucket. "`CASH` always exists"
>   holds from the moment the `REAL` wallet is opened.
> - **Auth Service opens both** for every account it creates, through its outbox (ADR-016 §4).
>
> `V2__wallets.sql` and the service README's *Wallets* are the detail.

### Currency

**Currency is a deployment constant, chosen and supplied by the operator.** Each cluster runs one
currency, configured alongside module enablement in that customer's Helm values (ADR-005). It is
therefore *not* a column on every ledger row.

This is a deliberate trade. The alternative — a `currency` column everywhere — buys flexibility no
contract currently asks for, and costs a correctness hazard on every query that forgets to filter by
it. A constant cannot be forgotten.

Three things follow, and all three are required:

**1. The journal must be self-describing.** A ledger export reading `amount: 100.0000` with no
currency anywhere is ambiguous forever, and financial records outlive the cluster that wrote them.
One immutable row records it:

```sql
CREATE TABLE ledger_config (
    singleton     BOOLEAN     PRIMARY KEY DEFAULT TRUE CHECK (singleton),
    currency      CHAR(3)     NOT NULL,          -- ISO 4217
    minor_units   SMALLINT    NOT NULL,          -- 2 for EUR, 0 for JPY, 3 for KWD
    initialized_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

Written once at provisioning; never updated. Changing a live deployment's currency is not a config
change, it is a migration, and the schema should refuse to pretend otherwise.

**2. The boundary must assert it.** Every API accepting an amount takes an explicit currency and
rejects any mismatch with `422 CURRENCY_MISMATCH`. Never coerce, never default. An operator
integration bug that posts RSD to a EUR deployment must fail on the first request, not book silently
at a 1:1 rate and surface months later in a reconciliation.

**3. Rounding is currency-specific.** `NUMERIC(20,4)` holds four decimal places, but a currency with
`minor_units = 0` (JPY, KRW) must never expose a fractional balance, and one with 3 (KWD, BHD)
rounds differently from EUR. Rounding policy reads `minor_units` from `ledger_config`; it is not
hardcoded to 2.

**If an operator ever needs multiple currencies**, the extension is one account per
`(user, currency)` — never mixed buckets inside one account. Bucket allocation, the `CHECK`
constraint, and the single-row lock all assume one unit of account; mixing currencies under one
`account_id` breaks all three simultaneously. Recording this now so the shape isn't rediscovered
under pressure later.

### Schema (vaullet_db)

```sql
-- Immutable journal. Source of truth. Append-only, never updated.
-- Single-sided: one row per (transaction, bucket, direction) against a user account.
-- The operator's own position is not ledgered here — see "Scope: a single-sided journal" below.
CREATE TABLE ledger_entries (
    entry_id        UUID PRIMARY KEY,
    account_id      UUID        NOT NULL,
    bucket_id       UUID        NOT NULL REFERENCES balance_buckets(bucket_id),
    direction       TEXT        NOT NULL CHECK (direction IN ('DEBIT','CREDIT')),
    amount          NUMERIC(20,4) NOT NULL CHECK (amount > 0),
    reservation_id  UUID        NULL REFERENCES reservations(reservation_id),
    transaction_id  UUID        NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE UNIQUE INDEX ON ledger_entries (transaction_id, bucket_id, direction);

-- Per-account aggregate AND the single concurrency anchor.
-- Every balance-changing operation locks this row first, before touching buckets.
-- Derived: rebuildable by replaying ledger_entries.
CREATE TABLE account_balances (
    account_id      UUID          PRIMARY KEY,
    posted_balance  NUMERIC(20,4) NOT NULL DEFAULT 0,   -- sum over FUNDABLE buckets (DEBT excluded)
    held_total      NUMERIC(20,4) NOT NULL DEFAULT 0 CHECK (held_total >= 0),
    debt_total      NUMERIC(20,4) NOT NULL DEFAULT 0 CHECK (debt_total >= 0),  -- sum over DEBT buckets
    updated_at      TIMESTAMPTZ   NOT NULL DEFAULT now(),
    CHECK (posted_balance - held_total >= 0)   -- overdraft is a schema violation
);

-- One row per grant. CASH is a singleton per account.
CREATE TABLE balance_buckets (
    bucket_id          UUID PRIMARY KEY,
    account_id         UUID          NOT NULL REFERENCES account_balances(account_id),
    bucket_type        TEXT          NOT NULL CHECK (bucket_type IN ('CASH','BONUS','LOYALTY','REFERRAL','DEBT')),
    source_module      TEXT          NULL,        -- NULL for CASH
    grant_id           UUID          NULL,        -- idempotency anchor for the granting event
    posted_balance     NUMERIC(20,4) NOT NULL DEFAULT 0,
    held_total         NUMERIC(20,4) NOT NULL DEFAULT 0 CHECK (held_total >= 0),
    withdrawable       BOOLEAN       NOT NULL,
    wagering_remaining NUMERIC(20,4) NOT NULL DEFAULT 0,
    spend_priority     SMALLINT      NOT NULL,    -- lower spends first
    expires_at         TIMESTAMPTZ   NULL,
    CHECK (posted_balance - held_total >= 0)
);
CREATE UNIQUE INDEX ON balance_buckets (grant_id) WHERE grant_id IS NOT NULL;
CREATE UNIQUE INDEX ON balance_buckets (account_id) WHERE bucket_type = 'CASH';
CREATE INDEX ON balance_buckets (account_id, spend_priority);

-- Authorization holds. Lifetime is chosen by the caller — see "Reservation lifetime".
CREATE TABLE reservations (
    reservation_id  UUID PRIMARY KEY,
    account_id      UUID          NOT NULL,
    amount          NUMERIC(20,4) NOT NULL CHECK (amount > 0),
    state           TEXT          NOT NULL CHECK (state IN ('HELD','SETTLED','RELEASED','EXPIRED')),
    idempotency_key TEXT          NOT NULL,
    expires_at      TIMESTAMPTZ   NOT NULL CHECK (expires_at > created_at),
    created_at      TIMESTAMPTZ   NOT NULL DEFAULT now(),
    UNIQUE (idempotency_key)
);
CREATE INDEX ON reservations (state, expires_at) WHERE state = 'HELD';

-- Which buckets a hold draws from, and how much from each.
CREATE TABLE reservation_allocations (
    reservation_id  UUID          NOT NULL REFERENCES reservations(reservation_id),
    bucket_id       UUID          NOT NULL REFERENCES balance_buckets(bucket_id),
    amount          NUMERIC(20,4) NOT NULL CHECK (amount > 0),
    PRIMARY KEY (reservation_id, bucket_id)
);
```

```mermaid
erDiagram
    ACCOUNT_BALANCES ||--o{ BALANCE_BUCKETS : "aggregates"
    ACCOUNT_BALANCES ||--o{ RESERVATIONS : "serializes on"
    BALANCE_BUCKETS  ||--o{ LEDGER_ENTRIES : "journalled in"
    BALANCE_BUCKETS  ||--o{ RESERVATION_ALLOCATIONS : "funds"
    RESERVATIONS     ||--|{ RESERVATION_ALLOCATIONS : "draws from"
    RESERVATIONS     ||--o{ LEDGER_ENTRIES : "settles into"

    ACCOUNT_BALANCES {
        uuid    account_id     PK "the concurrency anchor — locked FIRST, always"
        numeric posted_balance "sum over FUNDABLE buckets"
        numeric held_total
        numeric debt_total     "DEBT buckets, never spendable"
        check   overdraft      "posted_balance - held_total >= 0"
    }
    BALANCE_BUCKETS {
        uuid    bucket_id      PK
        uuid    account_id     FK
        text    bucket_type    "CASH BONUS LOYALTY REFERRAL DEBT"
        uuid    grant_id       UK "idempotency for the granting event"
        numeric posted_balance
        numeric held_total
        bool    withdrawable
        numeric wagering_remaining
        int     spend_priority "lower spends first"
        ts      expires_at
        check   overdraft      "posted_balance - held_total >= 0"
    }
    LEDGER_ENTRIES {
        uuid    entry_id       PK "immutable — append only, never updated"
        uuid    account_id
        uuid    bucket_id      FK
        text    direction      "DEBIT or CREDIT"
        numeric amount         "CHECK amount > 0"
        uuid    reservation_id FK
        uuid    transaction_id UK "unique with bucket_id + direction"
        ts      created_at
    }
    RESERVATIONS {
        uuid    reservation_id  PK
        uuid    account_id
        numeric amount
        text    state           "HELD SETTLED RELEASED EXPIRED"
        text    idempotency_key UK "a retry returns the existing hold"
        ts      expires_at      "caller-supplied lifetime"
        ts      created_at
    }
    RESERVATION_ALLOCATIONS {
        uuid    reservation_id PK
        uuid    bucket_id      PK
        numeric amount         "how much this hold takes from this bucket"
    }
    LEDGER_CONFIG {
        bool singleton    PK "one row, written at provisioning, never updated"
        char currency        "ISO 4217 — a constant cannot be forgotten"
        int  minor_units     "2 for EUR, 0 for JPY, 3 for KWD"
    }
```

`LEDGER_CONFIG` stands alone deliberately: currency is a deployment constant, not a column on every
row.

The two `CHECK (posted_balance - held_total >= 0)` constraints are the backstop, at both aggregate and bucket level: even with wrong application logic, Postgres refuses to record an overdraft. ADR-001 had no equivalent — nothing in its schema could have caught the bug.

`withdrawable_balance` is `SUM(posted_balance - held_total) WHERE withdrawable`, computed on read. It is deliberately not stored: it is a question, not a fact. A `DEBT` bucket is never `withdrawable`, so it is excluded here by construction.

**`DEBT` is not a funding source.** A `DEBT` bucket holds a positive amount representing what the user
owes the operator (ADR-009). It is a bucket so that the obligation is visible, aged and reportable
rather than hidden in a negative number — but it must never be spent, and it must never inflate the
aggregate the reserve path checks. Hence `posted_balance` sums *fundable* buckets only, `debt_total`
carries the obligation separately, and the allocation query filters `bucket_type <> 'DEBT'`. All three
have to agree: if debt were folded into `posted_balance`, the aggregate pre-check would approve an
amount the bucket allocation cannot actually cover, and the transaction would fail *after* passing the
gate that exists to reject it.

### Scope: a single-sided journal

`ledger_entries` records movement **against user accounts only**. There is no counterparty row, no
`CHECK` that debits equal credits per transaction, and no house or merchant account inside
`vaullet_db`. Prior documents described this as "double-entry"; that was inaccurate and is corrected
here. The design is a single-sided journal with a transactionally-maintained balance projection —
which is what a wallet needs and what the schema above actually implements.

This is a boundary, not an omission. The operator's own position — house exposure, settlement with
the PSP, revenue recognition — is the operator's accounting system's job, reconstructed downstream
from the event stream in `datawarehouse_db` (ADR-003). Vaullet answers one question: what is this
user's money, and may it be spent right now.

Stating it matters for throughput, not just vocabulary. A true two-sided journal would put a house
account on the other side of every transaction, and the account-first lock rule below would then
serialize **every transaction in the deployment** on that one row. The >1000 TPS target and the
"different accounts never contend" property both depend on the counterparty side staying out. If a
future decision introduces one, it must ship with balance-sharding for that account in the same
change — it is not an additive extension.

### Reserve (synchronous, one transaction)

```sql
BEGIN;

SELECT posted_balance, held_total
  FROM account_balances
 WHERE account_id = $1
   FOR UPDATE;              -- the ONLY lock taken; serializes this account only

-- if (posted_balance - held_total) < $amount: ROLLBACK, return 409 INSUFFICIENT_FUNDS

SELECT bucket_id, posted_balance - held_total AS available
  FROM balance_buckets
 WHERE account_id = $1
   AND bucket_type <> 'DEBT'                    -- an obligation is never a funding source (ADR-009)
   AND (expires_at IS NULL OR expires_at > now())
   AND posted_balance - held_total > 0
 ORDER BY spend_priority;   -- allocate greedily across buckets

INSERT INTO reservations (...) VALUES (..., 'HELD', $key, now() + $ttl);   -- caller-supplied
INSERT INTO reservation_allocations (...) VALUES ...;   -- one row per bucket touched

UPDATE balance_buckets  SET held_total = held_total + $alloc WHERE bucket_id = ANY($allocated);
UPDATE account_balances SET held_total = held_total + $amount, updated_at = now()
 WHERE account_id = $1;

COMMIT;
```

```mermaid
sequenceDiagram
    autonumber
    participant TX as Transaction Service
    participant V as Vaullet (ledger)
    participant PG as Postgres

    TX->>V: POST /v1/reservations<br/>Idempotency-Key, amount, currency, expires_in_seconds
    activate V
    V->>PG: BEGIN
    V->>PG: SELECT account_balances WHERE account_id FOR UPDATE
    Note over PG: the ONLY lock taken.<br/>Serializes this account — other accounts never contend
    PG-->>V: posted_balance, held_total
    alt available < amount
        V->>PG: ROLLBACK
        V-->>TX: 409 INSUFFICIENT_FUNDS
    else available >= amount
        V->>PG: SELECT buckets WHERE type != DEBT AND not expired<br/>ORDER BY spend_priority
        V->>PG: INSERT reservations + reservation_allocations
        V->>PG: UPDATE held_total on buckets and on the account
        V->>PG: COMMIT
        Note over PG: CHECK (posted - held >= 0) holds at both levels.<br/>An overdraft is a schema violation, not a lost race
        V-->>TX: 201 reservation_id, allocations, expires_at
    end
    deactivate V

    Note over TX,PG: later — settle or release, same account-first lock order
    TX->>V: transaction.completed.v1 (Kafka)
    V->>PG: lock account, write ledger_entries, decrement held, mark SETTLED
```

Read and write are in the same transaction, on the same rows, in the same service. There is no window.

**Deadlock safety — a rule for every write path, not just this one**: *every* operation that touches
an account's buckets must take `account_balances FOR UPDATE` **first**, before reading or writing any
bucket row, and must hold no other lock across a network call.

This applies to reserve, settle, release, grant and the expiry sweeper alike. Bucket rows are only
ever reached through that single gate, so two concurrent operations on one account serialize on one
row, and operations on different accounts never contend.

The rule exists because the paths naturally want opposite orders: reserve starts from the account and
allocates down into buckets, while settle and grant start from a known bucket and roll up to the
account. Left to their own logic, a reserve holding the account row and waiting on a bucket would
deadlock against a settle holding that bucket and waiting on the account row — on the money path,
under exactly the concurrency this ADR exists to handle. Postgres would detect and abort one, so it is
a failed transaction rather than corruption, but it is entirely avoidable.

Acquiring the account row first is therefore mandatory even where a path does not appear to need it.

**Spend priority** is set by the granting module and is a business decision (typically: expiring-soonest promotional money first, cash last, so users don't forfeit bonuses). The ledger sorts; it does not choose.

### Reservation lifetime

The caller supplies the hold's lifetime; the ledger does not assume one. `POST /v1/reservations`
takes an optional `expires_in_seconds`, defaulting to **300** and bounded by
`ledger_config.max_hold_seconds` (default 30 days), with `422 HOLD_TTL_TOO_LONG` beyond it.

A fixed short expiry was the original design and it is wrong for the product. Two hold classes exist
and their horizons differ by four orders of magnitude:

| Class | Typical lifetime | Settled by |
|---|---|---|
| **Authorization hold** — the ADR-004 default: check funds, complete within one request cycle | seconds | `transaction.completed.v1`, sub-second |
| **Two-phase transaction** — the wager exposed to operators as `authorize` then `capture` (ADR-012 §2) | hours to days | the operator, when the event resolves |

ADR-012 sells the second explicitly: a stake must be unavailable but not yet lost between placement
and resolution. Under a hardcoded five minutes every bet in the system expires long before it settles,
and the settlement/expiry race below stops being an edge case and becomes the normal path. The caller
knows its own settlement horizon; the ledger does not.

**An expired long hold is a business event, not noise.** When an authorization hold expires it means a
transaction stalled and the sweeper cleaned up. When a two-phase hold expires it means the operator
never captured — the user's stake is released, and the operator has a reconciliation problem they must
be told about. The sweeper therefore emits `ledger.hold-expired.v1` carrying `reservation_id`,
`account_id`, `amount` and the original `transaction_id`, and ADR-012 exposes it as a webhook event.
Silently releasing an operator's money is not an option.

`max_hold_seconds` is a deployment constant for the same reason currency is: an unbounded hold is
indistinguishable from a leak, and the ceiling belongs to the operator's contract rather than to a
request.

### Idempotency

`UNIQUE (idempotency_key)` — keyed on `transaction_id`. A retried reserve (client timeout, Transaction Service restart) hits the constraint and returns the *existing* reservation rather than placing a second hold. This is not optional: without it, retries double-hold and reject legitimate transactions.

Grants use `UNIQUE (grant_id)` on `balance_buckets` for the same reason — a redelivered `rewards.value-granted` event must not mint money twice. At-least-once delivery makes this mandatory, not defensive.

### Grants (asynchronous)

Bonus, Loyalty, and Referral each emit a common `rewards.value-granted` event (shared contract, per ADR-002). Vaullet consumes it and, in one transaction: **take `account_balances FOR UPDATE`**, insert a `balance_buckets` row, insert a `CREDIT` ledger entry, increment `account_balances.posted_balance`.

Grants are event-driven, not synchronous, which is what keeps all three modules independently removable — the rule from ADR-005 that optional functionality lives on the event side. Vaullet consuming an event type that no deployed module emits costs nothing.

### Settle and Release

On `transaction.completed`, in one transaction: **take `account_balances FOR UPDATE` first**, then insert one `ledger_entries` row per allocated bucket, decrement `held_total` on those buckets and on the account, apply amounts to `posted_balance`, and mark the reservation `SETTLED`. Idempotent via the unique index on `(transaction_id, bucket_id, direction)`.

On `transaction.failed` or cancellation: same lock first, then decrement `held_total` at both levels and mark `RELEASED`. No ledger entry — money that never moved leaves no journal record.

### Expiry

A sweeper releases abandoned holds:

```sql
UPDATE reservations SET state = 'EXPIRED'
 WHERE state = 'HELD' AND expires_at < now()
 RETURNING reservation_id, account_id;
-- then, per account and one account at a time:
--   BEGIN; SELECT ... FROM account_balances WHERE account_id = $1 FOR UPDATE;
--   release allocations, decrement bucket and account held_total; COMMIT;
```

The sweeper must lock **one account per transaction**, in the same account-first order as every other
path. Batching several accounts into one transaction would acquire account rows in whatever order the
sweep returned them, which is the one way this design can still deadlock against itself.

It runs every 30 seconds regardless of hold class — a long-lived hold simply is not due yet, so the
partial index `(state, expires_at) WHERE state = 'HELD'` keeps the sweep proportional to what has
actually expired rather than to how many holds are open.

Settlement uses `WHERE state = 'HELD'` and expiry uses `WHERE state = 'HELD' AND expires_at < now()`, so exactly one wins. If expiry wins, settlement affects zero rows and Vaullet publishes `ledger.settlement-rejected.v1`; Transaction Service fails the transaction. The state machine makes this safe rather than the timing: whichever transaction commits first, the other sees a row that is no longer `HELD` and takes the losing branch. There is no window in which both apply, and no path to an overdraft.

Timing only decides how *often* the loss happens. For an authorization hold at the 300-second default against a sub-second happy path it is rare and loud. For a two-phase hold the caller sets an expiry beyond its own settlement horizon, and an expiry that fires means the capture never came — which is the outcome the operator needs to hear about, hence `ledger.hold-expired.v1`.

Expired *buckets* are separate from expired *reservations*: a promotional bucket past `expires_at` is excluded from allocation and swept to zero with a `DEBIT` entry, so forfeiture is journalled rather than silent.

### Redis

Redis stays — for balance-read caching, rate limiting, and sessions. It is no longer load-bearing for correctness. Losing Redis degrades latency; it can no longer cause an overdraft.

## Consequences

### Positive Consequences

✅ **Overdrafts are actually prevented** — the invariant is enforced by an ACID transaction and a `CHECK` constraint, not by a lock whose scope excludes the write
✅ **One fewer critical dependency** — Redis leaves the correctness path; no lock TTL to tune against a latency budget, no split-brain, no lock lost on failover
✅ **Idempotent by construction** — a unique constraint, not a convention
✅ **Crash-safe** — a crashed caller leaves a hold that expires cleanly; ADR-001's design left a lock that expired *while the operation continued*
✅ **Holds are a product feature** — "pending balance" is something wallet users expect to see; ADR-001 had nowhere to represent in-flight money
✅ **Per-account serialization only** — different accounts never contend, so system-wide throughput is unaffected
✅ **Rewards modules stay independent** — Bonus, Loyalty and Referral each grant into the ledger without knowing about one another, and any subset can be deployed
✅ **Withdrawal rules are enforced where the money lives** — `withdrawable` is a ledger property, so no module can accidentally cash out locked promotional funds

### Negative Consequences

⚠️ **Vaullet exposes a synchronous command API.** It was described as append-only and producer-only (`arch.md`). It now accepts `POST /reservations` on the request path. `arch.md` must be updated; "producer-only" was never compatible with being the authority on available balance.

⚠️ **`account_balances` is mutable state inside the "immutable" service.** To be precise about what append-only means here: `ledger_entries` is the immutable journal and remains the source of truth. `account_balances` and `reservations` are a derived projection, maintained transactionally and fully rebuildable by replaying the journal. This is the standard journal-plus-balances split every real ledger uses; the previous docs overstated the invariant.

⚠️ **Vaullet is now on the latency critical path** for every transaction, and its availability bounds transaction availability.

⚠️ **Bucket allocation is materially more complex than a single balance** — greedy allocation across N buckets, per-bucket holds, and per-bucket settlement are the price of a real bonus platform. A deployment with no rewards module carries the schema but exercises only the one-bucket path.

⚠️ **A deployment is locked to one currency.** An operator who later needs a second currency
requires a migration, not a config change. Accepted: currency is a licensing-and-banking decision
that changes at contract speed, and the per-account-per-currency extension above is a documented
path rather than a redesign.

⚠️ **Two-phase lifecycle to implement** — reserve/settle/release/expire is more moving parts than a lock-and-check, and every path needs a test.

⚠️ **A caller can hold a user's funds for days.** Caller-supplied lifetimes are what ADR-012's wagering
flow requires, but the ledger has handed the duration of an unavailability to its client. `max_hold_seconds`
bounds it, `ledger.hold-expired.v1` reports it, and `vaullet.reservations.open.age.p99` watches it — but
the operator, not Vaullet, decides how long a stake stays locked.

### Risks

| Risk | Impact | Mitigation |
|------|--------|-----------|
| A house or merchant account is later added as a transaction counterparty | Every transaction serializes on one row; the throughput target fails | The journal is single-sided by decision (see *Scope*), and the operator's position lives downstream. Introducing a counterparty account requires balance-sharding it into N sub-accounts **in the same change**, not afterwards |
| Holds accumulate from stuck transactions | Legitimate transactions rejected as insufficient funds | Caller-supplied expiry (300s default), sweeper every 30s, alert on `held_total / posted_balance` ratio |
| A long two-phase hold is never captured | User's funds unavailable for as long as the caller asked for | `max_hold_seconds` ceiling, `ledger.hold-expired.v1` on release, and an alert on holds older than a deployment-set threshold |
| A caller sets a long TTL on an ordinary authorization | Funds locked far beyond the transaction that needed them | The default is short and explicit; a long TTL is a deliberate parameter, logged and visible in `GET /v1/reservations/{id}` |
| Settlement/expiry race | Transaction fails after user saw approval | Expiry ≫ happy-path duration; failure is explicit and compensable, never an overdraft |
| Long-running reserve transaction holds a row lock | Head-of-line blocking per account | Statement timeout of 1s; the transaction does no network I/O while holding the lock |
| Projection drifts from journal | Wrong balances | Nightly reconciliation replaying `ledger_entries`; alert on any mismatch |
| Bucket count grows unbounded on heavy promo users | Allocation scan slows | Sweep zero-balance and expired buckets; index on `(account_id, spend_priority)`; alert above ~50 active buckets per account |
| Spend priority misconfigured | Users forfeit bonuses, or cash spent before expiring promo money | Priority is module-owned and covered by tests; reconciliation reports forfeited amounts daily |

## Alternatives Considered

### Alternative 1: Keep the Redis lock, extend it to cover the ledger write

**Description**: Hold the lock until Vaullet has synchronously written the ledger entry.

**Pros**: Smallest change from ADR-001; keeps Vaullet's write path event-driven.

**Cons**: The lock would span a cross-service call plus a database write, making hold times *longer* exactly where ADR-001 needed them short. TTL-vs-latency gets worse, not better. And it still fails on Redis failover, because correctness would depend on a lock that async replication can lose. Using a distributed lock to protect a cross-service invariant is the case distributed locks handle worst.

**Why rejected**: Strictly more complexity than the reservation model for strictly weaker guarantees. If the database transaction can enforce the invariant, the lock is redundant; if it can't, the lock can't either.

---

### Alternative 2: Optimistic concurrency (version column + CAS + retry)

**Description**: `UPDATE account_balances SET ... WHERE version = $expected`; retry on zero rows affected.

**Pros**: No row locks held; same correctness guarantee; often better under low contention.

**Cons**: Retry storms on hot accounts; no fairness, so an unlucky request can starve; retry logic in every caller.

**Why rejected — provisionally**: This is a legitimate variant of the same decision, not a worse one. `SELECT FOR UPDATE` gives fairness and simpler code at our contention levels. If per-account contention proves low in load testing, switching is a localized change inside Vaullet with no API impact.

---

### Alternative 3: Kafka partitioning + stateful stream processing

**Description**: ADR-001's documented future path. Partition by account, single stateful processor per partition holds balances in memory.

**Pros**: Very high throughput; strong consistency without any shared-state coordination.

**Cons**: State recovery and rebalancing complexity; still needs a synchronous response path for a REST API, so it doesn't remove the request/response problem, only moves it.

**Why rejected for now**: Correct future path at much higher scale, unchanged from ADR-001's assessment. Postgres row locks handle 1000 TPS spread across accounts comfortably. Revisit above ~5000 TPS or when a single account exceeds ~50 TPS.

---

### Alternative 4: Eventual consistency with compensating reversals

**Description**: Approve optimistically, detect overdrafts asynchronously, reverse.

**Why rejected**: Unchanged from ADR-001 — zero tolerance for overdrafts is a hard requirement, and "approved then reversed" is unacceptable for a wallet.

---

### Note on ADR-001's rejection of database row locking

ADR-001 rejected `SELECT ... FOR UPDATE` as "5–10ms vs 1–2ms for Redis" and "database becomes bottleneck." That comparison does not hold up:

- It compares Redis lock *acquisition* against a full database round-trip. An uncontended `SELECT FOR UPDATE` on a primary key is well under 1ms.
- The Redis lock never replaced the Postgres write — it added a network hop *on top of* it. It was strictly more work per transaction, not less.
- "Database becomes bottleneck" assumes global contention. Row locks contend per account, and per-account transaction rates are inherently low.

The row-locking option was rejected on a performance argument that was never measured, in favor of an approach that was faster only because it wasn't doing the work that made it correct.

## Implementation Notes

### API (Vaullet)

```
POST   /v1/reservations           → 201 {reservation_id, allocations[], available_after, expires_at}
                                  | 409 INSUFFICIENT_FUNDS | 422 HOLD_TTL_TOO_LONG
GET    /v1/reservations/{id}      → 200 {state, amount, allocations[], expires_at}
DELETE /v1/reservations/{id}      → 204 (release)
GET    /v1/accounts/{id}/balance  → {posted, held, available, withdrawable,
                                     buckets: [{type, source_module, available,
                                                withdrawable, wagering_remaining,
                                                expires_at}]}
```

`Idempotency-Key` header required on `POST /v1/reservations`.

`POST /v1/reservations` accepts an optional `expires_in_seconds` (default 300, ceiling
`ledger_config.max_hold_seconds`, `422 HOLD_TTL_TOO_LONG` beyond it). `GET /v1/reservations/{id}`
returns state, allocations and `expires_at`, so a two-phase caller can see what it is holding.

`POST /v1/reservations` takes an explicit `currency` and returns `422 CURRENCY_MISMATCH` if it does
not equal `ledger_config.currency`. `GET /v1/accounts/{id}/balance` returns the currency alongside
every amount, so no caller has to infer it from configuration.

Added with wallets (amended 2026-10-22, see *Balance buckets*):

```
POST   /v1/wallets                → 201 the wallet, opened or existing | 409 WALLET_CLOSED
GET    /v1/wallets                → 200 {data[], next_cursor}
GET    /v1/wallets/{id}           → 200 the wallet
PATCH  /v1/wallets/{id}           → 200 (ACTIVE ↔ FROZEN) | 409 WALLET_CLOSED
DELETE /v1/wallets/{id}           → 204 (close, only when empty) | 409 WALLET_NOT_EMPTY
GET    /v1/wallets/{id}/balance   → {currency, posted, held, available, withdrawable, debt, buckets[]}
```

The scopes for every ledger endpoint are in [ADR-008](008-service-to-service-authentication.md) Layer 3.

### Revised transaction flow

```
Transaction Service
  1. Auth Service (REST)          — validate token
  2. Fraud Detection (REST)       — risk score
  3. Limits Service (REST)        — limit check
  4. Vaullet POST /reservations   — atomic check-and-hold  ← invariant enforced here
  5. Persist transaction (APPROVED), publish transaction.created
  ...
  6. On completion: publish transaction.completed → Vaullet settles
     On failure:    publish transaction.failed    → Vaullet releases
```

No lock acquire/release steps. Step 4 is the only place the balance invariant is decided.

### Migration path

None required. ADR-001 was never implemented — no code exists yet. This is a correction on paper, which is the cheapest place to make it.

### Monitoring

- `vaullet.reserve.latency.p95` — target <20ms
- `vaullet.reserve.rejected{reason=INSUFFICIENT_FUNDS}` — business signal, not an error
- `vaullet.reservations.expired.count{class=authorization}` — should be near zero; non-zero means transactions are stalling
- `vaullet.reservations.expired.count{class=two_phase}` — an operator failed to capture; a business signal, reported rather than alerted
- `vaullet.reservations.open.age.p99` — long holds are legitimate, unbounded ones are not
- `vaullet.held_ratio{account}` — alert above 0.5
- `vaullet.row_lock.wait.p95` — early warning for hot accounts
- `vaullet.reconciliation.mismatch.count` — must be zero
- `vaullet.buckets.active{account}` — allocation cost scales with this; alert above 50
- `vaullet.buckets.forfeited.amount` — promotional money expired unspent; a business metric the Bonus module owns

## References

- Supersedes [ADR-001: Use Redis Distributed Locking for Balance Consistency](001-distributed-locking-for-balance-consistency.md)
- Related: [ADR-003: Hybrid Database Strategy](003-hybrid-database-strategy-with-analytics.md) — `vaullet_db` isolation
- [Martin Kleppmann — How to do distributed locking](https://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html) — why Redlock is unsuitable for correctness-critical use
- [Designing Data-Intensive Applications](https://dataintensive.net/) — Ch. 7 (Transactions), Ch. 9 (Consistency and Consensus)
- Journal-plus-projection: an immutable single-sided journal with a transactionally-maintained balance
  aggregate. Not double-entry — there is no counterparty row; see *Scope: a single-sided journal*
- Related: [ADR-009: Payment Rails](009-payment-rails-deposits-and-withdrawals.md) — the `DEBT` bucket
- Related: [ADR-012: External API Surface](012-external-api-surface.md) — two-phase transactions, which
  set the hold lifetimes this ADR now takes from the caller

---

**Date**: 2026-08-27
**Author**: Predrag
**Reviewers**: TBD
