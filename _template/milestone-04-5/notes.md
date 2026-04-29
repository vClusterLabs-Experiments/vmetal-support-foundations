# Milestone 4.5 — Notes

Brief: [`../../milestone-04-5-observability.md`](../../milestone-04-5-observability.md)

## Files to produce

- [ ] `notes.md` (this file)
- [ ] `runbooks/01-bmc-unreachable.md`
- [ ] `runbooks/02-dhcp-misconfig.md`
- [ ] `runbooks/03-bad-ipxe-script.md`
- [ ] `runbooks/04-bad-machine-config.md`
- [ ] `signals-by-layer.md` — annotated table of where to look for what

## Anchor question

> Given only a customer report of "node won't provision," what's the minimum-cost diagnostic path that lands on the right layer?

(your answer)

## Conceptual questions

### 1. Why is serial console the gold standard for early-boot debugging? What can it see that OS-level logs can't?

(your answer)

### 2. Same symptom, five possible layers. Heuristic ordering for which to check first?

(your answer — likelihood × cost-to-check)

### 3. Production observability at AI Cloud scale: what replaces "SSH to the lab box and read journalctl"?

(your answer — centralized syslog, BMC event aggregation, fleet-wide DHCP lease databases)

### 4. What can't you see without physical access, even fully instrumented?

(your answer — pre-BMC failures: power, fan, MB-level POST)

### 5. Operational scale: 1,000 GPU nodes, flat L2 BMC network. What fails first, and what prevents it?

(your answer — broadcast domains, credential blast radius, firmware update storms, synchronous Redfish polling under fleet load)

## Observations
