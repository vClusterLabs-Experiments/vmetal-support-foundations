# Milestone 4.5: Notes

Brief: [`../../milestone-04-5-observability.md`](../../milestone-04-5-observability.md)

## Files to produce

- [ ] `notes.md` (this file)
- [ ] `runbooks/01-bmc-unreachable.md`
- [ ] `runbooks/02-dhcp-misconfig.md`
- [ ] `runbooks/03-bad-ipxe-script.md`
- [ ] `runbooks/04-bad-cloud-init.md` (split into installer-stage and target-stage sub-cases)
- [ ] `runbooks/05-kubelet-wont-start.md` (cloud-init succeeded, kubeadm exited 0, kubelet crashloops)
- [ ] `signals-by-layer.md`, your annotated diagnostic-surfaces table
- [ ] `installer-vs-target.md`, how to tell from a customer symptom whether they're SSH'd into the live-server installer or the deployed system

## Anchor question

> Given only a customer report of "node won't provision," what's the minimum-cost diagnostic path that lands on the right layer?

(your answer)

## Conceptual questions

### 1. Why is serial console the gold standard for early-boot debugging?

(your answer; what does it see that journalctl can't? Firmware PXE, iPXE chainload, kernel handoff, initrd panic, Subiquity startup all happen before journalctl exists.)

### 2. Diagnostic ordering: where to look first

(your answer; likelihood ÷ cost-to-check. Now that the OS is Ubuntu, SSH and journalctl available, how does that change the ordering vs. a fully-locked-down OS? When does SSH save you a serial-console hop, and when is it a *trap* that points you at the wrong layer?)

### 3. Observability at AI Cloud scale

(your answer; the path a single line of `cloud-init.log` takes from node to support engineer in a 1000-node fleet, aggregation, retention, search, alerting; what `cloud-init collect-logs` adds on top of `journalctl`.)

### 4. Failures invisible without physical access

(your answer; pre-BMC failures, BMC itself lying via Redfish.)

### 5. Operational scale: 1,000 GPU nodes, flat L2 BMC network

(your answer; broadcast domains, credential blast radius, firmware update storms, Redfish polling under fleet load, cloud-init log-volume bursts, kubelet TLS bootstrap CSR floods.)

## Induced failures (5 total)

For each: symptom (customer-visible), originating layer, first three diagnostic checks in order, which surface pinpointed the cause.

- [ ] 1. BMC unreachable
- [ ] 2. DHCP misconfiguration
- [ ] 3. Bad iPXE script
- [ ] 4. Bad cloud-init user-data, installer-stage *and* target-stage sub-cases (distinguish them)
- [ ] 5. Node provisions, kubelet won't start (cgroup-driver mismatch, kernel module missing, swap not actually disabled, CRI socket mismatch, pick one to induce)

## Observations
