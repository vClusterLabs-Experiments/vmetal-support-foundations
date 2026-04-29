# Milestone 4: Cluster Bootstrap (by hand)

**Goal:** Turn the provisioned Talos node from M3 into a working single-node Kubernetes cluster, then PXE-provision a second node and join it as a worker. No operators, no Cluster API. You drive every step.

**Expected: ~4h reading + ~6h lab.**

## What you're actually learning

- **The handoff between "node booted" and "node joined a cluster"**: what data crosses that boundary, who owns it, and what fails when it's wrong.
- **The control plane bootstrap chicken-and-egg**: the first node has no cluster to join, so it has to declare itself one. What state does it generate, and where does that state live?
- **Worker join as a trust operation**: a worker isn't just an OS that runs kubelet, it's an OS with credentials proving it belongs.
- **Why every cluster orchestrator** (CAPI, kubeadm, kops, Talos `omni`) is doing variations on what you're about to do manually.

## Anchor question

> What's the difference between "a node booted" and "a node joined a cluster," and who owns the data that bridges that gap?

**Hint:** something specific crosses that boundary. Figure out *what shape* it has, then *who owns it*, both answers depend on the bootstrap provider.

### Reference: the two stack flavours you'll meet

- **Talos (the stack you're running):** the worker's *machine config* (delivered via the M3 pipeline) already carries the cluster CA bundle, the API server endpoint, and a node-specific machine certificate signed by the cluster's machine-CA. There is **no kubeadm-style join token** in the Talos flow, `talosctl gen config` pre-generates the secrets.
- **kubeadm (the comparison stack):** the worker carries a short-lived **bootstrap token** + the cluster's CA bundle + the API server endpoint, and uses the kubelet TLS bootstrap flow (CSR → controller-manager signs → kubelet client cert).

Both flavours cross the same conceptual boundary; they fill it with different artefacts. Resist looking at the reference until after you've drafted your own answer.

## Stack

```text
Lima VM
├── libvirt + QEMU
│   ├── node01  (control plane)
│   └── node02  (worker, PXE-provisioned cold during this milestone)
├── network services           <-- dnsmasq + HTTP from M2
└── sushy-tools                <-- the BMC
```

## Reading list

| Topic | Source |
|---|---|
| Talos cluster bootstrap | [Talos: Getting Started](https://docs.siderolabs.com/talos/v1.12/getting-started/getting-started) |
| `talosctl` commands | [Talos: CLI reference](https://docs.siderolabs.com/talos/v1.12/reference/cli) |
| Kubernetes cluster components | [kubernetes.io/docs/concepts/overview/components/](https://kubernetes.io/docs/concepts/overview/components/) |
| kubeadm join flow (*comparison only*, your stack is Talos) | [kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-join/](https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-join/), read this to see the bootstrap-token + CSR flow that Talos *replaces* with machine-config-embedded certs |

## Success criteria

1. `node01` running a single-node Kubernetes control plane via Talos. `kubectl get nodes` from your Lima host returns it as Ready.
2. PXE-provision `node02` from cold (Redfish power-on → DHCP → iPXE → Talos → worker config).
3. `node02` joins the cluster and shows up as Ready.
4. Run a Pod (`kubectl run`) and confirm it lands on the worker.
5. Reboot the worker via Redfish. It rejoins automatically.
6. **Break it on purpose**: provide the worker with the wrong cluster CA in its config. Watch the join fail. Identify where in the boot/join flow the failure surfaces and what the symptom looks like.

## Conceptual questions

Each question has a **Read first** pointer to a source that teaches the underlying concept.

### 1. Bootstrap chicken-and-egg

The first control plane node needs etcd to start, but etcd needs a config that includes its own peer URL. How does Talos resolve this on first boot? Where does the initial cluster state come from?

**Read first:**
- [Talos: Getting Started](https://docs.siderolabs.com/talos/v1.12/getting-started/getting-started), read the `talosctl bootstrap` step and what Talos says it initializes.
- [etcd: Clustering Guide, new cluster](https://etcd.io/docs/v3.5/op-guide/clustering/), the "Static" and "Discovery" sections frame the same problem generically.

### 2. Join credentials: authority and blast radius

What authority does the credential a worker uses to join grant, and what's the blast radius if it leaks? How long should it live?

> **Stack note:** the *kind* of credential differs by stack, kubeadm uses a short-lived **bootstrap token**; Talos embeds machine-specific certs in the machine config. Read both pointers and then map them onto your Talos run.

**Read first:**
- [Talos: Secrets / `talosctl gen secrets`](https://docs.siderolabs.com/talos/v1.12/reference/cli), the Talos cluster secrets and machine-config-embedded credentials your stack actually uses.
- [Kubernetes: Bootstrap Tokens](https://kubernetes.io/docs/reference/access-authn-authz/bootstrap-tokens/) *(comparison, kubeadm-stack)*, official Kubernetes doc on TTL, group membership, and what a token authorizes.
- [kubeadm: TLS Bootstrapping](https://kubernetes.io/docs/reference/access-authn-authz/kubelet-tls-bootstrapping/) *(comparison, kubeadm-stack)*, documents the bootstrap-token-to-kubelet-client-certificate flow.

### 3. Kubelet certificates

A worker needs a kubelet client cert signed by the cluster CA. Who generates it, and at what point in the boot flow?

> **Stack note:** Talos pre-generates the kubelet's identity material as part of the machine config rather than running the upstream Kubernetes CSR flow at boot. Read the upstream CSR doc to understand the *general* shape of the problem, then read Talos's machine-config secrets to see how Talos answers it.

**Read first:**
- [Talos: Acquiring Machine Configuration](https://docs.siderolabs.com/talos/v1.12/configure-your-talos-cluster/system-configuration/acquire), how Talos delivers the per-node identity that kubelet ultimately uses.
- [Kubernetes: Kubelet TLS Bootstrapping](https://kubernetes.io/docs/reference/access-authn-authz/kubelet-tls-bootstrapping/) *(comparison, kubeadm-stack)*, the full CSR flow: token → CSR → controller-manager signs → cert.
- [Kubernetes: Certificate Signing Requests](https://kubernetes.io/docs/reference/access-authn-authz/certificate-signing-requests/) *(comparison, kubeadm-stack)*, the API object that backs the flow.

### 4. Second control plane node vs. second worker

What's different about adding another control plane node?

**Read first:**
- [Talos: Production Clusters](https://docs.siderolabs.com/talos/v1.12/getting-started/prodnotes), documents Talos production-cluster guidance for control plane, endpoints, and load balancing.
- [kubeadm: HA topology](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/high-availability/), official diagrams for stacked-etcd and external-etcd HA topologies.

### 5. Control plane permanently lost: what's recoverable?

The worker boots, joins, runs Pods. Then the control plane node dies permanently. What state is recoverable, and from where?

**Read first:**
- [etcd: Disaster Recovery](https://etcd.io/docs/v3.5/op-guide/recovery/), documents etcd snapshot restore, the core recovery path for Kubernetes control-plane state.
- [Talos: etcd Maintenance](https://docs.siderolabs.com/talos/v1.12/build-and-extend-talos/cluster-operations-and-maintenance/etcd-maintenance), Talos-specific backup/snapshot operations built on the etcd primitives.

### 6. Cluster join at scale

100 workers join simultaneously. What part of the control plane is the bottleneck?

**Read first:**
- [Kubernetes: Considerations for large clusters](https://kubernetes.io/docs/setup/best-practices/cluster-large/), official scaling guidance for large-cluster control-plane components.
- [kube-apiserver: API priority and fairness](https://kubernetes.io/docs/concepts/cluster-administration/flow-control/), documents request classification, queuing, and concurrency controls the API server can use under load.

## What is NOT in this milestone

- No HA control plane (one node).
- No CNI deep-dive, use whatever Talos defaults to.
- No persistent workloads, no storage class.
- No declarative orchestration. Every action is manual.
- No production-grade certificate or token rotation, that's M7.

## Exit artifact

`lab/vmetal/milestone-04/`:

- `notes.md`, six conceptual answers.
- `bootstrap-walkthrough.md`, exact sequence from cold node01 to 2-node cluster.
- `talosconfig` and `kubeconfig`, saved for later milestones.
- `failure-trace.md`, what happened with the wrong CA and how you diagnosed it.

---

Last Updated: 2026-04-29
