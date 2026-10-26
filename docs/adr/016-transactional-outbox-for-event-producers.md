# ADR-016: Transactional Outbox for Event Producers

## Status

**Proposed** (2026-10-22)

Decided on 2026-10-14 and built before it was written down: the outbox shipped in
wallet-auth-service 0.12.0 (wallet-auth-service#39), and production runs it as 0.12.2. This record
states what that release does and why. The number is already cited by the service's
`V3__outbox.sql` and README, which refer to §2 below.

Resolves **R8**. Amends [ADR-007](007-kafka-topics-and-event-schema-evolution.md) (a new §11, and the
§9 inventory), [ADR-014](014-user-management-in-local-identity-mode.md) §7,
[ADR-004](004-atomic-balance-reservations.md) (who opens an account's rows in the ledger) and
[ADR-008](008-service-to-service-authentication.md) Layer 3 (the ledger's scopes, closing **B4**).
Opens **B5**, **G6**, **G7** and **R9**.

## Context

[ADR-014](014-user-management-in-local-identity-mode.md) §7 decided that creating an account creates
its row in the ledger, by event: Auth Service publishes `identity.account-created.v1`, and the ledger
consumes it idempotently. It also recorded what that exposed — **there is no outbox anywhere in the
architecture**. Publishing after the commit loses the event when the process dies between the two,
and the register gave that a trigger: decide before a second producer exists (**R8**).

Three things changed before the second producer arrived:

- **The ledger grew wallets** (wallet-ledger-service#10, 0.1.0). An account's money sits in a `REAL`
  wallet (`CASH`, and `DEBT` once ADR-009 creates one) and a `BONUS` wallet (the promotional
  buckets). `POST /v1/wallets` opens one; with an account's first it creates the `account_balances`
  row, and a `REAL` wallet starts with its `CASH` bucket. "The ledger's row" became "the account's
  two wallets".
- **The ledger was deployed** (0.2.x, gitops#29) — with no route, called only from inside the
  cluster — and production already held accounts with no wallets. An account without a `REAL` wallet
  is a customer who cannot be paid.
- **There is still no Kafka.** The cluster is one machine ([ADR-015](015-cluster-topology-three-servers-on-one-machine.md)),
  and the event has exactly one consumer.

### Constraints

- **Creating an account never waits for the ledger, and never fails because of it.** This is why
  ADR-014 §7 rejected a synchronous call, and it still holds.
- **No event is lost.** [ADR-007](007-kafka-topics-and-event-schema-evolution.md) §10: financial
  events are never dropped. An account-created event moves no money, but its loss strands every
  deposit that follows.
- **Accounts that already exist are covered**, not only new ones.
- **Several replicas and rolling deploys.** Whatever delivers must be safe on any number of replicas
  at once, and across a rollout in which the previous version keeps creating accounts without
  writing any event.
- **No broker for one event.** Deploying Kafka on the one machine to carry one event type to one
  consumer is not justified yet.

### The question

How does a service make an event exist exactly when the change it describes does — and how is it
delivered while there is nothing to publish it to?

## Decision

### 1. Every producer writes its events to an outbox, in the transaction that makes the change

A producer never publishes after its commit, and never from inside its transaction. It inserts the
event into an outbox table **in its own schema**, in the same transaction as the change the event
describes; a relay in the same service delivers it afterwards.

- **Published after the commit**, the event is lost whenever the process dies in between — the **R8**
  gap.
- **Published inside the transaction**, it can announce a change that then rolls back, and it holds
  the transaction open across a network call.
- **Written in the transaction**, the event and the change commit or roll back together. Neither can
  exist without the other.

In wallet-auth-service the writer, `Outbox.accountCreated`, **refuses to run outside a transaction**
(`Propagation.MANDATORY`): committed on its own, an entry would bring back the gap it closes.

Each service owns its outbox the way it owns every other table ([ADR-003](003-hybrid-database-strategy-with-analytics.md)).
No service reads another's.

### 2. An outbox row is ADR-007's envelope, column for field

The row **is** the message: [ADR-007](007-kafka-topics-and-event-schema-evolution.md) §3's envelope,
one column per field, with the `producer` object flattened into two. Next to it sits the relay's
bookkeeping, which is no part of any event.

```sql
CREATE TABLE outbox (                       -- identity_schema, V3__outbox.sql
    -- the envelope (ADR-007 §3)
    event_id         UUID        NOT NULL,  -- consumers dedupe on it
    event_type       TEXT        NOT NULL,  -- 'identity.account-created'
    schema_version   TEXT        NOT NULL,  -- '1.0.0'
    occurred_at      TIMESTAMPTZ NOT NULL,  -- the account's created_at
    producer_service TEXT        NOT NULL,  -- 'auth-service'
    producer_version TEXT        NULL,      -- the build; NULL for backfilled and seeded rows
    correlation_id   TEXT        NULL,      -- trace id of the request; NULL where there was none
    causation_id     TEXT        NULL,      -- the event that caused this one; NULL here
    account_id       UUID        NOT NULL,
    payload          JSONB       NOT NULL,  -- an object; '{}' for this event
    -- delivery
    queued_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
    attempts         INTEGER     NOT NULL DEFAULT 0,
    next_attempt_at  TIMESTAMPTZ NOT NULL DEFAULT now(),   -- due time, and the lease
    last_attempt_at  TIMESTAMPTZ NULL,
    last_error       TEXT        NULL,      -- kept after delivery
    delivered_at     TIMESTAMPTZ NULL
);
```

Plus: a foreign key to the producer's own aggregate (`accounts`); a `CHECK` listing the event types,
so a new type is a migration and gets a reviewer; a partial unique index giving each account one
`identity.account-created`; and a partial index over the pending rows, which is all the relay polls.

**Why column for field**: when Kafka arrives the relay changes and the table does not. Nothing has to
be reshaped, backfilled or re-derived to publish a row that has been waiting.

### 3. A relay in the producer delivers, at least once

Every replica runs it. **No transaction is open across a delivery**; a round is three short steps:

1. **Claim** — one statement takes up to 20 due rows `FOR UPDATE SKIP LOCKED` and moves each one's
   `next_attempt_at` a lease (2 minutes) ahead, then commits. Two replicas claiming at once skip each
   other's rows, and a leased row is no longer due.
2. **Deliver** — outside any transaction (§4).
3. **Record** — one statement per row: delivered, or failed and due again after a backoff. **Only
   under the lease it was won under**: the claim returns the `next_attempt_at` it set, and the
   outcome statement requires the row to still hold it. A replica that stalled past its lease writes
   nothing, because the row is another replica's by then.

A replica that dies mid-round loses nothing: its rows come due when their lease ends. A round stops
starting deliveries once the lease left is shorter than the longest delivery could take.

**Failures are retried forever**, 10 seconds after the first and doubling to at most 15 minutes —
ADR-007 §10's rule applied to the relay. They are counted in two kinds: `unavailable` (no answer, or
`429`/`5xx`, or a `408` from Keycloak) mends itself; `refused` (anything else that is not the expected
answer) needs a person, and is logged as an error every time.

**At least once** means the receiving side must be idempotent: on `event_id` for a Kafka consumer
(ADR-007 §3), and for the HTTP stand-in below, by the ledger's own rule of one wallet per type.

### 4. Until there is Kafka, the relay delivers over HTTP, as the calls the consumer would make

`identity.account-created` is delivered as the ledger's `POST /v1/wallets`, `{"account_id", "type"}`,
first for `REAL` and then for `BONUS`:

- **Where**: the ledger's in-cluster Service (`LEDGER_URL`, `http://ledger-service.wallet-ledger.svc`).
  The ledger has no public route.
- **As whom**: a client-credentials token of `wallet-auth-service` — the client the Admin API already
  uses — asking for the optional scope `ledger:write` (gitops `realm/30-wallet-auth-service.yaml`).
  Cached apart from the Admin API's token, until 30 seconds before it expires; a `401` from the ledger
  drops it.
- **Done** means both wallets exist: a `201` for each that names the account and the type asked for,
  whether that call opened the wallet or an earlier one did — or `409 WALLET_CLOSED`, which is
  terminal and logged as a warning. Nothing else counts, so a `LEDGER_URL` that names something else
  fails loudly instead of marking rows delivered with no wallets behind them.
- **Idempotent by construction**: an account has at most one wallet of each type
  (`wallets_one_per_type`), and opening one that exists answers `201` with it. A repeated delivery
  changes nothing, which is why no ledger inbox was built.

**When Kafka arrives**, the relay publishes each row to `identity.account-created.v1`, keyed by
`account_id` (ADR-007 §6), and a consumer in the ledger opens the wallets. The table and its writers
do not change. Tracked as **R9**.

**What this accepts in the meantime**: the ledger takes a write from the identity adapter, which
ADR-014 §7 objected to. The write is the ledger's narrowest — opening a wallet moves no money, and
repeating it is harmless — but the scope that allows it also freezes and closes wallets, and the
token that carries it can do more than open wallets (**G7**). Until 2026-10-26 the same scope placed
and released holds too (**G6**); those need `ledger:reserve` now (ADR-008), which the relay is never
given.

### 5. Rows no request wrote: the backfill and the sweep

- **Backfill.** `V3__outbox.sql` gave every account that is not `CLOSED` the row it would have had,
  in the migration's own transaction. Creating the foreign key locks `accounts` against writes until
  V3 commits, so V3 waits at most two seconds to take that lock (`SET LOCAL lock_timeout = '2s'`)
  and fails whole rather than queue every account write behind a long transaction.
- **Sweep.** At the relay's start and every 10 minutes, one
  `INSERT … SELECT … ON CONFLICT DO NOTHING` queues every account that is not `CLOSED` and has no
  row. It covers the previous version's accounts during a rollout, and any path that forgets. It
  normally finds nothing; when it does, it warns and counts `auth_outbox_swept_total`.
- **Rollout order.** The migration Job is sync wave 0, with the relay off
  (`AUTH_OUTBOX_RELAY_ENABLED=false`); the Deployment is wave 1, so no pod of the new version starts
  before the table exists.

`CLOSED` accounts are neither backfilled nor swept. An account closed after its row was written still
gets its wallets: the event is that it was created.

### 6. The event: `identity.account-created` 1.0.0

- **Payload `{}`**: `account_id` and `occurred_at` — the account's `created_at` — are the whole fact.
  A field added later is a minor version on the same topic (ADR-007 §4).
- **`correlation_id`** is the trace id of the request that created the account, the same id its error
  bodies carry as `trace_id`, so one search finds the request and every delivery attempt.
  **`causation_id`** is always `NULL`: a request creates an account, never another event.
- **Writers**: `AccountService.create` and `EndUserService.create`, each in the transaction that
  inserts the anchor; V3's backfill; the demo seed (`k8s/test-users`); the sweep. The account at first
  sign-in (wallet-auth-service#24) must write it in its anchor's transaction too.

**The name is recorded as built, and its domain segment is open (B5).** ADR-007 names a topic's
domain after the producing service and renamed `user.registered` to `auth.user-registered.v1`;
ADR-014 §7 named this event `identity.account-created.v1`, and 0.12.0 shipped it. Renaming it is no
longer free: V3's backfill wrote `identity.account-created` into every production row, and the
table's `CHECK` and partial unique index name it, so a rename is now a data migration of `auth_db` —
and, once anything is published, a new topic under ADR-007 §4.

### 7. Delivered rows are kept

A delivered row stays, with its `last_error`, as the record of why it needed more than one attempt.
The service could not delete it anyway: V2's default privileges give `auth_app` `SELECT`, `INSERT`
and `UPDATE` on new tables, and no `DELETE`. Retention belongs with **D3**; at a few hundred bytes per
account it is not urgent.

## Consequences

### Positive Consequences

✅ **An event exists exactly when its change does** — no crash window loses one; **R8** closes
✅ **Account creation never waits for the ledger**; a ledger outage is a backlog, delivered within a quarter of an hour of recovery
✅ **Accounts that predate the outbox are covered** — V3's backfill queued every one that is not `CLOSED`
✅ **Safe on any number of replicas** without a leader or a coordinator: `SKIP LOCKED` plus a lease that doubles as a token
✅ **Kafka changes the relay, not the data** — every waiting row is already the message
✅ **The second producer copies a pattern** instead of rediscovering the gap

### Negative Consequences

⚠️ **The ledger accepts a write from the identity adapter until Kafka**, and Auth Service knows the
ledger's URL and contract — the coupling the event was meant to avoid. Bounded by **R9**.

⚠️ **`ledger:write` is the only scope that opens a wallet**, so the relay's token could also freeze and
close wallets on any account. It cannot hold money: since 2026-10-26 that takes `ledger:reserve`
(ADR-008), which closed **G6**.

⚠️ **The relay's token also passes Keycloak's Admin API**, because it comes from the client that
manages users (**G7**).

⚠️ **Polling costs something even when idle**: every replica asks for due rows every 5 seconds, and
the sweep reads every account every 10 minutes — an anti-join the unique index answers, which grows
with the number of accounts.

⚠️ **V3's foreign key held `accounts` against writes until the migration committed.** Milliseconds at
today's size; the same shape on a large table needs a different migration.

⚠️ **Every account gets a `BONUS` wallet**, whether or not the operator sells Bonus, and an account
closed before its row was delivered still gets both, empty.

⚠️ **Delivered rows accumulate** until D3 says otherwise.

### Risks

| Risk | Impact | Mitigation |
|---|---|---|
| A new creation path inserts an account without its row | That account has no wallets until the sweep, up to 10 minutes | The writer refuses to run outside a transaction; `AccountApiIT` and `EndUserApiIT` assert the row for each path; the sweep is the backstop and warns when it finds anything |
| The ledger is down, or refuses, for a long time | New accounts cannot be credited | `auth_outbox_oldest_pending_seconds` is the alert; `refused` is logged as an error on every attempt; rows are never given up on |
| `LEDGER_URL` names something that answers `2xx` | Rows marked delivered with no wallets | Only a `201` naming the account and type asked for counts as done |
| A replica stalls past its lease | The same row is delivered twice | An outcome is recorded only under its own lease; a repeated open is harmless |
| The relay's token leaks | Wallet changes on any account, and user management in the realm, until it expires | Short token lifetime; no public route to the ledger; no hold without `ledger:reserve`, which the relay is never given; **G7** records the rest |
| **B5** is settled after the first publish | A topic rename means dual-publishing (ADR-007 §4) | Settle B5 before the relay publishes to Kafka |

## Alternatives Considered

### Alternative 1: Publish after the commit

**Description**: ADR-014 §7 as written — commit the account, then publish.

**Pros**: Nothing to build beyond the publish call.

**Cons**: A crash between the commit and the publish loses the event for good, and nothing notices.

**Why rejected**: it is the gap R8 recorded. It was survivable for the first producer only by
accident.

---

### Alternative 2: Call the ledger in the request

**Description**: Auth Service opens the wallets before answering `201`.

**Pros**: Synchronous and simple; the wallets exist when the caller hears back.

**Cons**: Account creation fails whenever the ledger does, and the write spans three systems with no
transaction between them.

**Why rejected**: ADR-014 §7 rejected it for those reasons, and they have not changed.

---

### Alternative 3: Change data capture from the write-ahead log

**Description**: Debezium, or plain logical replication, reads the committed rows and publishes them.

**Pros**: No relay code; publishing cannot be forgotten by a code path.

**Cons**: Needs Kafka Connect and a replication slot per producer, per customer cluster. A slot whose
reader stops retains WAL until the disk fills — on a machine with roughly 220 GB for everything
(ADR-015). And there is no Kafka to publish to.

**Why rejected — for now**: heavier than the problem. It is a candidate for the relay's job once
Kafka exists, and the outbox table is what it would read.

---

### Alternative 4: Deploy Kafka now

**Description**: Install the broker and build the real consumer immediately.

**Pros**: No interim; ADR-014 §7 exactly as written.

**Cons**: A broker on one machine for one event type with one consumer — the memory, the operation
and RF=3 on three VMs sharing one disk array — before anything else needs it.

**Why rejected**: deferred, not refused. **R9** names the trigger.

---

### Alternative 5: The ledger reads the accounts

**Description**: The ledger polls `identity_schema.accounts`, or a view of it, and opens wallets for
accounts it has not seen.

**Pros**: No relay, no token.

**Cons**: One service reading another's database, against ADR-003's ownership; the ledger would learn
the auth schema, and every change to it would become a cross-service migration.

**Why rejected**: the dependency points the wrong way and lands in the most protected service.

---

### Alternative 6: A ledger inbox

**Description**: A ledger table of received `event_id`s, so a redelivered event is recognised and
ignored.

**Pros**: The general consumer-side answer to at-least-once delivery.

**Cons**: Nothing to protect here: opening a wallet is idempotent by the ledger's own unique
constraint.

**Why rejected**: dropped on 2026-10-14 as redundant. The Kafka consumer still needs durable dedupe on
`event_id` (ADR-007, CI rule 4), because the next event it consumes may not be idempotent by
construction.

## Implementation Notes

### Where it lives

wallet-auth-service, since 0.12.0:

- `src/main/resources/db/migration/V3__outbox.sql` — the table, its indexes and the backfill
- `src/main/resources/db/sql/outbox/` — `insert`, `claim-due`, `mark-delivered`, `mark-failed`,
  `queue-missing` (the sweep) and `backlog` (the gauges)
- `outbox/service/` — `Outbox` (the writer and the gauges), `OutboxRelay`, `OutboxRelaySchedule`,
  `Backoff`; `outbox/dao/` — `OutboxRepository` and `LedgerWallets` (the HTTP client)
- `k8s/migrate-job.yaml` (wave 0, relay off) and `k8s/deployment.yaml` (wave 1, relay on,
  `LEDGER_URL`)

The service README's *The account-created outbox* is the operating manual: configuration, the
`last_error` catalogue, and the queries for an incident.

### Configuration

| Property | Default | |
|---|---|---|
| `auth.outbox.relay.enabled` | `true`; `false` in `local`, `test` and `migrate` | Off, rows still queue; on again, they are delivered |
| `auth.ledger.url` | — | Required while the relay is on |
| `auth.outbox.relay.poll-interval` · `batch-size` · `lease` | `5s` · `20` · `2m` | The lease must outlast one delivery: 40 seconds at the default timeouts |
| `auth.outbox.relay.initial-backoff` · `max-backoff` | `10s` · `15m` | |
| `auth.outbox.relay.sweep-interval` | `10m` | |

The relay needs `KEYCLOAK_URL` and the client secret **in either identity mode**, and the client must
offer `ledger:write` as an optional scope (gitops#30).

### Signals

`auth_outbox_pending` and `auth_outbox_oldest_pending_seconds` are read from the table at each scrape,
so they are right on a replica whose relay is off; the second is the one to alert on.
`auth_outbox_delivered_total`, `auth_outbox_failed_total{reason}` and `auth_outbox_swept_total` count
per replica. Each delivery logs the account, the event and the correlation id.

### When Kafka arrives

1. Settle **B5**, then author the schema in `wallet-shared-contracts` (ADR-002, ADR-007 §5).
2. The relay publishes each row to the topic, keyed by `account_id`. With more than one event type
   per account it must keep an account's rows in order — the partition key gives that on Kafka, and
   the HTTP relay never had to.
3. The ledger's consumer opens the wallets, with durable dedupe on `event_id`.
4. Remove `ledger:write` from the `wallet-auth-service` client. That ends the relay's use of the
   wallets' scope, and **G7** with it. Its part of **G6**, holds, ended on 2026-10-26.

### The next producer

Copy the shape, not the code: an outbox table in its own schema with the envelope columns and the
bookkeeping, the event written in the transaction that makes the change, a relay with the same claim,
lease and backoff, and its event type added to its own `CHECK`.

## References

- [ADR-003: Hybrid Database Strategy](003-hybrid-database-strategy-with-analytics.md) — each service owns its schema
- [ADR-004: Atomic Balance Reservations](004-atomic-balance-reservations.md) — the ledger, and its wallets as amended
- [ADR-007: Kafka Topics and Event Schema Evolution](007-kafka-topics-and-event-schema-evolution.md) — the envelope (§3), never dropping (§10), and §11, added by this ADR
- [ADR-008: Service-to-Service Authentication](008-service-to-service-authentication.md) — Layer 3, and the ledger's scopes
- [ADR-014: User Management in Local Identity Mode](014-user-management-in-local-identity-mode.md) — §7, which this implements
- [ADR-015: Cluster Topology](015-cluster-topology-three-servers-on-one-machine.md) — why there is no broker yet
- wallet-auth-service README — *The account-created outbox*; `V3__outbox.sql`
- wallet-ledger-service README — *Wallets*; `V2__wallets.sql`
- gitops — `apps/keycloak-config/realm/15-vaullet-client-scopes.yaml`, `realm/30-wallet-auth-service.yaml`
- [Transactional outbox pattern](https://microservices.io/patterns/data/transactional-outbox.html)
- [PostgreSQL — `SELECT … FOR UPDATE SKIP LOCKED`](https://www.postgresql.org/docs/current/sql-select.html#SQL-FOR-UPDATE-SHARE)

---

**Date**: 2026-10-22 (decided 2026-10-14; built in wallet-auth-service 0.12.0)
**Author**: Predrag
