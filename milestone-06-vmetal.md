# Milestone 6: vmetal

**Goal:** Install vmetal on the management cluster from M5 and recognize every component. Map each of its CRDs and controllers to the layers below it. Articulate exactly what vmetal *adds* over CAPI + Metal3, and why those additions exist.

---

> ## 🛑 NOT AN ENDORSEMENT 🛑
>
> Nothing in this curriculum should be read as a recommendation or endorsement (implied or otherwise) of a particular vmetal architecture. The choices here exist to enforce a specific learning workflow. Some vmetal features (notably Auto Nodes and Private Node tenant clusters) have requirements and constraints that may differ from what this lab uses. For production guidance, consult the official documentation or your vCluster Labs contact.

---

## What you're actually learning

- **vmetal's value-add is the delta**, not the foundation. The foundation is the same Redfish/PXE/ignition/CAPI/Metal3 stack you built by hand.
- **The AI Cloud-specific concerns** that motivate that delta: GPU inventory, tenant isolation, image catalogs, scheduling across heterogeneous hardware, day-2 lifecycle.
- **How to read a product** by mapping its surface area onto primitives you already understand.

## Anchor question

> What does vmetal add on top of CAPI + Metal3, and why does each addition exist?

You should leave this milestone able to answer that crisply, with concrete examples for each addition. That's the whole point.

## Stack

```text
Lima VM
├── kind management cluster
│   ├── CAPI + Metal3              <-- the foundation vmetal builds on
│   ├── vmetal controllers
│   └── vmetal CRDs
├── libvirt + QEMU
└── sushy-tools
```

## Reading list

| Topic | Source |
|---|---|
| vmetal docs | [vmetal docs](https://vmetal.ai/docs) |
| vmetal architecture | [vmetal Architecture](https://vmetal.ai/docs/architecture/), study the diagram before installing |
| vmetal configuration | [vmetal Configuration](https://vmetal.ai/docs/configuration/) and [Limitations](https://vmetal.ai/docs/limitations/), read these before deciding what is product behavior vs. lab constraint |
| vmetal installed CRDs | Runtime source: after install, run `kubectl api-resources | grep -i vmetal`, `kubectl get crd | grep -i vmetal`, and `kubectl explain <kind>.spec --recursive` before reading controller code |

Read the docs *before* installing. Form a hypothesis about what each CRD does based on its name and schema. Then install and check whether the runtime behavior matches your hypothesis. Mismatches are the most valuable learning signal.

## Success criteria

1. vmetal installed cleanly on the M5 management cluster.
2. Hand-drawn or text diagram showing every running pod and its role.
3. **CRD mapping table**: for each vmetal CRD, identify whether it (a) replaces a CAPI/Metal3 primitive, (b) wraps one, or (c) is net-new with no analog. Justify each classification.
4. Provision a workload cluster *via vmetal* using the same fake nodes from M5. Compare the manifest you wrote to the M5 manifest, what changed, what got simpler, what got more opinionated?
5. Tear it down. Re-provision with a different config (different K8s version, different node roles). Note what vmetal makes easy that was painful in M5.
6. **Find a sharp edge**: induce a failure that vmetal handles, and one it doesn't. The latter is your map of where the abstraction leaks.

## vmetal vocabulary

vmetal sits on top of CAPI/Metal3 and inherits vCluster lineage for cluster topology. Two terms you should hold in your head before reading the conceptual questions, they map onto, but are not the same as, the upstream CAPI "management cluster / workload cluster" pair you used in M5:

- **Control Plane Cluster**, the management cluster from M5, recast as the cluster that hosts vmetal's controllers and the virtual control planes for tenants. (Don't say "host cluster.")
- **Tenant Cluster**, the cluster a tenant consumes. Each Tenant Cluster has its own Kubernetes API surface with isolation enforced by vmetal + vCluster underneath. (Don't say "virtual cluster" or "sharing cluster"; the **Virtual Control Plane** is the *component* that runs the tenant's API server, not the cluster itself.)
- **Tenant Isolation**, the operational property vmetal aims for: that one Tenant Cluster cannot see, schedule onto, or interfere with another's nodes, GPUs, or data plane. (Don't say "multi-tenancy.")
- **AI Cloud**, the operator/customer environment vmetal targets: bare-metal GPU fleets serving multiple tenants. (Don't say "neocloud.")

Carry these terms forward through the conceptual answers below.

## Conceptual questions

Each question has a **Read first** pointer. vmetal docs are the primary source for this milestone, every question grounds in them, but where helpful, a contrast source (CAPI/Metal3, NVIDIA, etc.) is included.

### 1. AI Cloud–specific concerns vs. generic CAPI+Metal3

What does vmetal address that a generic install doesn't? (GPU SKUs, NVLink topology, tenant isolation, image catalogs.)

**Read first:**
- [vmetal Architecture](https://vmetal.ai/docs/architecture/) and [vmetal Configuration](https://vmetal.ai/docs/configuration/), product architecture overview and documented configuration surfaces.
- [NVIDIA: GPU Operator](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/index.html), Kubernetes-side GPU operator documentation to use as a comparison point for vmetal's bare-metal layer.
- [NVIDIA: Topology-Aware GPU Selection](https://docs.nvidia.com/datacenter/nvtags/latest/nvtags-user-guide/index.html), NVIDIA's documentation for topology-aware GPU selection concepts and tooling.

### 2. Tenant isolation boundary

Where does "this fleet belongs to tenant X" live, physical, network, namespace, RBAC, all of the above?

**Read first:**
- [vmetal Configuration](https://vmetal.ai/docs/configuration/), review which isolation-related settings are explicitly documented in vmetal config, then compare that with Kubernetes-native controls.
- [Kubernetes: Multi-tenancy](https://kubernetes.io/docs/concepts/security/multi-tenancy/), Kubernetes guidance on tenant isolation across control plane, data plane, namespaces, and policy.

### 3. Image management: vmetal vs. raw Metal3

**Read first:**
- [vmetal Configuration](https://vmetal.ai/docs/configuration/), look for image, cluster, or node configuration surfaces actually exposed by the product.
- [Metal3: Provisioning and Deprovisioning](https://book.metal3.io/bmo/provisioning), the Metal3 baseline you're comparing against (already read in M5).

### 4. Day-2 lifecycle ownership

Which lifecycle events does vmetal opinionate (upgrades, decommission, firmware updates) and which does it leave to the operator?

**Read first:**
- [vmetal Limitations](https://vmetal.ai/docs/limitations/) and [vmetal Configuration](https://vmetal.ai/docs/configuration/), use documented product scope and exposed configuration to separate vmetal-owned automation from operator-owned work.
- [CAPI: MachineDeployment controller](https://cluster-api.sigs.k8s.io/developer/core/controllers/machine-deployment) and [CAPI: upgrading clusters](https://cluster-api.sigs.k8s.io/tasks/upgrading-clusters), generic Cluster API rollout and upgrade references for comparison.

### 5. Where does the abstraction leak?

This is empirical, answer from your observed failure. The reading is to ground "is this a fundamental limit or a roadmap gap?"

**Read first:**
- [vmetal Limitations](https://vmetal.ai/docs/limitations/), use documented limits first; if you need release-level behavior, inspect the installed chart/image versions alongside the public docs.
- [Joel Spolsky: The Law of Leaky Abstractions](https://www.joelonsoftware.com/2002/11/11/the-law-of-leaky-abstractions/), short, classic framing for leaky abstractions.

### 6. Customer evaluation: vmetal vs. roll-your-own

When does the abstraction earn its weight, and when is it overhead?

**Read first:**
- [vmetal Architecture](https://vmetal.ai/docs/architecture/), [Configuration](https://vmetal.ai/docs/configuration/), and [Limitations](https://vmetal.ai/docs/limitations/), note the documented architecture, configuration surfaces, and stated limitations.
- Re-read your own M5 `notes.md` and `reconciliation-trace.md`, the most honest answer comes from comparing the manual experience to vmetal's, not from any single doc.

## What is NOT in this milestone

- No production-realistic scale.
- No real GPUs (you're on Apple Silicon under TCG, there is no GPU passthrough story here).
- No multi-region.

## Exit artifact

`lab/vmetal/milestone-06/`:

- `notes.md`, six conceptual answers.
- `crd-mapping.md`, the table mapping vmetal CRDs to CAPI/Metal3 primitives.
- `pod-inventory.md`, what's running and why.
- `m5-vs-m6-diff.md`, manifests compared, opinions called out.
- `sharp-edges.md`, the failures vmetal handles vs. doesn't.

When you can sit in front of a customer and answer "what does vmetal do that I couldn't do myself, and why should I care?" without hedging, you're done.

---

Last Updated: 2026-04-29
