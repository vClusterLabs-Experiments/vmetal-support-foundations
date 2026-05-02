# Milestone 4: Cluster Bootstrap (by hand)

**Goal:** Turn the provisioned Ubuntu node from M3 into a working single-node Kubernetes cluster via `kubeadm init`, install a CNI, then PXE-provision a second Ubuntu node and join it as a worker via `kubeadm join`. cloud-init delivers everything `kubeadm` needs to succeed; you drive `kubeadm` itself. No operators, no Cluster API.

The cloud-init payload you wrote in M3 is the primitive. M4 extends it with the kubeadm prerequisites (containerd, swap off, kernel modules, sysctls, `pkgs.k8s.io` apt repo, `kubelet`/`kubeadm`/`kubectl`) and the `runcmd` line that actually invokes `kubeadm init` on the control plane and `kubeadm join` on the worker. Same delivery mechanism (NoCloud over HTTP), bigger payload.

## What you're actually learning

- **The handoff between "node booted" and "node joined a cluster."** What data crosses that boundary, who owns it, and what fails when it's wrong.
- **The control-plane bootstrap chicken-and-egg.** The first node has no cluster to join, so `kubeadm init` has to *manufacture* one, generate CAs, write static pod manifests, start a single-member etcd, surface a bootstrap token. Where does that state live, and what shape does it have?
- **kubeadm's phase model.** `kubeadm init` is not one step; it's eleven. Reading the phase output is how you triage a half-finished install.
- **Worker join as a trust operation.** A worker isn't just an OS that runs kubelet, it's an OS with credentials proving it belongs. Bootstrap token + CA hash + kubelet TLS bootstrap → signed kubelet client cert. Each piece has a name; learn them.
- **Where cloud-init ends and Kubernetes begins.** `cloud-init final` runs your `runcmd` line, which invokes `kubeadm`, which configures and starts kubelet, which talks to containerd. After that, cloud-init is done and Kubernetes owns the runtime. Re-running cloud-init on a working cluster member is a real ticket pattern; understand why it's bad.
- **CNI as a separate decision from `kubeadm`.** `kubeadm init` brings up the API server, etcd, scheduler, controller-manager, and `kube-proxy`. It does NOT install pod networking. Until a CNI is installed, no pod can run. Calico is the lab choice, locked.

## Anchor question

> What's the difference between "a node booted" and "a node joined a cluster," and who owns the data that bridges that gap?

**Hint:** the boundary is *not* `kubeadm join`, it's earlier. By the time `kubeadm join` runs, the worker already has the apiserver endpoint, the cluster's CA hash, and a short-lived bootstrap token. Where did each of those three come from, and which one is actually the *secret* that makes the join work?

## Stack

```text
Lima VM
├── libvirt + QEMU
│   ├── node01  (control plane, kubeadm init)
│   └── node02  (worker, PXE-provisioned cold during this milestone, kubeadm join)
├── network services           <-- dnsmasq + HTTP from M2/M3
└── sushy-tools                <-- the BMC
```

## Reading list

| Topic | Source |
|---|---|
| kubeadm init reference | [Kubernetes: kubeadm init](https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-init/), the phase model and the documented set of artifacts each phase produces |
| kubeadm join reference | [Kubernetes: kubeadm join](https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-join/), the worker side of the handshake |
| kubeadm config (v1beta4) | [Kubernetes: kubeadm config](https://kubernetes.io/docs/reference/config-api/kubeadm-config.v1beta4/), `ClusterConfiguration`, `InitConfiguration`, `JoinConfiguration` schema |
| Bootstrap tokens | [Kubernetes: Bootstrap tokens](https://kubernetes.io/docs/reference/access-authn-authz/bootstrap-tokens/), TTL, group membership, what the token authorizes |
| Kubelet TLS bootstrap | [Kubernetes: Kubelet TLS bootstrapping](https://kubernetes.io/docs/reference/access-authn-authz/kubelet-tls-bootstrapping/), the bootstrap-token → CSR → signed kubelet client cert flow |
| Node Authorizer | [Kubernetes: Using Node Authorization](https://kubernetes.io/docs/reference/access-authn-authz/node/), the RBAC mode that scopes a kubelet to its own node |
| Container runtime install | [Kubernetes: Container runtimes](https://kubernetes.io/docs/setup/production-environment/container-runtimes/), containerd config, cgroup driver, `SystemdCgroup`, CRI socket |
| K8s apt repository | [Kubernetes: Install kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/), the `pkgs.k8s.io` per-minor-version repo (legacy `apt.kubernetes.io` is frozen, do not teach it) |
| Calico Quick Start | [Calico: Quickstart for Calico on Kubernetes](https://docs.tigera.io/calico/latest/getting-started/kubernetes/quickstart), the manifest install, including the pod CIDR `192.168.0.0/16` |
| `kubeadm reset` | [Kubernetes: kubeadm reset](https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-reset/), the cleanup primitive for re-init/re-join |
| etcd disaster recovery (preserved from M4 prior) | [etcd: Disaster Recovery](https://etcd.io/docs/v3.5/op-guide/recovery/), snapshot + restore |

## Success criteria

1. **`node01` is a single-node Kubernetes control plane via `kubeadm init`.** Cloud-init's `runcmd` invoked `kubeadm init --config=/etc/kubernetes/kubeadm-init.yaml` on first boot. From the Lima host (or from `node01` itself with the admin kubeconfig), `kubectl get nodes` shows `node01` `Ready` once Calico finishes coming up.
2. **Calico is installed.** `kubectl get pods -n kube-system` and `kubectl get pods -n calico-system` (or `kube-system`, depending on the manifest you applied) show the Calico controller and per-node `calico-node` pods Running. `node01` flips to `Ready` only after Calico's CNI plugin lands on disk.
3. **PXE-provision `node02` from cold.** Redfish power-on → DHCP → iPXE → Ubuntu netboot → autoinstall → reboot → cloud-init `runcmd` invokes `kubeadm join`. The worker's user-data is the M3 payload extended with the `kubeadm join` line, including the API server endpoint, the bootstrap token, and the discovery CA cert hash you got from the control plane.
4. **`node02` joins the cluster and shows up as `Ready`.** `kubectl get nodes` shows two nodes. `kubectl run` schedules a Pod that lands on the worker.
5. **Reboot the worker via Redfish.** It rejoins automatically, kubelet uses the kubeconfig `kubeadm join` wrote, NOT the bootstrap token (which was one-shot, single-use, short-lived). Verify by watching `kubectl get nodes` go `NotReady` then `Ready` without you touching tokens.
6. **Break it on purpose, three ways:**
   - **Wrong CA hash.** Edit the worker's user-data to use a `--discovery-token-ca-cert-hash sha256:0000…0000`. Reprovision. Watch `kubeadm join` fail. Identify the line in `cloud-init-output.log` and the kubeadm error that pinpoints "the discovery server's CA does not match the expected hash."
   - **Expired token.** Wait 24h (or set `--ttl 5m` and wait), then try to join a new node with the stale token. Identify the failure path.
   - **Missing CNI.** Skip the Calico install on a fresh `kubeadm init`. Watch `node01` stay `NotReady`. `kubectl describe node node01` should call out the missing pod network. Install Calico; watch the node go `Ready`.

## Reference: the cloud-init payload that makes `kubeadm init` succeed

Your control-plane user-data is the M3 payload extended with the kubeadm prereqs, the apt repo install, the kubeadm config file, and the `runcmd` line that invokes `kubeadm init`. You'll write this in full and serve it from your HTTP seed server.

```yaml
#cloud-config
autoinstall:
  version: 1
  identity:
    hostname: node01
    username: ubuntu
    password: "$6$rounds=4096$REPLACE_ME$..."
  ssh:
    install-server: true
    allow-pw: false
    authorized-keys:
      - ssh-ed25519 AAAA...your-key...
  storage:
    layout:
      name: direct
  packages:
    - apt-transport-https
    - ca-certificates
    - curl
    - gpg
    - containerd
  late-commands:
    - curtin in-target --target=/target -- systemctl enable containerd
  user-data:
    timezone: UTC
    hostname: node01
    bootcmd:
      - swapoff -a
    write_files:
      - path: /etc/modules-load.d/k8s.conf
        content: |
          overlay
          br_netfilter
      - path: /etc/sysctl.d/k8s.conf
        content: |
          net.bridge.bridge-nf-call-iptables  = 1
          net.bridge.bridge-nf-call-ip6tables = 1
          net.ipv4.ip_forward                 = 1
      - path: /etc/kubernetes/kubeadm-init.yaml
        content: |
          ---
          apiVersion: kubeadm.k8s.io/v1beta4
          kind: ClusterConfiguration
          kubernetesVersion: v1.35.3
          controlPlaneEndpoint: "node01.lab:6443"
          networking:
            podSubnet: "192.168.0.0/16"
            serviceSubnet: "10.96.0.0/12"
          ---
          apiVersion: kubeadm.k8s.io/v1beta4
          kind: InitConfiguration
          nodeRegistration:
            criSocket: "unix:///var/run/containerd/containerd.sock"
    runcmd:
      # Disable swap permanently
      - sed -ri '/\sswap\s/s/^#?/#/' /etc/fstab
      # Kernel modules + sysctls
      - modprobe overlay
      - modprobe br_netfilter
      - sysctl --system
      # containerd: write default config (CRI enabled by default in `containerd config default`),
      # flip cgroup driver to systemd. We'll align sandbox_image after kubeadm is installed.
      - mkdir -p /etc/containerd
      - containerd config default > /etc/containerd/config.toml
      - sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
      # Kubernetes apt repo (pinned to v1.35; bump per the version you're teaching).
      - install -d -m 0755 /etc/apt/keyrings
      - curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.35/deb/Release.key | gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
      - echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.35/deb/ /' > /etc/apt/sources.list.d/kubernetes.list
      - apt-get update
      - apt-get install -y kubelet kubeadm kubectl
      - apt-mark hold kubelet kubeadm kubectl
      # Now that kubeadm is on disk, ask it which pause image this K8s minor wants
      # and pin sandbox_image to match. Avoids a kubelet warning + per-node pull on first boot.
      # Order matters: this must run AFTER `apt-get install kubeadm` and BEFORE `systemctl restart containerd`.
      - PAUSE_IMG=$(kubeadm config images list --kubernetes-version=v1.35.3 | grep pause)
      - sed -i "s#sandbox_image = .*#sandbox_image = \"${PAUSE_IMG}\"#" /etc/containerd/config.toml
      - systemctl restart containerd
      # kubeadm init
      - kubeadm init --config=/etc/kubernetes/kubeadm-init.yaml
      # CNI (Calico), apply with the admin kubeconfig kubeadm wrote
      - KUBECONFIG=/etc/kubernetes/admin.conf kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.30.0/manifests/calico.yaml
```

The worker user-data is structurally the same up through the apt-install line, but ends with `kubeadm join` driven by a `JoinConfiguration` (so `criSocket` is pinned the same way the CP pins it via `InitConfiguration`):

```yaml
    write_files:
      - path: /etc/kubernetes/kubeadm-join.yaml
        content: |
          ---
          apiVersion: kubeadm.k8s.io/v1beta4
          kind: JoinConfiguration
          discovery:
            bootstrapToken:
              token: "<token-from-CP>"
              apiServerEndpoint: "node01.lab:6443"
              caCertHashes:
                - "sha256:<hash-from-CP>"
          nodeRegistration:
            criSocket: "unix:///var/run/containerd/containerd.sock"
    runcmd:
      # ... (same prereqs as above through `apt-mark hold`) ...
      - kubeadm join --config=/etc/kubernetes/kubeadm-join.yaml
```

You get `<token>` and `<hash>` by running, on `node01` after `kubeadm init` has completed:

```bash
kubeadm token create --print-join-command
```

Paste the output into the worker user-data, serve it, then PXE the worker. The token has a 24-hour default TTL, so do this close to when you'll boot the worker. (Production stacks regenerate the token on demand for each join, that's exactly what CABPK does in M5.)

### Five details worth highlighting

- **`controlPlaneEndpoint` day 1.** Set it on the first `kubeadm init`, even on a single-CP cluster. Adding it later requires regenerating apiserver cert SANs and updating kubeconfigs across the cluster. `node01.lab:6443` works in this lab if dnsmasq resolves it; an IP works too.
- **CRI socket.** kubeadm auto-detects when only one runtime is installed. Setting `nodeRegistration.criSocket` explicitly removes ambiguity for the lab and matches what real ops teams pin.
- **Cgroup driver.** kubeadm defaults kubelet to the `systemd` cgroup driver. containerd MUST match (`SystemdCgroup = true`). Mismatch produces a kubelet that crashloops with vague kubelet logs and `kubeadm init` running well past the preflight check before failing, costly to diagnose if you don't know the trap.
- **Container runtime cgroup config differs across containerd 1.x and 2.x.** The `sed` line above targets containerd 1.x. On containerd 2.x the `[plugins.'io.containerd.cri.v1.runtime'.containerd.runtimes.runc.options]` block is what you edit. Pin the containerd version in your reading list and stay consistent.
- **CNI is a separate manifest apply.** Until Calico's manifest is applied and `calico-node` reaches `Ready`, `node01` itself will report `NotReady` (the kubelet condition `NetworkPluginNotReady` blocks it). This is normal. Watch the transition once so you recognize it.

## Conceptual questions

Each question has a **Read first** pointer to a source that teaches the underlying concept.

### 1. Bootstrap chicken-and-egg

`kubeadm init` runs on a node that has no cluster, no API server, no etcd. By the time it finishes, it has all three. What sequence of artifacts does `kubeadm` create, in order, to climb out of that hole? What's the role of static-pod manifests in `/etc/kubernetes/manifests/` and the `kubeadm-config` ConfigMap?

**Read first:**
- [Kubernetes: kubeadm init](https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-init/), the documented phase list (`preflight`, `certs`, `kubeconfig`, `control-plane`, `etcd`, `upload-config`, `upload-certs`, `mark-control-plane`, `bootstrap-token`, `kubelet-finalize`, `addon`).
- [Kubernetes: Static Pods](https://kubernetes.io/docs/tasks/configure-pod-container/static-pod/), the kubelet-managed pod mechanism kubeadm uses to run the apiserver, controller-manager, scheduler, and etcd before any cluster exists.

### 2. Bootstrap token: lifetime, scope, blast radius

What does a kubeadm bootstrap token authenticate as? What does it authorize? What's the default TTL and why is it that short? If a token leaks, what's the worst a malicious actor can do with it before it expires?

**Read first:**
- [Kubernetes: Bootstrap tokens](https://kubernetes.io/docs/reference/access-authn-authz/bootstrap-tokens/), the token format, the `system:bootstrappers` group, and the RBAC bindings that gate it.

### 3. Kubelet TLS bootstrapping: who signs the cert?

The worker authenticates to the apiserver with a bootstrap token long enough to file a CSR. The CSR is signed, the kubelet gets a long-lived client cert, and the bootstrap token is no longer used. Walk the flow: who files the CSR, who approves, who signs, where does the signed cert land on disk, and what RBAC binding makes auto-approval possible?

**Read first:**
- [Kubernetes: Kubelet TLS bootstrapping](https://kubernetes.io/docs/reference/access-authn-authz/kubelet-tls-bootstrapping/), the canonical end-to-end description.
- [Kubernetes: Using Node Authorization](https://kubernetes.io/docs/reference/access-authn-authz/node/), the authorization mode that scopes a kubelet to its own node and the pods bound to it.

### 4. cloud-init + kubeadm composition

Your `runcmd` line invokes `kubeadm init`. Why isn't there a declarative cloud-init module for `kubeadm`, the way there is for `users`, `apt`, or `write_files`? What does that say about kubeadm's design, and what's the cost of mixing imperative `runcmd` with the otherwise-declarative cloud-config? What happens if cloud-init re-runs (new `instance-id`) on a node that's already a working cluster member?

**Read first:**
- [cloud-init: Module frequency](https://cloudinit.readthedocs.io/en/latest/reference/modules.html), the per-instance vs. per-boot semantics that govern whether `runcmd` re-runs.
- [Kubernetes: kubeadm reset](https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-reset/), the cleanup that a re-`kubeadm-init`-on-an-existing-member would need to precede.

### 5. etcd disaster recovery

The control-plane node dies permanently, disk lost, BMC unreachable. What state is recoverable, and from where? What was the *last moment* you needed to have done something to make recovery possible? Sketch the snapshot-restore flow and the inputs it requires.

**Read first:**
- [etcd: Disaster Recovery](https://etcd.io/docs/v3.5/op-guide/recovery/), the snapshot + restore primitive that backs every "rebuild a Kubernetes control plane" recipe.
- [Kubernetes: Backing up an etcd cluster](https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/#backing-up-an-etcd-cluster), the kubeadm-flavored wrapper.

### 6. Forward pointer: how does CAPI replace what you just did?

In M5, you'll watch CAPI's `KubeadmControlPlane` provider stand up an HA control plane without anyone running `kubeadm init` by hand. CAPI doesn't replace kubeadm, it *drives* it. Specifically: Cluster API Bootstrap Provider Kubeadm (CABPK) reads a `KubeadmConfig` (or `KubeadmConfigTemplate`) and *generates a cloud-init user-data document* that runs the same `kubeadm init` / `kubeadm join` you ran. Where in this milestone's flow does CABPK slot in? What does it replace, and what stays the same?

**Read first:**
- [Cluster API: Bootstrap Provider Kubeadm](https://cluster-api.sigs.k8s.io/tasks/bootstrap/kubeadm-bootstrap.html), the official description of how CABPK turns a `KubeadmConfig` into cloud-init user-data.
- [Cluster API Book: Kubeadm-based control planes](https://cluster-api.sigs.k8s.io/developer/architecture/controllers/control-plane.html), the controller's role in lifecycle-managing CP `kubeadm init` and `kubeadm join --control-plane`.

## What is NOT in this milestone

- No HA control plane (one CP node). The `controlPlaneEndpoint` field is set day 1 anyway; that's the point.
- No declarative orchestration. Every action is manual or driven by your own user-data. CAPI/Metal3 is M5.
- No persistent workloads, no storage class, no LoadBalancer.
- No production-grade certificate or token rotation. That's M7.
- No image pre-baking. You're installing kubeadm/kubelet/kubectl from the apt repo on every boot. M7 introduces the rebuild-the-image pattern that AI Cloud production uses.
- No CNI deep dive. Calico is locked; we install it from the upstream manifest and move on. Why Calico over Flannel: matches AI Cloud production expectations and Cluster API + Metal3 examples.
- No Tenant Cluster, no vCluster, no Tenant Isolation work. M6 introduces vmetal's Tenant Cluster model on top of the kubeadm cluster you're building here.

## Exit artifact

`lab/vmetal/milestone-04/`:

- `notes.md`, your written answers to the six conceptual questions.
- `bootstrap-walkthrough.md`, exact sequence from cold `node01` (no disk) to a 2-node cluster with Calico and a Pod scheduled on the worker.
- `seed/cp/user-data` and `seed/worker/user-data`, the two user-data documents you actually served. Comment the `runcmd` line that invokes `kubeadm` and the line that applies Calico.
- `kubeconfig` (admin), saved for later milestones.
- `failure-trace.md`, the three induced failures (wrong CA hash, expired token, missing CNI) with symptom, log line that pinpointed cause, and remediation.
- `phase-walkthrough.md` (optional but recommended), an annotated transcript of `kubeadm init`'s phase output with one sentence per phase explaining what it did.

When `notes.md` answers all six questions, both nodes are `Ready`, a Pod is scheduled on the worker, and you've recovered the cluster from an induced failure, you're ready for **Milestone 4.5: Observability and Debugging**, which formalizes the diagnostic surfaces you've been using ad-hoc in M3 and M4.

---

Last Updated: 2026-05-01
