# Milestone 4 — Notes

Brief: [`../../milestone-04-cluster-bootstrap.md`](../../milestone-04-cluster-bootstrap.md)

## Files to produce

- [ ] `notes.md` (this file)
- [ ] `bootstrap-walkthrough.md` — exact sequence from cold node01 to 2-node cluster
- [ ] `talosconfig` and `kubeconfig` — saved for later milestones
- [ ] `failure-trace.md` — what happened with the wrong CA, how you diagnosed it

## Anchor question

> What's the difference between "a node booted" and "a node joined a cluster," and who owns the data that bridges that gap?

(your answer)

## Conceptual questions

### 1. Bootstrap chicken-and-egg: first CP node needs etcd, but etcd needs config. How does Talos resolve this on first boot?

(your answer)

### 2. Join tokens: what authority does one grant? Blast radius if leaked? Lifetime?

(your answer)

### 3. kubelet certs: who generates them, at what point in boot flow?

(your answer)

### 4. Adding a second control plane node vs. a worker — what's different?

(your answer — etcd membership, API server load balancing)

### 5. Worker boots, joins, runs Pods. CP dies permanently. What's recoverable, from where?

(your answer)

### 6. 100 workers join simultaneously. Bottleneck?

(your answer — API server admission, etcd write throughput, cert signing)

## Observations
