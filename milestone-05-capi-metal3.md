# Milestone 5: Cluster API + Metal3

**Goal:** Replace every manual step in M1–M4 with declarative Kubernetes resources. A `BareMetalHost` CR points at the BMC. A `Cluster` CR describes the desired cluster. Reconciliation loops do the work. You stop typing `curl` and start typing `kubectl apply`.

## What you're actually learning

- **Declarative bare-metal management**: the operator pattern applied to physical hardware.
- **The reconciliation loop on hardware**: what happens when desired state and observed state diverge for a physical thing that takes 10 minutes to power-cycle.
- **The CAPI architecture**: management cluster vs. workload cluster, infrastructure providers vs. bootstrap providers vs. control plane providers.
- **Where vmetal fits**: vmetal is in this lineage. Understanding CAPI + Metal3 means you can read vmetal's CRDs and immediately know what each one is for.

## Anchor question

> What does declarative bare-metal management actually mean, and what reconciliation loops are running to make it real?

Answer: at minimum, a **BareMetalHost controller** watching `BareMetalHost` CRs and driving Redfish + Ironic to converge each one to its target state (registered, inspected, available, provisioned, deprovisioned). A **Metal3Machine / Machine controller** binding `Machine` CRs to available BareMetalHosts. A **Cluster controller** (CAPI core) tracking topology references between `Cluster`, infrastructure-cluster, and control-plane CRs and triggering infra-cluster reconciliation, *not* directly orchestrating Machines; that's owned by the **MachineDeployment / KubeadmControlPlane / TalosControlPlane** controllers above it. Each is a stateless control loop reading desired state and issuing the same Redfish/iPXE/config calls you made by hand.

## Stack

```text
Lima VM
├── kind cluster                      <-- the management cluster
│   ├── CAPI core controllers
│   ├── Metal3 infrastructure provider
│   ├── Ironic (bare-metal provisioning service)
│   └── BareMetalHost CRs              <-- one per fake node
├── libvirt + QEMU                    <-- 2-3 fake nodes
├── network services                  <-- dnsmasq + HTTP
└── sushy-tools                       <-- the BMCs
```

The Lima VM hosts a Kubernetes management cluster (in kind/k3s) whose only job is to orchestrate the bare-metal nodes. The workload cluster you'll provision is reconstituted *by* the management cluster, your M4 `talosconfig` and `kubeconfig` are throwaway from this milestone forward; CAPI mints fresh ones as part of the declarative bootstrap.

> **Linkback to inventory (M3.5):** the `BareMetalHost` CRs you'll write here are exactly "strategy 1: manual inventory" from M3.5, with the same fields (BMC URL, credentials, boot MAC). If those feel hazy, re-skim M3.5's strategy table before you start.

## Reading list

| Topic | Source |
|---|---|
| Cluster API book | [Cluster API Book](https://cluster-api.sigs.k8s.io/) |
| CAPI core concepts | [CAPI: Concepts](https://cluster-api.sigs.k8s.io/user/concepts), Cluster, Machine, MachineDeployment, management vs. workload clusters |
| CAPI provider contracts | [CAPI: Developing providers](https://cluster-api.sigs.k8s.io/developer/providers/overview) and [InfraMachine contract](https://cluster-api.sigs.k8s.io/developer/providers/contracts/infra-machine) |
| Metal3 overview | [Metal3 introduction](https://book.metal3.io/) and [Bare Metal Operator](https://book.metal3.io/bmo/introduction.html) |
| Ironic under Metal3 | [Metal3: Ironic in Metal3](https://book.metal3.io/ironic/introduction.html) and [Ironic architecture](https://docs.openstack.org/ironic/latest/contributor/architecture.html) |
| BareMetalHost lifecycle | [Metal3: Host State Machine](https://book.metal3.io/bmo/state_machine) and [Provisioning/Deprovisioning](https://book.metal3.io/bmo/provisioning) |

## Success criteria

1. Management cluster (kind) running with CAPI core + Metal3 provider installed.
2. Three `BareMetalHost` CRs registered, each pointing at sushy-tools BMC URLs and credentials.
3. Watch one `BareMetalHost` go through the state machine: `registering` → `inspecting` → `available`. Observe Ironic driving Redfish under the hood.
4. Apply a `Cluster` + `KubeadmControlPlane` (or, for Talos, **`TalosControlPlane`** from [Sidero's CAPI control-plane provider](https://github.com/siderolabs/cluster-api-control-plane-provider-talos)) + `MachineDeployment` and watch the management cluster provision a workload cluster *automatically* across your three fake nodes.
5. Delete the workload cluster. Watch the BareMetalHosts deprovision and return to `available`.
6. **Reconciliation test**: while a node is `provisioned`, manually power it off via direct `virsh destroy`. Watch the controller notice and react. What does it do? (This is the most important exercise.)

## Conceptual questions

Each question has a **Read first** pointer to a source that teaches the underlying concept.

### 1. Why split Metal3 into CAPI provider + Ironic + IPA?

What does each component own, and why this division of labor?

**Read first:**
- [Metal3: Ironic in Metal3](https://book.metal3.io/ironic/introduction.html), describes Ironic's role inside Metal3 and the related services in that stack.
- [Metal3: Bare Metal Operator](https://book.metal3.io/bmo/introduction.html), documents the Kubernetes-side `BareMetalHost` API and controller responsibilities.
- [Ironic: Architecture](https://docs.openstack.org/ironic/latest/contributor/architecture.html), describes Ironic's service architecture, driver model, and node-management responsibilities.

### 2. Management cluster dies: are workload nodes orphaned?

**Read first:**
- [CAPI book: Pivoting (`clusterctl move`)](https://cluster-api.sigs.k8s.io/clusterctl/commands/move.html), documents the *mechanism* for moving Cluster API objects between management clusters.
- [CAPI: Glossary, pivot, management cluster, workload cluster](https://cluster-api.sigs.k8s.io/reference/glossary), names the *scenario* (workload clusters keep running on the data plane while their CAPI objects are moved). Pair the glossary's pivot definition with the `move` command above.
- [CAPI book: Concepts](https://cluster-api.sigs.k8s.io/user/concepts), documents the relationship between management clusters, workload clusters, and Cluster API objects.

### 3. Reconciliation cost on physical hardware

A software controller can re-reconcile every 10 seconds; a bare-metal controller cannot. What does this force the controller design to look like?

**Read first:**
- [Kubebuilder book: Controllers](https://book.kubebuilder.io/cronjob-tutorial/controller-overview), documents the reconcile loop pattern, requeue semantics, and retry/backoff behavior.
- [Metal3: Host State Machine](https://book.metal3.io/bmo/state_machine), the explicit state machine Metal3 uses for long-running host transitions.

### 4. Image management

Where does the OS image served to the node actually come from? Who builds it, where is it stored, and how does Metal3 reference it?

**Read first:**
- [Metal3: Provisioning and Deprovisioning](https://book.metal3.io/bmo/provisioning), documents the `image.url`, checksum, format, and deprovision/reprovision behavior on `BareMetalHost`.
- [Metal3: Ironic in Metal3](https://book.metal3.io/ironic/introduction.html), describes Ironic's role in the Metal3 provisioning path.

### 5. Locking: who owns a physical node?

**Read first:**
- [Metal3: Host State Machine](https://book.metal3.io/bmo/state_machine), documents the operational states where a host is available, provisioned, or being reclaimed.
- [CAPI: Machine controller](https://cluster-api.sigs.k8s.io/developer/core/controllers/machine), the Machine ↔ infrastructure object binding contract that all infrastructure providers implement.

### 6. vmetal mapping (preview for M6)

Read vmetal's docs and identify which of its CRDs map to CAPI/Metal3 primitives and which are net-new.

**Read first:**
- [vmetal Architecture](https://vmetal.ai/docs/architecture/) and [vmetal Configuration](https://vmetal.ai/docs/configuration/), start here before reading installed CRDs.
- [CAPI: Developing providers](https://cluster-api.sigs.k8s.io/developer/providers/overview) and [InfraMachine contract](https://cluster-api.sigs.k8s.io/developer/providers/contracts/infra-machine), the contract every infrastructure provider has to satisfy is the lens for reading vmetal's CRDs.

## What is NOT in this milestone

- No vmetal yet. M6.
- No tenant isolation or fleet management.
- No HA management cluster.

## Exit artifact

`lab/vmetal/milestone-05/`:

- `notes.md`, six conceptual answers.
- `manifests/`, the CRs you applied (BareMetalHost, Cluster, MachineDeployment, etc.).
- `reconciliation-trace.md`, what happened when you killed a node manually, with timeline + controller log excerpts.
- `capi-vs-vmetal.md`, first-pass map of CAPI/Metal3 primitives to vmetal concepts (you'll refine in M6).

---

Last Updated: 2026-04-29
