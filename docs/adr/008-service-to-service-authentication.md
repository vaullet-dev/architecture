# ADR-008: Service-to-Service Authentication

## Status

**Accepted** (2026-09-01)

Layers 1–3 as described below are adopted in full. **Alternative 6** (mesh-free: NetworkPolicy + CNI
encryption + Keycloak tokens on every call) was evaluated as a unit on 2026-09-01 and rejected — see
its entry for the reasoning, which reverses the initial lean.

| Layer | Decision |
|---|---|
| **Layer 1** — NetworkPolicy, default-deny | ✅ Adopted |
| **Layer 2** — Linkerd mTLS | ✅ Adopted |
| **Layer 3** — Keycloak service tokens, money path only | ✅ Adopted, scope unchanged |
| **Kafka** — SASL/OAUTHBEARER + ACLs | ✅ Adopted |
| **User context** — service identity + explicit `account_id` | ✅ Adopted |

Closes the item [ADR-006](006-authentication-and-identity.md) explicitly deferred: that ADR settled
*user* identity and assumed services inside a cluster were mutually authenticated, without saying how.

**Amended by [ADR-016](016-transactional-outbox-for-event-producers.md)** (2026-10-22): Layer 3 gains
the ledger's scopes as built, `ledger:read` and `ledger:write` — Auth Service's outbox relay among the
callers, and `GET /v1/reservations/{id}` among the reads, which closes **B4**.

## Context

Vaullet runs 14 services in one Kubernetes cluster per customer, communicating over REST for
synchronous validation and Kafka for everything else. Nothing currently authenticates one service to
another.

### Three questions, usually conflated

| Question | Example | Mechanism family |
|---|---|---|
| **Transport identity** | Is this really Transaction Service calling `POST /reservations`, or a compromised Notification pod? Can anything on the network read it? | mTLS |
| **Authorization** | Even if it *is* Transaction Service — should it be allowed to call that endpoint? Should Bonus Service? | policy / token scopes |
| **User context** | Transaction Service reserves for user X. How does Vaullet learn that trustworthily? | token propagation |

Most designs answer the first and quietly skip the other two.

### Threat model

Single-tenant clusters (ADR-005) remove cross-customer exposure, so the realistic threats are internal:

- **Lateral movement.** A vulnerability in Notification Service — a large dependency surface, low
  business criticality — becomes a path to the ledger if any pod can call any pod.
- **Traffic interception.** Node-level compromise, or shared/managed infrastructure where pod-to-pod
  traffic is not inherently private.
- **Repudiation.** An adjustment was posted; nothing proves which service posted it.

The blast radius that matters is Vaullet's `POST /v1/reservations`. It is the only synchronous
endpoint that commits money, and it is reachable from inside the cluster.

### Constraints

- **Per-customer cost multiplies.** ADR-005 already flags a high floor, raised again by Keycloak
  (ADR-006). Anything added here is paid once per customer, forever.
- **Keycloak is already core** and already issues tokens against a JWKS every service trusts.
- **Modules are conditionally deployed**, so any policy or ACL scheme must tolerate a service simply
  not existing.

## Decision

We will use **three layers**, because the three questions above have different right answers.

### Layer 1 — NetworkPolicy, default-deny

Every namespace denies ingress by default; each service declares who may reach it.

```yaml
kind: NetworkPolicy
spec:
  podSelector: { matchLabels: { app: vaullet } }
  policyTypes: [Ingress]
  ingress:
    - from:
        - podSelector: { matchLabels: { app: transaction-service } }
        - podSelector: { matchLabels: { app: admin-ui } }
```

Free, and it contains the blast radius of any failure in the layers above. It authenticates a
*network location* rather than a workload, which is precisely why it is not sufficient alone: anything
that lands inside an allowed pod inherits its reachability.

### Layer 2 — Linkerd, mTLS on all internal traffic

Linkerd is installed in every cluster. All meshed pod-to-pod traffic is mutually authenticated and
encrypted, with no application code and no certificates for us to manage.

Workload identity derives from the Kubernetes ServiceAccount, so `transaction-service` is a
cryptographic identity rather than a label anyone can copy. Certificates are issued and rotated by
`linkerd-identity` on a short lifetime; the trust anchor is the only thing with a long-lived rotation
schedule.

Authorization is declarative, and lives beside the service it protects:

```yaml
apiVersion: policy.linkerd.io/v1beta1
kind: Server
metadata: { name: vaullet-api }
spec:
  podSelector: { matchLabels: { app: vaullet } }
  port: http
  proxyProtocol: HTTP/2
---
apiVersion: policy.linkerd.io/v1alpha1
kind: MeshTLSAuthentication
metadata: { name: reserve-callers }
spec:
  identities:
    - "transaction-service.vaullet.serviceaccount.identity.linkerd.cluster.local"
---
apiVersion: policy.linkerd.io/v1alpha1
kind: AuthorizationPolicy
metadata: { name: vaullet-reserve }
spec:
  targetRef:  { kind: Server, name: vaullet-api }
  requiredAuthenticationRefs:
    - { kind: MeshTLSAuthentication, name: reserve-callers }
```

**Why Linkerd rather than Istio.** Istio's policy engine is richer, and at multi-cluster scale that
richness earns its keep. Here the deciding factor is the cost floor: roughly 30 sidecars per cluster
(14 services × 2 replicas) is on the order of 1.5–3 GB of sidecar memory under Istio versus 300–600 MB
under Linkerd — per customer, permanently. Vaullet needs mTLS and simple L7 authorization, which is
exactly Linkerd's scope. Istio ambient mode removes the sidecar entirely and would change this
calculation; it is worth revisiting when it has more operational history.

### Layer 3 — Keycloak service tokens, on the money path only

Services that move money additionally present a Keycloak-issued service token. Each such caller is a
confidential client with a service account:

```
transaction-service → POST /realms/vaullet/protocol/openid-connect/token
                      grant_type=client_credentials
                    ← { access_token, expires_in: 300 }

                    → POST vaullet/v1/reservations
                      Authorization: Bearer <token>
```

Vaullet validates it against the JWKS it already trusts for user tokens (ADR-006), and checks the
scope.

**Where this applies — and nowhere else:**

| Caller → callee | Scope |
|---|---|
| Transaction Service → Vaullet `POST /reservations`, `DELETE /reservations` | `ledger:reserve` |
| Admin UI → Risk Management account freeze | `risk:act` |
| Admin UI, operator backend → Auth `GET /v1/accounts/**` | `identity:read` |
| Admin UI, operator backend → Auth `POST`, `PATCH`, `PUT`, `DELETE /v1/accounts/**` | `identity:admin` |
| Auth Service, outbox relay → Ledger `POST /v1/wallets` | `ledger:write` |
| Operators by hand (client `ledger-ops`) → Ledger every `POST`, `PATCH`, `DELETE` | `ledger:write` |
| Any caller reading the ledger — today only operators, by hand, as `ledger-ops` → Ledger every `GET`: `/v1/reservations/{id}`, `/v1/accounts/{id}/balance`, `/v1/wallets/**` | `ledger:read` |

> **Added 2026-09-17** with [ADR-014](014-user-management-in-local-identity-mode.md). The two rows are
> not paperwork: this table is the source the Keycloak client scopes and the mesh policy are generated
> from, so an endpoint missing from it is unreachable in one direction and ungoverned in the other. No
> client would be configured to request `identity:admin`, so no real token would carry it and
> `@PreAuthorize` would refuse everyone — a 403 that reads like a bug in the service rather than a
> missing row here.
>
> Scope alone is not the whole rule. Auth Service also requires `SUPER_ADMIN` to create an account and
> `FRAUD_REVIEWER` to change its status, because ADR-006's separation of duties is what keeps user
> administration from being able to lift a fraud freeze.

> **Added 2026-10-22** with [ADR-016](016-transactional-outbox-for-event-producers.md). The last three
> rows are the ledger's scopes as wallet-ledger-service checks them and gitops'
> `realm/15-vaullet-client-scopes.yaml` defines them: `ledger:read` on every read, `ledger:write` on
> every change. Reading a reservation needs `ledger:read`, which closes **B4**.
>
> Two things here do not agree yet, and are recorded rather than resolved:
>
> - **The money path's scope.** The first row names `ledger:reserve`. No such scope exists in the
>   realm, and the ledger checks `ledger:write` on `POST` and `DELETE /v1/reservations`. Transaction
>   Service does not exist yet, so nothing is wrong today; but `ledger:write` opens, freezes and closes
>   wallets as well as holding money, and the outbox relay holds it. **G6** in [TODO.md](TODO.md).
> - **Layers 1 and 2 are not deployed yet.** There is no NetworkPolicy and no mesh in the cluster, so
>   on the ledger this scope check is the only control between a pod and any account; the ledger's
>   lack of a public route is what keeps it inside the cluster.

> **Removed (2026-09-01)**: an earlier draft also granted `ledger:adjust` for "Vaullet manual
> adjustments". No such endpoint exists — [ADR-004](004-atomic-balance-reservations.md)'s API is
> reservations and balance only. Manual ledger adjustments are a real operational need (correcting
> errors, goodwill credits, dispute resolution) but they have never been designed, and authorising an
> operation that does not exist is worse than not authorising it. Tracked as a pending decision.

**Why not everywhere.** Applying this to all inter-service calls means token plumbing in fourteen
services to duplicate a guarantee Layer 2 already provides cryptographically. That is cost without
benefit.

**Why here.** A Linkerd `AuthorizationPolicy` is one YAML mistake away from being wrong, and the
reserve endpoint is where that mistake commits money. An application-level check that fails
independently of the mesh is worth its cost on exactly these paths. The token also carries a
verifiable `azp` claim that goes into the audit record, which answers repudiation in a way network
identity alone does not.

### Kafka is a separate channel and needs its own answer

Layers 1–3 cover REST. Kafka carries most inter-service traffic and is not covered by any of them.

- **Authentication**: SASL/OAUTHBEARER against Keycloak. Services present the same client-credentials
  token they already obtain, so there is one identity system rather than a parallel set of Kafka
  credentials.
- **Authorization**: per-principal ACLs on topics and consumer groups. A service may produce only to
  the topics it owns and consume only what it needs — Bonus Service can read `transaction.created.v1`
  and cannot write it.
- **Encryption**: TLS to brokers.

ACLs are generated from the module composition (ADR-005): **a module that is not deployed has no
principal and no ACLs.** The topic ownership model in ADR-007 — the domain segment names the producer
— maps directly onto produce permissions, so the ACL set is derivable from the topic list rather than
maintained by hand.

### User context: service token plus explicit `account_id`

Vaullet learns which user a reservation is for from an explicit `account_id` in the request body,
authenticated by the caller's service identity rather than by a user token.

Transaction Service has already validated the user's token at the gateway (ADR-006) and authorized the
operation. Vaullet trusts Transaction Service — verified cryptographically by Layer 2 and by token at
Layer 3 — to have done so. Both services are inside one trust boundary, and the reserve call is
idempotent and keyed, so a replay is inert.

**Rejected: forwarding the user's token.** Vaullet would then accept user tokens, which makes a stolen
user token a direct path to the ledger, bypassing every check Transaction Service performs — fraud
scoring, limits, the transaction record itself.

**The hardening path, if this trust boundary ever needs tightening**: RFC 8693 token exchange.
Transaction Service exchanges the user's token for one scoped to *(user, vaullet, reserve)*, useless
anywhere else and carrying genuine user context. Keycloak's `token-exchange` feature is already enabled
for ADR-006's federated flow, so this is a change to two services and no new infrastructure. It is not
adopted now because it adds a Keycloak round trip to the latency-sensitive approval path to defend
against a threat — a compromised Transaction Service — that already has far worse options available
to it.

## Consequences

### Positive Consequences

✅ **Encryption and workload identity with no application code** — Layer 2 covers every service uniformly, including ones written later
✅ **Identity is cryptographic, not a shared secret** — a copied label or leaked config does not impersonate a service
✅ **Defence in depth where it matters** — a mesh policy error does not by itself open the money path
✅ **One identity system** — Keycloak serves users, services and Kafka principals; one JWKS, one rotation story
✅ **Composition-aware** — an undeployed module has no ServiceAccount, no policy and no ACLs, so nothing to misconfigure
✅ **Repudiation answered** — the audit record names a verified calling service, not an inferred one
✅ **Lateral movement is contained** — a compromised Notification pod fails at NetworkPolicy, then at mesh policy, then at the token check

### Negative Consequences

⚠️ **Linkerd is another component per cluster**, with its own upgrade cadence and its own trust-anchor
rotation — an expiry that is easy to forget and takes the whole cluster down when it lapses.

⚠️ **Two authorization mechanisms can drift.** Mesh policy says who may reach `POST /reservations`;
token scopes say who may reserve. Kept in one place and generated where possible, but they are two
statements of one intent and can disagree.

⚠️ **Keycloak sits on the money path.** Mitigated by caching tokens for their lifetime and refreshing
at 80%, plus JWKS caching with stale-if-error — a Keycloak outage stops *new* token issuance rather
than in-flight traffic — but it is a dependency the reserve call did not previously have.

⚠️ **Sidecar latency**, roughly a millisecond per hop. Immaterial against a 200ms approval budget,
measurable in a trace.

⚠️ **mTLS failures are opaque.** "Connection refused" with no application-level explanation is a
recognised operability cost of any mesh, and it will cost debugging time.

⚠️ **Service secrets need rotation**, which is real operational work multiplied per cluster.

### Risks

| Risk | Impact | Mitigation |
|------|--------|-----------|
| Linkerd trust anchor expires | Cluster-wide mTLS failure; total outage | Anchor expiry monitored with 30-day alerting; rotation rehearsed; expiry date is a tracked operational date, not a certificate detail |
| Mesh policy and token scope disagree | An intended-blocked call succeeds, or a legitimate one fails | Both generated from one module-composition source; an integration test asserts the negative case — Bonus Service *cannot* reserve |
| Service secret leaked | Impersonation until rotation | Short token lifetime; secrets from External Secrets Operator, never in Git; rotation on a schedule and on suspicion |
| A new service ships unmeshed | Silent gap — traffic works, unencrypted and unauthenticated | Namespace-level auto-injection; CI asserts every Deployment is meshed; unmeshed pods alert |
| Kafka ACLs drift from deployed modules | A removed module retains permissions, or a new one lacks them | ACLs generated from the same Helm values that decide deployment (ADR-005) |
| Policy blocks a legitimate call in production | Outage on a path nobody tested | Linkerd audit mode before enforcement on any new policy; the ADR-005 CI matrix exercises core-only and full compositions |

## Alternatives Considered

### Alternative 1: Istio

**Description**: Istio in place of Linkerd.

**Pros**: Richer policy engine (JWT validation at the sidecar, request-level authorization, egress
control); larger ecosystem; multi-cluster support Vaullet may eventually want.

**Cons**: Substantially heavier — roughly 1.5–3 GB of sidecar memory per cluster against Linkerd's
300–600 MB, plus a more demanding control plane, multiplied by every customer. The operational
learning curve is steeper on a platform already carrying Kafka, Keycloak, Elasticsearch and seven
databases per cluster.

**Why rejected**: Vaullet needs mTLS and simple L7 authorization. Istio's additional capability does
not offset a per-customer cost that recurs for the life of every contract. **Ambient mode changes this
materially** and should be revisited once it has more production history.

---

### Alternative 2: cert-manager plus Spring Boot mTLS, no mesh

**Description**: Issue per-service certificates with cert-manager; configure mTLS in each service.

**Pros**: No mesh; no sidecars; no additional runtime component.

**Cons**: Truststore and keystore configuration across fourteen services, rotation without restart, and
no authorization layer at all — a certificate says *who*, never *what they may do*. It is the mesh's
work done by hand, minus the mesh's benefits.

**Why rejected**: More application burden than Layer 2 for a strictly weaker result.

---

### Alternative 3: SPIFFE/SPIRE

**Description**: SPIRE issues SVIDs as platform-agnostic workload identity.

**Pros**: The standard for workload identity; portable across clusters and clouds; the correct
foundation at scale.

**Cons**: Another stateful component per cluster, and it still needs something to enforce policy — in
practice a mesh, which then supplies the identity anyway.

**Why rejected**: Solves a heterogeneity problem Vaullet does not have. Fourteen services in one
Kubernetes cluster are exactly the case a mesh's built-in identity covers.

---

### Alternative 4: NetworkPolicy only

**Description**: Layer 1 alone.

**Pros**: Free, native, no new components, no latency.

**Cons**: No encryption, and identity is network position rather than workload — a compromised pod
inherits its reachability, and nothing is recorded about who called what.

**Why rejected**: Retained as a layer, insufficient as the answer. It cannot address interception or
repudiation.

---

### Alternative 5: Keycloak service tokens on every inter-service call, no mesh

**Description**: Layer 3 everywhere; skip the mesh.

**Pros**: One mechanism; reuses Keycloak; fine-grained scopes throughout; no sidecars.

**Cons**: No transport encryption — a Bearer token over plaintext is a replayable password. Token
plumbing in every service, including ones with no security-relevant surface. Keycloak becomes a
dependency of *all* internal traffic rather than three endpoints.

**Why rejected**: Tokens authorize; they do not protect the wire. Without mTLS the tokens themselves
are interceptable, which undermines the mechanism doing the work.

### Alternative 6: NetworkPolicy + Keycloak tokens everywhere, no mesh (Alternatives 4 + 5 combined) — **EVALUATED AND REJECTED (2026-09-01)**

**Description**: Drop the mesh. Layer 1 for reachability, Keycloak client-credentials tokens on
*every* inter-service call for identity and authorization, and encryption at the CNI layer rather than
by sidecars.

Alternatives 4 and 5 were each rejected above for a gap the other one fills: 4 has no identity or
authorization, which 5 supplies; 5 has no encryption, which **CNI-level transparent encryption**
(Cilium or Calico with WireGuard) supplies without any sidecar. That combination was never evaluated
as a unit, and it deserved to be.

**Pros**:
- **No sidecars** — the smallest per-cluster footprint of any option
- **One identity mechanism** for users, services and Kafka principals; no second system to reason about
- **No mesh to operate or upgrade**, and no trust anchor whose expiry takes a cluster down
- **Authorization is explicit in application code**, so it is unit-testable rather than cluster YAML
- **Legible failures** — a 403 naming a scope beats an opaque connection reset

**Cons**:
- **Token plumbing in all fourteen services**, permanently
- **Keycloak becomes a dependency of all internal traffic** rather than three endpoints
- **Service identity becomes a bearer secret** — a leaked client secret impersonates a service until rotation
- **Loses the mesh's observability** — golden metrics and per-hop tracing would need building
- **CNI encryption is node-to-node**, so two pods on the same node may not traverse the encrypted path

**Why rejected — three reasons, in order of weight:**

**1. The footprint argument was weaker than it first appeared.** Linkerd is roughly 300–600 MB per
cluster. That is a real cost, but it sits beside Elasticsearch, Kafka, Keycloak, Argo CD (ADR-010) and
five to seven PostgreSQL databases. It is a small fraction of a cluster that already exists, and the
initial framing of this alternative overstated the saving by treating footprint as the deciding axis.
The Istio-versus-Linkerd comparison earlier in this ADR turns on a 1.5–3 GB difference and remains
sound; Linkerd-versus-nothing is a much smaller delta and does not carry the same weight.

**2. Mesh identity cannot be forgotten; token plumbing can.** Layer 2 applies to every service
uniformly, including ones written next year by someone who has not read this ADR. Alternative 6 makes
correct behaviour a property of each service's code — fourteen places to get right, and a permanent
surface where a new service can be written without the check and still work in every test. The ADR-010
CI gate "every `Deployment` is meshed" is a mechanically checkable fact; "every service correctly
obtains and validates a token on every outbound call" is not, and the difference between an invariant
the platform enforces and one each team must remember is the kind of thing this architecture has
repeatedly chosen to make structural.

**3. Its encryption story depends on an assumption we cannot verify.** CNI-level WireGuard is nearly
free *if the cluster already runs Cilium*. If it does not, adopting Alternative 6 means replacing the
CNI in every customer cluster — a far larger and riskier change than installing Linkerd. Making the
security posture contingent on someone else's platform choice is a poor trade when the alternative is
self-contained.

**What would flip this**: per-cluster resource cost becoming genuinely binding, *and* Cilium confirmed
across target clusters, *and* an accepted answer for the observability gap. Failing any one of those,
Layer 2 stands. Istio ambient mode, which removes sidecars while keeping mesh identity, is the more
likely future revision than going mesh-free.

## Implementation Notes

### Linkerd

Installed by the umbrella chart in every cluster. Namespace annotated
`linkerd.io/inject: enabled` so meshing is the default and opting out is explicit. Trust anchor stored
in the per-customer secret store with expiry tracked as an operational date.

New `AuthorizationPolicy` resources go to **audit mode** first — traffic that would be denied is logged
rather than blocked — and are promoted to enforcement after a clean window.

### Keycloak clients

```yaml
clients:
  - clientId: transaction-service
    serviceAccountsEnabled: true
    standardFlowEnabled: false          # no interactive login
    scopes: [ "ledger:reserve" ]
  - clientId: admin-ui
    serviceAccountsEnabled: true
    scopes: [ "risk:act" ]
```

Declarative, applied by `keycloak-config-cli` alongside the realm configuration from ADR-006. Secrets
are delivered by External Secrets Operator; none is committed.

### Spring Boot

Callers use `ClientCredentialsOAuth2AuthorizedClientProvider` with a cached token refreshed at 80% of
lifetime. Callees are resource servers validating against the Keycloak JWKS — the same configuration
already present for user tokens, with an additional scope check.

### Kafka

Strimzi with `type: oauth` listener authentication against the Keycloak realm, and ACLs generated from
the deployed module set. Topic ownership per ADR-007 gives produce permissions directly: a service may
produce only to topics whose domain segment names it.

### CI enforcement

1. Every `Deployment` is meshed — unmeshed workloads fail the build
2. Every service reaching Vaullet's reserve endpoint holds a Keycloak client with `ledger:reserve`
3. A negative test asserts a non-authorized service is refused at both Layer 2 and Layer 3
4. Kafka ACLs match the deployed module set for each configuration in the ADR-005 matrix

### Out of scope

**Human access** — `kubectl exec`, database credentials, break-glass procedures — is operational
access control, not service-to-service authentication. It deserves its own decision and is tracked
separately.

## References

- [ADR-005: Module Composition](005-module-composition-and-deployment-topology.md) — per-customer cost floor; conditional deployment driving policy and ACL generation
- [ADR-006: Authentication and Identity](006-authentication-and-identity.md) — Keycloak, the JWKS every service already trusts, and the deferral this ADR closes
- [ADR-007: Kafka Topics](007-kafka-topics-and-event-schema-evolution.md) — topic ownership mapping onto produce ACLs
- [ADR-004: Atomic Balance Reservations](004-atomic-balance-reservations.md) — why `POST /reservations` is the endpoint that matters
- [Linkerd — Automatic mTLS](https://linkerd.io/2/features/automatic-mtls/)
- [Linkerd — Authorization Policy](https://linkerd.io/2/features/server-policy/)
- [RFC 8693: OAuth 2.0 Token Exchange](https://datatracker.ietf.org/doc/html/rfc8693)

---

**Date**: 2026-09-01
**Author**: Predrag
**Reviewers**: TBD
