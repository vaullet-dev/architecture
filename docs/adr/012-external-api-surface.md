# ADR-012: External API Surface

## Status

**Accepted** (2026-09-01)
**Revised** (2026-09-02) — hold lifetimes are caller-supplied (ADR-004); spec fragments are generated
per module rather than authored (ADR-011)

## Context

[ADR-011](011-api-versioning-and-openapi.md) settled *how* the external API is versioned and specified.
It does not say what the API contains. That surface — the thing an operator's backend integrates
against — is the product, and it exists in no document.

### Who the caller is

The operator's **server**, not a browser. The operator owns the frontend and the end user's session
(ADR-006); their backend authenticates with client credentials and obtains user-scoped tokens by
exchange (ADR-008). This is a server-to-server API, which means it can be verbose, explicit and strict
in ways a public browser API cannot.

### The wrinkle: the surface is composition-dependent

[ADR-005](005-module-composition-and-deployment-topology.md) makes modules per-contract. An operator
without Bonus has no bonus endpoints — not endpoints that fail, endpoints that are *not there*. ADR-011
assumes one `openapi-v1.yaml` per major version. Those two facts are in tension, and nothing has
resolved it.

### What already constrains the design

- Money is decimal strings with explicit currency; one currency per deployment (ADRs 004, 011)
- `Idempotency-Key` required on state-changing POSTs (ADR-011)
- Cursor pagination; RFC 9457 errors with a stable `code` (ADR-011)
- Balances are typed buckets with a `withdrawable` flag (ADR-004)
- Deposits credit at capture; withdrawals hold funds at request (ADR-009)

## Decision

### 1. Resource design follows the integrator, not the service topology

Deposits, withdrawals and spends are all `TRANSACTION` types inside Transaction Service (ADR-009). They
are **separate resources** in the API:

```
/v1/deposits        /v1/withdrawals        /v1/transactions
```

They have different lifecycles, different fields, different failure modes and different permissions. An
integrator reading `POST /v1/transactions {"type": "DEPOSIT"}` has to learn which fields apply to which
type, and every SDK ends up with a union type nobody enjoys. Internal reuse is an implementation
detail; it is not a reason to make the caller model our services.

### 2. Two-phase transactions — the reservation model, exposed

The single most useful thing the ledger already does, and the flow a sportsbook actually needs.

A bet is placed now and settles hours later when the event resolves. Between those points the stake
must be unavailable but not yet lost. That is exactly an ADR-004 reservation, so it is exposed rather
than hidden:

```http
POST /v1/transactions
Idempotency-Key: bet_9f3c1a2
{
  "account_id":         "acct_0f9c",
  "type":               "WAGER",
  "amount":             "20.00",
  "currency":           "EUR",
  "capture":            false,          ← hold only
  "expires_in_seconds": 259200,         ← how long the stake stays held; default 300
  "reference":          "bet-8891",
  "metadata":           { "market": "match-winner" }
}
→ 201 { "transaction_id": "txn_01J8", "status": "AUTHORIZED",
        "expires_at":  "2026-09-05T18:00:00Z",
        "allocations": [ {"bucket_type": "BONUS", "amount": "12.00"},
                         {"bucket_type": "CASH",  "amount": "8.00"} ] }
```

Then one of:

```
POST /v1/transactions/{id}/capture   { "amount": "20.00" }   → settles (partial capture allowed)
POST /v1/transactions/{id}/void                              → releases the hold, no ledger entry
```

`capture: true` (the default) authorizes and settles in one call, for purchases that resolve
immediately.

**`allocations` is returned deliberately.** The operator needs to know that €12 of a €20 stake came
from bonus funds — it drives what they display, how they compute wagering contribution, and what a
void returns to which bucket. Hiding the bucket split would make the bonus platform, the flagship
module, invisible to the integrator using it.

**The caller sets the deadline, because only the caller knows it.** A purchase resolves in
milliseconds; a match-winner market resolves in three days. `expires_in_seconds` defaults to 300 and is
bounded by the deployment's `max_hold_seconds` (`422 HOLD_TTL_TOO_LONG` beyond it) — the ADR-004
reservation lifetime, exposed. A fixed short timeout would expire every wager in the system long before
its event resolved, which is the failure this section exists to prevent.

An authorization that reaches `expires_at` uncaptured is **released and announced**: the transaction
moves to `EXPIRED`, the stake returns to the buckets it came from, and `transaction.expired` is
delivered to the operator's webhook (from `ledger.hold-expired.v1`). An operator whose capture job
failed learns it from us rather than from a customer complaint.

> **Added 2026-10-26** with wallet-transaction-service 0.1.0 (architecture#7), its first build of
> `POST` and `GET /v1/transactions`. Where it differs from this decision, on purpose and for now:
>
> - **Types are generic, `DEBIT` and `CREDIT`.** `WAGER` and its relatives come when something treats
>   them differently.
> - **It authorizes and stops.** A debit is held, a credit recorded; capture and void are not built,
>   so an absent `capture` means a hold and `capture: true` is refused (`400 VALIDATION_FAILED`) rather
>   than held in silence. `true` as the default comes with capture.
> - **End users call it too**, not only the operator's server of *Who the caller is*: a debit of their
>   own account, and reading its transactions, with their own token (`transaction:self`, ADR-008).
> - **The codes are the platform's.** `RESOURCE_NOT_FOUND`, passed through from the ledger, and
>   `UPSTREAM_UNAVAILABLE`, where §7 lists `ACCOUNT_NOT_FOUND` and `TEMPORARILY_UNAVAILABLE`. One of
>   the two has to move, and renaming a published code is a breaking change under ADR-011.
> - **Idempotency keys never expire.** A key is part of the transaction's record, scoped to the
>   caller (the token's `sub`), where *Idempotency semantics* keeps it for 24 hours.

### 3. Core resources — present in every deployment

```
POST   /v1/accounts                          { external_ref?, metadata? } → account_id
GET    /v1/accounts/{id}
PATCH  /v1/accounts/{id}                     status: ACTIVE | SUSPENDED | CLOSED
GET    /v1/accounts/by-ref/{external_ref}

GET    /v1/accounts/{id}/balance             posted, held, available, withdrawable, buckets[]
GET    /v1/accounts/{id}/transactions        cursor-paginated

POST   /v1/transactions                      authorize (+capture)
GET    /v1/transactions/{id}
POST   /v1/transactions/{id}/capture
POST   /v1/transactions/{id}/void
POST   /v1/transactions/{id}/refund          voluntary reversal (ADR-009)

POST   /v1/deposits                          → { deposit_id, status, redirect_url? }
GET    /v1/deposits/{id}
POST   /v1/withdrawals                       → holds funds immediately (ADR-009)
GET    /v1/withdrawals/{id}
POST   /v1/withdrawals/{id}/cancel           while REQUESTED

GET    /v1/accounts/{id}/limits
PUT    /v1/accounts/{id}/limits              operator-set caps; responsible-gambling controls

GET    /v1/capabilities                      what this deployment actually exposes
POST   /v1/webhooks/subscriptions            manage delivery endpoints
```

`external_ref` lets the operator address accounts by their own user id, so they need not store a
mapping. It is unique per deployment and immutable once set.

`POST /v1/deposits` returns `redirect_url` when the PSP requires a hosted page or 3-D Secure. The
deposit is not credited until `deposit.captured.v1` — the API tells the truth about that rather than
returning an optimistic success.

### 4. Module-gated resources

Present only when the contract includes the module:

| Resource | Module |
|---|---|
| `GET /v1/accounts/{id}/bonuses` · `POST /v1/bonuses/grant` · `GET /v1/bonuses/{id}/wagering` | Bonus |
| `GET /v1/accounts/{id}/loyalty` · `POST /v1/loyalty/redeem` | Loyalty |
| `GET /v1/accounts/{id}/referrals` · `POST /v1/referrals/link` | Referral |
| `GET|POST /v1/subscriptions` | Subscription |
| `GET /v1/accounts/{id}/risk` · `POST /v1/accounts/{id}/freeze` | Risk Management |

**Account freeze is the exception.** Freezing is core — `accounts.status` is enforced locally in every
deployment (ADR-006), and a fraud lock cannot depend on a sellable module. `PATCH /v1/accounts/{id}`
with `status: SUSPENDED` always works. The Risk Management endpoints add *case management* on top:
investigation state, reviewer, resolution. The capability is core; the workflow is sold.

### 5. Composition and the specification

**Absent modules mean absent endpoints — `404`, not `501`.** A `501 Not Implemented` implies a feature
that exists and is unfinished. A module the operator did not buy is not unfinished; it is not part of
their deployment. `404` with `code: "ENDPOINT_NOT_AVAILABLE"` and a pointer to `/v1/capabilities` is
the honest answer.

**The OpenAPI spec is composed, not conditioned.** Rather than one spec with `x-module` annotations
that every integrator must filter mentally, the spec is assembled from fragments:

```
api/
  core.v1.yaml
  modules/bonus.v1.yaml
  modules/loyalty.v1.yaml
  ...
```

The ADR-010 pipeline composes `core + <enabled modules>` per customer and publishes **that customer's
spec** alongside their deployment. Their generated SDK contains exactly the endpoints they have.

This resolves the tension with ADR-011: there is still one `v1` contract per fragment, versioned by the
same additive-only rule. Composition selects fragments; it never alters them. An operator who later
buys Bonus receives a superset — additive, not a new major.

**`GET /v1/capabilities`** makes this discoverable at runtime:

```json
{
  "api_version": "v1",
  "currency":    "EUR",
  "modules":     { "bonus": true, "loyalty": true, "referral": false,
                   "subscription": false, "risk_management": true },
  "features":    { "two_phase_transactions": true, "partial_capture": true }
}
```

An operator integrating against several deployments — a group with multiple brands — reads this rather
than maintaining per-brand configuration.

### 6. Money and buckets in responses

```json
{
  "posted":       "150.00",
  "held":         "20.00",
  "available":    "130.00",
  "withdrawable": "80.00",
  "currency":     "EUR",
  "buckets": [
    { "bucket_id": "bkt_1", "type": "CASH",  "available": "80.00",
      "withdrawable": true },
    { "bucket_id": "bkt_2", "type": "BONUS", "available": "50.00",
      "withdrawable": false, "wagering_remaining": "200.00",
      "expires_at": "2026-10-01T00:00:00Z" }
  ]
}
```

`available` and `withdrawable` are both present and both necessary: a user has €130 to bet and €80 to
cash out, and an operator that conflates them will show the wrong number in one of two places.
`DEBT` buckets (ADR-009) appear in `buckets[]` and contribute to neither total.

### 7. Errors

The catalogue is part of the contract (ADR-011). Initial entries:

| HTTP | `code` | Meaning |
|---|---|---|
| 400 | `VALIDATION_FAILED` | Malformed request |
| 401 | `UNAUTHENTICATED` | Missing or invalid token |
| 403 | `ACCOUNT_SUSPENDED` | Account locked; the caller may not act for it |
| 404 | `ACCOUNT_NOT_FOUND` / `ENDPOINT_NOT_AVAILABLE` | Unknown account / module not deployed |
| 409 | `INSUFFICIENT_FUNDS` | Available balance below requested (ADR-004) |
| 409 | `IDEMPOTENCY_KEY_REUSED` | Same key, different payload |
| 409 | `TRANSACTION_NOT_CAPTURABLE` | Already captured, voided or expired |
| 422 | `CURRENCY_MISMATCH` | Currency differs from the deployment's (ADR-004) |
| 422 | `LIMIT_EXCEEDED` | Blocked by a configured limit |
| 429 | `RATE_LIMITED` | With `Retry-After` |
| 503 | `TEMPORARILY_UNAVAILABLE` | Dependency degraded; safe to retry with the same key |

`INSUFFICIENT_FUNDS` returns the available amount in `detail` but **never a full balance breakdown** on
a failed write — balance is a separate, separately-authorized read.

### 8. Rate limiting and quotas

Per API client, returned on every response as `RateLimit-Limit`, `RateLimit-Remaining`,
`RateLimit-Reset`. Read and write budgets are separate, because a reporting job polling balances must
not exhaust the budget that places bets.

## Consequences

### Positive Consequences

✅ **The product has a defined surface** — the largest documentation gap in the repository is closed
✅ **Two-phase transactions fit the domain** — the reservation model already built is exactly what pending bets need, exposed rather than reimplemented
✅ **The bonus platform is visible to integrators** — `allocations` shows the bucket split, so the flagship module is usable rather than merely present
✅ **Composition is honest** — absent modules are absent, and each customer's spec and SDK match their contract
✅ **`available` and `withdrawable` are distinct**, so operators cannot accidentally offer a cash-out of bonus funds
✅ **Resource design serves integrators**, not our service boundaries
✅ **Runtime discovery** via `/v1/capabilities` for multi-brand groups

### Negative Consequences

⚠️ **Per-customer specs and SDKs multiply artifacts.** Ten customers with different module sets means
ten specs and ten SDK builds. Generated by ADR-010's pipeline, but it is real build time and real
storage.

⚠️ **Two-phase transactions double the state an integrator must handle.** Authorize-then-capture is
more to get right than a single call, including the expiry case where neither happens.

⚠️ **`external_ref` creates a second identity space.** Immutable and unique, but operators will ask to
change it, and the answer has to be no.

⚠️ **Exposing bucket allocations couples integrators to the bucket model.** Adding a bucket type
becomes visible in responses — additive, but visible, and `DEBT` will surprise someone.

⚠️ **The surface is large** for a first version, and every endpoint is a permanent 12-month commitment
under ADR-011's deprecation policy.

### Risks

| Risk | Impact | Mitigation |
|------|--------|-----------|
| Integrator treats `available` as withdrawable | Users offered cash-outs that fail | Distinct field names, no single "balance" field, prominent in the integration guide, and the SDK exposes no combined accessor |
| Authorizations never captured or voided | Funds held until expiry; users see money missing | `expires_at` in the response; expiry releases automatically (ADR-004); held-ratio alerting |
| Idempotency key reused with a different payload | Silent wrong result | Key plus payload hash stored; a mismatch returns `409 IDEMPOTENCY_KEY_REUSED` rather than the cached response |
| Operator polls balance instead of consuming webhooks | Load, and stale reads | Separate read rate budget; webhooks documented as the primary integration path |
| Module bought later, integrator's SDK is stale | New endpoints unreachable | `/v1/capabilities` is authoritative at runtime; a new spec and SDK publish with the module's enablement |
| `metadata` used as a database | Unbounded storage, PII arriving unannounced | Size-capped, string values only, documented as opaque and never indexed |

## Alternatives Considered

### Alternative 1: One `/v1/transactions` resource for every money movement

**Description**: A single endpoint with a `type` discriminator covering deposits, withdrawals and spends.

**Pros**: Mirrors the internal model exactly (ADR-009: all are transaction types); fewer endpoints; one
lifecycle to document.

**Cons**: Deposits need `redirect_url` and PSP state; withdrawals need approval workflow and
cancellation; wagers need capture and void. A single schema becomes a union where most fields are
conditionally present, which every generated SDK renders as an unpleasant type and every integrator
mis-handles at least once.

**Why rejected**: It exports our internal convenience as the integrator's problem.

---

### Alternative 2: Hide reservations; single-phase transactions only

**Description**: Every transaction settles immediately. Operators model pending bets themselves.

**Pros**: A much simpler API; one call, one outcome; no expiry semantics to explain.

**Cons**: Every sportsbook operator would rebuild holds in their own system, badly — tracking which
stakes are committed against a balance that Vaullet believes is free. Two systems would disagree about
available funds, and the disagreement would surface as overdrafts.

**Why rejected**: The capability exists and the domain needs it. Withholding it would push a
correctness problem to the integrator, which is the opposite of what wallet infrastructure is for.

---

### Alternative 3: One spec with all modules; undeployed endpoints return 501

**Description**: Publish the full surface everywhere; unavailable modules answer `501`.

**Pros**: One spec, one SDK, one document; a purchased module works without redeploying anything.

**Cons**: The published contract would describe endpoints a customer cannot use, and their generated
SDK would offer methods that always fail. `501` also misrepresents the situation: nothing is
unimplemented.

**Why rejected**: A contract should describe what the customer has. This is also the composition model
ADR-005 already chose — undeployed means absent, at every layer including the API.

---

### Alternative 4: GraphQL

**Description**: A single graph endpoint instead of REST resources.

**Pros**: Clients fetch exactly the fields they need; no over-fetching across the account/balance/bonus
graph; one endpoint to version.

**Cons**: Idempotency, rate limiting per operation, and cache semantics all need rebuilding, and
ADR-011's versioning and OpenAPI decisions assume REST. Financial mutations benefit from the
explicitness REST enforces. Server-to-server integrators with fixed queries gain little from field
selection.

**Why rejected**: Its strengths address problems this API does not have, and adopting it would reopen
ADR-011.

---

### Alternative 5: Operator-supplied account identifiers as the primary key

**Description**: Address accounts by the operator's own user id throughout; no Vaullet `account_id`.

**Pros**: Nothing for the operator to store or map; simpler integration.

**Cons**: ADR-006 established why the ledger needs an identifier it controls — `ledger_entries` is
immutable for seven years and cannot key on a foreign, mutable identifier. Making the operator's id
primary would put that identifier into every financial record.

**Why rejected**: Same reasoning as ADR-006's account anchor. `external_ref` gives the convenience
without the coupling.

## Implementation Notes

### Spec fragment composition

`core.v1.yaml` plus one fragment per module, composed at build time by ADR-010's pipeline into each
customer's spec. Fragments share components via `$ref` into `common.v1.yaml`, so `Money`, `Bucket`,
`Problem` and `Cursor` are defined once.

**The fragments are generated, not written.** ADR-011 §5 configures springdoc with one `GroupedOpenApi`
per module, selected by controller package, so each fragment is a build output that is committed and
diffed like any other generated spec — and `common.v1.yaml` is emitted once from the `contracts-common`
types. Hand-authoring fragments alongside code-first generation would have produced two sources of
truth for the same endpoints; instead the module boundary lives in the package structure, enforced by
an architecture test.

The breaking-change gate (ADR-011) runs per fragment: composition cannot introduce a breaking change
that fragment-level checks would miss, because composition only selects.

### Idempotency semantics

The key is stored with a hash of the request body and the response, for 24 hours:

- Same key, same payload, completed → return the original response
- Same key, same payload, in flight → `409 REQUEST_IN_PROGRESS`
- Same key, different payload → `409 IDEMPOTENCY_KEY_REUSED`

Returning a cached response for a *different* payload is the dangerous failure, which is why the hash
is compared rather than the key alone.

### Webhook subscriptions

```
POST   /v1/webhooks/subscriptions   { url, events[], version }
GET    /v1/webhooks/subscriptions
DELETE /v1/webhooks/subscriptions/{id}
POST   /v1/webhooks/subscriptions/{id}/rotate-secret
```

Events are the ADR-007 names an operator cares about: `deposit.captured`, `withdrawal.confirmed`,
`chargeback.received`, `rewards.value-granted`, `transaction.completed`, `transaction.expired`.

Webhook event names drop the `.v1` suffix carried by the internal topic: the payload version is
negotiated per subscription (`version` on the subscription, §9 of ADR-011), so encoding it twice would
let the two disagree.

### Sandbox

Every deployment exposes a sandbox with the same surface, a PSP test adapter, and a way to force
outcomes (`X-Vaullet-Test-Scenario: capture_declined`). Chargebacks and payout failures are the paths
integrators most need to test and can least easily produce.

## References

- [ADR-004: Atomic Balance Reservations](004-atomic-balance-reservations.md) — buckets, holds, and the two-phase model this exposes
- [ADR-005: Module Composition](005-module-composition-and-deployment-topology.md) — why the surface varies per contract
- [ADR-006: Authentication and Identity](006-authentication-and-identity.md) — the account anchor; why `external_ref` is secondary
- [ADR-009: Payment Rails](009-payment-rails-deposits-and-withdrawals.md) — deposit and withdrawal semantics
- [ADR-010: Polyrepo CI/CD](010-polyrepo-cicd-and-gitops-delivery.md) — where per-customer specs and SDKs are composed
- [ADR-011: API Versioning and OpenAPI](011-api-versioning-and-openapi.md) — versioning, errors, pagination, idempotency

---

**Date**: 2026-09-01
**Author**: Predrag
**Reviewers**: TBD
