# ADR-005: Module Composition and Deployment Topology

## Status

**Accepted** (2026-08-27)

## Context

Vaullet is sold as a **single-tenant deployment**: each customer receives their own Kubernetes cluster running their own instance of the platform. Customers are wallet operators (initially betting platforms), and the commercial model sells capabilities separately — a contract determines which modules that customer's cluster runs.

This is not multi-tenancy. There is no shared infrastructure between customers, no `tenant_id`, and no cross-customer data path. Isolation is the cluster boundary, which is both stronger than row-level scoping and, for gambling licences with data-residency conditions, frequently mandatory.

### Forces

- **Commercial**: modules are line items. The Bonus platform is the primary differentiator and the expected upsell; the core wallet is the land.
- **Regulatory**: some capabilities cannot be optional regardless of contract. An audit trail is a licence condition, not a feature.
- **Operational**: a full cluster per customer is a high per-customer cost floor. Every component a customer didn't buy is cost with no revenue against it.
- **Architectural**: a customer's deployment must be a *subset* of one codebase, never a fork. Per-customer branches would be unmaintainable by the second customer.

### The Question

Given N optional modules, which combinations are valid, how is a combination expressed, and what stops an absent module from breaking a present one?

## Decision

We will classify every component into one of **four categories**, and enforce a structural rule that makes any combination of sellable modules valid by construction.

### The Rule

> **A synchronous dependency makes a component core. Optional functionality communicates only through events.**

Turning off an event consumer is safe: Kafka neither knows nor cares that a consumer group is absent, and the producer is unaffected. Turning off a synchronous dependency is not: the caller gets a connection failure, and both recoveries are wrong by default — fail-closed halts all transactions, fail-open silently disables a control in a financial system.

Rather than build machinery to handle that, we remove the case. Nothing optional sits in a synchronous path.

### Category 1: Core (always deployed)

| Component | Why core |
|---|---|
| Transaction Service (incl. Refund) | The orchestrator |
| Vaullet (Ledger) | Synchronous `reserve` on the approval path (ADR-004) |
| Auth Service | Synchronous token validation |
| Fraud Detection | Synchronous `POST /fraud/analyze-transaction` |
| Limits | Synchronous `POST /limits/check` |
| **Audit** | **Compliance override** — event-driven, so the rule says optional, but a deployment without an audit trail is not licensable. Core for regulatory reasons, not architectural ones. |
| Admin UI | Operating the platform is not optional |

Audit is the one deliberate exception, recorded here so it is not "rediscovered" during a certification review.

### Category 2: Platform Infrastructure (always deployed, never sold)

Kafka, Redis, PostgreSQL, Elasticsearch, **Keycloak** (ADR-006), and the **Scheduler Service**.

Scheduler warrants explanation: it is event-producing and nobody buys a cron. But Subscription cannot function without it, since `scheduler.payment-due` originates there. Classifying it as infrastructure rather than a module removes a sellable-to-sellable dependency — see the invariant below.

### Category 3: Sellable Modules

| Module | Notes |
|---|---|
| **Bonus** ⭐ | Flagship. Grants promotional value into ledger buckets (ADR-004) |
| Loyalty | Independent of Bonus |
| Referral | Independent of Bonus |
| Subscription | Requires Scheduler (infrastructure, always present) |
| Risk Management | Requires Fraud Detection (core, always present) |
| Notification | Pure event consumer |
| Reporting ETL | Pure event consumer |

### Category 4: Configurable Adapters

Not on/off, but one port with multiple implementations selected per contract. Currently one: **identity provider mode** — Keycloak realm users, or Keycloak identity-brokering to the operator's OIDC/SAML provider (ADR-006). Both issue identical tokens from the same realm, so no other service is aware of which is configured.

This category exists because "which implementation" and "whether present" are different questions that would otherwise be conflated in the same configuration mechanism.

### The Invariant

> **No sellable module may depend on another sellable module.**

Every dependency of every module in Category 3 points at Category 1 or 2 — components guaranteed present. The module graph is a star, not a mesh.

This is why Scheduler was reclassified and why Risk Management is acceptable despite depending on Fraud. The consequence is that **all 2⁷ = 128 combinations of sellable modules are valid**. There are no illegal configurations to detect, document, or guard against, and no contract can be sold that the platform cannot deploy.

This invariant must be checked whenever a module is added. Introducing one sellable-to-sellable edge converts a star into a graph and reintroduces invalid combinations.

### Enablement: build-time, not runtime

A module a customer did not buy is **not deployed at all** — no pod, no database schema, no Kafka consumer group. Enablement lives in per-customer Helm values, not in runtime feature flags.

```yaml
# contracts/acme-betting/values.yaml
modules:
  bonus:        { enabled: true }
  loyalty:      { enabled: true }
  referral:     { enabled: false }
  subscription: { enabled: false }
  riskMgmt:     { enabled: true }
  notification: { enabled: true }
  reportingEtl: { enabled: false }

auth:
  provider: federated          # local | federated (Keycloak broker configured or not)
  broker:
    issuer: https://id.acme-betting.example

platform:
  currency: EUR                # ISO 4217; one per deployment (ADR-004)
  minorUnits: 2
```

`platform.currency` is operator-supplied and immutable for the life of the deployment. It is
configuration in the same file as module enablement, but unlike a module it cannot be flipped later:
changing it is a data migration, not a redeploy. See [ADR-004](004-atomic-balance-reservations.md).

Rationale:

- **Auditable.** "Not installed" is demonstrable to a regulator. "Flag is false" requires trusting runtime state.
- **Least privilege.** A flagged-off service still holds database credentials, network reachability, and attack surface it has no business having.
- **Cost.** A customer without Bonus should not pay for the pod or the `rewards_db` schema.
- **Clean Kafka.** No deployment means no consumer group, so no lag alerts on a module nobody bought.

The trade-off is that enabling a module requires a deploy rather than a config change. Acceptable: contracts change at business speed, not runtime speed.

Runtime flags remain, scoped to behaviour *within* a deployed module (campaign toggles, rule thresholds). They never control module presence.

### Database provisioning follows enablement

`rewards_db` is provisioned only if at least one of Bonus, Loyalty, or Referral is sold; each schema only for its own module. Same for the conditional schemas within `support_db` (ADR-003). A deployment's database footprint is a function of its contract.

## Consequences

### Positive Consequences

✅ **Every contract is deployable** — the star invariant makes all 128 module combinations valid by construction, with no combination logic to maintain
✅ **Test matrix collapses** — because modules cannot interact, testing core-only, core + each module alone (7), and core + all covers the risk in 9 configurations rather than 128
✅ **One codebase** — a customer deployment is a subset, never a fork
✅ **Cost tracks revenue** — unsold modules cost nothing to run
✅ **Regulator-friendly** — the deployed manifest *is* the evidence of what does and doesn't run
✅ **The rule is checkable** — "is this call synchronous?" is a question about code, so drift is detectable in review

### Negative Consequences

⚠️ **The synchronous rule constrains design.** Any future capability wanted on the approval path must be core — sold to everyone or to no one. A customer wanting a bespoke synchronous check cannot be served without changing this ADR.

⚠️ **Per-customer cost floor is high.** Core alone is 7 services plus Kafka, Redis, Elasticsearch and several Postgres instances. Small operators may be unprofitable to serve on this topology.


> **Measured, 2026-09-17.** The floor now has a number rather than an adjective. A core-only
> deployment — 7 services, PostgreSQL, Kafka, Keycloak, Redis, Elasticsearch — is roughly **25 GiB of
> memory requests**. On the reference machine (62.7 GiB, three nodes per
> [ADR-015](015-cluster-topology-three-servers-on-one-machine.md), ~19% topology overhead) that leaves
> ~51 GiB usable, so core fits with room. Adding all seven sellable modules takes it to ~35 GiB.
>
> Two things this makes concrete. **A customer who buys nothing still pays for ~25 GiB of
> infrastructure**, which is the argument this consequence is making. And **the full 14-service set is
> a development-cluster shape, not a customer one** — ADR-005's own rule is that a deployment is a
> subset, so the box that has to run everything at once is the one that can least afford to.
⚠️ **Elasticsearch is core by inheritance.** Fraud Detection is core and ADR-003 gives it Postgres + Elasticsearch, so every customer runs an ES cluster. Worth revisiting: pattern search could degrade gracefully to Postgres-only for small deployments, making ES a Category 4 adapter.

⚠️ **Enabling a module needs a deploy** — accepted above.

⚠️ **Helm values become a commercial artifact.** A values file now encodes what a customer is contractually owed. It needs the review discipline of a contract, and must not drift from what sales actually sold.

### Risks

| Risk | Impact | Mitigation |
|------|--------|-----------|
| A sellable-to-sellable dependency is introduced | Invalid combinations become possible; test matrix explodes | Dependency direction asserted in CI: no module in Category 3 may reference another |
| A synchronous call to an optional module is added | Deployments without that module break at runtime | Architecture test on the call graph; the rule is mechanically checkable |
| Values file drifts from the signed contract | Customer billed for what they don't run, or runs what they didn't buy | Contract → values generation reviewed at sale; periodic reconciliation of deployed manifests against contracts |
| Untested module combination reaches production | Latent defect at a customer | The 9-configuration matrix above runs on every release |
| Per-customer cluster sprawl | Operational load scales linearly with customers | GitOps from a single repo of per-customer values; no manual cluster changes |

## Alternatives Considered

### Alternative 1: Runtime feature flags

**Description**: Deploy every module everywhere; a flag service decides what is active per customer.

**Pros**: Enable and disable without a deploy; one uniform artifact; trivial trials and rollbacks.

**Cons**: A customer's cluster runs code they never bought, holding credentials and consuming resources. "Disabled" becomes a runtime assertion rather than a structural fact, which is materially weaker to demonstrate to an auditor. Full cost regardless of contract. A flag bug becomes a licensing incident.

**Why rejected**: The deployment artifact is the strongest available evidence of what a customer runs, and in a regulated market that evidence is worth more than deploy-free toggling.

---

### Alternative 2: Per-customer branches or forks

**Description**: Each customer gets a branch containing only their modules.

**Pros**: Maximum flexibility for bespoke work.

**Cons**: Unmaintainable past the second customer — every fix cherry-picked N ways, no shared release, divergence guaranteed.

**Why rejected**: Not viable at any scale worth having.

---

### Alternative 3: Modular monolith with compile-time module selection

**Description**: One deployable, modules included or excluded at build time.

**Pros**: Far lower per-customer cost — one process instead of a dozen. Genuinely attractive for small operators, and would make the cost floor objection above largely disappear.

**Cons**: Module boundaries stop being enforced by the network and become a matter of discipline. Independent scaling of the transaction path is lost. Per-customer build artifacts complicate the release pipeline.

**Why rejected — with reservations**: The service split is retained because it makes module boundaries structural rather than conventional, and because sellable units and deployable units coinciding is what keeps the commercial and technical models aligned. This remains the strongest candidate for a future low-cost deployment tier, and should be revisited if small-operator economics matter.

---

### Alternative 4: Rules as data, single deployment

**Description**: One generic engine; module behaviour expressed as configuration rather than code.

**Pros**: Ultimate flexibility, no deployment variants.

**Cons**: Bonus mechanics — wagering requirements, tiering, campaign eligibility — are the flagship differentiator. Reducing them to configuration in a generic engine builds a worse version of the product's main selling point.

**Why rejected**: The differentiator should be code that can be developed, tested and sold, not configuration in an engine that resists both.

## Implementation Notes

### Repository and chart structure

An umbrella Helm chart with one subchart per module, each gated on `modules.<name>.enabled`. Per-customer values live in a separate GitOps repository, one directory per contract, deployed by Argo CD.

### CI matrix

Nine configurations, justified by the star invariant:

1. Core only
2. Core + Bonus
3. Core + Loyalty
4. Core + Referral
5. Core + Subscription
6. Core + Risk Management
7. Core + Notification
8. Core + Reporting ETL
9. Core + all sellable modules

Plus both Auth adapters (local, federated) against configuration 1 and 9.

### Architecture tests

Two rules, both mechanically enforceable and both run in CI:

1. No Category 3 module may hold a synchronous client for another Category 3 module.
2. No core service may hold a synchronous client for any Category 3 module.

Violations are build failures, not review comments. The rule that keeps 128 combinations valid should not depend on a reviewer noticing.

### Shared contracts

Per ADR-002, `wallet-shared-contracts` defines every event type regardless of which modules are deployed. Vaullet consuming a `rewards.value-granted` event that no deployed module emits costs nothing; the topic simply stays empty. Contracts are a compile-time dependency and are unaffected by deployment topology.

## References

- [ADR-002: Shared Contracts with Dual Publishing](002-shared-contracts-versioning-strategy.md) — event definitions independent of deployment
- [ADR-003: Hybrid Database Strategy](003-hybrid-database-strategy-with-analytics.md) — conditional database provisioning
- [ADR-004: Atomic Balance Reservations](004-atomic-balance-reservations.md) — ledger buckets; why grants are event-driven
- ADR-006: Authentication Strategy (pending) — the Category 4 adapter

---

**Date**: 2026-08-27
**Author**: Predrag
**Reviewers**: TBD
