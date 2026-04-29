# Milestone 5: Notes

Brief: [`../../milestone-05-capi-metal3.md`](../../milestone-05-capi-metal3.md)

## Files to produce

- [ ] `notes.md` (this file)
- [ ] `manifests/baremetalhost-*.yaml`
- [ ] `manifests/cluster.yaml`
- [ ] `manifests/machinedeployment.yaml`
- [ ] `reconciliation-trace.md`, what happened when you killed a node manually
- [ ] `capi-vs-vmetal.md`, first-pass map of CAPI/Metal3 primitives to vmetal concepts

## Anchor question

> What does declarative bare-metal management actually mean, and what reconciliation loops are running to make it real?

(your answer)

## Conceptual questions

### 1. Why is Metal3 split into CAPI provider + Ironic + IPA? What does each own?

(your answer)

### 2. Management cluster controls workload cluster's nodes. What if the management cluster dies?

(your answer)

### 3. Reconciliation cost: software loops every 10s; bare-metal loops can't. What does this force the controller design to look like?

(your answer, state machines, long timeouts, hard-to-test transitions)

### 4. Image management: where does the OS image come from? Who builds it, where's it stored, how does Metal3 reference it?

(your answer)

### 5. Two operators want to provision the same physical node. Locking model? Which CR field expresses ownership?

(your answer)

### 6. vmetal hint: which vmetal CRDs map to CAPI/Metal3 primitives, and which are net-new?

(initial map, refine in M6)

## Observations
