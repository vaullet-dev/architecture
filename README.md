# Vaullet

![Java 21](https://img.shields.io/badge/Java-21-007396?style=flat-square&logo=openjdk&logoColor=white)
![Spring Boot 3](https://img.shields.io/badge/Spring%20Boot-3.x-6DB33F?style=flat-square&logo=springboot&logoColor=white)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-4169E1?style=flat-square&logo=postgresql&logoColor=white)
![Apache Kafka](https://img.shields.io/badge/Apache%20Kafka-231F20?style=flat-square&logo=apachekafka&logoColor=white)
![Redis](https://img.shields.io/badge/Redis-DC382D?style=flat-square&logo=redis&logoColor=white)
![Elasticsearch](https://img.shields.io/badge/Elasticsearch-005571?style=flat-square&logo=elasticsearch&logoColor=white)
![Keycloak](https://img.shields.io/badge/Keycloak-4D4D4D?style=flat-square&logo=keycloak&logoColor=white)
![Kubernetes](https://img.shields.io/badge/Kubernetes-326CE5?style=flat-square&logo=kubernetes&logoColor=white)
![Argo CD](https://img.shields.io/badge/Argo%20CD-EF7B4D?style=flat-square&logo=argo&logoColor=white)
![Linkerd](https://img.shields.io/badge/Linkerd-2BEDA7?style=flat-square&logo=linkerd&logoColor=black)

**A closed-loop stored-value wallet platform.** Money enters by top-up, lives as a balance on an
internal ledger, is spent inside one operator's ecosystem, and may be withdrawn — the machinery behind
a game economy's purchased and earned currency, a marketplace's seller balances, or a loyalty
programme's stored value. Single-tenant: each customer gets their own Kubernetes cluster. Modules are
contract line items, and the promotional platform is the flagship.

**This repository contains the architecture and the decision record. There is no implementation yet —
deliberately.** The ledger design has already changed twice on paper, and paper is the cheap place for
that to happen. See [ADR-001 → ADR-004](#the-decision-that-was-wrong) for why that is the point rather
than the excuse.

```
        ╭────────────╮╭────────────╮╭────────────╮
        │    CASH    ││   BONUS    ││  LOYALTY   │
    ╭───┴────────────┴┴────────────┴┴────────────┴───╮
    │                                                │
    │   ╭────────────────────────────────────────╮   │
    │   │                                        │   │
    │   │          V  A  U  L  L  E  T           │   │
    │   │                                        │   │
    │   │    available · held · withdrawable     │   │
    │   │                                        │   │
    │   ╰────────────────────────────────────────╯   │
    │                                                │
    ╰────────────────────────────────────────────────╯
```

---

## Start here

| Document | What it is |
|---|---|
| [`arch.md`](arch.md) | System architecture — services, data flows, event topology, market positioning |
| [`docs/adr/README.md`](docs/adr/README.md) | Twelve architecture decision records, indexed |
| [`docs/adr/TODO.md`](docs/adr/TODO.md) | Everything unresolved, with IDs — contradictions, open decisions, accepted gaps |
| [`docs/adr/HOW-TO-WRITE-ADRS.md`](docs/adr/HOW-TO-WRITE-ADRS.md) | The writing standard these follow |

If you only read one thing, read
[ADR-004](docs/adr/004-atomic-balance-reservations.md).

## The decision that was wrong

[ADR-001](docs/adr/001-distributed-locking-for-balance-consistency.md) specified Redis distributed
locking to prevent overdrafts. Reviewing it before implementation, the lock turned out to guard a
balance *read* while the *write* happened asynchronously in another service, after the lock was
released. It never prevented the failure it was written to prevent.

Account holds €100. Two withdrawals of €60, arriving 50ms apart — not even concurrent:

| t | Request A | Request B | Ledger |
|---|---|---|---|
| 1ms | reads balance: **€100** | — | €100 |
| 2ms | €100 ≥ €60 → **approve** | — | €100 |
| 4ms | releases lock | — | €100 |
| 51ms | — | reads balance: **€100** | €100 |
| 52ms | — | €100 ≥ €60 → **approve** | €100 |
| ~200ms | Vaullet consumes A | | €40 |
| ~250ms | | Vaullet consumes B | **−€20** |

[ADR-004](docs/adr/004-atomic-balance-reservations.md) replaced it: `available = posted − held`,
check-and-reserve as one Postgres transaction, and
`CHECK (posted_balance - held_total >= 0)` — so an overdraft is a schema violation rather than a bug
that a lock was supposed to prevent. Redis left the correctness path entirely.

Nothing was implemented and nothing was migrated. The cost of the mistake was an afternoon, because
the reasoning had been written down plainly enough to be checked and found wrong. That is what the
decision record is *for*, and it is the reason this repository exists before any code does.

## What the architecture actually decides

- **Typed money.** Balances are per-grant buckets — `CASH`, `BONUS`, `LOYALTY`, `REFERRAL`, `DEBT` —
  each with its own `withdrawable` flag, expiry and spend priority, because a single scalar cannot
  express "€100, of which €60 is promotional credit with conditions still outstanding." With no rewards module
  deployed, every account has one `CASH` bucket and the model degrades to a plain balance.
  ([ADR-004](docs/adr/004-atomic-balance-reservations.md))
- **Modules that are genuinely removable.** *A synchronous dependency makes a component core; optional
  functionality communicates only through events.* No sellable module may depend on another, so the
  module graph is a star and all 2⁷ = 128 combinations are valid by construction — which collapses the
  CI matrix from 128 to 9. ([ADR-005](docs/adr/005-module-composition-and-deployment-topology.md))
- **Enablement is build-time, not a runtime flag.** "Not installed" is demonstrable to a regulator;
  "flag is false" requires trusting runtime state.
- **The wire contract outlives its bindings.** Event schemas are authored JSON Schema registered under
  `FULL_TRANSITIVE`, with Java and TypeScript generated from them — because a field removed in `v3`
  still exists in retained messages. REST DTOs keep Java as the source.
  ([ADR-002](docs/adr/002-shared-contracts-versioning-strategy.md),
  [ADR-007](docs/adr/007-kafka-topics-and-event-schema-evolution.md))
- **Identity is not written here.** Keycloak per cluster; Vaullet keeps an account anchor and nothing
  else, so under federation the operator keeps their PII and the seven-year ledger is never keyed on a
  foreign, mutable subject claim. ([ADR-006](docs/adr/006-authentication-and-identity.md))
- **Repositories build; Git deploys.** Immutable image tags, per-cluster Argo CD pulling only its own
  path — so the trust direction points outward and nothing reaches into a licensed operator's
  environment. ([ADR-010](docs/adr/010-polyrepo-cicd-and-gitops-delivery.md))

## Shape of the system

14 services plus an Admin UI. Seven are core and always deployed; seven are sellable and any subset is
valid. Six operational PostgreSQL databases plus an analytics database, Redis, Elasticsearch, Kafka
and Keycloak — nine infrastructure components in a full deployment, seven in a core-only one.

## Technology

Every choice below is argued in an ADR rather than assumed, and several were argued *against* the
obvious option.

### Runtime

| | | |
|---|---|---|
| **Java 21** · **Spring Boot 3** | Services | Virtual threads suit a system that is mostly waiting on Postgres and Kafka |
| **PostgreSQL** | System of record | The ledger's correctness rests on one `SELECT … FOR UPDATE` and two `CHECK` constraints — an overdraft is a schema violation, not a race the application has to win ([ADR-004](docs/adr/004-atomic-balance-reservations.md)) |
| **Apache Kafka** | Event backbone | Not just transport: it is what makes modules removable. A consumer group that isn't deployed costs nothing, which is why optional functionality is only ever reached through events ([ADR-005](docs/adr/005-module-composition-and-deployment-topology.md), [ADR-007](docs/adr/007-kafka-topics-and-event-schema-evolution.md)) |
| **Redis** | Cache, rate limiting, sessions | Deliberately *not* on the correctness path — it was, under the superseded ADR-001, and that was the bug |
| **Elasticsearch** | Fraud pattern search | Core by inheritance, and [flagged as a cost worth revisiting](docs/adr/TODO.md) rather than defended |
| **Keycloak** | Identity provider, per cluster | Credential storage, MFA, brute-force detection and OIDC conformance are not things to write yourself ([ADR-006](docs/adr/006-authentication-and-identity.md)) |

### Contracts and APIs

| | | |
|---|---|---|
| **JSON Schema** + registry | Event contracts | Authored, registered `FULL_TRANSITIVE`, with Java and TypeScript generated from them — because a field removed in `v3` still exists in retained messages ([ADR-002](docs/adr/002-shared-contracts-versioning-strategy.md), [ADR-007](docs/adr/007-kafka-topics-and-event-schema-evolution.md)) |
| **springdoc-openapi** + **oasdiff** | REST contracts | Code-first generation, one fragment per module, committed and diffed in CI — a breaking change to a published version fails the build ([ADR-011](docs/adr/011-api-versioning-and-openapi.md)) |
| **Maven** + **NPM** | Distribution | Six per-domain artifact pairs, so a rewards change does not version the whole fleet |
| **RFC 9457** Problem Details | Errors | Stable machine-readable codes, one catalogue |

### Platform

| | | |
|---|---|---|
| **Kubernetes** | One cluster per customer | Isolation is the cluster boundary — no `tenant_id`, no shared infrastructure, no cross-customer data path |
| **Helm** · **Terraform** | Composition | Module enablement is *build-time*: "not installed" is demonstrable to a regulator, "flag is false" requires trusting runtime state |
| **Argo CD** | Delivery | Per cluster, pull-based, so the trust direction points outward and nothing reaches into a customer's environment ([ADR-010](docs/adr/010-polyrepo-cicd-and-gitops-delivery.md)) |
| **GitHub Actions** | CI | One reusable workflow, fourteen callers, pinned by tag |
| **Linkerd** | mTLS + L7 authorization | Chosen over Istio on cost: ~30 sidecars per cluster is 300–600 MB under Linkerd versus 1.5–3 GB under Istio — per customer, permanently ([ADR-008](docs/adr/008-service-to-service-authentication.md)) |
| **cosign** · **SBOM** | Supply chain | Images signed, signatures verified at admission, immutable tags only |

### Observability

**Prometheus** + **Grafana**, **ELK**, **Jaeger**/**Zipkin**. The metrics that matter are named per
service — the ledger's include `vaullet.reserve.latency.p95`, `vaullet.reconciliation.mismatch.count`
(which must be zero) and `vaullet.held_ratio`. A platform-wide observability decision is
[still open](docs/adr/TODO.md).

## What this is not

Not an open-loop card issuer, not a payment gateway or PSP (it consumes rails, it does not provide
them), not a KYC/AML vendor, and not an FX or remittance engine. It holds value and moves it inside
one operator's boundary; it does not move money between parties.

## Market context

A **closed-loop stored-value wallet** is a distinct product from a payment gateway, which moves money
between parties and holds none, and from an open-loop card programme, which spends anywhere on a card
network. Value is preloaded, held on an internal ledger, spent inside one ecosystem, and sometimes
withdrawn.

**The technical comparables** — ledger and wallet infrastructure:

| Product | Shape | Relation |
|---|---|---|
| **Formance** | Open-source programmable double-entry ledger, with an explicit *programmable wallets* product built on **holds** | The closest analogue to ADR-004 — the same reservation model, arrived at independently. Formance is fully double-entry; this journal is single-sided, the deliberate scope difference between a wallet and a general ledger |
| **TigerBeetle** | Purpose-built financial accounting database, native debit/credit schema, very high throughput | What ADR-004's Alternative 3 becomes at scale — a candidate to sit *under* this rather than beside it |
| **Fragment**, **Twisp** | Ledger APIs | Ledger primitives without domain logic |
| **Modern Treasury** | Payment operations plus a ledger | Broader operational scope, US bank-rail oriented |

**The markets that need one**, all running largely the same machinery:

| Domain | Examples | Fit |
|---|---|---|
| **Game economies** | Xsolla, Tilia | Strongest fit — "purchased currency" versus "earned, non-cashable currency" *is* the bucket model with `withdrawable` |
| **Retail loyalty** | Starbucks app | Canonical closed-loop stored value: preload, earn, redeem |
| **Super-apps and delivery** | Ride and delivery wallets | Top-up plus credits plus referral; the Referral and Loyalty modules map directly |
| **Marketplace payouts** | Seller balances | Holds and rolling reserves map onto the reservation model |

**Where this sits**: the ledger correctness of a Formance or TigerBeetle, with the promotional
mechanics that platforms in those markets normally bolt on afterwards — deployed single-tenant, per
operator, with the modules that operator bought.

Two design choices that looked like invention turned out to be established practice, which is
reassuring rather than disappointing: licensing modules per contract is how platform vendors in this
space already sell, and holds against a journal-plus-projection ledger is exactly what Formance's
programmable wallets do.

**Deferred**: stored value means real money sits in a bank account outside the system, and operators
are generally required to keep it segregated and reconciled against the customer ledger. Safeguarding
and float reconciliation are not addressed — deliberately deferred as a licensing concern rather than
a design-phase one.

## Status

| | |
|---|---|
| Architecture | 12 ADRs, 1 superseded, reconciled against each other on 2026-09-02 |
| Implementation | Not started — `contracts-common` is next |
| Open items | [25 tracked](docs/adr/TODO.md) — 19 open, 6 deferred with triggers; 4 are blocking |

A private portfolio project, built to production-grade B2B standards: the goal is an architecture that
survives review by a senior engineer, not a system that currently ships.

## Licence

**© 2026 Predrag Arsić. All rights reserved.** Published for reading and evaluation — see
[LICENSE](LICENSE). Brief quotation with attribution is fine; reproducing these documents, or building
from them commercially, is not. The same terms apply across every repository in this project.

Ask if you want to do something the licence does not cover.
