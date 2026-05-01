# Milestone 5: Notes

Brief: [`../../milestone-05-capi-metal3.md`](../../milestone-05-capi-metal3.md)

## Files to produce

- [ ] `notes.md` (this file)
- [ ] `manifests/baremetalhost-*.yaml`
- [ ] `manifests/cluster.yaml`
- [ ] `manifests/kubeadmcontrolplane.yaml`
- [ ] `manifests/machinedeployment.yaml`
- [ ] `manifests/kubeadmconfigtemplate.yaml`
- [ ] `manifests/metal3machinetemplate.yaml`
- [ ] `userdata-trace.md` — **the central artifact**: kubectl walk from `KubeadmConfig` → decoded bootstrap Secret → BMH `spec.userData`, with the decoded cloud-init payload diffed against your M3 `seed/user-data`. Annotate every line CABPK added.
- [ ] `reconciliation-trace.md`, what happened when you killed a node manually (timeline + controller logs from BMO, CAPM3, KCP)
- [ ] `capi-vs-vmetal.md`, first-pass map of CAPI/Metal3/CABPK primitives to vmetal concepts (refine in M6)

## Anchor question

> The cloud-init payload you wrote by hand in M3 still runs at first boot — but you didn't write it this time. Who wrote it, where does it live in Kubernetes, and how did it reach the disk?

(your answer; CABPK → Secret → BMH `spec.userData` → Ironic → ConfigDrive → cloud-init ConfigDrive datasource → first-boot runcmd.)

## Conceptual questions

### 1. Why split Metal3 into CAPI provider + BMO + Ironic + IPA?

(your answer; what each component owns, why IPA is a separate ramdisk on the host being provisioned.)

### 2. Management cluster dies: are workload nodes orphaned?

(your answer; what state lives only on management vs. workload, `clusterctl move`, pivot scenario.)

### 3. Reconciliation cost on physical hardware

(your answer; software vs. bare-metal cost asymmetry, Metal3 host state machine, KCP/Machine controller long-tail handling.)

### 4. CABPK and the user-data delivery boundary

(your answer; why CABPK generates and Metal3 delivers; bootstrap Secret contract; what would change if the OS were Talos — and why the curriculum committed to Ubuntu/cloud-init instead.)

### 5. Image management

(your answer; image source, who builds, how Metal3 references — `image.url`/`checksum`/`format`; raw vs. qcow2 tradeoff; what `checksumType` prevents.)

### 6. Locking: who owns a physical node?

(your answer; `consumerRef` semantics, deprovision-then-reprovision lifecycle.)

### 7. vmetal mapping (preview for M6)

(initial map; pay attention to anything that *modifies the cloud-init payload* on its way to the host — Tenant Cluster registration scripts, GPU-specific bootstrap.)

## Reconciliation test

- [ ] Kill a `provisioned` node via `virsh destroy`. Watch BMO, CAPM3, and KCP each react. What does each layer do? Capture the controller logs.

## "Follow the user-data" trace

- [ ] `kubectl get kubeadmconfig <kc> -o jsonpath='{.status.dataSecretName}'`
- [ ] Decode: `kubectl get secret <name> -o jsonpath='{.data.value}' | base64 -d | gunzip` (fall back to base64-only if gunzip fails)
- [ ] Confirm: `kubectl get bmh <name> -o yaml | grep -A 2 userData`
- [ ] Diff decoded payload vs. `M3 seed/user-data`. Annotate.

## Observations
