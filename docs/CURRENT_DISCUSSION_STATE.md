# Current Discussion State - Vaullet Architecture

**Date**: 2026-09-02
**Status**: ADRs 002–012 reviewed against each other and reconciled — **the ledger is ready to build**
**Next topic**: implementation (see *Build order*); multi-currency and data retention remain deferred

---

## Product Context

Vaullet is a **private portfolio project built to production-grade B2B standards** — the goal is an
architecture that survives review by a senior engineer, not a system that currently ships.

The commercial model it is designed against:

- **Single-tenant deployments.** Each customer (a wallet operator, initially betting platforms) gets
  their own Kubernetes cluster. No shared infrastructure, no `tenant_id`, no cross-customer data
  path. Isolation is the cluster boundary — stronger than row-level scoping, and often mandatory
  under gambling licences with data-residency conditions.
- **Modules are line items.** A contract determines which modules that customer's cluster runs.
- **Bonus is the flagship.** The core wallet is the land; the bonus platform is the expected upsell.

---

## Decisions Made

| ADR | Decision | Status |
|-----|----------|--------|
| 001 | Redis distributed locking for balance consistency | ⛔ Superseded by 004 |
| 002 | Shared contracts — per-domain artifacts, dual publishing (Java + TypeScript) | ✅ Accepted (revised) |
| 003 | Hybrid database strategy with separate analytics DB | ✅ Accepted (revised) |
| 004 | Atomic balance reservations in the ledger | ✅ Accepted (revised) |
| 005 | Module composition and deployment topology | ✅ Accepted |
| 006 | Authentication and identity — **Keycloak** per cluster | ✅ Accepted (revised) |
| 007 | Kafka topic naming and event schema evolution | ✅ Accepted (revised) |
| 008 | Service-to-service authentication (NetworkPolicy + Linkerd + Keycloak) | ✅ Accepted |
| 009 | Payment rails — deposits, withdrawals, chargebacks | ✅ Accepted (revised) |
| 010 | Polyrepo CI/CD and GitOps delivery (GitHub Actions + Argo CD) | ✅ Accepted |
| 011 | API versioning and OpenAPI | ✅ Accepted (revised) |
| 012 | External API surface | ✅ Accepted (revised) |

### What ADR-004 changed

ADR-001's lock guarded a balance *read* while the *write* happened asynchronously in Vaullet after
the lock was released — so it did not prevent the overdraft it was written to prevent. Two
withdrawals 50ms apart both passed. Never implemented; corrected on paper.

Replacement: `available = posted − held`. Check-and-reserve is one atomic Postgres transaction inside
Vaullet, with `CHECK (posted_balance - held_total >= 0)` making an overdraft a schema violation.
Redis leaves the correctness path entirely.

Money is typed into **per-grant buckets** (`CASH`, `BONUS`, `LOYALTY`, `REFERRAL`), each with its own
`withdrawable` flag, expiry and spend priority — because a single scalar cannot express "$100, of
which $60 is bonus money with wagering outstanding." With no rewards module deployed, every account
has one `CASH` bucket and the model degrades to a plain balance.

### What ADR-005 established

**The rule**: *a synchronous dependency makes a component core; optional functionality communicates
only through events.* Disabling an event consumer is safe — Kafka doesn't care that a consumer group
is absent. Disabling a synchronous dependency is not, and both default recoveries are wrong
(fail-closed halts transactions, fail-open silently disables a financial control). So the case is
removed rather than handled.

**The invariant**: *no sellable module may depend on another sellable module.* The module graph is a
star, so all 2⁷ = 128 combinations are valid by construction and the CI matrix collapses to 9.

**Categories**:

- **Core**: Transaction (incl. Refund), Vaullet, Auth, Fraud Detection, Limits, Audit, Admin UI
- **Infrastructure**: Kafka, Redis, PostgreSQL, Elasticsearch, Scheduler
- **Sellable**: Bonus ⭐, Loyalty, Referral, Subscription, Risk Management, Notification, Reporting ETL
- **Adapters**: Auth identity provider (local | federated)

Two deliberate overrides: **Audit** is event-driven but core, because an audit trail is a licence
condition rather than a feature. **Scheduler** is infrastructure rather than a module, which removes
Subscription's only sellable-to-sellable dependency.

Enablement is **build-time** (per-customer Helm values), not runtime flags — "not installed" is
demonstrable to a regulator; "flag is false" requires trusting runtime state.

---

## Databases

6 operational PostgreSQL + 1 analytics + Redis + Elasticsearch = **9 components** full, **7** core-only.

| Database | Services | Provisioned |
|----------|----------|-------------|
| `vaullet_db` | Vaullet | Always |
| `transactions_db` | Transaction (incl. Refund) | Always |
| `fraud_db` | Fraud Detection, Risk Management | Always |
| `auth_db` | Auth | Always |
| `support_db` | Limits, Audit, Scheduler (+ Subscription, Notification if sold) | Always, schemas conditional |
| `rewards_db` | Bonus, Loyalty, Referral | Only if a rewards module is sold |
| `datawarehouse_db` | Reporting ETL | Only if sold |

**Boundary that matters**: `rewards_db` holds *rules* — wagering terms, tiers, campaigns. It never
holds balances. Promotional *money* lives in `vaullet_db` buckets, reached only by emitting
`rewards.value-granted`. Two services must not both believe they know what a user's money is.

---

## What ADR-006 settled

**Keycloak is deployed per cluster** as the identity provider. We do not write authentication —
credentials, MFA, brute-force detection and OIDC conformance come audited. Auth Service shrinks to a
thin adapter owning the account anchor, status enforcement, token-exchange orchestration and role
mapping.

Both identity models are Keycloak configuration rather than separate code paths:

- `local` — users live in the Keycloak realm
- `federated` — the operator's OIDC/SAML provider is registered as an identity **broker** in the same realm

Either way Keycloak issues the same token from the same realm, so **adapter invisibility is
structural** — there is no code path that could drift. That is a simplification over the original
bespoke design, and the reason the switch was cheap: nothing outside the Auth boundary changed.

Under federation Vaullet stores an **account anchor and nothing else**: `account_id`, issuer, subject,
status. No email, name or KYC data — the operator keeps their PII. The anchor exists because the ledger
is immutable for seven years and cannot key balances on a foreign, mutable subject claim.

Fraud locks are enforced **locally** in both modes: Vaullet will refuse a user the operator's IdP still
considers valid. Admin identity is **local by default**, so fraud response survives an operator IdP
outage.

This closed the last of the original four questions: user data stored (minimal, under federation),
admin roles (six roles with separation of duties), and the auth half of operator reporting access.

## The 2026-09-02 pre-implementation review

The ADRs were read against *each other* rather than individually, on the theory that a set of
individually sound decisions can still contradict. Six did, and all six are fixed:

1. **Hold lifetime is now caller-supplied** (`expires_in_seconds`, default 300, bounded by
   `max_hold_seconds`). ADR-012 sells two-phase wagers that settle hours or days later; ADR-004
   hardcoded `now() + interval '5 minutes'`. Every wager in the flagship use case would have expired
   before capture, and the settlement/expiry race ADR-004 calls rare would have been the normal path.
   An expired long hold now emits `ledger.hold-expired.v1` and reaches the operator as
   `transaction.expired` — releasing an operator's money silently is not acceptable.
2. **`DEBT` is excluded from funding.** ADR-004's allocation query selected every bucket with a
   positive balance, and ADR-009's `DEBT` bucket holds a *positive* amount representing what the user
   owes — so a user could have spent their own debt. `debt_total` is now a separate column on
   `account_balances`, `posted_balance` sums fundable buckets only, and the allocation filters
   `bucket_type <> 'DEBT'`. All three had to move together, because the aggregate pre-check and the
   bucket allocation must agree or the reserve path fails after passing its own gate.
3. **The journal is single-sided, and now says so.** `arch.md` claimed double-entry; the schema has one
   `account_id` per entry and no balanced-entry constraint. Beyond vocabulary this was a throughput
   trap: a real counterparty side would put a house account on every transaction and serialize the
   entire deployment on one row, against the account-first lock rule. Scope is now explicit — user
   accounts only; the operator's position is reconstructed downstream.
4. **Contracts publish per-domain artifacts** (`contracts-common`, `-ledger`, `-transaction`,
   `-payments`, `-rewards`, `-identity`) from the one repository. A single jar for fourteen services
   rebuilt, in the build graph, exactly the hub coupling ADR-005 removes at runtime.
5. **Event schemas are authored; Java and TypeScript are generated from them.** ADR-007 makes the
   registered JSON Schema the wire contract and argues events outlive their bindings — so the contract
   cannot be defined *by* a binding. REST DTOs keep Java as the source of truth, where the DTO and its
   controller ship together.
6. **OpenAPI fragments are generated per module.** ADR-011 mandated code-first generation, ADR-012
   mandated composed fragments, and nothing reconciled them. springdoc now emits one `GroupedOpenApi`
   per module, selected by controller package, with the package boundary enforced by an architecture
   test.

## Build order

1. `contracts-common` + `schemas/` + the generator and its CI gates — while there is still one consumer
2. `contracts-ledger`
3. **Vaullet**: migrations, then reserve / settle / release / expire, with ADR-001's failure trace as
   the first test. CI inline in this repo; extract `service-ci.yml` when a second service exists
4. Transaction Service, and the `core.v1.yaml` fragment behind it

## Still open

The full register is **[`/docs/adr/TODO.md`](adr/TODO.md)** — 4 blocking contradictions (B1–B4),
8 decisions needing an ADR (D1–D8), 5 accepted gaps (G1–G5), and 6 deferred re-evaluations with
triggers (R1–R6). The ones that shape what to build next:

- **B1 · Withdrawal holds have no lifetime set.** The 2026-09-02 fix made hold lifetime caller-supplied
  and gave the wagering path a value; the withdrawal path still inherits the 300-second default, so a
  manually reviewed withdrawal releases its hold while the payout is pending. Same failure class the
  review set out to remove, on a second path.
- **B2/B3 · The account anchor has two creation paths**, and the operator-facing one cannot succeed as
  specified (`keycloak_sub NOT NULL`); `external_ref` is specified with no column behind it.
- **D1 · Manual ledger adjustments** — ADR-006 assigns the role and the two-principal rule, ADR-008
  deliberately withdrew the scope because no endpoint exists. The control is written and the operation
  is not; that is the next ADR.
- **D2 · Human/operational access control** — `kubectl exec`, database credentials, break-glass.
  Explicitly out of ADR-008's scope, and usually the first thing a licence auditor asks about.
- **D6 · Disaster recovery** — ADR-003 asserts the warehouse is rebuildable; nothing says how
  `vaullet_db` itself is backed up or restored.

---

## Known Gaps

- **Per-customer cost floor is high** — core alone is 7 services plus Kafka, Redis, Elasticsearch and
  5 Postgres databases. Small operators may be unprofitable on this topology. ADR-005 Alternative 3
  (modular monolith) is the documented candidate for a low-cost tier.
- **Elasticsearch is core by inheritance** — Fraud Detection is core and ADR-003 gives it ES. Worth
  revisiting whether pattern search can degrade to Postgres-only.
- **14 services is a lot for a solo project.** The module boundaries have commercial justification, but
  "sellable unit" and "separately deployed service" are not the same thing.

---

## Files

- Architecture overview: `/arch.md`
- ADRs: `/docs/adr/0{01..12}-*.md` (index and review summary: `/docs/adr/README.md`)
- Database grouping: `/docs/database-grouping-strategy.md`
- Shared contracts example: `/docs/shared-contracts-example.md`
