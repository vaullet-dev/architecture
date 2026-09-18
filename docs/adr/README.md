# Architecture Decision Records (ADRs)

This directory contains Architecture Decision Records (ADRs) for the Vaullet walleting system.

## What are ADRs?

ADRs document important architectural decisions made during the development of the system. Each ADR captures:
- **Context**: Why we needed to make a decision
- **Decision**: What we chose to do
- **Consequences**: Trade-offs and implications
- **Alternatives**: What else we considered and why we rejected them

## Why ADRs?

- 📚 **Institutional memory**: Understand why decisions were made
- 🆕 **Onboarding**: Help new team members understand the system
- 🔄 **Evolution**: Easy to revisit and supersede when context changes
- 🤔 **Decision-making**: Forces us to think through trade-offs

## ADR Lifecycle

1. **Proposed**: Decision is being discussed
2. **Accepted**: Decision has been made and implemented
3. **Partially Accepted**: Some parts settled, others still under review — the ADR states which
4. **Deprecated**: Decision is outdated but not yet replaced
5. **Superseded**: Replaced by a newer decision (link to new ADR)

## How to Create an ADR

1. Copy `template.md` to a new file
2. Name it: `XXX-short-title.md` (e.g., `002-kafka-topic-naming.md`)
3. Use the next sequential number
4. Fill in all sections
5. Add it to the index below
6. Submit for review via pull request

## Index of ADRs

| ADR | Title | Status | Date |
|-----|-------|--------|------|
| [001](001-distributed-locking-for-balance-consistency.md) | Use Redis Distributed Locking for Balance Consistency | ⛔ Superseded by [004](004-atomic-balance-reservations.md) | 2026-08-26 |
| [002](002-shared-contracts-versioning-strategy.md) | Shared Contracts with Dual Publishing (Java + TypeScript) | ✅ Accepted (revised) | 2026-09-02 |
| [003](003-hybrid-database-strategy-with-analytics.md) | Hybrid Database Strategy with Separate Analytics Database | ✅ Accepted | 2026-08-27 |
| [004](004-atomic-balance-reservations.md) | Atomic Balance Reservations in the Ledger | ✅ Accepted (revised) | 2026-09-02 |
| [005](005-module-composition-and-deployment-topology.md) | Module Composition and Deployment Topology | ✅ Accepted | 2026-08-27 |
| [006](006-authentication-and-identity.md) | Authentication and Identity | ✅ Accepted | 2026-08-31 |
| [007](007-kafka-topics-and-event-schema-evolution.md) | Kafka Topic Naming and Event Schema Evolution | ✅ Accepted (revised) | 2026-09-02 |
| [008](008-service-to-service-authentication.md) | Service-to-Service Authentication | ✅ Accepted | 2026-09-01 |
| [009](009-payment-rails-deposits-and-withdrawals.md) | Payment Rails — Deposits, Withdrawals and Chargebacks | ✅ Accepted (revised) | 2026-09-02 |
| [010](010-polyrepo-cicd-and-gitops-delivery.md) | Polyrepo CI/CD and GitOps Delivery | ✅ Accepted | 2026-09-01 |
| [011](011-api-versioning-and-openapi.md) | API Versioning and OpenAPI | ✅ Accepted (revised) | 2026-09-02 |
| [012](012-external-api-surface.md) | External API Surface | ✅ Accepted (revised) | 2026-09-02 |
| [013](013-backend-common-shared-library.md) | `backend-common` — Shared Runtime Plumbing, Split by Dependency Footprint | ✅ Accepted (revised) | 2026-09-06 |
| [014](014-user-management-in-local-identity-mode.md) | User Management in Local Identity Mode | ✅ Accepted (revised) | 2026-09-17 |
| [015](015-cluster-topology-three-servers-on-one-machine.md) | Cluster Topology — Three RKE2 Servers on One Machine | ✅ Accepted | 2026-09-17 |

## The 2026-09-02 pre-implementation review

Before writing code, ADRs 002–012 were reviewed against each other for contradictions rather than
individually for soundness. Six were revised; the substantive changes:

| Fix | Where | Why it mattered |
|---|---|---|
| Hold lifetime is **caller-supplied** (default 300s, `max_hold_seconds` ceiling) | 004, 012 | ADR-012 sells two-phase wagers that settle in *hours*; ADR-004 hardcoded a 5-minute expiry, so every wager would have expired before capture |
| `DEBT` excluded from allocation; `debt_total` split out of `posted_balance` | 004, 009 | The reserve query would have allocated *from* a debt bucket, letting a user spend money they owe |
| The journal is **single-sided**, stated as scope | 004, arch.md | "Double-entry" was asserted but not implemented; leaving it implied a house account on every transaction, which would serialize the whole deployment on one row |
| Contracts publish **per-domain artifacts** | 002 | One jar for fourteen services rebuilt the coupling ADR-005 exists to remove |
| Event schemas are **authored**, Java generated from them | 002, 007 | ADR-007 makes the registered schema the wire contract; deriving it from a Java class makes wire compatibility an accident of refactoring |
| OpenAPI fragments are **generated per module** | 011, 012 | ADR-011 said code-first, ADR-012 said composed fragments; nothing said how both could be true |

Nothing in the reviewed set is now known to contradict anything else.

## Upcoming Decisions

Decisions that still need an ADR. Full detail, plus the contradictions, accepted gaps and deferred
re-evaluations that are not decisions in their own right, is in **[TODO.md](TODO.md)**.

- [ ] **D1 · Manual ledger adjustments** ⭐ (corrections, goodwill credits, dispute resolution) — the controls are already written against it in ADR-006 and ADR-008; the operation is not designed
- [ ] **D2** · Human/operational access control (kubectl, database credentials, break-glass)
- [ ] **D3** · Data retention and archival policy
- [ ] **D4** · PSP settlement matching (provider reports vs captured deposits)
- [ ] **D5** · Multi-currency within a single deployment (currently one currency per deployment — ADR-004)
- [ ] **D6** · Disaster recovery and backup — `vaullet_db` has no recovery story
- [ ] **D7** · Platform-wide observability
- [ ] **D8** · Operator reporting surface (dashboard vs API vs widgets)

TODO.md also tracks the items (**B1**, **B4**) that are *not* new decisions but contradictions between
accepted ones, and must be fixed before the code they touch is written. **B2** and **B3** closed with
[ADR-014](014-user-management-in-local-identity-mode.md), whose first implementation slice — the
account anchor in `wallet-auth-service` — is written and tested as of 2026-09-17.

Vaullet's ledger schema is **not** on this list: it is specified in [ADR-004](004-atomic-balance-reservations.md)
and is ready to implement.

## References

- [Open Items register](TODO.md) — everything unresolved, with IDs
- [ADR Template](template.md)
- [Architecture Overview](../../arch.md)
- [GitHub - ADR Tools](https://github.com/npryce/adr-tools)
- [Thoughtworks - Documenting Architecture Decisions](https://www.thoughtworks.com/radar/techniques/lightweight-architecture-decision-records)