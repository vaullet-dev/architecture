# Open Items

Everything known to be unresolved across `arch.md` and ADRs 001–012, as of **2026-09-02**.

This is the detailed register. [`README.md`](README.md) keeps the short index of decisions that still
need an ADR; this file also carries the contradictions, accepted gaps and deferred re-evaluations that
are not decisions in their own right.

**How to use it**: items are `B` (blocking implementation), `D` (needs a decision), `G` (known gap,
accepted), `H` (hygiene) and `R` (deferred re-evaluation with a trigger). Delete an item when it is
closed — a register nobody prunes stops being read.

---

## B — Blocking implementation

Fix before writing the code each one touches. Both are contradictions between accepted decisions,
not new decisions. **B2** (the anchor's two creation paths) and **B3** (`external_ref` has no column)
were closed by [ADR-014](014-user-management-in-local-identity-mode.md) on 2026-09-16, and **B4** (the
ledger's scope rows) by [ADR-016](016-transactional-outbox-for-event-producers.md) on 2026-10-22.

### B1 · Withdrawal holds have no lifetime set

- **Where**: [ADR-009](009-payment-rails-deposits-and-withdrawals.md) §4, against
  [ADR-004](004-atomic-balance-reservations.md) *Reservation lifetime*
- **What**: ADR-009 opens an ADR-004 reservation at `REQUESTED`. Above the auto-approval threshold the
  withdrawal waits in `REQUESTED` for an Admin UI action — human review, so hours or days. Hold
  lifetime became caller-supplied on 2026-09-02 and the wagering path was given a value; the
  withdrawal path was not, so it inherits the 300-second default.
- **Why it matters**: the hold releases five minutes after the request and the funds become spendable
  while the payout is still pending approval. A user can spend money already on its way out — the
  exact failure the hold at `REQUESTED` exists to prevent.
- **Fix**: ADR-009 §4 states the withdrawal hold TTL explicitly, as a deployment setting rather than a
  constant — the review SLA belongs to the operator. Blocks the withdrawal path, not the ledger core.

### B5 · `identity.account-created` or `auth.account-created`

- **Where**: [ADR-007](007-kafka-topics-and-event-schema-evolution.md) §1–§2 and §8, against
  [ADR-014](014-user-management-in-local-identity-mode.md) §7 and
  [ADR-016](016-transactional-outbox-for-event-producers.md) §6
- **What**: ADR-007 names a topic's domain after the service that produces it, renamed
  `user.registered` to `auth.user-registered.v1`, and lists `auth.*` among the operational topics.
  ADR-014 §7 named Auth Service's first event `identity.account-created.v1`, and wallet-auth-service
  0.12.0 shipped it under that name.
- **Why it matters**: the name is no longer free. V3's backfill wrote `identity.account-created` into
  every production outbox row, and the table's `CHECK` and its partial unique index name it, so a
  rename is now a migration of `auth_db` — every row, the constraint and the index — plus the constant
  in `Outbox`. Once the relay publishes to Kafka it becomes a new topic and a dual-publishing window
  (ADR-007 §4).
- **Fix**: pick one before the relay publishes to Kafka. Keeping `identity` reads ADR-007's domain as
  the bounded context rather than the service, and moves §2's `auth.user-registered.v1` and §8's
  `auth.*` with it; choosing `auth` costs the migration above. Blocks the Kafka relay, not the HTTP
  one running today.

---

## D — Needs a decision

Ordered by what blocks or exposes the most.

### D1 · Manual ledger adjustments ⭐

Corrections, goodwill credits, dispute resolution, write-offs.

- **Where**: promised in [ADR-006](006-authentication-and-identity.md) (`FINANCE` role, two-principal
  rule), explicitly *withdrawn* in ADR-008 (the `ledger:adjust` scope was removed because the endpoint
  does not exist), absent from ADR-004's ledger paths and ADR-012's surface.
- **Why first**: it is the only open decision that already has access controls and a
  separation-of-duties rule written against it. ADR-008 was right that authorising a non-existent
  operation is worse than not authorising it — but the resolution is to design the operation, not to
  leave the control dangling.
- **Shape it has to take**: an adjustment is a journal entry like any other (the journal is
  append-only — a correction is a new entry, never an edit), it needs a reason code and a
  two-principal approval trail, it must name which bucket it credits or debits, and `DEBT` write-off
  is one of its cases. Interacts with B1's operator-review flow.

### D2 · Human and operational access control

`kubectl exec`, database credentials, break-glass procedure.

- **Where**: explicitly out of scope in ADR-008; listed as open in `CURRENT_DISCUSSION_STATE.md`.
- **Why it matters**: ADR-008 secures service-to-service traffic thoroughly and says nothing about the
  humans who can reach the same data directly. Under a gambling licence this is usually the part an
  auditor asks about first, and "we have mTLS between services" is not an answer to it. ADR-010 adds
  to the surface: whoever can merge to `wallet-gitops` can change what every customer runs.
- **Depends on nothing** — can be written now.
- **As of 2026-09-30** there are two ways into a database, and neither is the answer: the service's
  own login (`auth_app`, password from OpenBao) through an SSH tunnel and `kubectl port-forward`, or
  `kubectl exec … psql` as the superuser. The strategy has to settle: personal, named roles instead of
  shared service logins; read-only by default (`pg_read_all_data`); how a write happens when one must
  (break-glass: time-boxed, logged, reviewed); and where each person's credential lives (OpenBao).

### D3 · Data retention and archival

- **Where**: `CURRENT_DISCUSSION_STATE.md`; interacts with ADR-007 §8 (money topics retained
  indefinitely, tiered) and the seven-year ledger immutability asserted in ADR-006.
- **Why it matters**: two retention claims already exist in the architecture and neither has a policy
  behind it. Under GDPR, indefinite retention needs a lawful basis and a deletion story, and "the
  ledger is immutable for seven years" and "the user may request erasure" have to be reconciled
  somewhere. That reconciliation is a decision, not an implementation detail.

### D4 · PSP settlement matching

Provider settlement reports vs captured deposits.

- **Where**: ADR-009 accepts the capture-to-settlement gap as "short, bounded, and priced in", and
  stops there.
- **Why it matters**: nothing currently detects a capture that never settles. That is the exposure
  window ADR-009 accepts, but accepting a window is different from being able to see into it.

### D5 · Multi-currency within one deployment

- **Where**: ADR-004 *Currency* documents the extension path (one account per `(user, currency)`) and
  defers the decision.
- **Why it matters**: it is genuinely deferred rather than forgotten, and the shape is recorded so it
  will not be rediscovered under pressure. Revisit when a contract asks — not before.

### D6 · Disaster recovery and backup

- **Where**: `arch.md` Next Steps item 6; untouched by any ADR.
- **Why it matters**: ADR-003 asserts `datawarehouse_db` is rebuildable from the event stream and
  ADR-007 makes retention support that claim — but no document says how `vaullet_db` itself is backed
  up, what the RPO is, or how a cluster is restored. The ledger is the system of record and it has no
  recovery story.

### D7 · Platform-wide observability

- **Where**: `arch.md` Next Steps item 8. ADR-004 specifies ledger metrics and ADR-003 a few alert
  thresholds; there is no platform decision.
- **Why it matters**: fourteen services across N single-tenant clusters, with no decision on where
  logs, metrics and traces go, or whether a vendor ever sees a customer's data. The last part is a
  data-residency question, which makes it more than a tooling preference.

### D8 · Operator reporting surface

Dashboard vs API vs embeddable widgets.

- **Where**: `CURRENT_DISCUSSION_STATE.md`. The auth half is settled (Admin UI is role-scoped; BI reads
  `datawarehouse_db` with read-only credentials).
- **Why it matters**: a product question rather than an architectural one, and flagged as such — but
  it determines whether `datawarehouse_db` needs a query API in front of it.

---

## G — Known gaps, accepted

Not scheduled. Recorded so they are not rediscovered as surprises.

| ID | Gap | Where | Status |
|---|---|---|---|
| **G1** | **Per-customer cost floor is high** — core alone is 7 services plus Kafka, Redis, Elasticsearch, Keycloak, Argo CD and 5 Postgres databases. Small operators may be unprofitable on this topology | ADR-005, ADR-010 | ADR-005 *Alternative 3* (modular monolith, compile-time module selection) is the documented candidate for a low-cost tier. Nobody has priced either |
| **G2** | **Elasticsearch is core by inheritance** — Fraud Detection is core and ADR-003 gives it ES, so every customer runs an ES cluster | ADR-003, ADR-005 | Candidate: degrade pattern search to Postgres-only and make ES a Category 4 adapter |
| **G3** | **Fourteen services is a lot for a solo project.** The module boundaries have commercial justification, but "sellable unit" and "separately deployed service" are not the same thing | ADR-005 | Accepted deliberately. The mitigation is build order, not architecture — ledger first, extract shared CI at the second service |
| **G4** | **Warehouse rebuild has no source beyond 90 days.** ADR-007 bounded money-topic retention on 2026-09-17, so ADR-003's "rebuildable from events" holds for 90 days only. The proposed resolution — rebuild from `vaullet_db`, which is the actual system of record — is written in ADR-007 §8 but **not yet recorded in ADR-003** | ADR-003, ADR-007 §8 | Decide before Reporting ETL is built. The retention number itself is settled |
| **G5** | **Nothing enforces a coherent contract version set.** Six independently versioned artifacts mean a service can run `contracts-ledger` 2.3 with `contracts-transaction` 1.9. An architecture test forbids a *cycle*; nothing checks the *set* | ADR-002 | Candidate: a BOM the services import, so the set is chosen once |
| **G6** | **`ledger:write` is one scope for every change on the ledger** — opening, freezing and closing wallets, and placing and releasing holds, on any account. Auth Service's outbox relay holds it to open wallets, so its token could also place holds, and no NetworkPolicy or mesh narrows who can reach the ledger yet (ADR-008 Layers 1–2 are not deployed). ADR-008's `ledger:reserve` for the money path does not exist in the realm | ADR-008 Layer 3, ADR-016 §4, gitops `realm/15-vaullet-client-scopes.yaml` | Accepted while the relay is the only service holding it. Candidate: split it — `ledger:reserve` for holds, as ADR-008 already names, and a wallet scope the relay holds alone — before a second service is given `ledger:write` |
| **G7** | **The relay's token also passes Keycloak's Admin API.** The relay authenticates as `wallet-auth-service`, whose service account holds `manage-users`, `view-users` and `view-realm`; with `fullScopeAllowed` those roles are in every token the client gets, so the token the ledger receives could manage the realm's users | ADR-016 §4, ADR-014 *Keycloak specifics*, gitops `realm/30-wallet-auth-service.yaml` | Accepted until Kafka, when the relay stops sending tokens anywhere. Candidate, checked on Keycloak 26.7.4: `fullScopeAllowed: false` with the three roles behind an optional scope that only the Admin API calls ask for — or a second client for the relay |

---

## H — Hygiene

- **H1** · `arch.md` writes event names without the `.v1` suffix in ~40 places. A convention note now
  says arch.md uses logical names and the topic appends the major version — defensible, but ADR-007's
  rule is about the topic name, and a reader checking conformance will bounce off it.
- **H2** · Every ADR ends with `**Reviewers**: TBD`. Fine for a solo project; it is the first thing a
  reviewer notices, and the honest fix is to remove the field rather than fill it in falsely.

---

## R — Deferred, with a trigger

Decisions that are correct now and have a named condition for revisiting. Not open items — listed so
the trigger is not forgotten.

| ID | Decision | Revisit when |
|---|---|---|
| **R1** | JSON Schema over Avro (ADR-007 §5) | Serialisation cost becomes measurable at throughput |
| **R2** | Linkerd over Istio (ADR-008 Layer 2) | Istio ambient mode has more operational history — it removes the sidecar cost that decided this |
| **R3** | Argo CD per cluster (ADR-010 §6) | Per-cluster resource cost becomes the binding constraint |
| **R4** | Path versioning over date-based (ADR-011 *Alternative 2*) | The integrator count grows into the hundreds |
| **R5** | PostgreSQL for `datawarehouse_db` (ADR-003) | Reporting queries exceed ~5s → ClickHouse or TimescaleDB |
| **R6** | 12 partitions on money topics (ADR-007 §6) | Increasing is a one-way door — changing it remaps keys to partitions |
| **R7** | Identity reads are a pure facade over the Keycloak Admin API (ADR-014 *Alternative 1*) | An Admin UI user list exceeds ~1s, or an operator asks for a filter spanning `status` and a profile field → add a rebuildable read projection |
| **R9** | Until Kafka, Auth Service's outbox relay delivers `identity.account-created` over HTTP, as the ledger's `POST /v1/wallets` (ADR-016 §4) | Kafka is deployed, a second consumer of the event appears, or a second event type is queued — whichever comes first. HTTP delivery reaches one consumer, by URL, and nothing keeps an account's rows in order |

---

**Maintenance**: reviewed whenever an ADR is accepted or revised. Last full pass 2026-09-02, against
ADRs 001–012 and `arch.md`. B2/B3 pruned 2026-09-16 on ADR-014; R8 added 2026-09-17 when the auth
service's first event producer made the outbox gap concrete. B4 and R8 pruned 2026-10-22 on ADR-016,
which added B5, G6, G7 and R9.
