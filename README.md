# Vaullet

![Java 21](https://img.shields.io/badge/Java-21-007396?style=flat-square&logo=openjdk&logoColor=white)
![Spring Boot 4](https://img.shields.io/badge/Spring%20Boot-4.1-6DB33F?style=flat-square&logo=springboot&logoColor=white)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-4169E1?style=flat-square&logo=postgresql&logoColor=white)
![Apache Kafka](https://img.shields.io/badge/Apache%20Kafka-231F20?style=flat-square&logo=apachekafka&logoColor=white)
![Redis](https://img.shields.io/badge/Redis-DC382D?style=flat-square&logo=redis&logoColor=white)
![Elasticsearch](https://img.shields.io/badge/Elasticsearch-005571?style=flat-square&logo=elasticsearch&logoColor=white)
![Keycloak](https://img.shields.io/badge/Keycloak-4D4D4D?style=flat-square&logo=keycloak&logoColor=white)
![Kubernetes](https://img.shields.io/badge/Kubernetes-326CE5?style=flat-square&logo=kubernetes&logoColor=white)
![Argo CD](https://img.shields.io/badge/Argo%20CD-EF7B4D?style=flat-square&logo=argo&logoColor=white)
![Traefik](https://img.shields.io/badge/Traefik-24A1C1?style=flat-square&logo=traefikproxy&logoColor=white)
![OpenBao](https://img.shields.io/badge/OpenBao-1B1B2E?style=flat-square&logo=openbao&logoColor=white)
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

Grouped by layer, as of 2026-09-14. Status says how real each piece is:

- **running**: deployed on the cluster that serves [vaullet.dev](https://vaullet.dev).
- **code**: built and tested in a repository, not deployed yet.
- **planned**: decided in an ADR, nothing built.

Versions are copied from the file that pins them.

### Frontend

| Technology | Version | Status | Why |
|---|---|---|---|
| **HTML, CSS, JavaScript** | — | running | vaullet.dev is one hand-written page with no framework and no build step |
| **nginx** | 1.27-alpine | running | Serves static files from a read-only, non-root container |

### Backend

| Technology | Version | Status | Why |
|---|---|---|---|
| **Java** | 21 | code | Virtual threads suit services that mostly wait on Postgres and Kafka |
| **Spring Boot** | 4.1.1 | code | Well understood, and deliberately not the interesting part |
| **backend-common** | 0.1.1 | code | Error bodies, the JWT security chain, OpenAPI and test setup, shared as Maven artifacts instead of copied into each service ([ADR-013](docs/adr/013-backend-common-shared-library.md)) |
| **Maven** · **japicmp** | japicmp 0.23.1 | code | A backend-common release that breaks its public API fails the build |
| **springdoc-openapi** | 3.1.0 | code | OpenAPI generated from code, major version in the path ([ADR-011](docs/adr/011-api-versioning-and-openapi.md)) |
| **RFC 9457** Problem Details | — | code | One error format with stable machine-readable codes |
| **Flyway** | Boot-managed | code | Numbered migrations. Changing an applied one stops the service from starting |
| **ArchUnit** | 1.4.1 | code | Enforces `api → service → dao` in the build |
| **Testcontainers** | Boot-managed | code | Integration tests run against real PostgreSQL. Not H2: the parts carrying the correctness argument are PostgreSQL-specific — `SELECT … FOR UPDATE`, plpgsql triggers, partial unique indexes, NULLs-distinct uniqueness — and an embedded database that *accepts* the migration without enforcing it gives a green suite proving nothing |
| **Jakarta Bean Validation** | Boot-managed | code | Rules that bind two fields live on a command object as a class-level constraint, so they hold for a Kafka listener as well as a controller |
| **JSON Schema** + registry | — | planned | Event contracts, with Java and TypeScript generated from them ([ADR-002](docs/adr/002-shared-contracts-versioning-strategy.md)) |

### Data and events

| Technology | Version | Status | Why |
|---|---|---|---|
| **PostgreSQL** | 18.6 | running | The ledger's correctness rests on one `SELECT … FOR UPDATE` and a `CHECK` constraint, so an overdraft is a schema violation ([ADR-004](docs/adr/004-atomic-balance-reservations.md)). Running since 2026-09-18: three clusters grouped by restore boundary, five databases, data checksums on and a version-stable `C.UTF-8` collation ([ADR-003](docs/adr/003-hybrid-database-strategy-with-analytics.md)) |
| **Apache Kafka** | — | planned | An undeployed consumer costs nothing, which is what makes modules removable ([ADR-005](docs/adr/005-module-composition-and-deployment-topology.md), [ADR-007](docs/adr/007-kafka-topics-and-event-schema-evolution.md)) |
| **Redis** | — | planned | Cache and rate limiting, kept off the correctness path. The superseded ADR-001 put it there, and that was the bug |
| **CloudNativePG** | 1.30.0 | running | Operator, not a StatefulSet: failover, switchover and minor upgrades stop being scripts somebody is on call for. Gives each instance its own volume, the only shape that works on node-local storage |
| **`pg_dump` CronJob** | — | running | Daily logical backup, verified at write time with `pg_restore -l`. No object store means no WAL archive, so the RPO is the dump interval — stated rather than implied ([ADR-003](docs/adr/003-hybrid-database-strategy-with-analytics.md)) |
| ~~**Elasticsearch**~~ | — | withdrawn | Out on 2026-09-17: SSPL / Elastic License 2.0 / AGPLv3, none permissive, and Vaullet ships into customer clusters. Replacement deferred to when Fraud Detection is built — OpenSearch, or PostgreSQL full-text alone ([ADR-003](docs/adr/003-hybrid-database-strategy-with-analytics.md)) |

### Security

| Technology | Version | Status | Why |
|---|---|---|---|
| **OpenBao** | 2.6.2 | running | Holds every credential. Restarts sealed and needs two of three key holders to open |
| **External Secrets Operator** | 2.10.0 | running | Copies credentials into Kubernetes Secrets, so no service talks to OpenBao. Each namespace reads only its own path |
| **cert-manager** · **Let's Encrypt** | 1.21.1 | running | The public certificate, plus a private CA for OpenBao's TLS |
| **Spring Security** | Boot-managed | code | One JWT resource-server chain in backend-common |
| **Host hardening** | — | running | ufw, fail2ban, SSH keys only, unattended security updates |
| **Keycloak** | — | planned | Identity provider per cluster. MFA and OIDC conformance are not things to write yourself ([ADR-006](docs/adr/006-authentication-and-identity.md)) |
| **Linkerd** | — | planned | mTLS between services. ~30 sidecars cost 300–600 MB, against 1.5–3 GB under Istio ([ADR-008](docs/adr/008-service-to-service-authentication.md)) |
| **cosign** · admission verification | — | planned | Signed images, checked before they run |

### Infrastructure

| Technology | Version | Status | Why |
|---|---|---|---|
| **Hetzner AX41** | — | running | One rented box: 6 cores, 64 GB ECC, mirrored NVMe. Production topology, not production grade — three VMs share one PSU, one board and one array |
| **Ubuntu** | 26.04 LTS | running | Host OS |
| **libvirt / KVM** | — | running | Three guests on the one machine, each with its own data disk so a full database cannot take the node's root filesystem with it ([ADR-015](docs/adr/015-cluster-topology-three-servers-on-one-machine.md)) |
| **RKE2** | v1.36.4+rke2r1 | running | Cluster state in etcd, not a SQLite file. **Three server nodes** since 2026-09-17, left schedulable: etcd quorum, working PodDisruptionBudgets and live drains are real rather than described |
| **Kubernetes** | 1.36 | running | One cluster per customer: isolation is the cluster boundary, no `tenant_id` |
| **Traefik** + **Gateway API** | 3.7.13 | running | No Ingress objects. Upstream retired ingress-nginx in March 2026 |
| **HAProxy** | 3.2.9 | running | On the host, TCP passthrough to all three nodes. The machine has one public IPv4 and the guests have none; health checks mean a node going down is a logged `DOWN` rather than an outage. TLS still terminates in Traefik |
| **local-path-provisioner** | 0.0.37 | running | The only StorageClass, node-local by design. Longhorn is rejected: replicating three ways onto one RAID1 array is write amplification for no durability ([ADR-015](docs/adr/015-cluster-topology-three-servers-on-one-machine.md)) |

### DevOps

| Technology | Version | Status | Why |
|---|---|---|---|
| **GitHub Actions** | — | running | One workflow per repo. Release versions come from commit messages |
| **GHCR** | — | running | Images and Maven artifacts. Tags are immutable, so `git revert` is a real rollback |
| **Docker Buildx** · **Cloud Native Buildpacks** | — | running | The web image ships with an SBOM and provenance. Services build with `spring-boot:build-image` |
| **Dependency review** | — | running | Pull requests adding vulnerable dependencies fail |
| **Terraform** | — | planned | A new customer environment is a plan and an apply |

### GitOps

| Technology | Version | Status | Why |
|---|---|---|---|
| **Argo CD** | 3.5.2 | running | Pulls each cluster from git, so nothing reaches into a customer's environment ([ADR-010](docs/adr/010-polyrepo-cicd-and-gitops-delivery.md)) |
| **Argo Rollouts** | 1.10.0 | running | vaullet.dev deploys blue-green: new pods start, then every request switches at once |
| **Helm** · **Kustomize** | — | running | Platform components are pinned chart versions in git. Services are Kustomize |

### Observability

| Technology | Version | Status | Why |
|---|---|---|---|
| **Prometheus** · **Grafana** | — | planned | The ledger's `vaullet.reconciliation.mismatch.count` must be zero |
| **Jaeger** | — | planned | Tracing across the synchronous approval path |

A platform-wide observability decision is [still open](docs/adr/TODO.md).

### Where older documents disagree

| Document | Says | What is true |
|---|---|---|
| ADR-010 §11 | ESO syncs from the customer's own secret store | ESO syncs from OpenBao in the cluster. A customer's own store stays possible by changing the store's provider |
| ADR-010 | Flyway runs as a Helm pre-upgrade Job | Flyway runs when the service starts |
| ADR-003 | One schema per service | Under review: one database per service |
| ADR-004 | Balance buckets inside the ledger | Being replaced by separate accounts (cash, bonus) |
| ADR-005, ADR-010 | Module composition with Helm and Terraform | Argo CD Applications over Helm and Kustomize, no Terraform |

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
| Architecture | 14 ADRs, 1 superseded, reconciled against each other on 2026-09-02 |
| Implementation | Two services with code, neither deployed — [ledger](../wallet-ledger-service) (reservations) and [auth](../wallet-auth-service) (the account anchor), on a shared [backend-common](../backend-common) |
| Tests | 44 integration tests against real PostgreSQL: 18 ledger, 26 auth |
| Cluster | Argo CD and OpenBao running; no PostgreSQL, no Keycloak, no service deployed |
| Open items | [24 tracked](docs/adr/TODO.md) — 16 open, 8 deferred with triggers; 2 are blocking |

A private portfolio project, built to production-grade B2B standards: the goal is an architecture that
survives review by a senior engineer, not a system that currently ships.

## Licence

**© 2026 Predrag Arsić. All rights reserved.** Published for reading and evaluation — see
[LICENSE](LICENSE). Brief quotation with attribution is fine; reproducing these documents, or building
from them commercially, is not. The same terms apply across every repository in this project.

Ask if you want to do something the licence does not cover.
