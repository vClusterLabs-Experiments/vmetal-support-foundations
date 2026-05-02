# Milestone 3: OS Provisioning with cloud-init

**Goal:** Take the network-boot pipeline from M2 and use it to install Ubuntu on disk and configure the deployed node declaratively, via cloud-init, without anyone ever logging in to set it up. PXE-boot the Ubuntu live-server netboot kernel/initrd, run Subiquity autoinstall to write Ubuntu to disk, then watch cloud-init apply your user-data on the deployed system's first boot. Both the install and the first boot are driven from a single set of files served over plain HTTP from your lab's web server.

The flow mirrors what vmetal does in production at lab scale: an installer fetches an OS image and writes it to disk, the host reboots, and cloud-init configures the deployed system from a user-data document. M5 will replace your hand-rolled HTTP delivery with Metal3 + Ironic + ConfigDrive. The payload, cloud-init, is the same in both places.

## What you're actually learning

- **cloud-init as the bare-metal config primitive.** It's how the major cloud distros (Ubuntu, RHEL, SUSE, Amazon Linux, Rocky) get configured on first boot. It's how Metal3/Ironic delivers per-node identity in vmetal. It's the mass-fleet alternative to "log in and run Ansible." The lesson here transfers directly to every customer ticket you'll see.
- **The five-stage cloud-init boot model.** `Detect → init-local → init (network) → config → final`. Different stages, different network state, different logs. Triage starts with knowing which stage owns which problem.
- **Two cloud-inits, one install.** Subiquity autoinstall is itself a cloud-init payload running in the live-server installer. The `autoinstall.user-data:` you nest inside it becomes the *target system's* cloud-init payload on first boot of the disk-installed OS. Same tool, two rootfses, two log paths with the same filename. Support engineers who don't internalize this split waste hours debugging the wrong system.
- **Module frequency and re-run semantics.** cloud-init does not "run once forever." Some modules run every boot, some run once per `instance-id`, some run once per system. Reboots, image clones, and `cloud-init clean` all change the picture.
- **The mutable-OS drift surface.** Ubuntu state lives across many places, apt, systemd, netplan, kubelet runtime state, containerd state, cloud-init's own cache. Knowing where it lives is the price of choosing a mutable distro; vmetal's customers chose it, and so do you.

## Anchor question

> What's the smallest amount of declarative state needed to turn a generic Ubuntu cloud image into a *specific* node?

**Hint:** the NoCloud datasource only requires two files, `meta-data` (carrying a unique `instance-id`) and `user-data` (the cloud-config document). Everything else, hostname, users, SSH keys, disks, network, packages, `kubeadm` prereqs, flows from those two files plus the kernel cmdline that points the installer at them. Verify against `cloud-init query` on a running node before you commit to an answer.

## Stack

```text
Lima VM
├── libvirt + QEMU
│   └── node01  (real disk that persists across reboots)
├── isolated network
│   ├── dnsmasq                     <-- DHCP + TFTP from M2
│   └── HTTP server (M2)
│       ├── ipxe.script              <-- chainloads the Ubuntu installer
│       ├── linux, initrd            <-- Ubuntu live-server netboot kernel + initrd
│       └── seed/
│           ├── meta-data            <-- one-line: instance-id + local-hostname
│           ├── user-data            <-- #cloud-config + autoinstall: + nested user-data:
│           └── (optional) vendor-data, network-config
└── sushy-tools                      <-- the BMC
```

Two cloud-init runs happen across one provisioning cycle:

1. **Installer-stage**, while the live-server ramdisk is running. Subiquity reads `autoinstall:` from the user-data, partitions the disk, runs `curtin` to install Ubuntu, then writes the *nested* `autoinstall.user-data:` to the target as the seed for the next stage.
2. **Target-system**, on the first boot of the deployed Ubuntu OS. cloud-init reads the seed Subiquity left for it and applies hostname, users, SSH keys, packages, sysctls, and any `runcmd` you wrote.

Both runs leave logs at `/var/log/cloud-init.log`, but on different rootfses. SSH into the wrong one and you're debugging the wrong system.

## The provisioning chain you're proving works

```text
1.  Redfish: PATCH boot device = Pxe; POST Reset = On       (you, from M1+M2)
2.  Firmware PXE → iPXE chainload                           (M2)
3.  iPXE fetches your iPXE script over HTTP                 (M2)
4.  iPXE fetches Ubuntu live-server kernel + initrd         (NEW)
5.  Kernel boots with cmdline: autoinstall ds=nocloud-net;s=http://server/seed/
6.  Live-server cloud-init runs, fetches meta-data + user-data, sees autoinstall:
7.  Subiquity partitions disk, curtin installs Ubuntu       (installer-stage)
8.  Subiquity writes nested user-data: to /var/lib/cloud/seed/nocloud/ on target
9.  Reboot into installed Ubuntu                            (boundary)
10. Target cloud-init runs: applies hostname, users, SSH, packages, runcmd
11. cloud-init final completes; node is ready               (target-stage)
```

Steps 1–3 are M1/M2. Steps 4–11 are this milestone. Every layer above (cluster bootstrap, vmetal) is a variation on what cloud-init runs in steps 10–11.

## Reading list

| Topic | Source |
|---|---|
| cloud-init boot stages | [cloud-init: Boot stages](https://cloudinit.readthedocs.io/en/latest/explanation/boot.html), the five-stage model and which systemd unit runs each stage |
| cloud-init module frequency | [cloud-init: Module frequency](https://cloudinit.readthedocs.io/en/latest/reference/modules.html), `always` vs. `instance` vs. `once`, and the role of `instance-id` |
| cloud-init NoCloud datasource | [cloud-init: NoCloud](https://cloudinit.readthedocs.io/en/latest/reference/datasources/nocloud.html), seed format, kernel-cmdline form, the trailing-slash gotcha |
| cloud-init logs and CLI | [cloud-init: Logging](https://cloudinit.readthedocs.io/en/latest/reference/logging.html) and [Reporting bugs / collect-logs](https://cloudinit.readthedocs.io/en/latest/howto/debug_user_data.html), the diagnostic surface |
| cloud-init schema validation | `cloud-init schema --config-file user-data.yaml`, run on the workstation before you serve it |
| Ubuntu live-server autoinstall | [Ubuntu Server: Automated install introduction](https://canonical-subiquity.readthedocs-hosted.com/en/latest/intro-to-autoinstall.html) and [Autoinstall reference](https://canonical-subiquity.readthedocs-hosted.com/en/latest/reference/autoinstall-reference.html), the format Subiquity reads |
| Ubuntu netboot installer | [Ubuntu Server: Netboot install](https://canonical-subiquity.readthedocs-hosted.com/en/latest/howto/install-netboot.html), kernel/initrd locations and required cmdline |
| Subiquity `late-commands` vs. target user-data | [Autoinstall reference: late-commands](https://canonical-subiquity.readthedocs-hosted.com/en/latest/reference/autoinstall-reference.html#late-commands), where each kind of "do this once" lives |
| Mutable-OS drift, for support context | [Google SRE Book: Configuration design and best practices](https://sre.google/workbook/configuration-design/), the operational case for treating config as data, not as a sequence of commands |

## Success criteria

1. **PXE-boot the Ubuntu live-server installer.** Your iPXE script (from M2) chainloads the Ubuntu netboot `linux` and `initrd` from your HTTP server, with kernel cmdline including `autoinstall ds=nocloud-net;s=http://<your-http-server>/seed/`. Console shows the live-server boot.
2. **Subiquity completes a hands-off install.** No keyboard input. The installer finds your `meta-data` and `user-data`, sees `autoinstall:`, partitions the disk, installs Ubuntu, and reboots. Time it: re-runs should be reproducible.
3. **Target cloud-init applies the nested user-data on first boot.** After reboot, `cloud-init status --wait --long` reports `done`. The hostname, your user account, and your SSH key are present. SSH in as the user you defined, not via password.
4. **You can read the diagnostic surface fluently.** From the deployed node: `cloud-init status --long`, `cloud-init query --list-keys`, `cloud-init query v1.instance_id`, `cloud-init analyze blame`. Locate `/var/log/cloud-init.log`, `/var/log/cloud-init-output.log`, `/run/cloud-init/instance-data.json`, and `/var/lib/cloud/instance/user-data.txt`. Be able to point at the file that proves cloud-init saw your seed.
5. **Re-provision with a changed user-data.** Change one field (hostname, an extra package, a new `runcmd`), wipe the disk, PXE again, observe the new identity. Prove the disk has no hidden state surviving a wipe.
6. **Validate user-data before serving it.** Run `cloud-init schema --config-file user-data.yaml` on your workstation. Catch one schema error on purpose (e.g., misspell `runcmd` as `runcmds`); confirm the validator surfaces it.

## Reference user-data shape

You'll write a fuller version of this in your `user-data.yaml`. The shape below is the kubeadm-prereq scaffolding M4 will build on, so you'll see most of it again. Keep it in front of you while drafting.

```yaml
#cloud-config
autoinstall:
  version: 1
  identity:
    hostname: node01
    username: ubuntu
    password: "$6$rounds=4096$REPLACE_ME$..."   # mkpasswd -m sha-512
  ssh:
    install-server: true
    allow-pw: false
    authorized-keys:
      - ssh-ed25519 AAAA...your-key...
  storage:
    layout:
      name: direct
  packages:
    - qemu-guest-agent
  user-data:                         # <-- nested: target-system cloud-init payload
    timezone: UTC
    hostname: node01
    bootcmd:
      - swapoff -a
    write_files:
      - path: /etc/modules-load.d/k8s.conf
        content: |
          overlay
          br_netfilter
      - path: /etc/sysctl.d/k8s.conf
        content: |
          net.bridge.bridge-nf-call-iptables  = 1
          net.bridge.bridge-nf-call-ip6tables = 1
          net.ipv4.ip_forward                 = 1
    runcmd:
      - sed -ri '/\sswap\s/s/^#?/#/' /etc/fstab
      - modprobe overlay
      - modprobe br_netfilter
      - sysctl --system
```

`meta-data` is one stanza:

```yaml
instance-id: node01-install-001
local-hostname: node01
```

Kernel cmdline (in your iPXE script, on the `kernel` line):

```text
autoinstall 'ds=nocloud-net;s=http://10.0.0.1/seed/' ip=dhcp ---
```

The single quotes are mandatory: without them, the shell that builds the iPXE/GRUB cmdline splits on `;` and only `autoinstall ds=nocloud-net` reaches the kernel.

Three traps you will hit and want to remember:

- **Trailing slash on `s=...`.** Without it, NoCloud silently fails to find `meta-data`.
- **Quote/escape the `;` in iPXE/GRUB** so the shell doesn't split the cmdline.
- **The nested `user-data:` does NOT need a `#cloud-config` header.** Subiquity wraps it. But the *outer* installer `user-data` file you serve at the seed URL DOES need `#cloud-config` on line 1.

## Conceptual questions

Each question has a **Read first** pointer to a source that teaches the underlying concept.

### 1. Why five boot stages?

cloud-init's stages (`Detect`, `init-local`, `init`, `config`, `final`) run under different systemd units, with different network availability, and read different parts of your user-data. Why split the work across five stages instead of running everything once at the end? What problem would you cause by moving network-dependent work into `init-local`?

**Read first:**
- [cloud-init: Boot stages](https://cloudinit.readthedocs.io/en/latest/explanation/boot.html), the canonical description of which unit runs which modules and when network is available.

### 2. First-boot detection: what gates a re-run?

cloud-init has to decide, at every boot, whether to apply `runcmd`/`users`/`write_files` again or skip them. What artifact on disk drives that decision, and what command resets it? What happens to a Kubernetes node if `cloud-init clean` is run on a working cluster member?

**Read first:**
- [cloud-init: First-boot detection / instance-id](https://cloudinit.readthedocs.io/en/latest/reference/datasources/nocloud.html), the role of `instance-id` and the `/var/lib/cloud/instances/` cache.
- `cloud-init clean --help` on the deployed node, the reset hammer and what it does *not* undo (packages, users, files in `/etc`, kubelet state).

### 3. user-data vs. meta-data vs. vendor-data: why three documents?

NoCloud expects (at minimum) `meta-data` and `user-data`, and optionally `vendor-data`. They look similar, all are cloud-config-shaped YAML, and a beginner's instinct is to merge them. Why does cloud-init keep them separate? Who is the intended author of each? What changes if a fleet operator wants to push a baseline config across thousands of nodes without touching per-node `user-data`?

**Read first:**
- [cloud-init: Instance metadata / vendor data](https://cloudinit.readthedocs.io/en/latest/explanation/instancedata.html), the layered model and merge semantics.

### 4. Datasource selection: how does cloud-init pick NoCloud?

cloud-init supports ~25 datasources (NoCloud, ConfigDrive, EC2, Azure, GCE, OpenStack, …). On a fresh boot it has to pick one before it can do anything else. What runs first, what hints does it consult (kernel cmdline, DMI/SMBIOS strings, filesystem labels), and what kernel arg or config file forces NoCloud explicitly? What does the matching log look like when it gets it wrong?

**Read first:**
- [cloud-init: Datasource detection / `ds-identify`](https://cloudinit.readthedocs.io/en/latest/reference/datasources.html), how detection narrows the list.
- `/run/cloud-init/ds-identify.log` on the deployed node, the actual decision trace, including which hints matched and which were rejected.

### 5. Triage on a misbehaving cloud-init node: where do you start?

You SSH into a node where `cloud-init status` reports `error`. What's the first command you run and why? Sketch the order: `status --long` → which log → which JSON → when do you reach for `cloud-init schema --config-file` vs. `cloud-init analyze blame` vs. `cloud-init collect-logs`? Be specific about the question each command answers.

**Read first:**
- [cloud-init: Debugging cloud-init](https://cloudinit.readthedocs.io/en/latest/howto/debug_user_data.html), the documented triage path.

### 6. Mutable-OS drift at fleet scale

Ubuntu's persistent disk surface is large: apt package state, systemd unit drops, netplan config, kubelet runtime state, containerd state, cloud-init's own cache, anything `runcmd` wrote. Across 500 nodes over a year, drift is inevitable. What patterns close the gap, golden images, periodic re-provision, config management (Ansible/Puppet), config snapshotting, and what's the operational tradeoff of each? Where does vmetal's "rebuild raw image, re-provision via Metal3" pattern (which you'll see in M7) sit in this spectrum?

**Read first:**
- [Google SRE Book: Configuration design](https://sre.google/workbook/configuration-design/), the operational case for treating config as data and re-provisioning as the upgrade primitive.

## Induced failures

Work through these in order. Each one isolates a different layer of the cloud-init pipeline.

1. **Bad YAML in user-data.** Introduce a tab character, or unquoted `:` in a value, or unbalanced indentation. Try to validate with `cloud-init schema --config-file`. Then serve it anyway; observe the failure mode in the live-server console. Where does the error surface, installer log, target log, or never-reaches-target?
2. **NoCloud fetch failure.** Stop your HTTP server (or strip the trailing slash from the `s=` cmdline). PXE the node. Observe `ds-identify`'s decision trace and the resulting timeout. Identify the file path and line in the log that points at "couldn't reach the seed URL."
3. **`runcmd` failure mid-execution.** Add a `runcmd` step that runs a command guaranteed to fail (`/bin/false`, or `apt-get install fakepackage`). Boot. Where does the failure show up, `cloud-init status --long`? `cloud-init-output.log`? Does cloud-init halt or continue? What's the impact on subsequent modules?
4. **`write_files` permission/path error.** Try to `write_files` to a path whose parent doesn't exist (`/opt/missing/dir/file.conf`). Try with a `permissions:` value that's invalid. Observe both. Does cloud-init fail loudly or silently skip?

For each, write down:
- The symptom on console.
- The file + line that pinpointed the cause.
- One-sentence remediation.

## A note on `runcmd`

`runcmd` is imperative inside an otherwise declarative format. Whenever you reach for it, ask first: is there a structured cloud-init module that does this declaratively? `users`, `ssh_authorized_keys`, `write_files`, `packages`, `apt`, `ntp`, `timezone`, `disk_setup`, `mounts`, `bootcmd`, these cover most "do this once" needs.

That said, `runcmd` is not a smell, and you'll see a lot of it. vmetal's production user-data uses `runcmd` to invoke the registration scripts that join private nodes to their tenant clusters, `kubeadm` itself is imperative, and wrapping it in `runcmd` is the honest way to express that. The rule of thumb: use `runcmd` for things that *are* genuinely imperative ("run this binary once at first boot"), and avoid using it for things that have a structured equivalent. Support engineers who recognize the difference triage faster.

## What is NOT in this milestone

- No Kubernetes cluster yet. cloud-init has installed `kubelet`/`kubeadm`/`containerd` (or staged the prereqs to install them), but nothing has run `kubeadm init`. That's M4.
- No CAPI, no Metal3, no Ironic. You are hand-rolling the cloud-init delivery via plain HTTP NoCloud. M5 replaces the delivery mechanism with Metal3 + ConfigDrive, but the cloud-init payload shape stays the same.
- No second node yet. Provision one. M4 brings a worker.
- No CNI, no pod CIDR, no `kubeadm` config. M4.
- No HTTPS, no signed metadata. The seed server is plain HTTP, that's a real production concern called out in M4/M5/M7, not solved here.
- No Talos, no Flatcar, no Ignition. The curriculum has committed to cloud-init as the primitive. The immutable-OS lessons are not preserved here.

## Exit artifact

`lab/vmetal/milestone-03/`:

- `notes.md`, your written answers to the six conceptual questions. **The artifact that matters.**
- `seed/meta-data`, `seed/user-data`, the documents your HTTP server actually served, with comments explaining each section.
- `ipxe-script.ipxe`, updated from M2 to chainload the Ubuntu netboot kernel/initrd with the autoinstall cmdline. Comment the cmdline, the trailing slash, and the `;`-escape.
- `reprovision-walkthrough.md`, exact sequence to wipe + PXE-again and end up with the changed identity.
- `failure-modes.md`, the four induced failures with symptom, log line that pinpointed cause, and remediation.
- `cloud-init-cheatsheet.md`, a one-page reference of the diagnostic commands you used most. You'll consult this in M4.5.
- `scaffolding/`, HTTP server config, kernel cmdline used, anything you'd throw away if you rebuilt the lab.

When `notes.md` answers all six questions, the four induced failures are written up, and you can re-provision the node in one shot from a fresh disk, you're ready for **Milestone 4: Cluster Bootstrap (by hand)**, which takes the cloud-init payload you wrote here and extends it with the `kubeadm init` / `kubeadm join` invocations that turn the node into a Kubernetes cluster.

---

Last Updated: 2026-05-01
