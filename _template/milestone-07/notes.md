# Milestone 7 — Notes (optional)

Brief: [`../../milestone-07-day2.md`](../../milestone-07-day2.md)

## Files to produce

Pick at least two of three exercises:

- [ ] `7a-os-upgrade/walkthrough.md` and `failure-log.md`
- [ ] `7b-decommission/walkthrough.md` and `failure-log.md`
- [ ] `7c-credential-rotation/walkthrough.md` and `failure-log.md`
- [ ] `notes.md` (this file)

## Anchor question

> What's the smallest-impact way to change anything about a running fleet, and what's the cost of doing it wrong?

(your answer)

## Conceptual questions

### 1. Drain semantics: what does `kubectl drain` actually do, and when does it lie?

(your answer — PDBs, finalizers, stuck terminating pods)

### 2. Upgrade ordering: control plane first or workers first? Constraints?

(your answer)

### 3. Credential rotation atomicity: window where management has old, BMC has new. How long? What can fire during it?

(your answer)

### 4. Decommission and identity: MAC, BMC URL, serial — which should change when re-racking hardware as a different fleet member?

(your answer)

### 5. What does vmetal automate in day-2, and what does it leave to you?

(your answer — compare to manual experience)

### 6. Why is the BMC the most dangerous component to compromise?

(your answer — what does Redfish admin give an attacker that root on the host doesn't? Consider in day-2 context: credentials rotate (7c), firmware flashes through `UpdateService` (7d), decommission must wipe BMC state (7b))

## Observations
