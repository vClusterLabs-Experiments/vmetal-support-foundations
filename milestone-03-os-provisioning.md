# Milestone 3: Immutable OS Provisioning

**Goal:** Take the network-boot pipeline from M2 and use it to deliver an actual operating system, Talos or Flatcar, configured declaratively from data served over HTTP. The node now has an identity (hostname, role, network config) without anyone ever logging in to set it up.

---

> ## 🛑 NOT AN ENDORSEMENT 🛑
>
> Nothing in this curriculum should be read as a recommendation or endorsement (implied or otherwise) of a particular vmetal architecture. The choices here exist to enforce a specific learning workflow. Some vmetal features (notably Auto Nodes and Private Node tenant clusters) have requirements and constraints that may differ from what this lab uses. For production guidance, consult the official documentation or your vCluster Labs contact.

---

## What you're actually learning

- **Immutable OS as a provisioning primitive**: why Talos and Flatcar exist, and why the immutable-image pattern keeps showing up in modern bare-metal stacks (alongside, not replacing, mutable-distro stacks like MAAS or generic Ironic flows).
- **Declarative node identity**: how a generic OS image becomes a *specific* node via config delivered at boot time.
- **The kernel command line as a control surface**: a kernel arg in the `talos.config*` family (e.g. `talos.config=`, `talos.config.inline=`) or `coreos.config.url=` is how you tell a generic kernel where to fetch its identity. Verify the exact arg against the Talos acquisition doc, the surface has expanded across recent releases.
- **Why disk-based persistence finally enters the picture**: ephemeral booting was fine for M2, but a Kubernetes node needs to remember things across reboots. What does it remember, and what stays ephemeral?

## Anchor question

> What's the smallest amount of declarative state needed to turn a generic OS image into a specific node?

**Hint:** look at the kernel-command-line family `talos.config*` (Talos) or `coreos.config.url=` (Flatcar/Ignition). The rest of node identity, hostname, network, disks, kubelet flags, certificates, flows from one document fetched via that arg. Verify with your lab; write the answer down only after you've proven it.

## Stack

```text
Lima VM
├── libvirt + QEMU
│   └── node01  (real disk that persists across reboots)
├── isolated network
│   ├── dnsmasq                <-- DHCP + TFTP
│   └── HTTP server
│       ├── ipxe.script        <-- points at a real kernel + initrd
│       ├── vmlinuz, initramfs <-- Talos or Flatcar
│       └── machine-config / ignition-config
└── sushy-tools                <-- the BMC
```

## Reading list

| Topic | Source |
|---|---|
| Talos overview and concepts | [Sidero: What is Talos Linux?](https://docs.siderolabs.com/talos/v1.12/overview/what-is-talos) |
| Talos machine config acquisition | [Talos: Acquiring Machine Configuration](https://docs.siderolabs.com/talos/v1.12/configure-your-talos-cluster/system-configuration/acquire) |
| Talos machine config reference | [Talos: configuration reference (v1.12)](https://docs.siderolabs.com/talos/v1.12/reference/configuration/v1alpha1/config), the v1alpha1 `Config` schema, the actual MachineConfig surface |
| Talos PXE boot | [Talos: PXE](https://docs.siderolabs.com/talos/v1.12/platform-specific-installations/bare-metal-platforms/pxe) |
| Flatcar Ignition | [flatcar.org/docs/latest/provisioning/ignition/](https://www.flatcar.org/docs/latest/provisioning/ignition/) |
| Ignition spec | [coreos.github.io/ignition/](https://coreos.github.io/ignition/) |

Pick one, Talos or Flatcar, and stick with it. Recommendation: **Talos**, because its API-only model forces you to confront the "no SSH" mindset immediately.

## Success criteria

1. Node PXE-boots into Talos/Flatcar using the iPXE pipeline from M2.
2. The node fetches its config from your HTTP server and applies it during first boot.
3. After first boot, the node has the hostname, IP, and disk layout you specified.
4. Power-cycle the node via Redfish; it comes back up with the same identity (state persisted to disk).
5. Change the config on the HTTP server, *re-provision* the node (wipe + PXE again), and see the new identity. Prove the disk has no hidden state surviving a wipe.
6. Inspect the node, for Talos, via `talosctl`; for Flatcar, via SSH, and locate where the config lives on disk.

## Conceptual questions

Each question has a **Read first** pointer to a source that teaches the underlying concept.

> **Stack note:** the readings below assume Talos (the recommended path). If you picked Flatcar, swap each Talos pointer for the corresponding Flatcar/Ignition doc, the *concept* under each question is identical.

### 1. Why immutable OS?

Why do Talos and Flatcar exist as separate distros instead of just "Ubuntu with cloud-init"? What's the operational argument for an immutable OS?

**Read first:**
- [Sidero: What is Talos Linux?](https://docs.siderolabs.com/talos/v1.12/overview/what-is-talos), Talos overview covering its API-managed model and immutable filesystem.
- [Flatcar: Getting Started](https://www.flatcar.org/docs/latest/installing) and [Flatcar Ignition](https://www.flatcar.org/docs/latest/provisioning/ignition/), Flatcar's install and provisioning entry points; use them as a second example of early-boot machine configuration.
- [Joel Spolsky: The Law of Leaky Abstractions](https://www.joelonsoftware.com/2002/11/11/the-law-of-leaky-abstractions/), frames *why* the operational case for an immutable OS is about removing config-drift surface area, not just "shiny new distro."

### 2. Plaintext config fetched at boot

The config document is fetched over HTTP at boot time, plaintext. What are the security implications, and what do production deployments do about it?

**Read first:**
- [Talos: Acquiring Machine Configuration](https://docs.siderolabs.com/talos/v1.12/configure-your-talos-cluster/system-configuration/acquire), Talos's documented config-acquisition methods and the order it checks them; the security tradeoffs become concrete here.

### 3. What persists on disk for a Kubernetes node?

Etcd data, kubelet certs, container images. What's safe to wipe on re-provision and what's not?

**Read first:**
- [Kubernetes: Components](https://kubernetes.io/docs/concepts/overview/components/), identifies the Kubernetes components involved, including etcd, kube-apiserver, kubelet, and container runtime.
- [Talos: Architecture](https://docs.siderolabs.com/talos/v1.12/learn-more/architecture), on the page, find `EPHEMERAL` and `STATE` for Talos's documented disk partition roles.

### 4. First-boot vs. subsequent boot detection

How does the OS know it's already been provisioned and shouldn't re-fetch + re-apply config? What disk artifact does it check?

**Read first:**
- [Talos: Acquiring Machine Configuration](https://docs.siderolabs.com/talos/v1.12/configure-your-talos-cluster/system-configuration/acquire), the acquisition order makes the first-boot vs. subsequent-boot distinction explicit, including the local `STATE` partition case.

### 5. Talos has no SSH and no shell: why?

Why is that an architectural choice, not a limitation? What does it force you to do differently when debugging?

**Read first:**
- [Sidero: What is Talos Linux?](https://docs.siderolabs.com/talos/v1.12/overview/what-is-talos), the API-managed section describes Talos's management model.
- [Talos: talosctl reference](https://docs.siderolabs.com/talos/v1.12/reference/cli), `talosctl dmesg`, `logs`, `read`, `restart`, inspect the API-driven debugging commands available without SSH.

### 6. Config-fetch storms at scale

500 nodes fetch their config from one HTTP server simultaneously. What's the failure mode, and what's the standard fix?

**Read first:**
- [iPXE: chainloading, HTTP delivery](https://ipxe.org/howto/chainloading), HTTP fetch with caching/CDN-fronting is the standard fix for config-fetch storms; TFTP-only does not survive.
- [MAAS: About images](https://maas.io/docs/about-images), MAAS's regional + rack-controller image distribution pattern; read it as a worked example of the "shard the fetch surface" answer.
- [Metal3: Provisioning and Deprovisioning](https://book.metal3.io/bmo/provisioning), Metal3's documented pattern for representing image URLs, checksums, image formats, and provisioning state (*what* is fetched, not how to scale it).

## What is NOT in this milestone

- No Kubernetes cluster yet, Talos boots and is ready to be a node, but no control plane has been bootstrapped.
- No second node.
- No declarative orchestration of provisioning, you're still triggering everything by hand via Redfish + edits to the HTTP-served config.

## Exit artifact

`lab/vmetal/milestone-03/`:

- `notes.md`, six conceptual answers.
- `machine-config.yaml` (or `ignition.json`), the config document, with comments explaining each section.
- `reprovision-walkthrough.md`, exact sequence to wipe + reboot + observe new identity.
- `scaffolding/`, HTTP server config, kernel cmdline used.

---

Last Updated: 2026-04-29
