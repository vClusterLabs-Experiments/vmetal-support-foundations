# Milestone 4: Notes

Brief: [`../../milestone-04-cluster-bootstrap.md`](../../milestone-04-cluster-bootstrap.md)

## Files to produce

- [ ] `notes.md` (this file)
- [ ] `bootstrap-walkthrough.md`, exact sequence from cold node01 (no disk) to a 2-node cluster with Calico and a Pod scheduled on the worker
- [ ] `seed/cp/user-data`, control-plane cloud-init payload (with kubeadm prereqs + `kubeadm init` + Calico apply)
- [ ] `seed/worker/user-data`, worker cloud-init payload (with kubeadm prereqs + `kubeadm join` line)
- [ ] `kubeconfig` (admin), saved for later milestones
- [ ] `failure-trace.md`, the three induced failures (wrong CA hash, expired token, missing CNI) with symptom + log line + remediation
- [ ] `phase-walkthrough.md` *(optional but recommended)*, annotated transcript of `kubeadm init`'s phase output

## Anchor question

> What's the difference between "a node booted" and "a node joined a cluster," and who owns the data that bridges that gap?

(your answer; the boundary is *not* `kubeadm join`. Identify which of {API server endpoint, CA hash, bootstrap token} is actually the *secret* making the join work.)

## Conceptual questions

### 1. Bootstrap chicken-and-egg

(your answer; the kubeadm init phase list, preflight → certs → kubeconfig → control-plane → etcd → upload-config → upload-certs → mark-control-plane → bootstrap-token → kubelet-finalize → addon. Role of static-pod manifests in `/etc/kubernetes/manifests/` and the `kubeadm-config` ConfigMap.)

### 2. Bootstrap token: lifetime, scope, blast radius

(your answer; what the token authenticates as, what it authorizes, default TTL, leak impact, `system:bootstrappers`.)

### 3. Kubelet TLS bootstrap: who signs the cert?

(your answer; CSR filed by kubelet, signed by controller-manager, RBAC binding for auto-approval, where the signed cert lands on disk.)

### 4. cloud-init + kubeadm composition

(your answer; why `runcmd kubeadm init` instead of a declarative cloud-init module; cost of mixing imperative `runcmd` with declarative cloud-config; what happens if cloud-init re-runs on a working cluster member.)

### 5. etcd disaster recovery

(your answer; snapshot + restore, what state was the *last moment* you needed to do something to make recovery possible.)

### 6. Forward pointer to M5: how does CABPK replace what you just did?

(your answer; CABPK reads KubeadmConfig and *generates* cloud-init user-data that runs the same `kubeadm init`/`kubeadm join`. What does it replace, what stays the same?)

## Induced failures (write up in `failure-trace.md`)

- [ ] Wrong CA hash on worker (`--discovery-token-ca-cert-hash sha256:0000…0000`)
- [ ] Expired token (`--ttl 5m`, then wait)
- [ ] Missing CNI (skip Calico apply, watch node stay `NotReady`)

## Observations
