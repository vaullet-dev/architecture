# ADR-015: Cluster Topology — Three RKE2 Servers on One Machine

## Status

**Accepted** (2026-09-17) · **Amended** (2026-09-17, §4 — *As built*)

Resolves the topology item deferred in `gitops/TODO.md` on 2026-09-11, whose stated trigger was
"revisit at the first stateful service". PostgreSQL is that service.

## Context

The cluster is one Hetzner AX41 running single-node RKE2. That was a deliberate choice, stated
plainly in `gitops/README.md` — *"production topology, not production grade — one PSU, one board, one
node"* — and it has been correct up to now, because nothing deployed on it held data anybody would
miss.

That changes with PostgreSQL. And several accepted decisions already assume a cluster this one cannot
be:

- [ADR-007](007-kafka-topics-and-event-schema-evolution.md) specifies Kafka **RF=3**. On one node
  that is three replicas on one disk — the word "replication" without the property.
- [ADR-005](005-module-composition-and-deployment-topology.md) assumes modules can be drained and
  redeployed independently. With `PodDisruptionBudgets defined: 0` there is nothing to drain against.
- [ADR-003](003-hybrid-database-strategy-with-analytics.md) sets an RTO of 15 minutes for
  `vaullet_db`. A kernel upgrade on a single node is a cluster outage of longer than that.

**`etcd members: 1`** is the root of all three. Quorum is one, so the cluster tolerates zero node
failures, and every availability mechanism downstream is decoration.

### What the machine actually has

Measured on 2026-09-17, not estimated:

| | |
|---|---|
| CPU | AMD Ryzen 5 3600 — 6 cores / 12 threads, one NUMA node |
| Memory | 62.7 GiB ECC. `kubectl top node`: **4551 Mi used, 7%** |
| Whole cluster, 34 pods | **2.34 GiB actual** — of which `kube-apiserver` is 1187 Mi and everything else is under 100 Mi each |
| Disk | 2 × 477 GB NVMe, RAID1 (`md2`, healthy). **401 GB free** |
| Virtualisation | AMD-V enabled, `/dev/kvm` present. libvirt **not installed** |
| Public addressing | **one IPv4, `65.109.58.119/32`**, gateway off-subnet. IPv6 `/64` included |
| Storage | `local-path` only, node-local. **No `VolumeSnapshotClass`** |

### Constraints

- **No additional cost.** Whatever is chosen runs on the machine already rented. This rules out extra
  IPv4 addresses, object storage, and a second machine.
- **One IPv4, routed as a `/32`.** With an off-subnet gateway there is no spare address to bridge a
  VM onto, so nodes cannot hold public IPv4s.
- **It ships to customers.** Anything adopted here is deployed into someone else's cluster under
  contract (ADR-005), so licences matter — the reasoning that chose OpenBao over BSL-licensed Vault.
- **Nothing valuable is deployed yet.** One PVC exists cluster-wide, holding 236 KB.

### The question

How many Kubernetes nodes should this machine present, of what kind, and what does that actually buy?

## Decision

### 1. Three RKE2 **server** nodes, left schedulable. No worker agents.

Three VMs on the one machine, each an RKE2 server — in Kubernetes terms, three nodes with
`ROLES: control-plane,etcd`. `etcd` therefore has three members, quorum two, and the cluster survives
losing one node.

**They are not tainted, so they run workloads.** This is worth stating because it contradicts the
usual textbook shape: `kubeadm` applies `node-role.kubernetes.io/control-plane:NoSchedule` by default
and you are then obliged to add workers. RKE2 does not, and this cluster already proves it — the
single node reports `Taints: <none>` while running Argo CD, Traefik, OpenBao, cert-manager and
External Secrets.

So three servers give three *usable* scheduling domains. Dedicated worker agents would be three more
VMs buying no additional etcd members — see *Alternative 2*.

### 2. The measured cost, and what is left

A control-plane node costs **≈2.75 GiB**: 1.55 GiB of control plane (dominated by `kube-apiserver`)
plus ≈1.2 GiB of guest kernel, systemd, containerd and kubelet.

```
62.7 GiB
 −8.25   3 nodes x 2.75
 −3.50   host: libvirt, ~300 MB qemu overhead per VM, OS, page cache
 ──────
≈51 GiB usable for workloads       (single node today: ≈60 GiB)
```

The tax is **19%**. Against a core Vaullet deployment of roughly 25 GiB of requests — 7 core services
plus PostgreSQL, Kafka, Keycloak, Redis and Elasticsearch — that leaves comfortable headroom.

**The number to watch is `kube-apiserver`.** It is 1187 Mi against 34 pods and grows with object count
and watchers. Three of them at 250 pods could be 2–3 GiB each. Re-measure rather than trusting this
figure as the cluster fills.

### 3. Sizing

| | |
|---|---|
| Per VM | **4 vCPU, 16 GiB, 60 GB thin-provisioned qcow2** |
| Total | 12 vCPU (1:1 with threads), 48 GiB, ~180 GB nominal disk |

vCPU is allocated 1:1 with hardware threads rather than oversubscribed, because **etcd is
latency-sensitive and CPU steal between guests is the classic cause of spurious leader elections**.
The server VMs' vCPUs are pinned; nothing else on the host competes.

Disk is thin-provisioned: a 60 GB qcow2 consumes only what is written, so the nominal 180 GB does not
come out of the 401 GB free until it is actually used. That matters, because at this topology **disk
is the binding constraint long before RAM is** — see *Risks*.

### 4. Networking: private bridge, NAT, ingress forwarded from the host

The `/32` decides this. Nodes take private addresses on a libvirt bridge and the host forwards
inbound traffic:

```
                internet
                    │
       65.109.58.119  (host — the only public IPv4)
                    │  DNAT :80 :443 :6443
                    ▼
     virbr0 ── k8s-1 192.168.122.11
              k8s-2 192.168.122.12
              k8s-3 192.168.122.13
```

Every port Kubernetes needs between nodes — etcd 2379/2380, apiserver 6443, kubelet 10250, the canal
overlay — runs inside that bridge and never reaches the internet. This is how managed clusters are
shaped: EKS and GKE nodes hold private addresses and no public IP at all.

**Ingress is staged, deliberately.**

1. **At cutover: DNAT to one node.** The fewest moving parts at the moment most likely to go
   wrong, and reversible by deleting the rules. See *As built* below: the count was wrong, and the
   `nodeSelector` turned out to be unnecessary.
2. **Once proven: HAProxy on the host**, TCP passthrough to all three nodes. Ingress then survives
   draining a node, which is half the reason for the split. **TLS still terminates in Traefik** — the
   host proxies TCP and never holds a certificate.

The included IPv6 `/64` could give each VM a real public address with no NAT. Not used: `vaullet.dev`
is IPv4-only with an A record, so the IPv4 path is needed regardless and a second ingress route would
be a second thing to keep correct.

#### As built (2026-09-17)

This ADR said "two `iptables` rules". Publishing a libvirt-hosted cluster on a single public address
takes **three rules across two tables**, and the two that were missing here fail in ways that read as
something other than a firewall problem. Recorded because the shape generalises to any NAT-ed guest
network, not because the commands belong in an ADR:

| Rule | Why it is load-bearing |
|---|---|
| `nat/PREROUTING` DNAT, **inserted at position 1**, scoped `-d <public ip>` | Calico's `cali-PREROUTING` and RKE2's `CNI-HOSTPORT-DNAT` occupy the same chain, so an *appended* rule is never reached and a cutover silently keeps serving from the old cluster. The `-d` scope is not tidiness: packets *leaving* the guests are forwarded and traverse `PREROUTING` too, so an unscoped rule bends all outbound HTTPS back into the cluster's own ingress — ACME, registry pulls and git fetches all fail, reporting a TLS mismatch against Traefik's default certificate. |
| `filter/LIBVIRT_FWI` ACCEPT | libvirt's default network is outbound-only. It accepts `RELATED,ESTABLISHED` and ends the chain with `REJECT --reject-with icmp-port-unreachable`, so a new inbound connection is accepted by the host and then refused internally. Indistinguishable from a dead service except that it fails instantly rather than hanging. |
| `nat/POSTROUTING` MASQUERADE | NAT loopback. Without it a guest resolving the public hostname is DNAT-ed to a neighbour on the same bridge, which replies directly and outside conntrack. cert-manager's HTTP-01 self-check runs over precisely this path, so certificate renewal fails while the site is healthy from outside. |

Two further corrections to the staging above:

**Pinning Traefik was unnecessary.** `enable-servicelb` places a klipper-lb (`svclb`) pod on every
node, each forwarding into the Traefik Service, so any node address is a valid entry point wherever
Traefik is scheduled. At cutover the DNAT named `k8s-1` while Traefik ran on `k8s-3` and it served
correctly throughout. The `nodeSelector` would have cost scheduling freedom for nothing.

**The single-node DNAT is still a single point of failure** — because the rule *names* a node, not
because Traefik lives on one. That is what stage 2 (HAProxy) fixes, and it is why a statistical
spread across the three addresses is not a shortcut to it: without health checks it black-holes a
third of requests the moment a node goes down.

**Stage 2 landed the same day, and it subsumed the defect above.** The DNAT left one known fault:
the node it named could not reach the public address through the host, because its packet was
DNAT-ed back to itself, never SNAT-ed, and arrived with `src == dst`, where the kernel discards it
as a martian. cert-manager's HTTP-01 self-check runs over exactly that path.

HAProxy on the host (`mode tcp`, TLS still terminating in Traefik) fixed both that and the single
point of failure, because a proxy does not forward the original packet — it terminates the
connection and opens a new one sourced from the host, so source and destination can never be equal.
All three nodes now reach the public address. The failover was drilled rather than assumed: the site
served with two of three backends disabled. Health checks are HTTP on `:80` for both pools, since a
bare TCP connect succeeds against klipper-lb even with no Traefik behind it.

The remaining cost is the one a TCP proxy always has: **backends see the host, not the client.**
Nothing depends on the client address in this cluster yet, so it was accepted rather than solved.
PROXY protocol is the answer when rate limiting, IP allowlists or honest access logs arrive, and it
has to land on the proxy and the Traefik entrypoints together.

### 5. Built in parallel, cut over, not migrated

VMs cannot be slid underneath a running bare-metal RKE2 node; this is a new cluster. But it does not
have to be a rebuild-in-place, because the machine is 93% idle:

```
now       AX41 ── RKE2 (bare metal)                    serving vaullet.dev
build     AX41 ── RKE2 (bare metal)                    still serving
               └─ libvirt ── k8s-1 / k8s-2 / k8s-3     new cluster, verified
cut over  flip the host's DNAT rules to the bridge
rollback  flip them back
reclaim   destroy the bare-metal RKE2
```

Downtime is the length of an `iptables` change, and every step before the flip is reversible by doing
nothing. During the overlap both clusters coexist: 48 GiB of VMs plus ~5 GiB of the existing node
fits, though it is the one moment the machine is tight.

### 6. `system-reserved` is set on every node

The current node reports `allocatable == capacity` — RKE2 reserves nothing. The scheduler will
therefore place pods requesting the entire machine, leaving nothing for the kernel, containerd or the
RKE2 supervisor, and the OOM killer's next choice may be etcd.

Each VM sets `--kubelet-arg=system-reserved=cpu=200m,memory=1Gi`. This is a latent fault on the
current single node too, and it is fixed here rather than carried forward.

### 7. What this explicitly does not buy

Stated so nobody later mistakes the topology for resilience it does not have:

- **Not hardware resilience.** Three VMs share one PSU, one board, one host kernel and one disk array.
  Every failure that takes the machine takes all three nodes.
- **Not capacity.** It costs 19% and adds nothing.
- **Not a production-grade deployment.** `gitops/README.md` keeps the line *"production topology, not
  production grade"*, which after this change is more accurate rather than less: the topology becomes
  production-shaped while the hardware stays a single point of failure.

What it buys is that **PDBs, drains, `topologySpreadConstraints`, rolling upgrades and Kafka RF=3 stop
being decoration** — and that a kernel upgrade becomes a rolling operation rather than an outage.

## Consequences

### Positive Consequences

✅ **etcd tolerates one node failure** — quorum two of three, instead of zero of one
✅ **ADR-007's RF=3 and ADR-005's drain assumptions become deliverable** rather than aspirational
✅ **Node upgrades stop being cluster outages** — cordon, drain, patch, uncordon, next
✅ **Three real scheduling domains**, so anti-affinity and spread constraints do something
✅ **No additional cost** — one rented machine, KVM already enabled
✅ **Cheapest possible moment**: one 236 KB PVC exists cluster-wide
✅ **The rebuild retires the exposed OpenBao unseal keys** as a side effect, since the new cluster is initialised fresh

### Negative Consequences

⚠️ **19% of the machine goes to overhead** — ~11.75 GiB to duplicated kernels, kubelets and three
copies of `kube-apiserver`. Usable memory drops from ~60 GiB to ~51 GiB.

⚠️ **libvirt is a new layer to operate** — VM lifecycle, a bridge, NAT rules, and guest images to
patch alongside the cluster.

⚠️ **Ingress becomes a host concern.** Two `iptables` rules today, a HAProxy later. Both live outside
GitOps, which is a genuine departure from "the whole cluster, as git".

⚠️ **A networking mistake can cost remote access.** A bridge or `ip_forward` error can black-hole the
host's own routing, and recovery is then Hetzner's rescue system.

⚠️ **Three `kube-apiserver` processes** instead of one, each with its own watch cache. This is the
line item most likely to grow.

### Risks

| Risk | Impact | Mitigation |
|---|---|---|
| Host networking broken during setup, SSH lost | Recovery via Hetzner rescue, plus a reboot | Verify from a second SSH session before closing the first; arm `sleep 60 && reboot` before any change touching `enp41s0` |
| etcd leader elections from CPU steal | Cluster instability under load | vCPU pinned 1:1 with threads; no oversubscription on server VMs |
| Three etcd members fsyncing to one RAID1 array | Write latency, and it triples with member count | Accepted at three; a reason not to go to five (*Alternative 3*) |
| Disk exhaustion | Everything stops at once | Thin qcow2; ~220 GB left for data after VM roots. Kafka retention must be sized to the box, not to ADR-007's maximum |
| `local-path` pins volumes to nodes | A stateful pod cannot reschedule elsewhere | Accepted. CloudNativePG gives each instance its own volume, which is the pattern it recommends; Kafka behaves the same way. Longhorn is rejected — replicating three ways onto one RAID1 array is write amplification for no durability |
| Cutover fails | vaullet.dev down | The old cluster is still running and serving until the DNAT flip; rollback is reversing two rules |
| Overlap period is tight on memory | Build stalls | Build VMs at 12 GiB, grow to 16 after the old node is reclaimed |

## Alternatives Considered

### Alternative 1: Stay on one node

**Description**: Change nothing. Accept `etcd members: 1`.

**Pros**: ~60 GiB usable instead of 51. No libvirt, no NAT, no new failure modes. Honest, and already
documented.

**Cons**: Every availability mechanism in the accepted ADRs stays decoration. A kernel upgrade is an
outage. Kafka RF=3 is meaningless. And the cost of changing later is a database migration across a
cluster rebuild rather than a flip of two firewall rules.

**Why rejected**: the capacity argument for staying — which was the strongest one — collapsed on
measurement. The tax is 19%, not the ~40% first estimated, and against a ~25 GiB core deployment
there is ample room. What remains is a real gap between what the ADRs specify and what the cluster can
do.

---

### Alternative 2: One control-plane node plus worker nodes

**Description**: The conventional shape — one RKE2 server, two or three agents.

**Pros**: Matches what most people expect. Isolates workloads from the API server.

**Cons**: Worker nodes hold no etcd. Three or four VMs, the same overhead, and **`etcd members: 1`
unchanged** — lose the server and the cluster stops accepting any change, while pods keep running with
no way to replace them. It also traps you: RKE2 can add an agent to a running cluster with one
command, but **there is no supported promotion from agent to server** — that is a drain, delete and
rebuild.

**Why rejected**: it pays the full price of virtualisation and fixes the single point of failure it
was presumably adopted to fix. Three servers cost the same or fewer VMs and give quorum as well. Two
of the four textbook reasons for the split — independent hardware scaling and hardware blast radius —
do not survive putting every node on one machine; the security argument does, and is the reason
worker agents may be added *later*, on top of three servers.

---

### Alternative 3: Five or seven control-plane nodes

**Description**: More etcd members for more failure tolerance.

**Pros**: Fits — five at 8 GiB is 40 GiB. Tolerates two simultaneous failures rather than one.

**Cons**: Every failure capable of taking two VMs on this machine — host kernel panic, disk array,
PSU, board — takes all of them. The independent failures that do occur, a VM crash or a node drained
for patching, arrive one at a time and three already covers those. Meanwhile five members means five
etcd instances fsyncing every Raft proposal to the same two NVMes, and every write waiting on three
acknowledgements instead of two.

**Why rejected**: past three, the dial stops doing anything the hardware can deliver. Tolerance of
correlated failure comes from a second machine, not more guests.

---

### Alternative 4: Three servers plus three worker agents

**Description**: The full conventional topology, virtualised.

**Pros**: API servers isolated from workloads — the one textbook reason that survives virtualisation.

**Cons**: Six VMs, ~16.65 GiB of overhead — **27%** — leaving ~46 GiB. Six guest images to patch.

**Why rejected**: not yet. At 34 pods and 7% memory the isolation buys little, and agents can be added
to a running cluster trivially. Revisit when `kube-apiserver` memory or a security review justifies
it; the three-server layout is the one that keeps that door open.

---

### Alternative 5: A second physical machine

**Description**: Rent another Hetzner box and build a genuinely distributed cluster.

**Pros**: The only option that survives losing hardware. Everything else here is topology on a single
point of failure.

**Cons**: Costs money.

**Why rejected**: excluded by constraint, not by argument. It is worth recording that this is the only
real answer to hardware resilience, so that nothing in this ADR is later mistaken for having provided
it.

## Implementation Notes

### Order

1. Install libvirt/qemu; define the bridge and NAT **without touching `enp41s0`'s own routing**.
2. Provision three VMs by cloud-init: Ubuntu 26.04, static addresses `.11`/`.12`/`.13`.
3. RKE2 server on `k8s-1`; join `k8s-2` and `k8s-3` with the shared token and `server:` pointing at
   `k8s-1`. Set `system-reserved` on all three.
4. Bootstrap Argo CD from `gitops`, then the platform wave. **Initialise OpenBao fresh** — new unseal
   shares, new root token, stored offline.
5. Verify: three `Ready` nodes, three etcd members, every Argo Application `Synced`/`Healthy`.
6. Flip the host DNAT to the bridge. Verify `vaullet.dev` end to end.
7. Reclaim the bare-metal RKE2 and grow the VMs to 16 GiB.

The commands belong in a runbook in `gitops`, with the rollback for each step written next to it.

### OpenBao

The new cluster means `03-openbao.sh init` runs again, producing new unseal shares. That retires the
ones exposed in a plaintext file in the workspace on 2026-09-15 without needing a rekey on a live
instance, where a mis-copied share would mean permanent loss. **The init output must not pass through
an assistant terminal** — run it directly on the box.

### What stays outside GitOps

libvirt domain definitions, the bridge, the NAT rules and later the HAProxy configuration live on the
host, not in `gitops`. That is a real departure from *"the whole cluster, as git"* and should be
managed by a checked-in script under `gitops/bootstrap/`, in the same spirit as the three existing
bootstrap scripts, rather than by hand.

### Out of scope

**Where PostgreSQL's WAL archive goes**, which is the next decision and blocked on this one. **Kafka
retention sizing** against ~220 GB of remaining disk. **Worker agents**, deferred to *Alternative 4*'s
trigger.

## References

- [ADR-003: Hybrid Database Strategy](003-hybrid-database-strategy-with-analytics.md) — the RTO this topology has to support
- [ADR-005: Module Composition and Deployment Topology](005-module-composition-and-deployment-topology.md) — drains and independent module deployment
- [ADR-007: Kafka Topics and Event Schema Evolution](007-kafka-topics-and-event-schema-evolution.md) — RF=3
- `gitops/README.md` — "production topology, not production grade"
- `gitops/TODO.md` — the 2026-09-11 deferral this resolves

---

**Date**: 2026-09-17 (amended the same day, §4 *As built*, after execution)
**Author**: Predrag
