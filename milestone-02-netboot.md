# Milestone 2: Network Boot (PXE / DHCP / TFTP / iPXE)

**Goal:** Take the diskless server from Milestone 1 and make it boot something, anything, off the network. You're not installing an OS yet. The conceptual payoff is understanding how a machine that knows nothing about itself except its MAC address ends up executing code you served it.

---

> ## 🛑 NOT AN ENDORSEMENT 🛑
>
> Nothing in this curriculum should be read as a recommendation or endorsement (implied or otherwise) of a particular vmetal architecture. The choices here exist to enforce a specific learning workflow. Some vmetal features (notably Auto Nodes and Private Node tenant clusters) have requirements and constraints that may differ from what this lab uses. For production guidance, consult the official documentation or your vCluster Labs contact.

---

## What you're actually learning

- **The PXE handshake**: how a NIC ROM, with no disk and no config, finds a bootloader on a network it has never seen before.
- **Why iPXE exists**: the original PXE ROM is a 1990s artifact with no scripting, no HTTP, no TLS. iPXE is the modern bootloader you chainload into so the rest of the stack is sane.
- **The DHCP-as-control-plane pattern**: DHCP is not just "give me an IP." Options 66/67 (and later, 175 for iPXE) turn DHCP into the first dispatch mechanism in the boot chain.
- **The contract that the BMC's `boot=PXE` from Milestone 1 actually invokes.** You set a flag in Redfish; here you see what that flag *causes*.
- **Why every bare-metal orchestrator** (vmetal, Metal3, Tinkerbell, MAAS) ships a DHCP/TFTP/HTTP stack, or assumes one exists. This is the layer they live on.

## The smallest-state question

The anchor question for this milestone:

> What is the smallest amount of state a server needs to know about itself before it can boot?

**Hint:** the NIC's burned-in MAC address is the only fact the firmware has at power-on. Prove the rest can be delivered over the wire; if a node had to know anything else first, you couldn't provision it from zero.

## Stack

```text
Lima VM (Ubuntu)
├── libvirt + QEMU                     <-- the diskless server (node01)
├── isolated libvirt network
│   ├── dnsmasq                        <-- DHCP + TFTP in one process
│   │   ├── DHCP: hands out IP + boot filename
│   │   └── TFTP: serves the initial iPXE binary
│   └── HTTP server (nginx / python -m http.server)
│       ├── iPXE script (the "what to do next" instructions)
│       └── test payload (kernel/initrd or netboot.xyz menu)
└── sushy-tools                        <-- the BMC
```

The BMC and the server are the same scaffolding you built in M1. The new layer is the network services that exist *between* power-on and "something is running."

## The boot chain you're proving works

```text
1. Redfish: PATCH boot device = Pxe          (you, via curl)
2. Redfish: POST ComputerSystem.Reset = On   (you, via curl)
3. QEMU powers on the VM                     (libvirt)
4. NIC ROM does PXE: broadcasts DHCPDISCOVER (firmware)
5. dnsmasq replies with IP + next-server + boot filename
6. NIC ROM TFTPs the iPXE binary             (firmware)
7. iPXE runs, does its OWN DHCP, and is dispatched to a script URL via iPXE-aware DHCP signalling, most commonly option 175 (encapsulated iPXE options) *or* a `user-class` match on the string `iPXE`. Either route works; pick one and be consistent.
8. iPXE HTTP-fetches the script
9. Script tells iPXE to HTTP-fetch a kernel + initrd
10. iPXE boots into them
```

Steps 1–2 are Milestone 1. Steps 3–10 are this milestone. Every layer above (OS install, cluster bootstrap, vmetal) is a variation on step 9.

## Reading list

| Topic | Source |
|---|---|
| PXE/DHCP boot options | [RFC 4578](https://datatracker.ietf.org/doc/html/rfc4578) and [RFC 5970](https://datatracker.ietf.org/doc/html/rfc5970), use as protocol references, not first-pass tutorials |
| iPXE: what and why | [iPXE documentation](https://ipxe.org/docs), start with the "Starting iPXE" and "DHCP" how-tos |
| iPXE scripting | [ipxe.org/scripting](https://ipxe.org/scripting), the language is tiny, learn all of it |
| iPXE chainloading | [ipxe.org/howto/chainloading](https://ipxe.org/howto/chainloading), the PXE-to-iPXE handoff |
| iPXE DHCP server config | [iPXE: Using ISC dhcpd](https://ipxe.org/howto/dhcpd), examples for chainloading, user-class detection, and option 175 |
| dnsmasq as DHCP+TFTP | `man dnsmasq`, focus on `--dhcp-boot`, `--dhcp-match`, `--enable-tftp`, `--tftp-root` |
| DHCP options 66/67 and iPXE option 175 | [RFC 2132](https://datatracker.ietf.org/doc/html/rfc2132) for 66/67, [iPXE dhcpd guide](https://ipxe.org/howto/dhcpd) for option 175, and [iPXE user-class](https://ipxe.org/cfg/user-class) for detecting stage 2 |
| Production DHCP/TFTP/iPXE service | [Tinkerbell: Smee](https://tinkerbell.org/docs/v0.22/services/smee/), Tinkerbell's DHCP and TFTP service for network boot |
| netboot.xyz (optional, useful test payload) | [netboot.xyz](https://netboot.xyz) |

## Success criteria

1. **Boot-device override (deferred from M1).** Set `Boot.BootSourceOverrideTarget=Pxe` via Redfish with `BootSourceOverrideEnabled=Once`, power-cycle, observe one PXE attempt, and confirm the override clears. Repeat with `Continuous` and confirm the override survives multiple cycles. The flag's downstream consequence, actually doing PXE, is what the rest of this milestone proves.
2. Power-cycle `node01` via Redfish (Milestone 1 still works) and watch it PXE-boot end-to-end into an iPXE prompt or a netboot.xyz menu.
3. Capture the DHCP exchange (`tcpdump -i virbr-prov -n port 67 or port 68`) and identify each of the four packets: DISCOVER, OFFER, REQUEST, ACK. Identify which DHCP options carry the boot instructions.
4. Demonstrate the **two-stage DHCP**: the firmware's first DHCP, then iPXE's second DHCP after it loads. Explain why iPXE asks again instead of reusing the first lease.
5. Serve a custom iPXE script over HTTP that prints a unique string and halts. Prove the node executed *your* script, not a default.
6. Change one byte of your iPXE script, power-cycle the node, and see the change reflected, proving the node holds no boot state of its own between cycles.
7. Break it on purpose: stop the TFTP server, power-cycle, observe the failure mode in the QEMU console. Then stop dnsmasq entirely and observe the different failure mode. You should be able to diagnose either failure from the symptoms alone.

## Conceptual questions

Each question has a **Read first** pointer to a source that teaches the underlying concept.

### 1. Why does iPXE do its own DHCP after being loaded by PXE?

What information does iPXE need that the firmware DHCP didn't necessarily provide, and why is reusing the first lease a bad idea?

**Read first:**
- [iPXE: chainloading](https://ipxe.org/howto/chainloading), documents the PXE-to-iPXE handoff and DHCP configuration from iPXE after the handoff.
- [iPXE: `ifconf` command](https://ipxe.org/cmd/ifconf), documents the command iPXE uses to configure a network interface, including DHCP.
- [iPXE: Using ISC dhcpd](https://ipxe.org/howto/dhcpd), option 175 and user-class examples you will mirror in dnsmasq.

### 2. DHCP option 67 vs. iPXE's two-step

DHCP option 67 says "boot filename", but you're serving iPXE, which then loads a *different* file (the script). Why this two-step? What goes wrong if you try to skip iPXE and have the firmware PXE ROM load your kernel directly?

**Read first:**
- [iPXE: chainloading](https://ipxe.org/howto/chainloading), documents the two-stage PXE ROM to iPXE flow and the richer boot methods available after iPXE is loaded.
- [RFC 5970](https://datatracker.ietf.org/doc/html/rfc5970), UEFI/PXE DHCP options. Skim the option definitions to see what firmware bootstrapping gets from DHCP.

### 3. No persistent state on the node

The node has no disk and no persistent config. What guarantees that two consecutive power cycles produce identical boot behavior? What would you have to change to make boot behavior vary per-node?

**Read first:**
- [iPXE: scripting](https://ipxe.org/scripting), note the documented `${mac}`, `${serial}`, and `${uuid}` variables used in scripts.
- [Tinkerbell: Hardware data model](https://tinkerbell.org/docs/v0.22/concepts/hardware/), documents the hardware record shape Tinkerbell uses for per-node network and provisioning metadata.

### 4. DHCP trust on a shared L2 segment

DHCP is broadcast-based and trusts the first reply. What does that imply for security in a shared L2 segment? How do AI Cloud operators prevent rogue DHCP servers from hijacking provisioning?

**Read first:**
- [Wikipedia: DHCP snooping](https://en.wikipedia.org/wiki/DHCP_snooping), short, vendor-neutral explanation of trusted-port / untrusted-port semantics and the binding table.
- [RFC 7610: DHCPv6-Shield](https://datatracker.ietf.org/doc/html/rfc7610), the IPv6 analog, codified. Same threat model, same defense pattern.

### 5. PXE provisioning at scale

You provision 500 nodes simultaneously. They all PXE-boot at once. What part of this stack melts first, and what's the standard mitigation?

**Read first:**
- [iPXE: chainloading](https://ipxe.org/howto/chainloading), on the page, find "HTTP" for examples of fetching later-stage boot assets over HTTP after the PXE handoff. The HTTP-fetch route is what survives a thundering herd; TFTP-only does not.
- [MAAS: About images](https://maas.io/docs/about-images), MAAS distributes images via per-rack streams to limit the blast radius of a simultaneous boot wave; read for the "regional + rack controller" pattern that Foreman, MAAS, and Tinkerbell all converge on.
- [Tinkerbell: Smee](https://tinkerbell.org/docs/v0.22/services/smee/), documents Tinkerbell's DHCP and network-boot service; pair with the MAAS doc to see two takes on the same scale problem.

### 6. Redfish handoff: where does `boot=Pxe` actually live?

When the BMC says `boot=Pxe`, where does that flag actually live, and how does the NIC ROM know to honor it on next power-on? Is this a UEFI thing, a BIOS thing, or both?

**Read first:**
- [Ironic: Redfish driver, boot mode and management](https://docs.openstack.org/ironic/latest/admin/drivers/redfish.html), focus on the boot-mode and virtual-media sections. Production driver context for `BootSourceOverrideTarget`, `BootSourceOverrideEnabled`, `Pxe`, and vendor interoperability notes, short, concrete, and the place real implementations get this wrong.
- [sushy-tools source](https://opendev.org/openstack/sushy-tools), optional source dive for the simulator implementation if you want to trace how this mock handles Redfish System state.
- [Talos: Bare-metal PXE installation](https://docs.siderolabs.com/talos/v1.12/platform-specific-installations/bare-metal-platforms/pxe), Talos's documented PXE flow, the M3-onwards stack you'll exercise next.

## What is NOT in this milestone

- No real OS installed to disk. The node boots into iPXE / a test payload and stops.
- No per-node configuration. Every node would boot identically right now. (Milestone 3 fixes this with MAC-based dispatch.)
- No HTTPS, no signed boot. (Milestone 5+ when we look at supply-chain concerns.)
- No Talos / Flatcar / ignition yet. That's Milestone 3.

## Exit artifact

A directory `lab/vmetal/milestone-02/` containing:

- `notes.md`, answers to the six conceptual questions. The durable artifact.
- `boot-chain-trace.md`, annotated `tcpdump` output showing the DHCP exchange, with each option called out and explained in your own words.
- `ipxe-script.ipxe`, the custom script you served, with comments explaining each line.
- `failure-modes.md`, what you saw when you broke TFTP vs. broke DHCP, and how you'd diagnose each from symptoms alone.
- `scaffolding/`, dnsmasq config, HTTP server config, any iPXE build notes. Throwaway.

When all of that exists and you can power-cycle the node and watch your custom iPXE script run end-to-end, you're ready for Milestone 3: immutable OS provisioning (Talos or Flatcar via iPXE + ignition/machine config over HTTP).

---

Last Updated: 2026-04-29
