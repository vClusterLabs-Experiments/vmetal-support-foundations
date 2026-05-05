# Milestone 1: BMC / Redfish

**Goal:** Drive a Redfish API to power one fake server on/off and read its current state. The contract you exercise here is what every BMC, real or simulated, exposes.

## What you're actually learning

- **The BMC as a separate compute element**: its own network, its own credentials, its own state, capable of controlling a host that is powered off or wedged.
- **The Redfish `Systems` resource**: how a server appears as an HTTP tree you can `GET` and act on.
- **Async control**: a Redfish `POST` is a request, not a command. State converges later, so the caller has to be a loop, not a one-shot RPC.

The lab tools mock the contract. Real iDRAC / iLO / OpenBMC implements the same contract. Anything you learn about the contract transfers; anything you learn about libvirt internals does not.

## Stack (disposable scaffolding)

```text
Mac (Apple Silicon)
└── Lima VM (Ubuntu)
    ├── libvirt + QEMU              <-- stands in for "the server"
    │   └── domain: node01           (diskless, NIC on isolated net, boot=network,hd)
    ├── isolated libvirt network     (no DHCP yet, pure L2)
    └── sushy-tools                  <-- stands in for "the BMC"
```

In production this whole box collapses into: a real server with a real BMC chip on a real management network. The Redfish API surface is identical.

## Apple Silicon constraint

No nested KVM, so inner VMs run under TCG (software emulation). Pick one guest arch and stick with it across all milestones. Recommendation: **x86_64**, because vmetal/Metal3/Ubuntu cloud-image docs assume amd64 by default. Slow but tolerable for 1–3 nodes.

## Pre-flight: scaffolding setup

This is one-time work; you'll reuse the same Lima VM through M7. None of it is the subject of study, get it working, then forget it. If you already have Lima with libvirt and sushy-tools running, skip ahead.

1. **Lima VM with Ubuntu.** Install Lima; launch an Ubuntu VM with enough disk and CPU for QEMU inside it.
   *Verify:* `limactl list` shows the VM `Running`; `limactl shell <name>` drops you into a shell.
   **Reference:** [lima-vm.io: Getting started](https://lima-vm.io/docs/), the minimum to launch one Ubuntu VM.
2. **libvirt + QEMU inside the VM.** Install via apt; enable and start `libvirtd`; add your user to the `libvirt` group.
   *Verify:* `virsh list --all` returns without errors.
   **Reference:** [libvirt.org](https://libvirt.org/docs.html), skim only the install + `virsh` basics.
3. **Isolated libvirt network + `node01` domain.** Define an isolated network (no DHCP, no NAT, pure L2) and a libvirt domain `node01` with one NIC on that network, no disk, and boot order `network,hd`.
   *Verify:* `virsh net-list` shows the network active; `virsh dumpxml node01` shows the NIC bound to it.
4. **sushy-tools bound to `node01`.** Install `sushy-tools`; configure the dynamic emulator to drive your libvirt domain via the libvirt URI.
   *Verify:* `curl -u <user>:<pass> http://<lima-ip>:<port>/redfish/v1/Systems/` returns the System tree containing `node01`.
   **Reference:** [sushy-tools dynamic emulator](https://docs.openstack.org/sushy-tools/latest/user/dynamic-emulator.html), the virtual Redfish BMC that drives a libvirt domain.

If step 4's `curl` returns a System tree, scaffolding is done and the BMC contract work below can begin.

## Reference index

| Topic | Source | Type |
|---|---|---|
| BMC vs host responsibility | [OpenBMC: Host management](https://github.com/openbmc/docs/blob/master/host-management.md), sensors, event logs, boot options, host power control, watchdog | (read) |
| Redfish `Systems` shape | [DMTF Published Mockups](https://redfish.dmtf.org/redfish/mockups/v1), open `public-rackmount1` and click through `Systems/` to see what an HTTP-rendered server looks like | (read) |
| Redfish state ownership | [Ironic: Redfish hardware type](https://docs.openstack.org/ironic/latest/admin/drivers/redfish.html), the boot-mode and virtual-media sections make BMC vs. host state ownership concrete | (read) |
| Async control as a control loop | [Kubernetes: Controllers](https://kubernetes.io/docs/concepts/architecture/controller/), the thermostat-analogy framing for every async API in the curriculum | (read) |
| sushy-tools (the mock you'll run) | [sushy-tools dynamic emulator](https://docs.openstack.org/sushy-tools/latest/user/dynamic-emulator.html), the virtual Redfish BMC that drives a libvirt domain | (reference) |
| Lima / libvirt scaffolding | [lima-vm.io](https://lima-vm.io/docs/) and [libvirt.org](https://libvirt.org/docs.html), skim only what you need to define one domain and one network | (reference) |

## Success criteria

From inside the Lima VM, you should be able to:

1. Show your fake server in the Redfish tree: `GET /redfish/v1/Systems/`.
   **Reference:** [DMTF Published Mockups](https://redfish.dmtf.org/redfish/mockups/v1), open `public-rackmount1` to see what a populated `Systems/` tree should look like.
2. Read its current `PowerState` via Redfish.
3. Power it on with a Redfish action; verify the libvirt domain is running.
4. Demonstrate idempotence: the same calls drive the host to the desired state from any starting state.

You may use `curl`, [HTTPie](https://httpie.io), or a small script. The point is the API, not the client.

> Boot-device override (`BootSourceOverrideEnabled = Once` vs. `Continuous`) is exercised at the start of M2, where PXE actually boots and you can observe the consequence. Don't tackle it here.

## Conceptual questions (write your answers down)

Three questions. Read the pointer, draft an answer, then run the lab checkpoint to confirm your model matches reality.

### 1. What does a BMC fundamentally provide that the host OS cannot, and why must it be a separate compute element?

Think about: the host is off, the host is wedged, the host's NIC driver crashed, someone needs to install an OS for the first time.

**Read first:** [OpenBMC: Host management](https://github.com/openbmc/docs/blob/master/host-management.md), sensors, event logs, boot options, host power control, watchdog behavior.

**Lab checkpoint:** `GET /redfish/v1/Systems/<id>`. Confirm `PowerState` reflects what `virsh list` shows. If they disagree, sushy-tools isn't bound to the right libvirt domain, fix before moving on.

### 2. What state does the BMC own vs. the host?

What lives on the BMC and persists across a host power cycle? What lives on the host? What survives a BMC reset?

**Read first:** [Ironic: Redfish hardware type](https://docs.openstack.org/ironic/latest/admin/drivers/redfish.html), read only the boot-mode and virtual-media sections; that's where state ownership becomes concrete.

**Lab checkpoint:** Read `Boot.BootSourceOverrideTarget`. Restart sushy-tools (`systemctl restart` or kill + relaunch). `GET` it again. Did the value persist? Power-cycle the libvirt domain. Did the value persist? The answers map directly to your written model.

### 3. Power actions are asynchronous. How should a consumer handle that?

A `POST` to `ComputerSystem.Reset` can return before the state change is complete. What *shape of caller* does an async API like this imply? Once you have the right mental model, every Redfish-specific detail (task monitors, `Retry-After`, polling cadence) is just instantiation.

**Hint:** think loop, not one-shot RPC.

**Read first:** [Kubernetes: Controllers (concepts)](https://kubernetes.io/docs/concepts/architecture/controller/), short prose explanation of the control-loop pattern with the thermostat analogy. Carry this forward to every async API in the curriculum.

**Lab checkpoint:** `POST` `ComputerSystem.Reset` with `ResetType=On`. Immediately poll `PowerState` every 1s and log timestamps until it reaches `On`. Record the convergence delay. Repeat from `On` → `ForceOff`. The polling loop you just wrote is what every controller in M5 does, scaled up.

## What is NOT in this milestone

- No DHCP, TFTP, HTTP server. The fake server will fail to boot, expected and correct.
- No OS image yet (cloud-init enters in M3).
- No second node. One is enough to learn the contract.
- No vmetal, Metal3, or Cluster API.

## Exit artifact

A directory `lab/vmetal/milestone-01/` containing:

- `notes.md`, your written answers to the three conceptual questions above. **This is the artifact that matters.**
- `redfish-power.sh`, the HTTP calls that satisfy the four success criteria, in order, with comments asserting what each one proves about the contract.
- `scaffolding/`, the libvirt XML, sushy-tools config, and any Lima setup notes. Kept for reproducibility but explicitly marked as throwaway; you will not study it for milestone 2.

When `notes.md` answers all three questions and `redfish-power.sh` runs cleanly end-to-end, you're ready for Milestone 2: network boot (PXE / DHCP / TFTP / iPXE), which opens with the boot-device override exercise deferred from here.

---

Last Updated: 2026-05-01
