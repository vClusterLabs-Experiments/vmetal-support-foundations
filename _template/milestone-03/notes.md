# Milestone 3 — Notes

Brief: [`../../milestone-03-os-provisioning.md`](../../milestone-03-os-provisioning.md)

## Files to produce

- [ ] `notes.md` (this file)
- [ ] `machine-config.yaml` (Talos) or `ignition.json` (Flatcar) — the config document
- [ ] `reprovision-walkthrough.md` — wipe + reboot + observe new identity
- [ ] `scaffolding/http-server.conf`
- [ ] `scaffolding/kernel-cmdline.txt`

## Anchor question

> What's the smallest amount of declarative state needed to turn a generic OS image into a specific node?

(your answer)

## Conceptual questions

### 1. Why do Talos / Flatcar exist instead of "Ubuntu with cloud-init"?

(your answer — operational argument for an immutable OS)

### 2. The config is fetched plaintext over HTTP at boot. Security implications and production mitigations?

(your answer — signed configs, HTTPS, encryption at rest)

### 3. What state must persist on disk for a Kubernetes node? What's safe to wipe?

(your answer — etcd data, kubelet certs, container images)

### 4. First-boot vs. subsequent boot: how does the OS know not to re-fetch + re-apply config?

(your answer — what disk artifact does it check?)

### 5. Talos has no SSH and no shell. Why is that an architectural choice, not a limitation?

(your answer)

### 6. 500 nodes fetch config from one HTTP server simultaneously. Failure mode and fix?

(your answer)

## Observations