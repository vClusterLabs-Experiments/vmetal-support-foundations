# Milestone 6: Notes

Brief: [`../../milestone-06-vmetal.md`](../../milestone-06-vmetal.md)

## Files to produce

- [ ] `notes.md` (this file)
- [ ] `crd-mapping.md`, table mapping vmetal CRDs to CAPI/Metal3 primitives
- [ ] `pod-inventory.md`, what's running and why
- [ ] `m5-vs-m6-diff.md`, manifests compared, opinions called out
- [ ] `sharp-edges.md`, failures vmetal handles vs. doesn't

## Anchor question

> What does vmetal add on top of CAPI + Metal3, and why does each addition exist?

(your answer)

## Conceptual questions

### 1. AI Cloud-specific concerns: what does vmetal address that generic CAPI+Metal3 doesn't?

(your answer, GPU SKUs, NVLink topology, tenant isolation, image catalogs)

### 2. Tenant isolation model: where does the boundary live? Physical, network, namespace, RBAC, all?

(your answer)

### 3. Image management: how does vmetal differ from raw Metal3?

(your answer)

### 4. Day-2 operations: which lifecycle events does vmetal own opinions about, and which does it leave to the operator?

(your answer)

### 5. Where did the abstraction leak in your lab? Fundamental limitation or roadmap gap?

(your answer)

### 6. runcmd-driven Tenant Cluster registration: where in the M5 cloud-init chain (KubeadmConfig → CABPK → Secret → BMH userData → Ironic ConfigDrive → cloud-init) does vmetal's Private Node registration script slot in?

(your answer, prepended/appended/separate document, and where you'd SSH first when a Private Node provisions but never appears in its Tenant Cluster)

### 7. Customer evaluating vmetal vs. rolling CAPI+Metal3 themselves: case for each?

(your answer, when does the abstraction earn its weight, when is it overhead?)

## Observations
