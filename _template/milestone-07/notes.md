# Milestone 7: Notes (optional)

Brief: [`../../milestone-07-day2.md`](../../milestone-07-day2.md)

## Files to produce

Pick at least two of three exercises (7a/7b/7c). 7d is design-only on paper.

- [ ] `7a-os-upgrade/`, image build artifacts, rollout walkthrough, failure trace, scale analysis
  - [ ] `image-build/` (build scripts + the two image artifacts, or pointers to where they live)
  - [ ] `rollout-walkthrough.md` (timestamps, commands, what reconciled when)
  - [ ] `failure-trace.md` (deliberate failure: bad checksum / missing image URL / kubeadm-join failure)
  - [ ] `scale-analysis.md` (what changes at 100x, image bandwidth, drain time, quorum windows)
- [ ] `7b-decommission/walkthrough.md` and `failure-log.md`
- [ ] `7c-credential-rotation/walkthrough.md` and `failure-log.md`
- [ ] `7d-firmware-rollout-design.md` (paper design, not executed)
- [ ] `notes.md` (this file)

## Anchor question

> What's the smallest-impact way to change anything about a running fleet, and what's the cost of doing it wrong?

(your answer)

## Conceptual questions

### 1. Drain semantics

(your answer; PDBs, finalizers, stuck terminating pods.)

### 2. Image rebuild vs. in-place upgrade

(your answer; what each side of the tradeoff buys you; why production AI Cloud lands on rebuild + re-provision.)

### 3. Upgrade ordering: control plane first or workers first?

(your answer; version skew rule, what specifically breaks if a worker's kubelet is newer than the apiserver.)

### 4. Quorum, capacity, and the cost of a CP rebuild

(your answer; 3-node CP during rollout has 0 tolerance, what does that mean operationally? Single-node CP rebuild = downtime, for how long? Production answer.)

### 5. Credential rotation atomicity

(your answer; window where management has old, BMC has new; what fires during it.)

### 6. Decommission and identity

(your answer; MAC, BMC URL, serial, which should change when re-racking; what Ironic cleans by default.)

### 7. vmetal day-2 automation vs. operator responsibility

(your answer; reuse M6 `sharp-edges.md`; differences between Auto Nodes and Private Node Tenant Clusters.)

### 8. Why is the BMC the most dangerous component to compromise?

(your answer; what does Redfish admin give an attacker that root on the host doesn't? Place in day-2 context: 7c rotates BMC credentials, 7d flashes firmware via UpdateService, 7b must wipe BMC state on decommission.)

## 7a-specific checklist (image-rebuild + re-provision)

- [ ] Built image N+1 with a meaningful difference (kernel bump, K8s minor version bump, or both)
- [ ] Published `.img` + `.sha256` to HTTP server
- [ ] Drained worker (or updated `Metal3MachineTemplate`)
- [ ] Updated `BareMetalHost.spec.image.url` (or template ref) to point at image N+1
- [ ] Watched: deprovision → IPA wipe → IPA write image N+1 → ConfigDrive regenerated → reboot → cloud-init `kubeadm join` → node Ready
- [ ] CP rollout sequenced and quorum-bounded (or, on single-CP lab, downtime measured)
- [ ] Forced detour: pointed `image.url` at a 404 or a wrong checksum, identified which controller surfaced the error first

## Observations
