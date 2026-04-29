# Milestone 7: Day 2 Operations (optional)

**Goal:** Confront where most real fleet pain lives, not provisioning, but everything that happens after. Upgrades, decommission, credential rotation. The operations that have to work *while the cluster is in use* and that don't have a clean blank-slate fallback.

**Expected: ~3h reading + ~6h lab per chosen exercise.**

## What you're actually learning

- **Why day-2 is harder than day-0.** Day-0 has no users. Day-2 has SLOs, in-flight traffic, and a tenant who notices when their GPU job dies mid-epoch.
- **The patterns that work**: drain-and-replace, rolling updates with surge, blue/green at the cluster level.
- **The credential lifecycle problem**: BMC passwords, join tokens, certificates, every secret has an expiration, a rotation procedure, and a blast radius.
- **Where vmetal earns its keep** vs. where it offloads to the operator.

## Anchor question

> What's the smallest-impact way to change *anything* about a running fleet, and what's the cost of doing it wrong?

## Lab exercises

Pick at least two of 7a/7b/7c. The 7d firmware sketch is design-only, do it last and on paper. All four is overkill for a learning lab.

### 7a: OS Upgrade

Upgrade Talos from version N to N+1 on a 2-node cluster (1 CP, 1 worker) without losing the Pod running on the worker.

- What's the upgrade primitive vmetal/CAPI exposes?
- What happens to the Pod during the worker upgrade?
- How does the control plane verify the new version is healthy before proceeding?

### 7b: Node Decommission

Permanently remove `node02` from the fleet.

- What state on the node must be wiped before re-using the hardware?
- What state in the management cluster must be cleaned up?
- What happens if you skip a step? (Try it. Observe the orphan state.)

### 7c: BMC Credential Rotation

Change the Redfish password on `node01`.

- Where is the password stored on the management-cluster side?
- What's the order of operations, change BMC first, then management cluster, or vice versa? What goes wrong with the wrong order?
- How do you do this on 1,000 nodes without an outage window?

### 7d: Firmware / Microcode Update (sketch only)

The fourth day-2 operation, included as a sketch because doing it on libvirt/sushy-tools is meaningless, there is no firmware to flash. But the *design* matters:

- M1 Q5 flagged "firmware update storms" as a scale-failure mode. Sketch the rollout strategy you'd use across 1,000 nodes: which fraction in flight, what defines "healthy after flash," what's the rollback?
- Where does the artifact live? (vendor repo, signed bundle, Redfish `UpdateService`?)
- Whose CR, vmetal's, Metal3's, vendor-specific, owns the desired state?
- What's the equivalent of `kubectl drain` for "the BMC is about to reboot for 8 minutes"?

Write the design as `firmware-rollout-design.md`; do not attempt to execute it.

## Reading list

| Topic | Source |
|---|---|
| Talos upgrade flow | [Talos: Upgrading Talos Linux](https://docs.siderolabs.com/talos/v1.12/configure-your-talos-cluster/lifecycle-management/upgrading-talos) |
| Kubernetes node drain | [kubernetes.io/docs/tasks/administer-cluster/safely-drain-node/](https://kubernetes.io/docs/tasks/administer-cluster/safely-drain-node/) |
| CAPI MachineDeployment rollout strategies | [CAPI: MachineDeployment controller](https://cluster-api.sigs.k8s.io/developer/core/controllers/machine-deployment) and [CAPI: upgrading clusters](https://cluster-api.sigs.k8s.io/tasks/upgrading-clusters) |
| etcd day-2 maintenance (compaction, defrag, snapshot cadence) | [etcd: Maintenance](https://etcd.io/docs/v3.5/op-guide/maintenance/), closes the loop with M4's etcd-disaster-recovery reading; steady-state hygiene is what keeps disaster recovery from being the only option |
| vmetal upgrade surface | [vmetal Configuration](https://vmetal.ai/docs/configuration/) and [vmetal Limitations](https://vmetal.ai/docs/limitations/), read for what vmetal opinionates on cluster upgrades vs. defers to CAPI; if the docs don't cover an upgrade primitive you expected, that's a *vmetal sharp edge* (see M6 `sharp-edges.md`), record it in your 7a notes |

## Success criteria

For each exercise you choose:

1. Successful execution end-to-end with no manual intervention beyond `kubectl apply`.
2. **One deliberate failure** induced mid-operation, with observation of the recovery (or lack thereof).
3. Written tradeoff analysis: what would change at 100x scale?

## Conceptual questions

Each question has a **Read first** pointer to a source that teaches the underlying concept.

### 1. Drain semantics

What does `kubectl drain` actually do, and when does it lie? (PodDisruptionBudgets, finalizers, stuck terminating Pods.)

**Read first:**
- [Kubernetes: Safely Drain a Node](https://kubernetes.io/docs/tasks/administer-cluster/safely-drain-node/), official Kubernetes task doc for node drain behavior, including PDB interaction.
- [Kubernetes: Pod Disruption Budgets](https://kubernetes.io/docs/concepts/workloads/pods/disruptions/), documents voluntary disruption limits that can block evictions during drain.
- [Kubernetes: Finalizers](https://kubernetes.io/docs/concepts/overview/working-with-objects/finalizers/), documents how finalizers delay deletion until cleanup completes.

### 2. Upgrade ordering: control plane first or workers first?

**Read first:**
- [Kubernetes: Version skew policy](https://kubernetes.io/releases/version-skew-policy/), documents supported component skew and upgrade-order constraints; current policy says kubelet must not be newer than kube-apiserver and may be up to three minor versions older.
- [kubeadm: Upgrade](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/), official kubeadm upgrade procedure and ordering.

### 3. Credential rotation atomicity

There's a window where the management cluster has the old password and the BMC has the new one. How long is that window, and what happens if a reconcile fires during it?

**Read first:**
- [Metal3: Bare Metal Operator](https://book.metal3.io/bmo/introduction.html), how `BareMetalHost` references BMC address and credential secret names.
- [Kubernetes: Secrets](https://kubernetes.io/docs/concepts/configuration/secret/), Kubernetes primitive used for storing credentials such as the BMC credential secret referenced by `BareMetalHost`.
- [HashiCorp Vault: Database secrets engine, root credential rotation](https://developer.hashicorp.com/vault/docs/secrets/databases), not BMC-specific; use it as an example of an external credential-rotation workflow.

### 4. Decommission and identity

A node's MAC, BMC URL, and serial number are all "identity." When re-racking, which should change?

**Read first:**
- [Metal3: Provisioning and Deprovisioning](https://book.metal3.io/bmo/provisioning), documents the fields removed during deprovisioning and the transition back to an available host.
- [Ironic: cleaning steps](https://docs.openstack.org/ironic/latest/admin/cleaning.html), documents automated and manual cleaning steps such as erasing devices and applying deploy/clean step operations.
- [Tinkerbell: Hardware](https://tinkerbell.org/docs/v0.22/concepts/hardware/), a different stack's documented hardware and network metadata shape, useful as a contrast to Metal3.

### 5. vmetal day-2 automation vs. operator responsibility

**Read first:**
- [vmetal Configuration](https://vmetal.ai/docs/configuration/) and [vmetal Limitations](https://vmetal.ai/docs/limitations/), compare documented behavior and limitations against what you did by hand in 7a/b/c.
- Your own M6 `sharp-edges.md`, the failure modes you found there often map directly to day-2 gaps.

### 6. Why is the BMC the most dangerous component to compromise?

What does an attacker with Redfish admin gain that an attacker with root on the host does not? Place this question in day-2 context: BMC credentials rotate (7c), firmware updates flow through `UpdateService` (7d), and decommission must wipe BMC state (7b). Each is a moment when the BMC's blast radius matters more than usual.

**Read first:**
- [Eclypsium: BMC&C, Lights Out Forever](https://eclypsium.com/research/bmcc-lights-out-forever/), concrete BMC-compromise impact: remote server control, firmware implanting, bricking.
- *Optional:* [NIST SP 800-193: Platform Firmware Resiliency Guidelines](https://csrc.nist.gov/pubs/sp/800/193/final), federal guidance on firmware resiliency, protection, detection, and recovery.

## Exit artifact

`lab/vmetal/milestone-07/`:

- `notes.md`, six conceptual answers.
- One subdirectory per exercise attempted, each with a walkthrough + failure log + scale analysis.

When you finish this milestone, you have done, at small scale, the full lifecycle of a bare-metal Kubernetes fleet, and you can read any vmetal/Metal3/CAPI bug report and locate it on the map.

---

Last Updated: 2026-04-29
