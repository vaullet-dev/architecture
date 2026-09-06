# ADR-013: `backend-common` — Shared Runtime Plumbing, Split by Dependency Footprint

## Status

**Accepted** (2026-09-04)

## Context

The ledger service was migrated onto the `@vaullet-io` template shape on 2026-09-04. Doing so made
something visible that a single service could hide: of the 2,763 lines of Java in
`wallet-ledger-service`, 766 across fourteen files were not about the ledger at all. They are the
error-response shape, the security chain, the CORS policy, the OpenAPI metadata, the Testcontainers
wiring and the ArchUnit rules — and every line of them already existed, near-identically, in
`predrag-arsic/spring-boot-template`.

A diff of the two copies is the argument. `ResourceNotFoundException` is byte-identical.
`SecurityConfig`, `LocalSecurityConfig`, `MethodSecurityConfig`, `WebMvcConfig`, `OpenApiConfig`,
`ApiExceptionHandler`, `ApplicationProperties`, `IntegrationTest`, `TestcontainersConfiguration` and
`TestSecurityConfiguration` differ only in the package name, the scopes named in one string literal,
and Javadoc that has been reworded rather than changed in meaning.

Two copies is a smell. [ADR-005](005-module-composition-and-deployment-topology.md) plans fourteen
services, and `arch.md` names them; fourteen copies is a different kind of problem.

### What copying actually costs here

Not keystrokes. The failures are specific:

| Copied thing | How the copy goes wrong | How you find out |
|---|---|---|
| JWT claim mapping | The ledger's copy read roles from a top-level `roles` claim. [ADR-006](006-authentication-and-identity.md) has Keycloak emit them under `realm_access.roles` | You do not. `hasRole('FINANCE')` silently never matches, and an endpoint that looks locked down denies everyone |
| Error body shape | One service adds `trace_id`, another does not; a third names it `correlationId` | An integrator writes per-service error parsing, or [ADR-011](011-api-versioning-and-openapi.md) §7's "`code` is the contract" stops being true |
| `@IntegrationTest` | One service adds a `@TestPropertySource`, doubling its Spring context count | A slow build nobody attributes to it |
| Layering rules | A service loosens one rule locally rather than arguing for the change | The platform boundary quietly becomes per-service preference |

The claim-mapping row is the one that settles it: that bug shipped, in the service that guards the
money, and it is invisible to every test that does not specifically look for it. A defect class that
is silent and per-copy is a defect class you remove by having one copy.

### The constraint from ADR-002

[ADR-002](002-shared-contracts-versioning-strategy.md) already rejected exactly this shape — its
Alternative 5, "one artifact for every domain", was rejected because a single fat jar consumed by all
fourteen services rebuilds the build-graph hub that polyrepo and ADR-005 exist to remove:

> Every service upgrades through the same version stream; an unrelated change is still a version bump
> the whole fleet eventually takes. […] The version number stops carrying information: `2.9.0` says
> something changed somewhere.

Any shared-library decision here has to answer that, or it reopens a settled one.

## Decision

We will create **`vaullet-io/backend-common`**: one repository, publishing **four artifacts split by
dependency footprint**, plus a BOM.

### 1. It shares plumbing, never contracts

`backend-common` and `wallet-shared-contracts` (ADR-002) are disjoint by construction:

| | `wallet-shared-contracts` | `backend-common` |
|---|---|---|
| Contains | DTOs, event schemas — **what goes on the wire** | Error handling, security, test scaffolding — **how a service is built** |
| Changes when | A contract changes | Spring changes, or a platform-wide practice does |
| Breaks | Producers and consumers of that domain | Nothing on the wire; a compile error in a service |
| Split by | Business domain | Dependency footprint |

The one place they touch is the RFC 9457 problem shape, and the split is clean:
`contracts-common`'s `Problem` is the type a **client** deserialises; `backend-common-web` produces
that body **server-side**. Neither generates the other.

### 2. Four artifacts, split by what they drag onto a classpath

ADR-002's objection is answered by applying ADR-002's own remedy — split the published unit, keep one
repository. The axis differs because the coupling differs: contracts couple by domain, plumbing
couples by dependency.

| Artifact | Contains | Drags in | Who skips it |
|---|---|---|---|
| `backend-common-core` | `ErrorType`, `ApplicationException` and general-purpose subclasses | `spring-web` (for `HttpStatus`), JSpecify | Nobody |
| `backend-common-web` | problem+json advice, `ProblemDetails`, CORS, OpenAPI metadata, `platform-defaults.yaml` | `spring-boot-starter-webmvc`, validation, micrometer-tracing | A worker with no HTTP surface |
| `backend-common-security` | JWT resource-server chain, Keycloak claim mapping, method security | `spring-boot-starter-security`, oauth2-resource-server | A service behind the mesh with no application-level authn |
| `backend-common-test` | `@IntegrationTest`, PostgreSQL container, `JwtDecoder` stand-in, ArchUnit rules | Testcontainers, Boot test starters, ArchUnit | Nobody, and it is `test`-scoped |
| `backend-common-bom` | Dependency management for the four | — | — |

**`core` carries the weight of the design.** A service layer throws `ApplicationException` subclasses
and stays free of the web stack, which is what keeps the same rules callable from the Kafka listener
that ADR-004's revised flow settles money from. A fat jar would put `spring-webmvc` on that
listener's classpath and the distinction would erode in a month.

### 3. Domain error codes stay in the service that owns them

`ErrorType` is an **interface**, not the enum it was. `backend-common-core` ships `CommonErrorType`
(validation, not-found, conflict, busy, upstream, denied, internal); each service declares its own
enum for its own vocabulary.

The test for membership is **not** "does this sound platform-wide" but *"would a service that knows
nothing about money still raise it"*. `INSUFFICIENT_FUNDS` and `CURRENCY_MISMATCH` fail it and stay
in the ledger, even though ADR-011 §7 names both as catalogue entries — a code used by exactly one
service is that service's vocabulary. They graduate the day a second service raises them.

Without this inversion, every new domain error in any of fourteen services is a pull request against
`backend-common` and a version bump the whole fleet inherits. That is ADR-002's hub, rebuilt one
enum constant at a time.

### 4. Consumed as auto-configuration, overridden by declaring a bean

Adding the dependency is the whole installation step. Every bean is `@ConditionalOnMissingBean`, so
overriding one is declaring your own — an auto-configuration is a default, never a constraint. The
ledger uses exactly one override: `LedgerApiExceptionHandler` renames the lock-contention code back
to its published `ACCOUNT_BUSY`, in ten lines, because renaming it would be a breaking change under
ADR-011 §4.

### 5. Shared `application.yaml` defaults are imported, not injected

The platform's Boot configuration — virtual threads, snake_case JSON, problem+json for framework
errors, Actuator exposure, tracing sampling, log correlation — ships as a classpath resource a
service imports:

```yaml
spring:
  config:
    import: "classpath:vaullet/platform-defaults.yaml"
```

**Explicitly, not through an auto-configured property source.** An imported document ranks below the
file importing it, so local settings still win; and one greppable line beats a reader of
`application.yaml` seeing no mention of the Jackson naming strategy and reasonably concluding nothing
sets it.

### 6. Versioning and publishing

Semantic versioning, one version across the four modules — they share the `ErrorType` interface and
the `vaullet.*` property prefix, so a mixed combination is not supported and the BOM makes it hard to
assemble by accident. Published to GHCR Maven from a tag, as ADR-002 §"Publishing Workflow" does for
contracts.

Unlike a contract artifact, an upgrade here is **never forced**: nothing about `backend-common` is on
the wire, so a service on `1.2.0` and a service on `1.4.0` interoperate exactly as before. That is
the property that makes the version stream tolerable.

## Consequences

### Positive Consequences

✅ **The claim-mapping bug is fixed once, for everyone** — and covered by a test that asserts the
nested `realm_access.roles` path ADR-006 specifies
✅ **The error contract is testable as a contract** — `ApiExceptionHandlerTest` asserts the ADR-011 §7
body shape at the wire, in the library; no service's own suite fails when that shape drifts
✅ **766 lines across fourteen files leave the ledger** (net −658 after what replaced them), and
will not be written again by the next thirteen services
✅ **The layering rules become a platform boundary** rather than thirteen local copies that drift
✅ **A service's dependency list says what it is** — a worker declaring `core` only cannot
accidentally acquire the web stack
✅ **ADR-002 stays intact** — its remedy is applied, not its rejected alternative

### Negative Consequences

⚠️ **One more repository in the release path.** A change spanning the library and a service is two
pull requests and a version bump, and cannot be one atomic commit. This is the recurring cost, and
it is the same one ADR-002 and ADR-010 already accept for the polyrepo.

⚠️ **`mvn install` before a service build, until the first publish.** `0.1.0-SNAPSHOT` resolves from
the local repository only, so CI for a consuming service is red until `backend-common` is published
to GHCR. Tracked below.

⚠️ **Auto-configuration is action at a distance.** A bean nobody declared is a bean nobody greps for.
Mitigated by conditional-report visibility (`--debug`), by every bean being overridable, and by
keeping shared YAML an explicit import rather than an injected property source — but a developer
does need to know the library is there.

⚠️ **A shared library invites unshareable things into it.** The pressure to move "just this one"
domain type in is constant, and each one narrows who can use the module. §3's test is the defence,
and it needs enforcing in review.

### Risks

- **The library becomes the hub anyway**, one graduated error code at a time. *Mitigation*: §3's
  membership test, and the rule that a code graduates on its **second** consumer, never in
  anticipation of one.
- **A Spring Boot major version splits the fleet**, because the library must target one. *Mitigation*:
  the library depends on `spring-boot-starter-parent` for dependency management only and is never
  repackaged, so it is usable across patch and minor versions; a major version means a major version
  of the library, and services migrate on their own schedule.
- **Version skew between the four modules** produces a `NoSuchMethodError` rather than a clean
  failure. *Mitigation*: the BOM, and one version across all four.

## Alternatives Considered

### Alternative 1: Keep copying (status quo)

- **Pros**: Zero coordination cost; every service is self-contained and independently readable; no
  new repository, no publishing pipeline, no version skew.
- **Cons**: The claim-mapping bug, and every future member of that class — silent, per-copy, and
  invisible to tests. Fourteen copies of a security chain is fourteen chances to get
  deny-by-default wrong.
- **Why rejected**: the defects are silent and safety-relevant. Copying is defensible for code whose
  failure is loud.

### Alternative 2: One fat `backend-common` jar

- **Pros**: Simplest possible build and consumption — one dependency, one version, no module
  boundaries to get wrong.
- **Cons**: Puts `spring-webmvc` and `spring-security` on the classpath of a worker that has neither,
  which erodes exactly the service-layer/HTTP separation ADR-004's Kafka settlement path depends on.
  Rebuilds the build-graph hub ADR-002 rejected.
- **Why rejected**: ADR-002 already decided this, on the same reasoning. The split costs an aggregator
  POM.

### Alternative 3: A Maven parent POM instead of a library

- **Pros**: Shares dependency versions and plugin configuration with no code coupling at all; a
  service inherits and overrides freely.
- **Cons**: Shares no *behaviour*. The claim mapping, the error body and the ArchUnit rules are code,
  and a parent POM cannot carry them. Also makes every service's build depend on a POM that must be
  released before any of them can build.
- **Why rejected**: it solves the smaller half of the problem. The library still inherits from
  `spring-boot-starter-parent`, which is where dependency management belongs.

### Alternative 4: Put the plumbing in `wallet-shared-contracts`

- **Pros**: One shared repository rather than two; one publishing pipeline already designed in
  ADR-002.
- **Cons**: The two change for entirely unrelated reasons — a Spring upgrade would bump a *contract*
  version and force consumers to re-evaluate wire compatibility that did not change. It would also
  put `spring-boot-starter-security` on the classpath of every TypeScript-generating contract module.
- **Why rejected**: the version number would stop meaning anything, which is the failure mode ADR-002
  named.

### Alternative 5: A Spring Boot starter published per concern, with no shared core

- **Pros**: Maximum independence; a service takes only what it wants, and no module depends on
  another.
- **Cons**: `ApplicationException` and `ErrorType` are needed by the web module, the security module
  and the service layer alike. Without a shared core each starter carries its own copy of the error
  vocabulary, and a service using two of them has two incompatible `ErrorType` types.
- **Why rejected**: the core is genuinely shared; pretending otherwise reintroduces the duplication at
  the library level.

## Implementation Notes

### Adopted by

| Service | Status |
|---|---|
| `wallet-ledger-service` | ✅ Migrated 2026-09-04. Fourteen files and 766 lines removed; Java down 2,763 → 2,105. 20 unit/slice + 18 integration tests green — the 18 unchanged in count and in what they assert, the 20 being the previous 19 plus a regression test for the one deviation |
| `spring-boot-template` | ⬜ Pending. The template is the second copy the extraction was diffed against, and it should be the first consumer after the ledger — a template that does not use the library teaches every new service not to |

### Follow-ups

- [ ] Publish `0.1.0` to GHCR Maven and switch consumers off `-SNAPSHOT`. Until then a consuming CI
      run needs `backend-common` built first.
- [ ] Migrate `spring-boot-template` onto the library and delete its copies.
- [ ] Decide where `PageResponse` belongs. It is in the template and not in the ledger, and ADR-011 §8
      mandates **cursor** pagination for external APIs — an offset envelope may be the wrong shape to
      standardise on at all.

## References

- [ADR-002](002-shared-contracts-versioning-strategy.md) — per-domain artifacts; the reasoning this
  ADR applies on a different axis
- [ADR-005](005-module-composition-and-deployment-topology.md) — why a build-graph hub matters
- [ADR-006](006-authentication-and-identity.md) — the Keycloak claim set the security module maps
- [ADR-008](008-service-to-service-authentication.md) — the scopes on the money path
- [ADR-011](011-api-versioning-and-openapi.md) — §7's error catalogue, §8's wire conventions
- `vaullet-io/backend-common` — the repository, whose README is the adoption guide
