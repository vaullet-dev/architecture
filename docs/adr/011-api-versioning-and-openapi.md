# ADR-011: API Versioning and OpenAPI

## Status

**Accepted** (2026-09-01)
**Revised** (2026-09-02) — generation is per module fragment, reconciling code-first with ADR-012's
composed specification

## Context

Vaullet exposes three distinct API surfaces, and they have been treated as one:

| Surface | Consumer | Can we force an upgrade? |
|---|---|---|
| **External** | The operator's backend integrating the wallet | **No.** They ship on their own schedule under a contract |
| **Internal** | Service-to-service REST (Transaction → Vaullet, → Fraud, → Limits) | Effectively yes — one umbrella chart per cluster (ADR-010) |
| **Outbound webhooks** | Vaullet → the operator's endpoint | No, and we do not even control the receiver |

Endpoints already specified in ADRs 004 and 009 use `/v1/` paths, so path versioning is de facto in
place. It has never been decided, and no rule says what may change inside a `v1`.

### The gap nobody has written down

**The external API does not exist in any document.** ADR-004 defines Vaullet's internal reserve
endpoint; ADR-009 defines the PSP adapter port. Neither is the surface an operator integrates against
— the thing actually being sold. `arch.md`'s first open item is still "define detailed API contracts."

### The constraint from ADR-002

ADR-002 rejected OpenAPI as the source of truth for DTOs, keeping Java as the source and generating
TypeScript. Any OpenAPI decision here must be **code-first**, or it reopens a settled decision.

## Decision

### 1. Two regimes, because the surfaces differ

**External APIs and webhooks are versioned for the long term. Internal APIs are versioned for one
release.**

Treating them alike is the common mistake in either direction: either internal calls carry
compatibility machinery nobody needs, or external contracts get broken with the casualness appropriate
to an internal refactor.

### 2. External: major version in the URI path

```
https://api.<operator>.vaullet/v1/accounts/{id}/balance
```

Not headers, not query parameters, not content negotiation. The path is greppable in logs, trivially
testable with `curl`, cacheable, unambiguous in a support conversation, and already in use across
ADRs 004 and 009.

**Additive-only within a major version.** The rule is identical to ADR-007's for event schemas, and
deliberately so — one rule for the whole architecture:

| Change | Allowed in `v1`? |
|---|---|
| New optional request field | ✅ |
| New response field | ✅ |
| New endpoint | ✅ |
| New optional query parameter | ✅ |
| New enum value in a **request** | ✅ |
| New enum value in a **response** | ⚠️ Only if clients were told to tolerate unknowns |
| Removing or renaming a field | ❌ `v2` |
| Making an optional field required | ❌ `v2` |
| Narrowing a type or changing units | ❌ `v2` |
| Changing an HTTP status or error code | ❌ `v2` |

The response-enum row is the one that catches people. Adding `WITHDRAWAL_PENDING_REVIEW` to a status
enum breaks any integrator with an exhaustive `switch`. It is permitted only because the integration
guide requires clients to treat unknown enum values as a documented fallback — and that requirement
must exist from `v1.0`, not be added later.

### 3. Deprecation: 12 months, announced in-band

When `v2` ships, `v1` continues for **at least 12 months**, and every `v1` response carries:

```http
Deprecation: @1788220800
Sunset: Wed, 01 Sep 2027 00:00:00 GMT
Link: <https://docs.vaullet.dev/migrate/v1-to-v2>; rel="deprecation"
```

Per RFC 9745 (`Deprecation`) and RFC 8594 (`Sunset`). Machine-readable, so an integrator's monitoring
can catch it without anyone reading an email.

Twelve months because these are contracted B2B integrations. An operator's roadmap is not ours, and a
shorter window makes the platform's release schedule their problem.

**Usage is tracked per client per version.** A version is not retired on a date; it is retired when
the date has passed *and* traffic is zero. Sunsetting an endpoint a paying customer still calls is a
worse outcome than carrying it another quarter.

### 4. Internal: compatible with N-1, and no further

Internal service APIs keep `/v1/` paths for consistency but carry **one guarantee only: compatibility
with the immediately previous release.**

That guarantee is not optional and it is not larger than it needs to be. It exists because a rolling
update runs old and new pods simultaneously — during a deploy, Transaction Service `2.4` may call
Vaullet `2.3`. Without N-1 compatibility every deploy is a brief outage. With more than N-1, services
accumulate compatibility code for versions that no cluster runs, since ADR-010 ships one umbrella
chart per customer and versions do not drift far.

Breaking an internal API is therefore a two-release operation: add the new form in release N, remove
the old in N+1. The same expand–contract discipline ADR-010 requires of database migrations.

### 5. OpenAPI: generated from code, per module fragment, committed as an artifact, diffed in CI

Code-first via springdoc-openapi, from annotated controllers and the ADR-002 shared-contract DTOs.
This honours ADR-002 rather than reopening it.

**Generation is per fragment, not per service.** [ADR-012](012-external-api-surface.md) composes each
customer's specification from `core.v1.yaml` plus one fragment per module they bought. Those two facts
only fit together if the *generator* emits fragments, so it does: springdoc is configured with one
`GroupedOpenApi` per module, selected by the controller package.

```java
@Bean GroupedOpenApi core()  { return GroupedOpenApi.builder()
        .group("core").packagesToScan("com.wallet.api.core").build(); }
@Bean GroupedOpenApi bonus() { return GroupedOpenApi.builder()
        .group("bonus").packagesToScan("com.wallet.api.bonus").build(); }
```

Each group generates one file — `api/core.v1.yaml`, `api/modules/bonus.v1.yaml` — and each is
committed and diffed independently by the gate below. Shared schema objects (`Money`, `Bucket`,
`Problem`, `Cursor`) come from `contracts-common` and are emitted once into `api/common.v1.yaml`, which
the fragments `$ref` rather than inlining; springdoc is pointed at it so the same Java type never
produces two definitions.

Nothing is hand-authored, and nothing is composed at generation time. Composition is a build step in
the ADR-010 pipeline that selects fragments per customer — it never rewrites one, which is why a
fragment-level compatibility check is sufficient for the composed result.

**The package boundary is therefore load-bearing**: a controller in the wrong package puts an endpoint
in the wrong fragment and ships a module to a customer who did not buy it. An architecture test asserts
that every controller lives under the package matching its module, and that a core controller
references no module type.

Code-first has one well-known failure: the public contract changes silently when someone edits a
class. The fix is to stop treating the spec as a build output:

```
build → generate openapi.yaml → diff against committed spec
                                      │
              ┌───────────────────────┴───────────────────────┐
              │                                               │
       no difference                              difference detected
              │                                               │
         build passes                    additive → commit the update, pass
                                         breaking → FAIL unless the path major changed
```

`oasdiff` (or `openapi-diff`) classifies the change. **A breaking change to a published version fails
the build**, exactly as ADR-007's `FULL_TRANSITIVE` registry rejects an incompatible event schema. The
committed spec makes every API change a reviewable line in a pull request rather than an emergent
property of a refactor.

The spec is generated but **not** authoritative-by-accident: it is a checked-in artifact with the same
review discipline as code.

### 6. Published spec and generated SDKs

Each external major version publishes a specification to the docs site and to GHCR as an OCI artifact.
Two things are published, and the distinction matters:

- **The full `openapi-v1.yaml`** — every fragment composed together. This is the reference
  documentation, and what a prospective integrator reads.
- **Per-customer `openapi-v1.yaml`** — `core` plus the modules in that customer's contract, composed by
  the ADR-010 pipeline and published alongside their deployment (ADR-012 §5). Their generated SDK
  contains exactly the endpoints they have, and no method that would return `404`.

From those, CI generates and publishes client SDKs — TypeScript first, since ADR-002 already commits to
TypeScript for frontends.

For a B2B product this is not a nicety. "Here is your typed SDK" materially shortens an operator's
integration, and integration time is a sales consideration.

### 7. Errors: RFC 9457 Problem Details with a stable code catalogue

```json
{
  "type":     "https://docs.vaullet.dev/errors/insufficient-funds",
  "title":    "Insufficient funds",
  "status":   409,
  "detail":   "Available balance 40.00 EUR is less than requested 60.00 EUR",
  "instance": "/v1/withdrawals/wd_01J8...",
  "code":     "INSUFFICIENT_FUNDS",
  "trace_id": "cor_01J8..."
}
```

**`code` is the contract; `detail` is for humans.** Codes are stable within a major version and
enumerated in the spec — adding one is additive, changing the meaning of one is breaking. `INSUFFICIENT_FUNDS`
(ADR-004) and `CURRENCY_MISMATCH` (ADR-004) are already in use and become catalogue entries.

`trace_id` is ADR-007's `correlation_id`, so a customer support ticket quoting an error is traceable
straight through the event stream.

### 8. Platform-wide API conventions

**Idempotency.** Every state-changing `POST` on the external API requires an `Idempotency-Key` header;
a replay returns the original response, never a second effect. Already required by ADR-004
(reservations) and ADR-009 (PSP callbacks) — this makes it a platform rule rather than three local ones.

**Cursor pagination**, never offset:

```
GET /v1/accounts/{id}/transactions?limit=50&cursor=eyJ0cyI6...
→ { "data": [...], "next_cursor": "eyJ0cyI6..." }
```

Offset pagination on an append-only, high-volume table (`ledger_entries`, ADR-004) both skips and
duplicates rows as new entries arrive mid-pagination, and degrades badly at depth. Cursors are stable
under concurrent writes.

**Money is never a float.** Decimal string plus explicit currency — `{"amount": "40.00", "currency": "EUR"}`
— matching `NUMERIC(20,4)` and `ledger_config` from ADR-004. A JSON number for money is a rounding
defect waiting for a language with binary floats on the other end.

**Timestamps** are RFC 3339 UTC with explicit offset.

### 9. Outbound webhooks are a versioned, signed API

Operators need to know when a withdrawal completed or a chargeback arrived. Webhooks are as much a
public contract as the REST surface, and are usually treated as less:

- **Versioned per subscription** — the operator pins `v1`; a new payload shape is `v2` and they migrate
- **Signed** — HMAC over the raw body with a per-client secret, plus a timestamp to bound replay
- **At-least-once with exponential backoff** — the receiver must be idempotent on `event_id`, which is
  the same envelope ADR-007 defines internally
- **Ordering is not guaranteed.** Each payload carries `occurred_at` and `event_id`; a receiver that
  assumes order will eventually be wrong

Payload shapes are the ADR-007 envelope, so an operator sees the same event vocabulary the platform
uses internally.

## Consequences

### Positive Consequences

✅ **One rule across the architecture** — additive within a major, new major for breaking, applied to REST, events and webhooks alike
✅ **Breaking changes cannot ship silently** — `oasdiff` in CI fails the build, matching how ADR-007 gates event schemas
✅ **The spec is reviewable** — an API change is a diff in a pull request, not a discovery after release
✅ **ADR-002 stays intact** — Java remains the source of truth; OpenAPI is generated
✅ **Integrators get typed SDKs** and machine-readable deprecation
✅ **Internal APIs carry only the guarantee a rolling update needs**, and no compatibility code beyond it
✅ **Errors are machine-actionable** and traceable into the event stream via `correlation_id`

### Negative Consequences

⚠️ **Twelve months of `v1` support after `v2` ships.** Two implementations of every changed endpoint,
or a translation layer, for a year. This is the real cost of the decision and it recurs with every
major version.

⚠️ **Committed specs create merge conflicts.** Two branches touching adjacent endpoints will both
regenerate the file. Mechanical to resolve, and irritating.

⚠️ **Code-first constrains API design.** Some shapes are awkward to express through annotations, and
there is a standing pull toward exposing internal model structure — the reason many teams go
design-first.

⚠️ **N-1 internal compatibility is a discipline, not a mechanism.** Nothing structural prevents someone
breaking an internal API in one release; only the contract test catches it.

⚠️ **Webhook delivery is an operational burden** — retries, dead letters, and per-customer endpoint
failures become support load.

### Risks

| Risk | Impact | Mitigation |
|------|--------|-----------|
| Response enum extended, integrator's exhaustive switch breaks | Customer-visible failure from an "additive" change | Unknown-value tolerance mandated in the integration guide from v1.0 and asserted in the SDK's own tests |
| `v1` sunset while a customer still calls it | Broken integration for a paying customer | Per-client per-version usage tracked; retirement requires the date *and* zero traffic |
| Committed spec drifts from the running service | The published contract is a lie | Spec regenerated and diffed on every build; drift fails CI |
| Internal API broken without the N-1 step | Rolling update causes a brief outage | Contract tests run the previous release's client against the new server in the ADR-010 pipeline |
| Webhook secret leaked | Forged events accepted by the operator | Per-client secrets, rotatable without redeploy; timestamp bounds replay |
| SDK generated from a stale spec | Customers integrate against something that no longer exists | SDKs generated in the same job that publishes the spec, from that artifact |

## Alternatives Considered

### Alternative 1: Header or media-type versioning

**Description**: `Accept: application/vnd.vaullet.v1+json`, URLs unversioned.

**Pros**: Purist REST — one resource, one URI; version is metadata rather than identity.

**Cons**: Invisible in logs and dashboards; awkward to test by hand; a missing header means an implicit
default, which is the worst failure mode available; caching requires `Vary` discipline that
intermediaries get wrong.

**Why rejected**: For a B2B API whose consumers are integrating under contract, being obvious beats
being correct-by-REST-doctrine. It is also already contradicted by `/v1/` paths in ADRs 004 and 009.

---

### Alternative 2: Date-based versioning (the Stripe model)

**Description**: Clients pin `Vaullet-Version: 2026-09-01`; the server transforms responses between
dated versions.

**Pros**: Genuinely excellent for a fast-evolving API. Many small breaking changes become tractable,
each client upgrades independently, and no `v2` cliff ever arrives. The best-in-class answer for a
platform at Stripe's scale.

**Cons**: Requires a version-transformation layer for every breaking change, forever, plus test
coverage for every dated version against every endpoint. The maintenance cost is proportional to
change count, not to major versions.

**Why rejected — with reservations**: The right answer for an API changing weekly with thousands of
integrators. Vaullet has a handful of contracted operators and a domain that changes at the speed of
banking. **Worth revisiting if the integrator count grows into the hundreds**, and the path-version
scheme does not preclude it.

---

### Alternative 3: Design-first OpenAPI as the source of truth

**Description**: Author OpenAPI YAML; generate Java servers and clients.

**Pros**: Language-agnostic contract; API design gets deliberate attention; parallel client and server
development; no risk of leaking internal model shapes.

**Cons**: Directly reverses ADR-002, which chose Java DTOs as the source of truth after weighing this
exact trade. Generators produce boilerplate, and YAML expresses validation less richly than Java.

**Why rejected**: Reopening a settled decision would need a stronger reason than API-design hygiene,
and the CI diff gate recovers most of design-first's real benefit — a contract that cannot change
without review.

---

### Alternative 4: No external versioning; break with notice

**Description**: One unversioned API; announce breaking changes and coordinate migration.

**Pros**: No duplicate implementations; smallest surface; nothing to maintain.

**Cons**: Every breaking change becomes a synchronised migration across all integrators — the same
flag-day model ADR-007 rejected for events, for the same reason: consumers do not deploy on our
schedule.

**Why rejected**: Consistency with ADR-007, and it makes the platform's release cadence its customers'
problem.

## Implementation Notes

### Generation and gate

```yaml
# in wallet-infrastructure/.github/workflows/service-ci.yml
- name: Generate OpenAPI fragments
  run: ./mvnw springdoc-openapi:generate     # one file per GroupedOpenApi

- name: Detect breaking changes, per fragment
  run: |
    for f in api/common.v1.yaml api/core.v1.yaml api/modules/*.v1.yaml; do
      oasdiff breaking \
        --base     "$f" \
        --revision "target/openapi/$(basename "$f")" \
        --fail-on ERR
    done

- name: Fail on uncommitted spec drift
  run: |
    cp target/openapi/*.yaml api/ && cp target/openapi/modules/*.yaml api/modules/
    git diff --exit-code api/ \
      || { echo "OpenAPI fragment changed — commit the regenerated file"; exit 1; }
```

The second step fails on breaking changes; the third fails when the committed spec is stale. Both are
required, and neither substitutes for the other.

Running the check per fragment is what makes ADR-012's composition safe: composition only *selects*
fragments, so a composed specification cannot contain a breaking change that no fragment contains.

### External API surface

This ADR settles *how* the external API is versioned and specified. **What it contains is settled by
[ADR-012](012-external-api-surface.md)** — core resources, module-gated resources, the two-phase
transaction flow, and how composition interacts with the specification. The sketch that stood here has
been superseded by that decision and removed rather than left to drift.

### Contract testing for N-1

The ADR-010 pipeline runs the previous release's generated client against the new server. A failure
means the N-1 guarantee is broken and a rolling update would drop requests.

### Versioning the webhook payloads

Webhook subscriptions carry a version pinned per client. A new payload shape publishes as `v2`;
existing subscriptions stay on `v1` until the operator migrates, under the same 12-month policy.

## References

- [ADR-002: Shared Contracts](002-shared-contracts-versioning-strategy.md) — Java as source of truth, which makes this code-first
- [ADR-004: Atomic Balance Reservations](004-atomic-balance-reservations.md) — `/v1/reservations`, `INSUFFICIENT_FUNDS`, `CURRENCY_MISMATCH`
- [ADR-007: Kafka Topics](007-kafka-topics-and-event-schema-evolution.md) — the additive-within-major rule this mirrors; the envelope webhooks reuse
- [ADR-009: Payment Rails](009-payment-rails-deposits-and-withdrawals.md) — deposit and withdrawal endpoints
- [ADR-010: Polyrepo CI/CD](010-polyrepo-cicd-and-gitops-delivery.md) — where the spec gate and contract tests run
- [RFC 9457: Problem Details for HTTP APIs](https://datatracker.ietf.org/doc/html/rfc9457)
- [RFC 8594: The Sunset HTTP Header Field](https://datatracker.ietf.org/doc/html/rfc8594)
- [RFC 9745: The Deprecation HTTP Header Field](https://datatracker.ietf.org/doc/html/rfc9745)

---

**Date**: 2026-09-01
**Author**: Predrag
**Reviewers**: TBD
