# Milestone 7: Day 2 Operations (optional)

**Goal:** Confront where most real fleet pain lives — not provisioning, but everything that happens after. Upgrades, decommission, credential rotation. The operations that have to work *while the cluster is in use* and that don't have a clean blank-slate fallback.

## What you're actually learning

- **Why day-2 is harder than day-0.** Day-0 has no users. Day-2 has SLOs, in-flight traffic, and a tenant who notices when their GPU job dies mid-epoch.
- **The image-rebuild + re-provision upgrade primitive.** This is how Metal3 — and therefore vmetal — actually upgrades a node. You don't `apt upgrade` in place; you build a new raw OS image, swap `BareMetalHost.spec.image.url`, let Metal3 deprovision and reprovision, and the node rejoins the cluster running the new image. That's the production model M5 set up; M7 exercises it.
- **Patterns that work at fleet scale:** drain-and-replace, rolling updates with surge, blue/green at the cluster level. CP rollout (drain-aware, one-at-a-time, quorum-bounded) vs. worker rollout (parallelism allowed up to PDB constraints).
- **The credential lifecycle problem.** BMC passwords, kubeadm join tokens, certificates, image checksums — every secret has an expiration, a rotation procedure, and a blast radius.
- **Where vmetal earns its keep** vs. where it offloads to the operator.

## Anchor question

> What's the smallest-impact way to change *anything* about a running fleet, and what's the cost of doing it wrong?

## Lab exercises

Pick at least two of 7a/7b/7c. The 7d firmware sketch is design-only — do it last and on paper. All four is overkill for a learning lab.

### 7a: OS Upgrade via image rebuild and re-provision

Upgrade the Ubuntu OS image on a 2-node cluster (1 CP, 1 worker) without losing the Pod running on the worker. The lab is small enough that you can watch the entire flow; the lessons scale to any size.

**The flow you're proving works:**

```text
1.  Build a new raw Ubuntu image (image N+1) with a different kernel
    or a different Kubernetes minor version baked in.
2.  Publish the new image to your HTTP server. Generate the sha256
    checksum file alongside it.
3.  `kubectl drain` the worker (or update the Metal3MachineTemplate
    that owns it, depending on which path you take).
4.  Update `BareMetalHost.spec.image.url` (or the Metal3MachineTemplate
    referenced by the MachineDeployment) to point at image N+1.
5.  Metal3 deprovisions the host: powers off, IPA wipes the disk,
    BMH transitions back to `available`.
6.  Metal3 re-provisions: IPA writes image N+1 to disk, materializes
    the same ConfigDrive (CABPK regenerates the bootstrap Secret
    against the live cluster's join state), node reboots.
7.  Cloud-init runs `kubeadm join` against the existing CP. Node
    rejoins the cluster, this time running image N+1.
8.  CP rollout follows the same flow but is sequential and respects
    quorum.
```

**Image build pipeline.** You need two distinct Ubuntu images. Options:

- **[image-builder](https://github.com/kubernetes-sigs/image-builder)**, the Cluster API project's image builder. Supports raw output, baked-in Kubernetes packages, customizable kernel selection. Closest to what AI Cloud production stacks use.
- **[Packer](https://www.packer.io/)** + an Ubuntu cloud-image base. More general-purpose; you write the provisioning script yourself.

Build image N+1 with a meaningful difference — bump the kernel (`linux-image-generic-hwe-…`), bump the Kubernetes minor (`kubelet`/`kubeadm`/`kubectl` v1.30 → v1.31, respecting upstream skew policy), or both. Capture the build artifact (the `.img` and its `.sha256`) under your HTTP server's image directory.

**Rollout semantics.**

- **Worker rollout:** parallelism allowed up to PodDisruptionBudget. Drain → image swap → reprovision → rejoin. PDB is the throttle.
- **CP rollout:** sequential, drain-aware, **bounded by etcd quorum tolerance.** A 3-node CP can lose one member and stay healthy. During a CP rebuild on a 3-node cluster you've reduced quorum tolerance to zero — *if a second CP fails for any reason*, the cluster is read-only until quorum returns. Discuss the implications. Now imagine you're rolling a 5-node CP, then a single-CP cluster (which has *no* tolerance — the cluster is unavailable for the duration).

**What to write up:**
- The image build (commands you ran, time it took, output).
- The exact resource you edited to trigger the rollout (BMH directly, or `Metal3MachineTemplate`, or `KubeadmControlPlane.spec.version` for a coordinated K8s upgrade).
- The Pod's behavior on the worker during the rebuild (was it evicted cleanly? did the deployment reschedule it onto the CP, given the lab is 2-node?).
- What you'd change at 100x scale.

**Forced detour to learn from:** intentionally point `BareMetalHost.spec.image.url` at a URL that returns 404, or whose `.sha256` doesn't match. Watch Metal3 fail the deploy. Identify the controller (BMO? CAPM3? Ironic?) that surfaces the error first, and the field on the BMH status that pins the failure.

### 7b: Node Decommission

Permanently remove `node02` from the fleet.

- What state on the node must be wiped before re-using the hardware? (Hint: `kubeadm reset`, then Ironic cleaning, then BMC password rotation if the hardware moves between trust zones.)
- What state in the management cluster must be cleaned up? (Machine, BareMetalHost, kubeadm join references, any CSRs, lingering Secrets.)
- What happens if you skip a step? (Try it. Observe the orphan state.)

### 7c: BMC Credential Rotation

Change the Redfish password on `node01`.

- Where is the password stored on the management-cluster side? (BMC credentials Secret referenced by `BareMetalHost.spec.bmc.credentialsName`.)
- What's the order of operations — change BMC first, then management cluster, or vice versa? What goes wrong with the wrong order?
- How do you do this on 1,000 nodes without an outage window?

### 7d: Firmware / Microcode Update (sketch only)

The fourth day-2 operation, included as a sketch because doing it on libvirt/sushy-tools is meaningless — there is no firmware to flash. But the *design* matters:

- M1 Q5 flagged "firmware update storms" as a scale-failure mode. Sketch the rollout strategy you'd use across 1,000 nodes: which fraction in flight, what defines "healthy after flash," what's the rollback?
- Where does the artifact live? (vendor repo, signed bundle, Redfish `UpdateService`?)
- Whose CR — vmetal's, Metal3's, vendor-specific — owns the desired state?
- What's the equivalent of `kubectl drain` for "the BMC is about to reboot for 8 minutes"?

Write the design as `firmware-rollout-design.md`; do not attempt to execute it.

## Reading list

| Topic | Source |
|---|---|
| image-builder for Cluster API | [Kubernetes SIGs: image-builder](https://github.com/kubernetes-sigs/image-builder), the canonical CAPI image builder; supports raw output for Metal3 |
| Metal3 image pivot | [Metal3: Provisioning and Deprovisioning](https://book.metal3.io/bmo/provisioning), specifically `image.url`, `image.checksum`, `image.format`, and the deprovision/reprovision triggered by changing them |
| Cluster API rolling updates | [CAPI: MachineDeployment controller](https://cluster-api.sigs.k8s.io/developer/core/controllers/machine-deployment) and [CAPI: upgrading clusters](https://cluster-api.sigs.k8s.io/tasks/upgrading-clusters), surge/maxUnavailable, KubeadmControlPlane rollout semantics |
| Kubernetes node drain | [Kubernetes: Safely Drain a Node](https://kubernetes.io/docs/tasks/administer-cluster/safely-drain-node/) |
| K8s version skew policy | [Kubernetes: Version skew policy](https://kubernetes.io/releases/version-skew-policy/), kubelet ≤ apiserver, three-minor-version window |
| etcd day-2 maintenance | [etcd: Maintenance](https://etcd.io/docs/v3.5/op-guide/maintenance/), compaction, defrag, snapshot cadence; closes the loop with M4's etcd-disaster-recovery reading |
| `kubeadm reset` | [Kubernetes: kubeadm reset](https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-reset/), the cleanup primitive for re-init/re-join during 7b decommission |
| Ironic cleaning | [Ironic: cleaning steps](https://docs.openstack.org/ironic/latest/admin/cleaning.html), automated + manual cleaning during deprovision |
| vmetal upgrade surface | [vmetal Configuration](https://vmetal.ai/docs/configuration/) and [vmetal Limitations](https://vmetal.ai/docs/limitations/), what vmetal opinionates on cluster upgrades vs. defers to CAPI; if the docs don't cover an upgrade primitive you expected, that's a *vmetal sharp edge* (see M6 `sharp-edges.md`) — record it in your 7a notes |

## Success criteria

For each exercise you choose:

1. Successful execution end-to-end with no manual intervention beyond `kubectl apply` (and the image build for 7a).
2. **One deliberate failure** induced mid-operation, with observation of the recovery (or lack thereof). For 7a: bad checksum, missing image URL, or `kubeadm join` failing on the rebuilt worker.
3. Written tradeoff analysis: what would change at 100x scale?

## Conceptual questions

Each question has a **Read first** pointer to a source that teaches the underlying concept.

### 1. Drain semantics

What does `kubectl drain` actually do, and when does it lie? (PodDisruptionBudgets, finalizers, stuck terminating Pods.)

**Read first:**
- [Kubernetes: Safely Drain a Node](https://kubernetes.io/docs/tasks/administer-cluster/safely-drain-node/), node drain behavior including PDB interaction.
- [Kubernetes: Pod Disruption Budgets](https://kubernetes.io/docs/concepts/workloads/pods/disruptions/), voluntary disruption limits that can block evictions.
- [Kubernetes: Finalizers](https://kubernetes.io/docs/concepts/overview/working-with-objects/finalizers/), how finalizers delay deletion until cleanup completes.

### 2. Image rebuild vs. in-place upgrade

Ubuntu *can* be `apt upgrade`'d in place. Metal3 / vmetal opts to rebuild the image and re-provision instead. Why? What does in-place upgrade buy you (faster, less data movement)? What does image-rebuild-and-replace buy you (clean state, deterministic outcome, drift elimination)? Which side of the tradeoff does production AI Cloud land on, and why?

**Read first:**
- [Metal3: Provisioning and Deprovisioning](https://book.metal3.io/bmo/provisioning), the documented lifecycle.
- [Google SRE Book: Configuration design](https://sre.google/workbook/configuration-design/), already cited in M3 — same lesson reapplied to images.

### 3. Upgrade ordering: control plane first or workers first?

What's the version-skew rule, and what's the failure mode if you reverse it? Walk through what specifically breaks if a worker's kubelet is newer than the apiserver.

**Read first:**
- [Kubernetes: Version skew policy](https://kubernetes.io/releases/version-skew-policy/), the canonical skew rules.
- [kubeadm: Upgrade](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/), the documented upgrade procedure for the kubeadm path you've been running since M4.

### 4. Quorum, capacity, and the cost of a CP rebuild

A 3-node CP tolerates 1 failure. During a CP rollout, you've taken the tolerance to 0 *for the duration of one node's rebuild*. What does that mean operationally? On a single-node CP (your lab), a CP rebuild means cluster downtime — for how long? What's the production answer? (Hint: 5-node CP, but it's not just "more nodes.")

**Read first:**
- [etcd: Clustering Guide](https://etcd.io/docs/v3.5/op-guide/clustering/), the quorum math.
- [Kubernetes: Considerations for large clusters](https://kubernetes.io/docs/setup/best-practices/cluster-large/), CP-sizing guidance.

### 5. Credential rotation atomicity

There's a window where the management cluster has the old BMC password and the BMC has the new one. How long is that window, and what happens if a Metal3 reconcile fires during it?

**Read first:**
- [Metal3: Bare Metal Operator](https://book.metal3.io/bmo/introduction.html), how `BareMetalHost` references BMC address and credential secret names.
- [Kubernetes: Secrets](https://kubernetes.io/docs/concepts/configuration/secret/), the storage primitive for credentials.
- [HashiCorp Vault: Database secrets engine, root credential rotation](https://developer.hashicorp.com/vault/docs/secrets/databases), not BMC-specific; an example of an external credential-rotation workflow worth contrasting against.

### 6. Decommission and identity

A node's MAC, BMC URL, and serial number are all "identity." When re-racking, which should change? What does Ironic clean by default, and what do you need to add to a cleaning step list to be safe across trust boundaries?

**Read first:**
- [Metal3: Provisioning and Deprovisioning](https://book.metal3.io/bmo/provisioning), the fields removed during deprovisioning and the transition back to `available`.
- [Ironic: cleaning steps](https://docs.openstack.org/ironic/latest/admin/cleaning.html), automated + manual cleaning, including device wipes.
- [Tinkerbell: Hardware](https://tinkerbell.org/docs/v0.22/concepts/hardware/), a different stack's hardware-metadata shape, useful as a contrast.

### 7. vmetal day-2 automation vs. operator responsibility

Reuse your M6 `sharp-edges.md`: which day-2 operations does vmetal opinionate, and which does it leave to you? Where does the answer change between Auto Nodes and Private Node Tenant Clusters?

**Read first:**
- [vmetal Configuration](https://vmetal.ai/docs/configuration/) and [vmetal Limitations](https://vmetal.ai/docs/limitations/), compared against what you did by hand in 7a/b/c.
- Your own M6 `sharp-edges.md`, the failure modes you found there often map directly to day-2 gaps.

### 8. Why is the BMC the most dangerous component to compromise?

What does an attacker with Redfish admin gain that an attacker with root on the host does not? Place this question in day-2 context: BMC credentials rotate (7c), firmware updates flow through `UpdateService` (7d), and decommission must wipe BMC state (7b). Each is a moment when the BMC's blast radius matters more than usual.

**Read first:**
- [Eclypsium: BMC&C, Lights Out Forever](https://eclypsium.com/research/bmcc-lights-out-forever/), concrete BMC-compromise impact: remote server control, firmware implanting, bricking.
- *Optional:* [NIST SP 800-193: Platform Firmware Resiliency Guidelines](https://csrc.nist.gov/pubs/sp/800/193/final), federal guidance on firmware resiliency.

## Exit artifact

`lab/vmetal/milestone-07/`:

- `notes.md`, eight conceptual answers.
- One subdirectory per exercise attempted. For 7a specifically:
  - `image-build/`: the build script and the two image artifacts (or pointers to where they live).
  - `rollout-walkthrough.md`: timestamps, commands, what reconciled when.
  - `failure-trace.md`: the deliberate failure you induced.
  - `scale-analysis.md`: what changes at 100x, with concrete numbers (image bandwidth, drain time, quorum windows).

When you finish this milestone, you have done — at small scale — the full lifecycle of a bare-metal Kubernetes fleet, and you can read any vmetal/Metal3/CAPI bug report and locate it on the map.

---

Last Updated: 2026-05-01
