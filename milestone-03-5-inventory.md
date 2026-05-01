# Milestone 3.5: Inventory and Discovery

**Required for M5.** Without this milestone you'll be hand-writing `BareMetalHost` CRs in M5 with no model for where they came from in production.

**Goal:** Confront the question every bare-metal stack glosses over: how does the orchestrator *learn* what hardware exists? Until now you've been hand-pointing tools at `node01`. Real fleets have thousands of nodes and no human typing them in.

## What you're actually learning

- **The discovery problem**: a server in a rack has no way to announce itself except by powering on and emitting DHCP. Everything else is manual data entry or out-of-band aggregation.
- **The three real-world strategies** and their tradeoffs:
  1. **Manual / ticket-driven**: ops files a CR per server. Accurate, slow, doesn't scale.
  2. **DHCP fingerprinting + auto-enrollment**: nodes PXE-boot into a discovery image that registers them. Scales, but trust boundary is fragile.
  3. **Redfish aggregation**: a chassis manager or top-of-rack controller exposes all BMCs through one API. Vendor-specific, common in modern AI Cloud builds.

## Anchor question

> If a server is bolted into a rack right now and nobody has touched a keyboard, what's the chain of events that makes it appear in the orchestrator's database?

## Lab exercise

Add a second fake node (`node02`) and a third (`node03`) to your libvirt setup. Then:

1. Implement **strategy 1** (manual): write a YAML inventory file by hand listing each node's MAC, BMC URL, and credentials. Feed it into a script that hits Redfish on each.
2. Implement **strategy 2** (auto-enrollment): boot a node into a tiny Linux image (Alpine + a script) that POSTs its MAC, CPU, memory, and disk info to your HTTP server. Watch the inventory grow without you typing anything.
3. Sketch (don't fully build) **strategy 3**: how would you aggregate three sushy-tools instances behind one Redfish endpoint? What does the data model look like? Note that in production, aggregation usually happens *above* Redfish, at a chassis manager (Dell OME, HPE OneView) or at the orchestrator (Metal3's Ironic conductor pool), not via a true Redfish `AggregationService`. Sketch the pure-Redfish version anyway; the contrast is the point.

## Reading list

| Topic | Source |
|---|---|
| Tinkerbell hardware data model | [Tinkerbell: Hardware](https://tinkerbell.org/docs/v0.22/concepts/hardware/), the hardware record shape for network and provisioning metadata |
| Tinkerbell workflow binding | [Tinkerbell: Workflows](https://tinkerbell.org/docs/v0.22/concepts/workflows/), how hardware gets bound to work |
| Metal3 BareMetalHost discovery | [Metal3: Bare Metal Operator](https://book.metal3.io/bmo/introduction.html), [Host State Machine](https://book.metal3.io/bmo/state_machine), and [Controlling Inspection](https://book.metal3.io/bmo/inspect_annotation) |
| MAAS commissioning | [MAAS: About commissioning machines](https://canonical.com/maas/docs/about-commissioning-machines), MAAS's documented commissioning flow for booting a temporary environment and gathering machine data |
| Redfish aggregation service | [DMTF Redfish Developer Hub](https://redfish.dmtf.org/) and [`AggregationService` schema](https://redfish.dmtf.org/schemas/v1/AggregationService.json), schema reference only |

## Success criteria

1. Three nodes registered in your manual inventory, all controllable via Redfish.
2. Auto-enrollment script: power on a node with no prior knowledge, see it appear in your inventory store within seconds.
3. Written tradeoff analysis: which strategy fits which operational reality (small lab, mid-size shop, AI Cloud at scale).

## Conceptual questions

Each question has a **Read first** pointer to a source that teaches the underlying concept.

### 1. Trust in auto-enrollment

In auto-enrollment, what stops a malicious DHCP responder on the network from registering fake nodes? What stops a real node from lying about its specs?

**Read first:**
- [Tinkerbell: Hardware](https://tinkerbell.org/docs/v0.22/concepts/hardware/), documents the fields Tinkerbell uses to describe hardware, network data, and provisioning metadata.
- [MAAS: About commissioning machines](https://canonical.com/maas/docs/about-commissioning-machines), documents MAAS's commissioning sequence: boot an ephemeral environment, collect hardware details, and move the machine toward "Ready."

*Deepen after M5:* [Metal3: Controlling Inspection](https://book.metal3.io/bmo/inspect_annotation), once you've seen `BareMetalHost` in M5, this doc shows how Metal3 requests, re-runs, or disables host inspection. Skip on first pass.

### 2. Identity stability when hardware changes

If a NIC is replaced, the MAC changes. How should an inventory system handle that?

**Read first:**
- [Ironic: Architecture](https://docs.openstack.org/ironic/latest/contributor/architecture.html), describes Ironic's separation of node lifecycle, driver interfaces, hardware management, and networking. The data-model thinking here transfers to any inventory you build.

*Deepen after M5:* [Metal3: Bare Metal Operator](https://book.metal3.io/bmo/introduction.html), `BareMetalHost` fields for BMC address, credential secret, boot MAC, and discovered host status. Most useful once you've actually run M5.

### 3. Enrollment race conditions

Two nodes boot simultaneously and try to enroll. Where are the ordering hazards?

**Read first:**
- [Kubernetes: optimistic concurrency](https://kubernetes.io/docs/reference/using-api/api-concepts/), on the page, find "resource version" for Kubernetes's version-based update and conflict behavior.
- [Tinkerbell: Workflow](https://tinkerbell.org/docs/v0.22/concepts/workflows/), documents how Tinkerbell binds work to hardware and tracks workflow execution state.

### 4. Schema drift over time

BMC firmware gets upgraded and the Redfish API exposes new fields. How does inventory stay current?

**Read first:**
- [DMTF Redfish Profiles](https://www.dmtf.org/standards/profiles), interoperability profiles define the *contract* a fleet of heterogeneous BMC firmware versions must hold to; this is the stable mechanism for managing schema drift across BMC vendors.
- [Redfish schemas index](https://redfish.dmtf.org/schemas/v1/), versioned schema bundle; read to see how DMTF tags compatibility across schema revisions.
- [OpenBMC: Redfish cheatsheet](https://github.com/openbmc/docs/blob/master/REDFISH-cheatsheet.md), open-source BMC perspective on the practical Redfish surface exposed by an implementation, useful as a contrast to closed-firmware vendor Redfish surfaces.

## Exit artifact

`lab/vmetal/milestone-03-5/`:

- `notes.md`, answers + the tradeoff analysis.
- `inventory.yaml`, your manual data model.
- `enroll.sh`, the auto-enrollment script.
- `aggregation-sketch.md`, your design for strategy 3.

---

Last Updated: 2026-05-01
