# vmetal Learning Lab

A bottom-up path to understanding the full stack vmetal runs on. The goal is conceptual mastery, not a working vmetal install — though you'll have one by the end. Each milestone uses disposable scaffolding (libvirt + sushy-tools on a Lima VM) to teach contracts that transfer to real hardware.

## How to use this

- Work milestones in order. Each assumes the prior one's exit artifact exists.
- Read each milestone's `## Reading list` and navigate the linked docs yourself. The briefs intentionally do not include copy-pasteable commands.
- The durable artifact at every milestone is `notes.md` — your written answers to the conceptual questions. Configs and scripts are throwaway.
- Each milestone declares an expected reading and lab budget; ignore links beyond it.
- Only M7 is optional; the `.5` interludes (M3.5, M4.5) are required — M5 depends on both.

## Milestones

| # | File | One-line summary | Time |
|---|---|---|---|
| 1 | [milestone-01-bmc.md](./milestone-01-bmc.md) | Drive a Redfish API to power a fake server on/off. Learn the BMC contract. | ~9h (incl. ~4h one-time scaffolding setup) |
| 2 | [milestone-02-netboot.md](./milestone-02-netboot.md) | PXE / DHCP / TFTP / iPXE. Boot a diskless node off the network. | ~10h |
| 3 | [milestone-03-os-provisioning.md](./milestone-03-os-provisioning.md) | Deliver Talos or Flatcar with a config-over-HTTP. Generic image becomes a specific node. | ~8h |
| 3.5 | [milestone-03-5-inventory.md](./milestone-03-5-inventory.md) | How does the orchestrator learn what hardware exists? | ~6h |
| 4 | [milestone-04-cluster-bootstrap.md](./milestone-04-cluster-bootstrap.md) | Bring up single-node K8s and join a PXE-provisioned worker, by hand. | ~10h |
| 4.5 | [milestone-04-5-observability.md](./milestone-04-5-observability.md) | Induce failures across the chain. Build diagnostic runbooks. | ~5h |
| 5 | [milestone-05-capi-metal3.md](./milestone-05-capi-metal3.md) | Replace every manual step with declarative CRs. CAPI + Metal3 + Ironic. | ~13h |
| 6 | [milestone-06-vmetal.md](./milestone-06-vmetal.md) | Install vmetal. Map every CRD to a layer below. Articulate the delta. | ~8h |
| 7 | [milestone-07-day2.md](./milestone-07-day2.md) | *(optional)* Upgrades, decommission, credential rotation. Where real fleet pain lives. | ~9h per exercise |

## Why this order

The path is **bottom-up by abstraction layer**: BMC → network boot → OS → cluster → declarative orchestration → product. Each layer's contract is a black box to the layer above and a learned skill at its own level. By the time you install vmetal in M6, every component it spins up corresponds to a layer you've operated by hand.

The two `.5` interludes are not enrichment — they are load-bearing. M5 will assume you've thought about inventory (M3.5) and built diagnostic instinct for the boot chain (M4.5). Skipping them makes M5 noticeably harder to reason about.

## Apple Silicon notes

- All work happens inside one Lima VM running Ubuntu.
- Inner VMs run x86_64 under TCG (no nested KVM on M-series). Slow but workable for 1–3 fake nodes.
- Don't try to scale node count past what TCG can comfortably handle; the lab is for understanding the contract, not perf.

## Directory layout

The curriculum (briefs + learner-workspace template) lives at the repo root. Your active work goes under `lab/vmetal/`, which is gitignored — copy `_template/` to start.

```text
.
├── README.md                            (this file)
├── milestone-01-bmc.md                  (brief)
├── milestone-02-netboot.md              (brief)
├── ...
├── milestone-07-day2.md                 (brief)
├── _template/                           (learner workspace skeleton — copy this)
│   ├── README.md
│   ├── milestone-01/notes.md
│   └── ...
└── lab/                                 (gitignored — your active work)
    └── vmetal/
        ├── milestone-01/
        │   ├── notes.md                 (the durable artifact)
        │   ├── redfish-power.sh
        │   └── scaffolding/             (throwaway)
        ├── milestone-02/
        └── ...
```

---

Last Updated: 2026-04-29