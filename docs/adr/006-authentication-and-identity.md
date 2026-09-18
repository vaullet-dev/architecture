# ADR-006: Authentication and Identity

## Status

**Accepted** (2026-08-31)

Implements the Category 4 adapter described in [ADR-005](005-module-composition-and-deployment-topology.md).

**Amended by [ADR-014](014-user-management-in-local-identity-mode.md)** (2026-09-16, extended 2026-09-17): the `accounts`
DDL below is superseded there — `keycloak_sub` is nullable, `external_ref` is added, and the status
values are `ACTIVE | SUSPENDED | CLOSED`. ADR-014 also specifies user management under
`auth.provider: local`, which this ADR names as a selling point but does not design.

## Context

Vaullet is deployed one cluster per operator (ADR-005). Operators arrive in two shapes, and the
commercial model sells to both:

- **Operators without an identity system.** They want the wallet *and* user management: registration,
  credentials, MFA, roles. Vaullet is their identity provider.
- **Operators with an existing identity system.** Their users already exist, often with KYC already
  performed under their gambling licence. They want Vaullet to *recognise* those users, not to own
  them. Vaullet is an auth provider over someone else's directory.

The second is the more common shape for an established betting platform, and the first is the more
common shape for a new one. Neither can be treated as the exception.

### Constraints

- **No service outside Auth may know which mode is deployed.** ADR-005 Category 4 exists precisely to
  keep "which implementation" invisible. If Transaction Service has to branch on identity mode, the
  abstraction has failed.
- **The ledger needs a stable identifier.** `ledger_entries` is immutable and retained seven years.
  Whatever keys a balance must outlive an operator changing identity providers.
- **Fraud locks must be enforceable against the operator's own directory.** When Risk Management
  freezes an account, that decision cannot be overridable by a customer's IdP saying the user is fine.
- **Data minimisation.** Under GDPR — and because the operator holds KYC anyway — storing personal
  data we do not need is a liability, not a feature.

## Decision

We will implement **two identity adapters behind one internal contract**, selected per deployment by
`auth.provider` in the customer's Helm values.

### The internal contract

**Keycloak is the only token signer, in both modes.** Auth Service brokers; it never issues tokens
itself. Every other service validates exactly one token format, against one JWKS, with one claim set:

```json
{
  "sub":        "f47ac10b-…",       // Keycloak user id
  "account_id": "acct_0f9c…",       // Vaullet's own id, injected by a protocol mapper
  "realm_access": { "roles": ["END_USER"] },
  "scope":      "wallet:read wallet:transact",
  "exp":        1756300000,          // 15 minutes
  "iss":        "https://auth.<cluster>/realms/vaullet"
}
```

**`account_id` is injected by a Keycloak protocol mapper** from a user attribute, so downstream
services key on Vaullet's identifier and never on Keycloak's. If Keycloak is ever replaced, or a realm
re-imported, `sub` changes and `account_id` does not.

The modes differ *only* in how the user got into the realm. Downstream they are indistinguishable —
which is the whole point.

### The provider: Keycloak, deployed per cluster

**We will not build an identity provider.** Credential storage, MFA enrolment, brute-force detection,
password policy, session management and OIDC conformance are solved problems with a long history of
subtle vulnerabilities in bespoke implementations. Keycloak is deployed as part of the platform in
every cluster, self-hosted — which SaaS IdPs (Auth0, Okta) cannot satisfy under the data-residency
conditions that motivated single-tenant deployment in the first place (ADR-005).

**Auth Service does not disappear; it shrinks.** It is no longer an identity provider — it is a thin
adapter owning the four things Keycloak cannot own for us:

1. Just-in-time provisioning of the account anchor (below)
2. Account status enforcement, including fraud locks
3. Token exchange orchestration for operator backends
4. Mapping Keycloak roles onto Vaullet's separation-of-duties rules

### The two modes collapse into one mechanism

This is the strongest argument for Keycloak, and it simplifies the original design.

**Mode A — local identity (`auth.provider: local`)**: users live in the Keycloak realm. Registration,
credentials, MFA, password reset and account lockout are Keycloak's, configured per realm.

**Mode B — federated identity (`auth.provider: federated`)**: the operator's OIDC or SAML provider is
registered as an **identity broker** in the same realm. Keycloak validates upstream, and the user
becomes a realm user linked to that external identity.

Either way Keycloak issues **the same token, from the same realm, signed by the same key, with the
same claim set**. The adapter invisibility that ADR-005 Category 4 requires is no longer something we
implement and must keep honest — it is a property of the mechanism.

The primary machine integration remains **token exchange** (RFC 8693): the operator's backend presents
its own client credential plus the user's subject and receives a user-scoped token. This matches how
these integrations actually work — the operator owns the frontend, and their server calls Vaullet.
Direct browser flows use authorization code + PKCE against Keycloak.

### The account anchor (required in both modes)

Even under federation, Vaullet keeps a minimal local row per user:

```sql
CREATE TABLE accounts (
    account_id     UUID PRIMARY KEY,            -- what the ledger keys on, forever
    keycloak_sub   UUID NOT NULL,               -- Keycloak user id, both modes
    status         TEXT NOT NULL CHECK (status IN ('ACTIVE','LOCKED','CLOSED')),
    created_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (keycloak_sub)
);
```

Created **just-in-time** on first successful authentication, in *both* modes, and written back to the
Keycloak user as an attribute so the protocol mapper can emit it thereafter.

> **Superseded by [ADR-014](014-user-management-in-local-identity-mode.md) §5.** `keycloak_sub` is
> nullable, because an operator may provision an account before the user has ever logged in;
> `external_ref` is a column; and `LOCKED` became `SUSPENDED` to match ADR-012 and to stop colliding
> with Keycloak's own brute-force *temporarily locked* state. The reasoning below is unchanged.

**Adopting Keycloak makes this table more necessary, not less.** There is now a third-party system's
identifier in the chain. `keycloak_sub` is stable only as long as that Keycloak realm is; a realm
re-import, a migration to another IdP, or an operator switching their upstream broker all change it.
`ledger_entries` is immutable for seven years and cannot be keyed on any of that. The anchor is the
seam that lets the identity layer be replaced without touching a financial record — which is exactly
the property that made switching to Keycloak cheap in the first place.

This row is not optional, and the reason is the ledger. Balances cannot be keyed on the operator's
`sub` claim: subjects are opaque strings owned by a foreign system, they change when an operator
migrates IdP or re-issues identifiers, and `ledger_entries` is immutable for seven years. Keying
immutable financial records on a mutable external identifier guarantees an unfixable migration later.
With the anchor, the mapping can be re-pointed and the ledger never moves.

### What we store under federation

In Vaullet's own tables: `account_id`, `keycloak_sub`, status, timestamps. **That is the complete
list.** No email, name, date of birth, address, or document data.

Keycloak under brokering holds whatever the upstream IdP asserts. Mapper configuration is set to
import **only** the subject and the claims required for role assignment — brokered profile data is not
persisted. Data minimisation is a realm configuration item, reviewed like any other, not an emergent
property of the default setup.

Attributes that Fraud Detection needs — country, device, session context — are passed **per request**
by the operator, evaluated, and not persisted as a profile. This is a selling point rather than a
limitation: the operator keeps their PII, and Vaullet's breach surface excludes it.

### Revocation and account locking — stays local

Tokens are short-lived (15 minutes) and **`accounts.status` is checked on every request** at the API
Gateway, cached in Redis with explicit invalidation on change.

`risk.account-locked` from Risk Management sets `status = 'SUSPENDED'` (ADR-014 §5) in Vaullet's own
table and takes effect within one cache invalidation, in both modes.

**Keycloak's `enabled` flag is not the primary control**, and this is worth being explicit about
because it is the obvious-looking shortcut. Three reasons it is insufficient:

- Disabling a Keycloak user prevents *new* logins but does not invalidate tokens already issued —
  a locked user would keep transacting for up to the token lifetime.
- It would require Risk Management to hold write credentials against the identity provider, giving a
  sellable module administrative authority over the directory.
- Under brokering, the upstream IdP can re-assert the user on next login.

Keycloak is disabled as a secondary measure. The authoritative lock is Vaullet's, which means Vaullet
will refuse a user the operator's IdP still considers valid. That asymmetry is deliberate and
non-negotiable: a fraud freeze a customer can override is not a fraud freeze.

### Admin identity is local by default

The operator's staff authenticate against Vaullet's local directory even when end users are federated.

Three reasons. **Break-glass**: if the operator's IdP is down, an administrator must still be able to
log in and freeze an account — an outage in the customer's infrastructure must not disable fraud
response. **Privilege containment**: federating admin roles hands the operator's IdP control over who
can approve refunds and adjust limits. **Population and lifecycle**: dozens of accounts with a
different joiner-mover-leaver process from millions of end users.

Federated admin SSO is available for operators with a hard SSO mandate, on the condition that at least
one local emergency account always exists and is exercised quarterly.

### Roles

Realm roles in Keycloak, emitted in `realm_access.roles`. Keycloak assigns them; **Vaullet enforces
what they mean.** Separation of duties is business logic operating on ledger and audit state, not
something an IdP can express.

| Role | Capability |
|---|---|
| `END_USER` | Own wallet only |
| `SUPPORT_AGENT` | Read transactions and balances; move no money |
| `FRAUD_REVIEWER` | Review cases, freeze/unfreeze accounts |
| `FINANCE` | Approve refunds and manual adjustments |
| `CONFIG_ADMIN` | Limits, campaigns, module configuration |
| `SUPER_ADMIN` | User and role administration |

**Separation of duties**: no single role both initiates and approves a money movement. Manual
adjustments require two distinct `FINANCE` principals; `SUPER_ADMIN` can grant the role but cannot
itself approve an adjustment. Every role assignment and every adjustment is an `Audit` event — which
is core in every deployment (ADR-005), so this holds regardless of contract.

### Machine clients

Operators integrating server-to-server authenticate as `api_clients` via OAuth2 client credentials
over mTLS, with scopes distinct from user scopes. A machine client acting for a user obtains a
user-scoped token through token exchange; it never carries ambient authority over all accounts.

## Consequences

### Positive Consequences

✅ **Both operator shapes are sellable** without forking the platform
✅ **Adapter invisibility is structural, not maintained** — both modes are realm users issued identical tokens; there is no code path that could drift
✅ **We did not write authentication** — credentials, MFA, brute-force detection, password policy and OIDC conformance come audited, with a CVE process and a security track record
✅ **Self-hosted per cluster** — satisfies the data-residency conditions that rule out SaaS identity providers
✅ **The ledger is insulated from identity churn** — an operator can replace their IdP without touching a financial record
✅ **Minimal PII under federation** — smaller breach surface, and a genuine differentiator in a regulated market
✅ **Fraud locks are enforceable** even against the operator's own directory
✅ **Fraud response survives a customer IdP outage** via local admin identity
✅ **Separation of duties is structural**, backed by an audit trail that is core in every deployment

### Negative Consequences

⚠️ **Keycloak is a stateful component in every cluster.** A JVM with its own database, needing at
least two replicas for HA and Infinispan cache configuration. ADR-005 already flags the per-customer
cost floor; this raises it, per customer.

> **Now deliverable, 2026-09-17.** "Two replicas for HA" and the `PodDisruptionBudget` in the risk
> table below were not achievable on a single-node cluster — two replicas on one node survive nothing,
> and a PDB has nothing to protect against. [ADR-015](015-cluster-topology-three-servers-on-one-machine.md)
> moves the reference cluster to three nodes, which makes both real: the replicas land on different
> nodes and a drain respects the budget.
>
> The limit worth stating: those three nodes are VMs on one machine, so Keycloak survives a **node**
> going away, not the hardware.

⚠️ **We inherit Keycloak's upgrade cadence.** Major versions have historically carried breaking
configuration changes, and the upgrade must be tested per customer cluster.

⚠️ **Token exchange is not enabled by default** in Keycloak and has spent a long time as a
preview-flagged feature. The primary machine-integration path depends on a non-default feature, which
is a real dependency to track across upgrades.

⚠️ **Just-in-time provisioning creates accounts on first login.** A misconfigured issuer could mint
account rows for users of an unrelated system. Mitigated by pinning `external_issuer` to the configured
OIDC issuer and rejecting any token from another.

⚠️ **Vaullet can lock a user the operator considers active.** Correct, but it needs to be in the
contract and in the operator's runbook, or it will arrive as a support surprise.

⚠️ **Fraud Detection loses stored user history under federation.** Attributes arrive per request, so
long-horizon behavioural profiles are weaker than in local mode unless the operator supplies them.

⚠️ **Local admin identity means a second directory** for operators who mandate SSO everywhere.

### Risks

| Risk | Impact | Mitigation |
|------|--------|-----------|
| Operator's upstream IdP unreachable | Brokered logins fail | Keycloak caches broker metadata; local admin realm unaffected, so fraud response continues |
| Keycloak unavailable | No new logins cluster-wide | 2+ replicas, PodDisruptionBudget; existing tokens valid for their 15-minute life; Vaullet reserve path does not call Keycloak |
| Realm config drifts from source control | Environments diverge; a security setting silently differs | Realm is declarative config applied by `keycloak-config-cli` in CI; console changes in production are a break-glass action that triggers a drift alert |
| Keycloak major upgrade breaks a realm | Auth outage for that customer | Upgrade tested against the ADR-005 CI matrix before any cluster; per-customer rollout, never fleet-wide |
| Stolen internal token | Account access until expiry | 15-minute lifetime; gateway status check; bind token to client where transport allows |
| Subject collision after an operator IdP migration | Two accounts for one person, split balances | `UNIQUE (external_issuer, external_subject)`; migration is a deliberate re-point of the anchor, never a silent re-provision |
| Redis status cache stale after a lock | Locked user transacts briefly | Explicit invalidation on write plus short TTL; Vaullet re-checks status inside the reserve transaction for money movement specifically |
| Admin credentials weaker than end-user credentials | Highest-privilege path is the softest | MFA mandatory for every non-`END_USER` role; no exceptions, including break-glass accounts |
| Break-glass account unused and expired | Unusable in the outage it exists for | Quarterly exercise, recorded as an audit event |

## Alternatives Considered

### Alternative 1: Pass the operator's token through to every service

**Description**: Skip minting an internal token; each service validates the operator's OIDC token.

**Pros**: No token exchange; one fewer moving part.

**Cons**: Every service becomes aware of the identity mode, breaking ADR-005 Category 4. Each would
need issuer configuration, JWKS caching and per-operator claim mapping — identity logic replicated
twelve times and drifting. Local-mode deployments have no such token at all, so services would need
two validation paths regardless.

**Why rejected**: It moves identity concerns into every service to save one hop in Auth Service.

---

### Alternative 2: No local account row under federation

**Description**: Key everything on `(issuer, subject)` directly; store nothing locally.

**Pros**: Maximum data minimisation; nothing to provision.

**Cons**: The ledger's immutable primary identifier becomes a foreign, mutable string. Any IdP
migration orphans seven years of financial records. Account status has nowhere to live, so fraud locks
become unenforceable without asking the operator's IdP — exactly the dependency that must not exist.

**Why rejected**: Saves one narrow table at the cost of the ledger's integrity and the fraud lock.

---

### Alternative 3: Always federate; drop the local adapter

**Description**: Require every operator to bring an identity provider.

**Pros**: One adapter; no credential storage anywhere; smallest security surface.

**Cons**: Excludes new operators who have no IdP — precisely the customers for whom a turnkey wallet
platform is most attractive, and for whom user management is part of what they are buying.

**Why rejected**: Commercially, it discards a segment the product is well suited to.

---

### Alternative 4: Build a bespoke Auth Service

**Description**: Implement credential storage, MFA, session management and OIDC in Vaullet's own Auth
Service.

**Pros**: No third-party component per cluster; no upgrade cadence inherited; complete control of the
token contract; one less stateful service to operate.

**Cons**: It means writing authentication. Argon2id parameter selection, MFA enrolment and recovery
flows, brute-force detection, password policy, session fixation, token replay, OIDC conformance — each
is a well-documented source of vulnerabilities in hand-rolled implementations, and none is a
differentiator. For a platform whose selling point is a bonus engine, engineering effort spent
reimplementing an IdP is effort not spent on the product.

**Why rejected**: The build-versus-buy question is decided by what is actually being sold. Vaullet
competes on ledger correctness and bonus mechanics, not on identity. Keycloak's identity brokering
also collapses the two modes into one native mechanism, so buying *removes* a component we would
otherwise have had to build and keep honest.

**What keeps this reversible**: nothing outside the Auth boundary depends on the choice. The account
anchor, local status enforcement, separation of duties and the claim contract are all ours regardless.
If Keycloak ever stops fitting, it can be replaced without touching another service — and that
property is worth more than the build-versus-buy answer itself.

---

### Alternative 5: Auth0, Okta or another SaaS identity provider

**Description**: Hosted identity as a service.

**Pros**: No component to operate at all; the strongest managed security posture available.

**Cons**: Player identity would leave the customer's cluster, breaking the data-residency conditions
that motivated single-tenant deployment (ADR-005). Per-tenant SaaS pricing multiplies by customer
count with no volume leverage.

**Why rejected**: Incompatible with the deployment model. Self-hosting is a hard requirement here, not
a preference.

## Implementation Notes

### Configuration

```yaml
auth:
  keycloak:
    realm: vaullet
    replicas: 2
    features: [ "token-exchange", "admin-fine-grained-authz" ]
  provider: federated          # local | federated (broker configured or not)
  broker:                      # only when provider == federated
    alias:    acme-idp
    protocol: oidc             # oidc | saml
    issuer:   https://id.acme-betting.example
    syncMode: IMPORT           # subject + role claims only; no profile data
  tokenLifetime: 15m
  admin:
    realm: vaullet-admin       # separate realm, local users, no broker
    mfaRequired: true
```

**Admin identity is a separate Keycloak realm** with no identity broker configured. A realm boundary
enforces the break-glass property structurally: an outage or misconfiguration of the operator's
upstream IdP cannot reach a realm that does not reference it. The reasoning for local admin identity
above is unchanged; Keycloak simply gives it a stronger boundary than a configuration flag would.

### Realm configuration is code

Realms, clients, roles, mappers, brokers and password policy are declarative YAML applied by
`keycloak-config-cli` from `wallet-infrastructure`. **No production realm is configured through the
admin console.** Console access exists for inspection and break-glass only, and a drift check against
the declared state runs on a schedule.

This matters more than it sounds: realm configuration includes the password policy, the brute-force
settings, the token lifetime and the mapper that decides what leaves the IdP. Those are security
controls, and security controls that live only in a database someone clicked belong nowhere near a
regulated deployment.

### Database placement

Keycloak gets its **own schema in `auth_db`** (`keycloak`), owned entirely by Keycloak and never
written to by us — it runs its own migrations on upgrade. Vaullet's `accounts` anchor and `api_clients`
live in a separate `identity_schema` with separate credentials, consistent with the schema-per-service
isolation in [ADR-003](003-hybrid-database-strategy-with-analytics.md).

The coupling to watch is upgrade ordering: a Keycloak major upgrade migrates its schema, so the
`identity` schema must never hold a foreign key into Keycloak's tables. `accounts.keycloak_sub` is
deliberately a plain `UUID` and not a reference.

### Token exchange flow (federated)

```
Operator backend            Auth Service (adapter)        Keycloak
      │                              │                        │
      │  POST /v1/token/exchange     │                        │
      │  client_credentials (mTLS)   │                        │
      │  + subject_token             │                        │
      │─────────────────────────────>│                        │
      │                              │  RFC 8693 exchange     │
      │                              │───────────────────────>│
      │                              │                        │ validate broker token
      │                              │                        │ link/create realm user
      │                              │  access_token          │
      │                              │<───────────────────────│
      │                              │ upsert accounts (JIT)  │
      │                              │ write account_id attr  │
      │                              │ check status != LOCKED │
      │  200 { access_token }        │                        │
      │<─────────────────────────────│                        │
```

Auth Service brokers rather than mints. It never issues tokens itself — Keycloak is the only signer in
the system, so there is exactly one key to rotate and one JWKS for every service to trust.

### Gateway responsibilities

Validate the internal token, check `accounts.status` (Redis-cached), enforce scope, propagate
`account_id` downstream. Services trust the gateway's validated context and do not re-verify tokens —
except Vaullet, which re-checks status inside the reserve transaction, because money movement warrants
the extra read.

### Out of scope

**Service-to-service authentication** (mTLS vs JWT between internal services) is a separate decision,
tracked in the ADR index. This ADR assumes services inside a cluster are mutually authenticated by the
mesh and concerns itself only with *user* identity.

**Operator-facing reporting surfaces** — dashboard versus API versus embeddable widgets — is a product
question, not an auth one. The auth half is settled here: Admin UI is role-scoped, and BI tools read
`datawarehouse_db` through read-only database credentials rather than through Auth Service.

## References

- [ADR-005: Module Composition and Deployment Topology](005-module-composition-and-deployment-topology.md) — Category 4 adapters
- [ADR-004: Atomic Balance Reservations](004-atomic-balance-reservations.md) — why the ledger needs a stable `account_id`
- [ADR-003: Hybrid Database Strategy](003-hybrid-database-strategy-with-analytics.md) — `auth_db` isolation
- [RFC 8693: OAuth 2.0 Token Exchange](https://datatracker.ietf.org/doc/html/rfc8693)
- [OpenID Connect Core 1.0](https://openid.net/specs/openid-connect-core-1_0.html)
- [OWASP ASVS v4 — Authentication Verification Requirements](https://owasp.org/www-project-application-security-verification-standard/)

---

**Date**: 2026-08-31
**Author**: Predrag
**Reviewers**: TBD
