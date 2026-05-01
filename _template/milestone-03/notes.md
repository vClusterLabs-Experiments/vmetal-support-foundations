# Milestone 3: Notes

Brief: [`../../milestone-03-os-provisioning.md`](../../milestone-03-os-provisioning.md)

## Files to produce

- [ ] `notes.md` (this file)
- [ ] `seed/meta-data`, the NoCloud meta-data (instance-id + local-hostname)
- [ ] `seed/user-data`, the cloud-config + autoinstall + nested target user-data
- [ ] `ipxe-script.ipxe`, updated from M2 to chainload Ubuntu netboot kernel/initrd with the autoinstall cmdline
- [ ] `reprovision-walkthrough.md`, exact sequence to wipe + PXE-again with a changed user-data
- [ ] `failure-modes.md`, the four induced failures (bad YAML, NoCloud fetch failure, runcmd failure, write_files error)
- [ ] `cloud-init-cheatsheet.md`, one-page reference of diagnostic commands you used most
- [ ] `scaffolding/http-server.conf`
- [ ] `scaffolding/kernel-cmdline.txt`

## Anchor question

> What's the smallest amount of declarative state needed to turn a generic Ubuntu cloud image into a *specific* node?

(your answer; verify with `cloud-init query` on a running node before you commit)

## Conceptual questions

### 1. Why five cloud-init boot stages?

(your answer; Detect / init-local / init / config / final, different units, different network state, what would break if you moved network-dependent work into init-local?)

### 2. First-boot detection: what artifact gates a re-run? What does `cloud-init clean` do, and what does it NOT undo?

(your answer; instance-id, /var/lib/cloud/instances/, reset hammer; impact on a working K8s node)

### 3. user-data vs. meta-data vs. vendor-data: why three documents?

(your answer; intended author of each, layered model, fleet-operator-pushed baseline scenario)

### 4. Datasource selection: how does cloud-init pick NoCloud over ConfigDrive vs. ~25 alternatives?

(your answer; ds-identify, kernel cmdline, DMI/SMBIOS hints, filesystem labels; what arg forces NoCloud?)

### 5. Triage on a misbehaving cloud-init node: the first three commands and why?

(your answer; status --long → which log → which JSON → schema vs. analyze blame vs. collect-logs)

### 6. Mutable-OS drift at fleet scale

(your answer; apt + systemd + netplan + kubelet + containerd + cloud-init cache; golden-image / re-provision / config-management tradeoffs; where does vmetal's image-rebuild model in M7 sit?)

## Induced failures (write up in `failure-modes.md`)

For each: symptom on console, file + line that pinpointed cause, one-sentence remediation.

- [ ] 1. Bad YAML in user-data (tab character, unquoted `:`, bad indentation)
- [ ] 2. NoCloud fetch failure (HTTP server stopped, or trailing slash dropped from `s=`)
- [ ] 3. `runcmd` failure mid-execution (`/bin/false` or `apt-get install fakepackage`)
- [ ] 4. `write_files` permission/path error (parent dir missing, bad permissions value)

## Observations
