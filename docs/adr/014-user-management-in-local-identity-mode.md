# ADR-014: User Management in Local Identity Mode

## Status

**Accepted** (2026-09-16)

Extends the Category 4 identity adapter in [ADR-006](006-authentication-and-identity.md).
Amends ADR-006's `accounts` DDL and closes open items **B2** and **B3**.

**Amended by [ADR-016](016-transactional-outbox-for-event-producers.md)** (2026-10-22): §7 — the
account-created event is written to an outbox in the account's own transaction and, until there is
Kafka, delivered as the ledger's `POST /v1/wallets` for `REAL` and then `BONUS`.

## Context

[ADR-006](006-authentication-and-identity.md) sells to two operator shapes: those who bring their own
identity provider (`auth.provider: federated`), and those who want Vaullet to *be* their identity
provider (`auth.provider: local`). The second group is buying user management — registration,
credentials, MFA, roles — as part of the wallet, and ADR-006 says so in its first paragraph.

ADR-006 then answers only the authentication half. Keycloak issues the tokens in both modes and the
`accounts` anchor keys the ledger, but nothing says how an operator in local mode *creates* a user,
lists users, resets a password, or assigns a role. Today that is Keycloak's admin console — which the
same ADR forbids using against a production realm.

### Constraints

- **We did not write authentication, and must not start now.** ADR-006 *Alternative 4* rejected a
  bespoke identity provider. Any design that puts a credential in a Vaullet table reopens it.
- **One writer to the realm.** Auth Service already holds Keycloak admin credentials for
  just-in-time provisioning and the `account_id` attribute write-back. A second component writing the
  same realm has no way to order its writes against the first.
- **The federated mode must not notice.** ADR-005 Category 4 exists so no service branches on identity
  mode. Whatever this adds must be invisible to a federated deployment.
- **Two open contradictions block the first call of every integration.** `POST /v1/accounts`
  (ADR-012 §3) cannot satisfy ADR-006's `keycloak_sub NOT NULL` for a user who has never logged in
  (**B2**), and `external_ref` is promised as unique and immutable with no column to hold it (**B3**).
- **Data minimisation is a selling point** under federation (ADR-006), and must survive this.

### The question

Where does the user record live — Keycloak or our own tables — and what is the source of truth?

## Decision

### 1. Source of truth is split by fact, not by system

There is no single source of truth for "the user", because Keycloak and Vaullet hold **two different
objects that describe the same person**. Ownership follows one rule: *whoever owns a fact's failure
mode owns the fact.*

| Fact | Owner | Why |
|---|---|---|
| Password hashes, MFA enrolment, recovery codes, sessions, brute-force counters, password policy | **Keycloak, exclusively** | ADR-006 *Alternative 4*. Never mirrored, never read, never backed up by us. |
| Username, email, name, phone, locale, email-verified | **Keycloak** (local mode only) | Registration data Keycloak already stores; a second copy is a sync problem bought for nothing. |
| Role assignment | **Keycloak assigns**; Vaullet enforces meaning | ADR-006 *Roles*. Separation of duties is ledger and audit state, not a directory claim. |
| `account_id` | **Vaullet, permanently** | Keys seven years of immutable `ledger_entries` (ADR-004). |
| `status` — the freeze | **Vaullet, authoritative** | ADR-006: Keycloak's `enabled` flag is explicitly *not* the primary control. |
| `external_ref` | **Vaullet** | The operator's own user id (ADR-012 §3). |
| Balances, holds, journal | **Ledger** | ADR-004. |

Stated in one line: **Keycloak owns the identity, Vaullet owns the account.** Both modes obey this
table; federated mode simply has no rows in the two middle bands.

### 2. This is a Category 4 capability, not a sellable module

User management turns on with `auth.provider: local` and is unavailable under `federated`. It is not
bought and cannot be declined, so it is not a Category 3 module (ADR-005). It ships inside **Auth
Service** — repository `wallet-auth-service` — and introduces no new deployable component.

ADR-005's rule ("a synchronous dependency makes a component core") is not violated: these endpoints
are synchronous for the *operator*, and no Vaullet service calls them. The module graph is unchanged.

Operators discover availability through the mechanism ADR-012 §4 already defines — absent endpoints
return `404 ENDPOINT_NOT_AVAILABLE`, and `GET /v1/capabilities` reports `user_management`.

### 3. Identity is a sub-resource of the account

The account is the resource in every mode; identity hangs off it, and is present only in local mode.
A parallel `/v1/users` resource keyed on the same identifier is rejected below.

```
POST   /v1/accounts                       local:     { username, email, password?, external_ref?, … }
                                          federated: { external_ref?, metadata? }
GET    /v1/accounts/{id}
PATCH  /v1/accounts/{id}                  status — BOTH modes, core, the fraud freeze
GET    /v1/accounts/by-ref/{external_ref}

GET    /v1/accounts/{id}/identity         local only  → Keycloak
PATCH  /v1/accounts/{id}/identity         local only  → Keycloak
POST   /v1/accounts/{id}/password-reset   local only  → Keycloak execute-actions-email
GET    /v1/accounts/{id}/roles            local only  → Keycloak realm roles
PUT    /v1/accounts/{id}/roles            local only  → Keycloak realm roles
DELETE /v1/accounts/{id}                  local only  → CLOSED + identity erased; ledger untouched
GET    /v1/accounts?email=&status=&…      local only  → Keycloak search, cursor-paginated
```

`PATCH /v1/accounts/{id}` stays outside user management deliberately. Freezing is core in both modes
(ADR-012 §4), belongs to `FRAUD_REVIEWER`, and user administration belongs to `SUPER_ADMIN`. Two
endpoints, two roles, two audit streams — ADR-006's separation of duties falls out of the resource
layout instead of being enforced on top of it.

### 4. Auth Service is a facade. It stores no profile.

Every identity read is a call to the **Keycloak Admin REST API**, answered from Keycloak and returned;
every identity write is an Admin API call. Vaullet persists no email, no name, no phone, no role
assignment and no credential — in either mode.

The alternative, a locally projected read model, is rejected below and deferred rather than refused:
it is purely additive, so it can be introduced later without touching this contract.

### 5. The anchor, amended

This supersedes the DDL in ADR-006 *The account anchor* and lives in `identity_schema` (ADR-003).

```sql
CREATE TABLE accounts (
    account_id   UUID        NOT NULL,
    keycloak_sub UUID        NULL,        -- null while unlinked; set at first authentication
    external_ref TEXT        NULL,        -- operator's own user id; immutable once set
    status       TEXT        NOT NULL,
    created_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at   TIMESTAMPTZ NOT NULL DEFAULT now(),

    CONSTRAINT accounts_pk              PRIMARY KEY (account_id),
    CONSTRAINT accounts_keycloak_sub_uk UNIQUE (keycloak_sub),
    CONSTRAINT accounts_external_ref_uk UNIQUE (external_ref),
    CONSTRAINT accounts_status_ck       CHECK (status IN ('ACTIVE','SUSPENDED','CLOSED')),

    -- Every account must be addressable by something other than the id we just minted.
    -- This is what makes POST /v1/accounts retry-safe without an Idempotency-Key.
    CONSTRAINT accounts_natural_key_ck
        CHECK (external_ref IS NOT NULL OR keycloak_sub IS NOT NULL),

    -- An empty string is not a missing value; allowing both would give two spellings of
    -- "no operator reference", one of which silently defeats the constraint above.
    CONSTRAINT accounts_external_ref_not_blank_ck
        CHECK (external_ref IS NULL OR length(btrim(external_ref)) > 0)
);
```

Plus a `BEFORE UPDATE` trigger holding two invariants the database owns rather than the service:
`external_ref` is immutable once set, and a `CLOSED` account does not reopen. A trigger and not a
`RULE` — a rule with `DO INSTEAD NOTHING` makes a forbidden write succeed *silently*, which is worse
than the failure it prevents.

This is implemented as `V1__identity.sql` in `wallet-auth-service`, and the generated catalogue
(`docs/schema/`) is the review surface.

Three changes, each closing a known contradiction:

- **`keycloak_sub` becomes nullable** (**B2**). A null is not a defect; it is the *unlinked anchor*, a
  state the design genuinely has — an account provisioned by the operator for a user who has not yet
  logged in. Readers must handle it. The ledger never does: it keys on `account_id` only.
- **`external_ref` exists, `UNIQUE` and nullable** (**B3**), with no `UPDATE` path in the service.
  ADR-012 promises operators it is immutable; that promise is now a constraint rather than a hope.
- **`status` is `ACTIVE | SUSPENDED | CLOSED`.** ADR-006 said `LOCKED`, ADR-012 said `SUSPENDED`, for
  one column. `SUSPENDED` wins on two grounds: it is the value already published to operators in
  ADR-012, and `LOCKED` collides with Keycloak's own brute-force *temporarily locked* user state
  inside the same system — the exact confusion ADR-006 spends a section preventing.

### 6. One creation endpoint, two modes, repairable

`POST /v1/accounts` means "make me an account" in both modes. What differs is how much identity exists
at the time.

**Local mode** — create the Keycloak user, then the anchor, then write `account_id` back as a Keycloak
user attribute so the protocol mapper can emit it (ADR-006). Three writes across two systems with no
transaction between them.

**Federated mode** — insert the anchor alone, `keycloak_sub` null, linked at first authentication by
the just-in-time path ADR-006 already requires.

The partial-failure cases are closed by **making the operation idempotent rather than transactional**:

| Fails after | Leaves | Repaired by |
|---|---|---|
| Keycloak user created | A realm user with no anchor | JIT provisioning at first login, which must exist for federated mode anyway |
| Anchor inserted | An account whose tokens carry no `account_id` | The attribute write-back is re-attempted on next authentication |
| Nothing | Nothing | Retry |

No compensating-transaction machinery is written, because the self-healing path is one the platform
is obliged to have regardless.

#### `POST /v1/accounts` carries no `Idempotency-Key`

ADR-011 makes the header a platform rule for every state-changing POST. **This endpoint is exempt,
and it is the only one entitled to be**, because the resource has natural keys that a reservation
does not: `external_ref` and `keycloak_sub` are both unique, and `accounts_natural_key_ck` guarantees
at least one is present. A retry is therefore recognisable from the request itself.

The reason generalises: an account is always created for somebody who already exists in another
system — the operator's user table, or the Keycloak realm — so the caller holds an identifier for
them *before* they hold ours. A hold has no prior identity, which is why the ledger needs a key.

Three outcomes, all decided from the request alone:

| Second request arrives with | Meaning | Answer |
| --- | --- | --- |
| A natural key matching, nothing contradicting | A retry | `201` with the **existing** account |
| A key matching, the other key disagreeing | A collision | `409`, named for **the key that matched** |
| Neither matching | New | `201`, inserted |

The conflict is reported against the key that *found* the account, not the field that differs.
Getting that backwards produces error bodies that are confidently wrong — `IDENTITY_ALREADY_LINKED`
naming a subject linked to nothing, or `EXTERNAL_REF_TAKEN` naming a reference nobody holds. The
first implementation did exactly that; a unit test caught it.

An `Idempotency-Key` header is accepted and logged when sent, so an integrator who puts one on every
POST is not punished for it. It is not stored and not required.

### 7. Creating an account creates the ledger's row

Nothing in the architecture currently creates `account_balances`, and the ledger has no create
endpoint — the gap the money-model review surfaced as *"no document says who creates the ledger's
account row or its cash bucket"*. It is this service's, because this service mints the identifier.

**By event, with an idempotent consumer.** Auth Service publishes
`identity.account-created.v1`; the ledger consumes it and upserts `account_balances` **and** the
singleton `CASH` bucket, keyed on `account_id`.

A synchronous call was rejected: account creation would then fail whenever the ledger is down, the
write would span a third system with no transaction, and it inverts the dependency — the ledger is
the platform's most protected service and should not accept writes from an identity adapter.

Because the consumer is an idempotent upsert, lazy creation falls out as a free fallback: any credit
path upserts the same way, so if money somehow arrives before the event, whichever gets there first
creates the row and the other is a no-op. That closes the eventual-consistency window the event
otherwise opens.

> **This exposes that there is no outbox anywhere in the architecture.** Publishing after the commit
> means a crash between the two loses the event permanently. Here it is survivable — the first
> deposit repairs it — but by accident rather than by design, and every event producer added after
> this one meets the same gap. It deserves a decision before the second producer exists.

> **Amended 2026-10-22 by [ADR-016](016-transactional-outbox-for-event-producers.md) — as built in
> wallet-auth-service 0.12.0.**
>
> - **The event is an outbox row**, written in the transaction that inserts the anchor, so it exists
>   exactly when the account does. The gap in the note above is closed (it was **R8**).
> - **The ledger opens wallets rather than upserting a row.** It grew wallets
>   (wallet-ledger-service#10): `POST /v1/wallets` opens an account's `REAL` or `BONUS` wallet, and
>   with the account's first, its `account_balances` row; a `REAL` wallet starts with the singleton
>   `CASH` bucket. Every account gets both wallets.
> - **There is no Kafka yet**, so a relay in Auth Service delivers the row as those two calls, `REAL`
>   then `BONUS`, with a token carrying `ledger:write`. Until Kafka the ledger therefore accepts the
>   write from the identity adapter that the rejection above objected to; creating an account still
>   never waits for the ledger. When Kafka arrives the relay publishes `identity.account-created.v1`
>   and the ledger's consumer opens the wallets (**R9**).
> - **The lazy fallback does not exist yet**: the ledger has no credit path for it to ride on.
> - **Accounts that predate the outbox** were queued by V3's backfill; one the previous version
>   created during the rollout is queued by the relay's sweep.

### 8. Closing an account

`DELETE /v1/accounts/{id}` sets `status = 'CLOSED'`, deletes the Keycloak user, and touches no
financial record. `account_id`, `ledger_entries` and the seven-year retention are unaffected.

This is the anchor earning its keep a second time. ADR-006 justified it by IdP migration; it is also
what makes GDPR erasure and financial retention coexist, because the immutable records are keyed on an
identifier that carries no personal data. Closure is irreversible and emits an `Audit` event.

### 9. Configuration

```yaml
auth:
  provider: local
  userManagement:
    hostedPages:      true        # Keycloak's own registration and reset screens
    provisioningApi:  true        # POST /v1/accounts
    passwordReset:    keycloak
```

**Both paths, and an operator may run either or both** (decided 2026-09-16). They answer different
needs rather than competing: Keycloak's hosted pages cost no code and give end users registration,
login and password reset that a self-service product needs, while `POST /v1/accounts` is how an
operator provisions on someone's behalf and how a migration imports in bulk. Two booleans rather than
one enum, because "either" was never the real question.

Under `auth.provider: federated` the whole block is rejected at chart-render time rather than
ignored — a setting that silently does nothing is worse than one that fails the render.

## Consequences

### Positive Consequences

✅ **No credential ever enters a Vaullet table** — ADR-006 *Alternative 4* stays decided
✅ **Nothing can drift**, because nothing is duplicated; there is no reconciliation job and no rebuild
✅ **Data minimisation survives user management** — Vaullet's breach surface still excludes PII in both modes
✅ **No new deployable component**, so ADR-005's per-customer cost floor is unchanged
✅ **B2 and B3 close**, and with them the first call of every integration
✅ **The fraud freeze stays separated** from user administration, by resource and by role
✅ **GDPR erasure and seven-year retention coexist** without a special case
✅ **Reversible** — a read projection is additive and needs no contract change

### Negative Consequences

⚠️ **Every identity read is a network call to Keycloak.** Admin UI list views become as fast as the
Keycloak Admin API is, which is not very.

⚠️ **Search is Keycloak's search.** Queries spanning both systems — "every `SUSPENDED` account with an
`@acme.com` address" — cannot be one query. The API filters on one side and intersects in the service,
which paginates badly.

⚠️ **A Keycloak outage removes user administration**, not just new logins. ADR-006 already accepts the
login half; this widens it. The money path is unaffected, and the freeze still works because `status`
is ours.

⚠️ **Auth Service becomes a privileged Keycloak client**, holding realm-management credentials. It was
already such a client for JIT provisioning; the scope widens to user create and delete.

⚠️ **Bulk import is unaddressed.** A local-mode operator migrating from an existing system will drive
a per-user REST loop until this is designed.

### Risks

| Risk | Impact | Mitigation |
|---|---|---|
| Keycloak Admin API changes across a major upgrade | User management breaks for a customer | Already inside ADR-006's per-customer upgrade test; add admin-path coverage to the ADR-005 CI matrix |
| Realm-management credential leaks | Full directory control | Held only by Auth Service, sourced from OpenBao via ESO, scoped to the `vaullet` realm and never the `vaullet-admin` realm |
| Orphan realm user after a partial create | A user who can log in with no account | JIT provisioning links it on first authentication; a scheduled report lists realm users with no anchor |
| Operator scripts against Keycloak directly | Two writers, unordered | Console and Admin API access to the production realm is break-glass with a drift alert (ADR-006) |
| Admin list view too slow to use | The capability is nominally present, practically unusable | The named trigger for **Alternative 1**, below |
| `external_ref` reused after closure | A closed account's reference cannot be re-registered | Deliberate: `UNIQUE` is not filtered on status. Operators re-point rather than re-create |

## Alternatives Considered

### Alternative 1: A local read projection of the Keycloak user

**Description**: Same single writer, plus an `identity_schema.user_profiles` table fed by a Keycloak
`EventListenerProvider`, declared a cache and rebuildable from Keycloak on demand.

**Pros**: Admin search becomes one SQL query joined against `accounts.status`; reads survive a
Keycloak blip; no cross-system pagination.

**Cons**: A projection is a cache, so it needs a rebuild job, a staleness alarm and an out-of-order
guard on Keycloak's event sequence. It also copies PII into a database that has none today, and it
depends on a Keycloak SPI plugin — a second thing to carry across major upgrades.

**Why rejected**: it buys a sync problem before the query that needs it has been met. Writes are
identical under both designs, so this is additive.

**Revisit when**: an Admin UI user list exceeds ~1s, or an operator asks for a filter spanning
`status` and a profile field. Tracked as **R7**.

---

### Alternative 2: Vaullet's own user table as source of truth, Keycloak as a mirror

**Description**: A full `users` table — email, name, phone, roles — written first, pushed into
Keycloak.

**Pros**: One place to query; the CRUD reads like any other service's.

**Cons**: Two masters for one record, with no transaction between them and an inevitable divergence
policy. Credentials would still have to live in Keycloak, so the "single" source of truth would be
partial from day one. It also puts PII in Vaullet in a mode where it is avoidable.

**Why rejected**: it is the design this ADR exists to refuse. "One record stored twice" is the failure
mode, not the goal.

---

### Alternative 3: A sellable User Management module

**Description**: Category 3 module in the ADR-005 sense, a line item on the contract.

**Pros**: Priced separately; matches how Bonus and Loyalty are sold.

**Cons**: It is not optional in the way a module is — a local-mode operator cannot decline it and a
federated operator cannot buy it. It would need a second writer to the Keycloak realm and a write into
Auth Service's `identity_schema`, so either a cross-service database write or a synchronous
sellable-to-core dependency.

**Why rejected**: the enablement question is already answered by `auth.provider`. Two mechanisms for
one condition is how configurations drift.

---

### Alternative 4: A separate `/v1/users` resource

**Description**: `/v1/users/{account_id}` alongside `/v1/accounts/{account_id}`.

**Pros**: Registration reads as its own thing, which matches how integrators think about signup.

**Cons**: Two nouns, one identifier, one row. The platform already carries one naming overload
("Vaullet" is both the product and the ledger service) and it costs a paragraph of explanation in
every document.

**Why rejected**: a sub-resource puts the ownership boundary in the URL — the account is ours in every
mode, the identity only sometimes — which is the thing readers most need to understand.

## Implementation Notes

### Build order

`wallet-auth-service` does not exist yet. Build it in this order, because each step is independently
useful:

1. The anchor: migration, repository, `POST`/`GET /v1/accounts`, `by-ref` lookup. Federated mode is
   complete at this point, and the ledger gains the account row it currently has no creator for.
2. Token validation and the `account_id` protocol mapper, plus JIT provisioning.
3. `status` enforcement and the Redis-cached gateway check (ADR-006).
4. The Keycloak admin client and the identity sub-resources — this ADR's subject.
5. `api_clients` and token exchange (ADR-006), which no other step depends on.

### Scopes

The identity endpoints need entries in ADR-008's Layer 3 table. Proposed: `identity:admin` for the
write paths and `identity:read` for `GET`. `SUPER_ADMIN` carries both; `SUPPORT_AGENT` carries
`identity:read` only. **B4** is the same kind of omission and should be fixed in the same pass.

### Keycloak specifics

Realm-management client with `manage-users`, `view-users` and `manage-realm` on the `vaullet` realm
and **nothing** on `vaullet-admin` — the break-glass realm boundary in ADR-006 is only a boundary if
no Vaullet component can write across it. Password reset uses `execute-actions-email` with
`UPDATE_PASSWORD` rather than setting a credential, so no password ever transits Vaullet.

### Out of scope

**Bulk import** — needed by a migrating local-mode operator, and a per-user REST loop until designed.
**Self-service profile editing by end users** — this ADR's endpoints are operator- and admin-facing;
end users edit their profile in Keycloak's account console.
**Admin identity lifecycle** — the `vaullet-admin` realm is local, small, and out of this surface.

### Follow-up edits

- ADR-006: the account anchor DDL and `LOCKED` are superseded here; a pointer is added there. The
  schema is spelled `identity_schema`, matching ADR-003 — corrected 2026-09-17, before the first
  migration ran anywhere.
- ADR-012: `PATCH /v1/accounts/{id}` status values already read `SUSPENDED` and need no change.
- ADR-011: this ADR's §6 records the one endpoint exempt from the `Idempotency-Key` rule, and why.
- ADR-007: §7 needs a topic for `identity.account-created.v1`, plus a decision on the missing outbox
  — both made 2026-10-22 by [ADR-016](016-transactional-outbox-for-event-producers.md): the topic is in
  ADR-007's §9 inventory, and the outbox is its new §11.
- ADR-008: Layer 3 gains `identity:read` and `identity:admin` — added 2026-09-17.
- TODO.md: **B2** and **B3** close with this ADR. **B4** remains — closed 2026-10-22 by ADR-016's
  Layer 3 rows for the ledger.

## References

- [ADR-003: Hybrid Database Strategy](003-hybrid-database-strategy-with-analytics.md) — `identity_schema`
- [ADR-004: Atomic Balance Reservations](004-atomic-balance-reservations.md) — why `account_id` is permanent
- [ADR-005: Module Composition and Deployment Topology](005-module-composition-and-deployment-topology.md) — Category 4
- [ADR-006: Authentication and Identity](006-authentication-and-identity.md) — the mode this extends
- [ADR-008: Service-to-Service Authentication](008-service-to-service-authentication.md) — scope table
- [ADR-012: External API Surface](012-external-api-surface.md) — `/v1/accounts`, `external_ref`, capabilities
- [Keycloak Admin REST API](https://www.keycloak.org/docs-api/latest/rest-api/index.html)

---

**Date**: 2026-09-16
**Author**: Predrag
