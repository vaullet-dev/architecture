# ADR-007: Kafka Topic Naming and Event Schema Evolution

## Status

**Accepted** (2026-08-27)
**Revised** (2026-09-02) — schemas are authored, not generated from Java (ADR-002); topic inventory
extended with the events introduced by ADRs 004 and 009

Extends [ADR-002](002-shared-contracts-versioning-strategy.md), which versions the shared-contracts
*artifact* but says nothing about the wire.

## Context

Vaullet is event-driven by design, and [ADR-005](005-module-composition-and-deployment-topology.md)
makes that load-bearing commercially: modules are removable precisely *because* they communicate only
through events. That only holds if the event layer itself has rules.

### The gap ADR-002 leaves

ADR-002 gives the `wallet-shared-contracts` artifact a semantic version and tests backward
compatibility at build time. That is a **library** guarantee, not a **wire** guarantee. They are
different problems:

- A consumer running contracts `v2.1` will read events produced by a service on `v2.9`. Semver on the
  jar says nothing about whether that deserialises.
- Events already written to a topic cannot be recompiled. A field removed in `v3.0` still exists in
  three weeks of retained messages.
- Under polyrepo deployment there is **no controlled upgrade order**. Producer-first and
  consumer-first must both be safe, because either can happen.

### What is actually in use today

Auditing every event named across `arch.md` and ADRs 001–006 turned up a naming layer that grew by
accretion:

| Pattern | Examples | Problem |
|---|---|---|
| PascalCase | `ValueGranted` | The single most load-bearing event in the system, and it matches nothing else |
| Kebab in a segment | `recurring-payment.due` | Mixes separators against the dot-case majority |
| Two segments | `transaction.created`, `bonus.applied` | Fine |
| Three segments | `transaction.status.updated`, `fraud.suspicion.detected` | Inconsistent depth; `fraud.case.created` and `fraud.pattern.detected` disagree about what the middle segment means |
| No version anywhere | all of them | No path to a breaking change that is not an outage |

None of this is broken yet because none of it is built. It is exactly the moment to fix it.

### Constraints

- No controlled deploy order between services
- A customer's cluster runs only the modules they bought — topics may have zero consumers, and that
  must cost nothing
- Financial events may not be silently dropped
- ADR-003 asserts `datawarehouse_db` is rebuildable from the event stream

## Decision

### 1. Topic naming

```
<domain>.<event-name>.v<major>
```

- **`<domain>`** — the owning bounded context, singular, lowercase. The service that *produces* the
  event, never the one that consumes it.
- **`<event-name>`** — past tense, describing something that already happened. An event is a fact.
- **`v<major>`** — schema major version. See §4.

**Separator rule**: dots separate segments; hyphens separate words *within* a segment. Never
underscores — Kafka collides `.` and `_` in JMX metric names, and a topic named `a.b_c` and one named
`a_b.c` produce the same metric.

Exactly three segments. Always.

### 2. The renaming

| Current | Becomes |
|---|---|
| `ValueGranted` | `rewards.value-granted.v1` |
| `transaction.created` | `transaction.created.v1` |
| `transaction.approved` | `transaction.approved.v1` |
| `transaction.completed` | `transaction.completed.v1` |
| `transaction.failed` | `transaction.failed.v1` |
| `transaction.status.updated` | *removed* — superseded by the explicit lifecycle events above |
| `refund.initiated` / `.completed` / `.failed` | `refund.initiated.v1` / `refund.completed.v1` / `refund.failed.v1` |
| `ledger.entry.recorded` | `ledger.entry-recorded.v1` |
| `ledger.balance.updated` | `ledger.balance-updated.v1` |
| `ledger.settlement.rejected` | `ledger.settlement-rejected.v1` |
| `fraud.analysis.completed` | `fraud.analysis-completed.v1` |
| `fraud.suspicion.detected` | `fraud.suspicion-detected.v1` |
| `fraud.pattern.detected` | `fraud.pattern-detected.v1` |
| `fraud.case.created` / `.reviewed` | `risk.case-created.v1` / `risk.case-reviewed.v1` |
| `account.locked` / `.unlocked` | `risk.account-locked.v1` / `risk.account-unlocked.v1` |
| `bonus.applied` / `.revoked` / `.expired` | `bonus.applied.v1` / `bonus.revoked.v1` / `bonus.expired.v1` |
| `bonus.wagering.completed` | `bonus.wagering-completed.v1` |
| `loyalty.tier.changed` | `loyalty.tier-changed.v1` |
| `loyalty.points.redeemed` | `loyalty.points-redeemed.v1` |
| `referral.completed` | `referral.completed.v1` |
| `subscription.created` / `.suspended` | `subscription.created.v1` / `subscription.suspended.v1` |
| `recurring-payment.due` | `scheduler.payment-due.v1` |
| `recurring-payment.processed` | *removed* — the outcome is `transaction.completed.v1` |
| `limit.threshold.reached` | `limits.threshold-reached.v1` |
| `user.registered` | `auth.user-registered.v1` |

Two deletions are substantive rather than cosmetic:

**`transaction.status.updated` is removed.** A generic "something changed" event forces every consumer
to re-derive what happened by inspecting the payload, which is how consumers end up coupled to the
producer's state machine. The explicit lifecycle events already carry the meaning.

**`recurring-payment.processed` is removed.** `arch.md` previously had Subscription Service processing
its own payment and announcing the result; that duplicated the money path and was corrected so
Subscription requests a transaction like any other client. There is one event for money having moved,
and it is `transaction.completed.v1`. Two would drift.

Note `fraud.case.*` and `account.*` move to the `risk.` domain: Risk Management produces them, and the
domain segment names the producer.

### 3. The envelope

Every event carries the same envelope. Only `payload` varies.

```json
{
  "event_id":       "evt_01J8…",
  "event_type":     "transaction.completed",
  "schema_version": "1.4.0",
  "occurred_at":    "2026-08-27T14:32:11.482Z",
  "producer":       { "service": "transaction-service", "version": "2.3.1" },
  "correlation_id": "cor_01J8…",
  "causation_id":   "evt_01J8…",
  "account_id":     "acct_0f9c…",
  "payload":        { }
}
```

- **`event_id`** makes consumers idempotent. Kafka is at-least-once; every consumer that changes state
  must dedupe on this. Vaullet's ledger already enforces this at the database level (ADR-004) — the
  envelope generalises it to every consumer.
- **`correlation_id`** spans one user-initiated flow; **`causation_id`** is the `event_id` that caused
  this one. Together they reconstruct a full causal chain across services, which is what makes an
  event-driven system debuggable at 3am.
- **`schema_version`** is the full semver of the payload schema, distinct from the `v<major>` in the
  topic name.

### 4. Versioning: topic-per-major

**Additive, optional changes stay on the same topic.** New optional field, new enum value handled by a
documented default — same topic, minor version bump.

**Anything else gets a new topic.** Removing a field, renaming one, tightening a type, changing a
meaning: produce to `<domain>.<event>.v2`, dual-publish to both during migration, retire `v1` when its
consumer lag is zero and stays zero.

Breaking changes become a visible, gradual, reversible operational event rather than a coordinated
flag-day across a polyrepo. The cost is dual-publishing during migration; the alternative is
synchronised deploys, which ADR-005's independent-module model rules out.

**Schema Registry compatibility: `FULL_TRANSITIVE`.** Not `BACKWARD` (assumes consumers upgrade first)
and not `FORWARD` (assumes producers do). With independent deploys there is no order to assume, so
both directions must hold, against *all* prior versions rather than just the previous one — a chain of
individually-compatible steps is not itself compatible. `FULL_TRANSITIVE` mechanically enforces "only
additive and optional", which is precisely the rule above, so the registry rejects violations at
publish time rather than a reviewer catching them.

### 5. Format: JSON Schema

JSON on the wire, JSON Schema in the registry — and **the schema is authored, not derived**.

`schemas/<domain>/<topic>.json` in `wallet-shared-contracts` is the contract; the Java and TypeScript
event types are generated from it ([ADR-002](002-shared-contracts-versioning-strategy.md), *Two sources
of truth*). This follows directly from the argument this ADR is built on: events outlive the code that
produced them, money topics are retained indefinitely, and a contract that outlives every binding must
not be defined by one of them. Generating the schema *from* a Java class would also make wire
compatibility an accident of refactoring — rename a field, regenerate, and the registry is asked to
accept a break nobody wrote down. Authoring it puts the break in a pull request instead.

REST DTOs keep Java as their source of truth; they ship with the controller that serves them and do
not have this problem.

JSON keeps one representation end to end. Avro is more compact and has the better registry story, but its TypeScript support is
weak, and at >1000 TPS the payload-size difference is not the constraint. Debuggability — reading a
message off a topic during an incident without tooling — matters more at this scale.

Revisit if throughput reaches the point where serialisation cost is measurable.

### 6. Partitioning and ordering

**Key every account-scoped event by `account_id`.** Kafka orders within a partition only, so this is
what guarantees a grant lands before the spend that draws on it.

Events not scoped to an account key on their own aggregate id (`campaign_id`, `job_id`). Never key on
`event_id` — a unique key spreads one aggregate across every partition and destroys ordering.

**Partitions**: 12 for money-movement topics, 3 elsewhere. Partition count can be increased but never
decreased, and increasing it changes key-to-partition mapping — so this is closer to a one-way door
than it looks, and 12 is deliberate headroom rather than a current requirement.

### 7. Consumer groups

`<service-name>` — one group per service, matching the deployment unit. A module that isn't deployed
has no consumer group, so it raises no lag alerts and costs nothing (ADR-005).

Where one service consumes a topic for two independent purposes, `<service-name>.<purpose>`.

### 8. Retention

| Topic class | Retention | Rationale |
|---|---|---|
| Money movement (`transaction.*`, `ledger.*`, `refund.*`, `rewards.*`, `bonus.*`) | **90 days** *(revised 2026-09-17, was "tiered, indefinite")* | See below |
| Risk and fraud (`fraud.*`, `risk.*`) | 90 days | Investigation window |
| Operational (`limits.*`, `auth.*`, `scheduler.*`) | 7 days | Transport only |
| Config / state-carrying | Compacted | Latest value per key is the point |

**Kafka is transport, not the system of record.** `vaullet_db` is the ledger; Audit Service is the
compliance record. Retention exists for replay and rebuild, not durability of truth.

**On indefinite retention for money topics** — this is not a preference, it is a dependency.
[ADR-003](003-hybrid-database-strategy-with-analytics.md) states `datawarehouse_db` is rebuildable
from events with a 2-hour recovery objective. Under a 7-day retention that claim silently degrades to
"rebuildable for the last 7 days", which is not a rebuild; it is a gap nobody notices until a rebuild
is attempted under pressure. Either the events outlive the warehouse or ADR-003's recovery story is
false. Tiered storage (local SSD hot, object storage cold) makes indefinite retention affordable, and
the alternative — sourcing rebuilds from operational databases — reintroduces exactly the coupling
ADR-003 separated them to avoid.

> **Revised 2026-09-17: money topics retain for 90 days, not indefinitely.**
>
> Tiered storage needs an object store for the cold tier. There is none on the machine and
> [ADR-015](015-cluster-topology-three-servers-on-one-machine.md) admits no spend. The array has
> 401 GB free, of which roughly **220 GB remains after the three VM root disks** — for PostgreSQL,
> every Kafka topic, and everything else. The soak-test plan had already sized bounded retention at
> ~1.2 TB: **five times what exists.**
>
> 90 days is chosen as a deliberate limit rather than a number the disk forced. Indefinite retention
> on a staging-grade cluster is storage nobody is reading, and an unbounded topic is a slow outage
> with a date on it.
>
> **This breaks the paragraph above, and the break has to be resolved rather than noted.** ADR-003
> claims `datawarehouse_db` is rebuildable from events; with a 90-day bound that is true for 90 days
> and false before. The resolution follows from this ADR's own first sentence — *"Kafka is transport,
> not the system of record"*:
>
> **Rebuild the warehouse from `vaullet_db`, not from Kafka.** `ledger_entries` is append-only and
> retained seven years by decision (ADR-004); it *is* the record. Depending on a transport buffer to
> still be holding seven-year-old messages was treating Kafka as the system of record this ADR
> explicitly says it is not. Retention then only has to cover replay, reprocessing and consumer lag —
> days, not years — and 90 is generous for that.
>
> The cost is honest and should be stated: the warehouse ETL gains a batch path from an operational
> database, which is coupling ADR-003 separated them to avoid. **Needs a decision recorded in ADR-003
> before Reporting ETL is built.**
>
> **RF=3 is unaffected and becomes deliverable** with three nodes; on one node it was three replicas
> on one disk. Tracked as **G4** in [TODO.md](TODO.md).

### 9. Topics introduced after this ADR

Later decisions add topics; they follow the rule above rather than amending it, and are listed here so
the inventory stays in one place:

| Topic | Introduced by | Notes |
|---|---|---|
| `deposit.initiated.v1` · `deposit.captured.v1` · `deposit.failed.v1` | [ADR-009](009-payment-rails-deposits-and-withdrawals.md) | Money movement; `deposit.captured.v1` is what Bonus consumes for deposit-match promotions |
| `withdrawal.requested.v1` · `withdrawal.approved.v1` · `withdrawal.confirmed.v1` · `withdrawal.failed.v1` | ADR-009 | Money movement |
| `chargeback.received.v1` · `chargeback.resolved.v1` | ADR-009 | Money movement; produces the `DEBT` bucket |
| `ledger.hold-expired.v1` | [ADR-004](004-atomic-balance-reservations.md) | A reservation released without capture. For a long two-phase hold this is a business event the operator must be told about, not cleanup noise |

The `deposit.*`, `withdrawal.*` and `chargeback.*` domains are money movement for retention purposes
(§8) and key on `account_id` (§6).

### 10. Retry and dead letters

`<topic>.retry` (delayed redelivery, exponential backoff, capped at 5 attempts) and `<topic>.dlq`.

**Financial events are never dropped.** A message reaching a DLQ pages someone; it does not increment
a counter on a dashboard nobody watches. DLQ depth > 0 on a money-movement topic is an alert, not a
metric. Replay is manual and audited, because automatic replay of a poison pill reproduces the
failure.

## Consequences

### Positive Consequences

✅ **Breaking changes have a safe path** — a new topic and a migration window instead of a flag-day
✅ **No deploy-order assumption** — `FULL_TRANSITIVE` makes producer-first and consumer-first equally safe
✅ **Consumers can be idempotent** — `event_id` in every envelope, which at-least-once delivery makes mandatory
✅ **Flows are traceable across services** — correlation and causation reconstruct the causal chain
✅ **Ordering is guaranteed where money depends on it** — account-keyed partitioning
✅ **Undeployed modules stay free** — no consumer group, no lag, no alert
✅ **The registry enforces the rule** — violations fail at publish, not at review
✅ **One naming convention** — `ValueGranted` no longer stands alone

### Negative Consequences

⚠️ **`FULL_TRANSITIVE` is strict.** Some genuinely reasonable changes will require a new topic. That
strictness is the point, but it will feel heavy the first time a trivial-looking rename needs a v2.

⚠️ **Dual-publishing during migrations** means a period of writing every affected event twice, with
the storage and complexity that implies.

⚠️ **Indefinite retention on money topics costs money.** Tiered storage reduces it substantially but
does not remove it, and under single-tenant deployment the cost is per customer cluster.

⚠️ **Envelope overhead on every message** — roughly 300 bytes before payload. Immaterial at current
volume; worth revisiting alongside the Avro question if it ever isn't.

⚠️ **A rename touches every document.** `ValueGranted` appears in ADRs 004 and 005 and in `arch.md`,
and all of them must move together or the docs contradict each other again.

### Risks

| Risk | Impact | Mitigation |
|------|--------|-----------|
| Producer bypasses the registry | Unvalidated schema reaches consumers | Registry is mandatory in the serialiser config; CI asserts every produced type is registered |
| Topic created ad hoc by auto-creation | Wrong partitions, wrong retention, name off-convention | `auto.create.topics.enable=false`; topics are Terraform, reviewed like any other infrastructure |
| v1 retired while a consumer still reads it | Silent data loss for that consumer | Retire only after consumer lag is zero *and* the group has been inactive for 7 days |
| DLQ accumulates unnoticed | Money events lost in practice | Depth > 0 on a money topic pages; dashboard-only alerting is explicitly insufficient |
| Partition count increased on a keyed topic | Ordering breaks across the resize | Treat as a migration with a new topic, not an in-place change |
| Consumers dedupe in memory | Duplicates after a restart | Dedupe state must be durable — a database constraint, as ADR-004 does |

## Alternatives Considered

### Alternative 1: Version in the payload only, never in the topic name

**Description**: One topic per event type forever; consumers branch on `schema_version`.

**Pros**: Fewer topics; no dual-publishing; no migration choreography.

**Cons**: Every consumer accumulates version-handling branches that are never removed, because nobody
can prove the old version has stopped arriving. The compatibility burden shifts from the platform to
every consumer, and it never expires.

**Why rejected**: It makes breaking changes *possible* rather than *safe*, and pushes the cost onto
the consumers least equipped to carry it.

---

### Alternative 2: Avro with Schema Registry

**Description**: Binary Avro instead of JSON.

**Pros**: Compact; the canonical Kafka pairing; strongest schema evolution tooling; registry
integration is first-class rather than bolted on.

**Cons**: TypeScript support is materially weaker, and ADR-002 commits to generated TypeScript for
frontends. Messages become unreadable without tooling, which matters during incidents.

**Why rejected — provisionally**: The right choice at high throughput, and a contained change if
throughput arrives. At current volume, debuggability and one representation end-to-end are worth more
than bytes on the wire.

---

### Alternative 3: No schema registry; rely on ADR-002's shared contracts

**Description**: The shared-contracts artifact is the schema; compatibility is a build-time concern.

**Pros**: One less component per cluster; ADR-002 already tests compatibility.

**Cons**: Compiles against a schema, but does not validate what is actually on the topic. A service
that skips a contracts upgrade, or a hand-rolled producer, is caught by nothing. Retained messages
outlive the artifact version that wrote them.

**Why rejected**: Build-time compatibility and runtime compatibility are different guarantees, and
only the runtime one prevents the incident.

---

### Alternative 4: Event names as `<verb>.<noun>` or command-style

**Description**: `create.transaction`, or imperative names.

**Pros**: Reads naturally in code.

**Cons**: Imperative names invite treating events as commands, which is how choreography quietly
becomes orchestration — a producer starts caring who acts on its event. Past tense keeps events facts
about the past that anyone may ignore, which is exactly what makes modules removable under ADR-005.

**Why rejected**: The naming would undermine the property the architecture depends on.

## Implementation Notes

### Topic provisioning

Terraform, in `wallet-infrastructure`, with partition count and retention per topic class. Auto-creation
disabled in every environment. A module's topics are provisioned with the module (ADR-005), so a
customer without Bonus has no `bonus.*` topics.

### Registry configuration

```properties
schema.registry.url=http://schema-registry:8081
value.subject.name.strategy=io.confluent.kafka.serializers.subject.TopicRecordNameStrategy
auto.register.schemas=false      # CI registers; runtime never does
use.latest.version=false
```

`auto.register.schemas=false` is the important line: a service that could register its own schema
could register an incompatible one at 3am.

### CI enforcement

1. Every schema under `schemas/` maps to a registered subject, and every generated event class maps
   back to a schema — fail the build otherwise. A hand-edited generated file fails the same check
2. Schema compatibility checked against `FULL_TRANSITIVE` before publish
3. Topic names linted against `^[a-z]+\.[a-z][a-z-]*\.v[0-9]+$`
4. Every consumer of a state-changing event has a durable dedupe on `event_id` — asserted by an
   architecture test, since this is the rule most likely to be quietly skipped

### Follow-up: the rename must propagate

`ValueGranted` appears in ADR-004, ADR-005 and `arch.md`; the `fraud.case.*` → `risk.*` move and the
`recurring-payment.*` retirement affect `arch.md`. These are documentation changes to make alongside
this ADR, not after it.

## References

- [ADR-002: Shared Contracts with Dual Publishing](002-shared-contracts-versioning-strategy.md) — artifact versioning, which this extends to the wire
- [ADR-003: Hybrid Database Strategy](003-hybrid-database-strategy-with-analytics.md) — the warehouse-rebuild dependency that drives retention
- [ADR-004: Atomic Balance Reservations](004-atomic-balance-reservations.md) — database-level idempotency, generalised here
- [ADR-005: Module Composition](005-module-composition-and-deployment-topology.md) — why events, not synchronous calls
- [Confluent — Schema Evolution and Compatibility](https://docs.confluent.io/platform/current/schema-registry/fundamentals/avro.html)
- [Kafka topic naming: the `.` vs `_` metric collision](https://github.com/apache/kafka/blob/trunk/core/src/main/scala/kafka/common/Topic.scala)

---

**Date**: 2026-08-27
**Author**: Predrag
**Reviewers**: TBD
