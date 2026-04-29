# Milestone 3.5 — Notes

Brief: [`../../milestone-03-5-inventory.md`](../../milestone-03-5-inventory.md)

## Files to produce

- [ ] `notes.md` (this file)
- [ ] `inventory.yaml` — manual data model (strategy 1)
- [ ] `enroll.sh` — auto-enrollment script (strategy 2)
- [ ] `aggregation-sketch.md` — design for strategy 3

## Anchor question

> If a server is bolted into a rack right now and nobody has touched a keyboard, what's the chain of events that makes it appear in the orchestrator's database?

(your answer)

## Conceptual questions

### 1. Trust: in auto-enrollment, what stops a malicious DHCP responder from registering fake nodes? What stops a real node from lying?

(your answer)

### 2. Identity stability: if a NIC is replaced, the MAC changes. How should inventory handle that?

(your answer)

### 3. Race conditions: two nodes boot simultaneously and try to enroll. Where are the ordering hazards?

(your answer)

### 4. Drift: BMC firmware upgraded, Redfish exposes new fields. How does inventory stay current?

(your answer)

## Tradeoff analysis

Manual / fingerprinting / aggregation — which fits which operational reality?

(your analysis)

## Observations