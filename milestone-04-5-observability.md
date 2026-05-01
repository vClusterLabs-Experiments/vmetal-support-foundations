# Milestone 4.5: Observability of the Boot Chain

**Required for M5.** M5's reconciliation exercise (kill a provisioned node and watch the controller react) is unreadable without the diagnostic surfaces this milestone teaches.

**Goal:** Build the diagnostic muscle that distinguishes a support engineer from someone who just installs things. When 1 of 500 nodes fails to provision, where do you look, in what order, and what do you look for?

The boot chain has more layers now than in M1/M2: BMC → DHCP/TFTP → HTTP → iPXE → kernel → Subiquity (installer-stage cloud-init) → reboot → first-boot cloud-init → containerd → kubelet → cluster join. Each layer has its own log surface. This milestone makes you fluent in all of them, in order.

## What you're actually learning

- **The boot chain has many layers; a failure surfaces at one and points back to another.** Symptom location ≠ root cause location.
- **The signals available at each layer** and which one you reach for first.
- **The installer-stage / target-stage split** from M3 is a real fork in your diagnostic path. SSHing into the wrong rootfs answers the wrong question.
- **The cloud-init / kubelet boundary.** "OS booted" and "node joined" are different events. Many cloud-init successes precede kubelet failures; many kubelet failures look like cloud-init failures and aren't.
- **Pattern recognition.** Each failure mode has a fingerprint. After this milestone you should be able to look at a symptom and predict the layer.

## Anchor question

> Given only a customer report of "node won't provision," what's the minimum-cost diagnostic path that lands on the right layer?

## Lab exercise

Induce five failures. For each, record:
- The exact symptom (what does the customer see?)
- The layer where the failure actually originates
- The first three things you'd check, in order, and why
- Which of the diagnostic surfaces below pinpointed the cause

| # | Failure to induce | Hint |
|---|---|---|
| 1 | BMC unreachable | Stop sushy-tools, attempt power-on |
| 2 | DHCP misconfiguration | Wrong `next-server`, observe firmware behavior |
| 3 | Bad iPXE script | Typo in URL, observe where the chain breaks |
| 4 | Bad cloud-init user-data | Two sub-cases: (a) installer-stage failure (e.g., bad `autoinstall:` shape, install never completes); (b) target-stage failure (e.g., bad nested `runcmd` or unparseable `write_files` content, install completes but `cloud-init status --long` reports `error` after first boot). Distinguish them. |
| 5 | Node provisions, kubelet won't start | cloud-init `runcmd` ran `kubeadm init` (or `join`) and exited 0, but kubelet crashloops. Common causes: containerd cgroup driver mismatch, missing kernel module, swap not actually disabled, CRI socket mismatch. The fingerprint here is "cloud-init says success, `kubectl get nodes` never sees the node." |

## Diagnostic surfaces to learn

| Layer | Where the signal lives | How to read it |
|---|---|---|
| BMC | sushy-tools logs / Redfish event log on real hardware | `journalctl -u sushy-emulator`, `GET /redfish/v1/Systems/{id}/LogServices/` |
| DHCP/TFTP | dnsmasq logs, `tcpdump` on the bridge | `journalctl -u dnsmasq`, `tcpdump -i virbrN port 67 or port 69` |
| HTTP (iPXE script + kernel/initrd + NoCloud seed + kubeadm packages) | HTTP server access log, `tcpdump` for the fetch | `journalctl -u nginx` or `python -m http.server` stdout, `tcpdump -i virbrN port 80 or port 443`. Most M3+ failures land here. |
| iPXE | Serial console of the booting node | QEMU `-serial stdio`, real hardware via IPMI SOL |
| Installer-stage cloud-init (Subiquity live-server) | Live-server console + the live-server's own filesystem | `/var/log/cloud-init.log` *inside the installer*, `/var/log/installer/subiquity-server-debug.log`, `/run/cloud-init/ds-identify.log`. SSH to live-server is possible if `ssh.install-server: true` and you set a key. |
| Target-stage cloud-init (deployed Ubuntu, first boot) | Same filenames, deployed rootfs | `cloud-init status --long --wait`, `/var/log/cloud-init.log`, `/var/log/cloud-init-output.log`, `/run/cloud-init/instance-data.json`, `cloud-init analyze blame`, `cloud-init collect-logs` for tickets |
| Container runtime | systemd journal | `journalctl -u containerd`, `crictl ps`, `crictl pods`, `crictl logs <id>`, `crictl info` |
| kubelet | systemd journal | `journalctl -u kubelet -f`, `systemctl status kubelet`, `/var/lib/kubelet/config.yaml`, `/etc/kubernetes/kubelet.conf` |
| Cluster join | apiserver audit log + kubelet logs | `kubectl get events -A`, `kubectl describe node <name>`, `kubectl get csr` for pending TLS bootstrap CSRs |
| Recovery primitive | the node itself, before re-PXE | `kubeadm reset -f`, `cloud-init clean --logs --seed`, then re-provision. Without one of these, re-runs hit half-converged state. |

## Reading list

| Topic | Source |
|---|---|
| cloud-init triage | [cloud-init: Debugging cloud-init](https://cloudinit.readthedocs.io/en/latest/howto/debug_user_data.html), the documented triage path; pair with `cloud-init status --long --wait` and `cloud-init collect-logs` |
| Subiquity install logs | [Ubuntu Server: Subiquity logs](https://canonical-subiquity.readthedocs-hosted.com/en/latest/intro-to-autoinstall.html), the file paths under `/var/log/installer/` and what each contains |
| containerd + crictl | [Kubernetes: Debugging Kubernetes nodes with crictl](https://kubernetes.io/docs/tasks/debug/debug-cluster/crictl/), the canonical debug surface for the CRI runtime |
| kubelet logs and node troubleshooting | [Kubernetes: Troubleshooting clusters](https://kubernetes.io/docs/tasks/debug/debug-cluster/), node-level signals and how to walk them |
| Serial-over-LAN concept | [OpenBMC: Host serial console](https://github.com/openbmc/docs/blob/master/console.md) and [OpenBMC: IPMITOOL cheatsheet](https://github.com/openbmc/docs/blob/master/IPMITOOL-cheatsheet.md) |
| Reading DHCP traces | [Wireshark: DHCP](https://wiki.wireshark.org/DHCP), `tcpdump -vv port 67 or port 68`, and [iPXE: Using ISC dhcpd](https://ipxe.org/howto/dhcpd) |
| Effective troubleshooting | [Google SRE Book: Effective Troubleshooting](https://sre.google/sre-book/effective-troubleshooting/), the systematic ordering you'll cite in Q2 below |

## Success criteria

1. Successfully induce all five failures.
2. For each, produce a one-page failure runbook: symptom → diagnostic path → fix → "what did the support ticket look like before vs. after you triaged it?"
3. Demonstrate you can identify the layer from symptom alone, before touching anything, on at least 4 of 5.
4. Be able to articulate the difference between an installer-stage and a target-stage cloud-init failure on inspection of the symptom alone.

## Conceptual questions

Each question has a **Read first** pointer to a source that teaches the underlying concept.

### 1. Why is serial console the gold standard for early-boot debugging?

What can it see that the OS-level logs can't? What part of the chain (firmware PXE, iPXE chainload, kernel handoff, initrd panic, Subiquity startup) produces output *only* on the serial console and never reaches `journalctl` because the system that would run journalctl doesn't exist yet?

**Read first:**
- [OpenBMC: Host serial console](https://github.com/openbmc/docs/blob/master/console.md), documents how the host UART console is shared with console clients via the BMC.
- [OpenBMC: IPMITOOL cheatsheet](https://github.com/openbmc/docs/blob/master/IPMITOOL-cheatsheet.md), command-level examples for IPMI Serial-over-LAN (`sol activate`).

### 2. Diagnostic ordering: where to look first

The same symptom ("node never appears") can come from any layer. What's a heuristic ordering, likelihood ÷ cost-to-check? Now that the OS is Ubuntu (SSH + `journalctl` available, unlike Talos), how does that change the ordering compared to a fully-locked-down OS? When does SSH save you a serial console hop, and when is it actually a *trap* that points you at the wrong layer?

**Read first:**
- [Google SRE Book: Effective Troubleshooting](https://sre.google/sre-book/effective-troubleshooting/), the framing of "check the cheap, common things first" and how to narrow a failure layer.
- [Brendan Gregg: USE Method](https://www.brendangregg.com/usemethod.html), a reusable methodology for narrowing down where in a stack a problem lives.

### 3. Observability at AI Cloud scale

What replaces "SSH to the lab box and read `journalctl`" when you have thousands of nodes? Sketch the path a single line of `cloud-init.log` should take from the node to a support engineer's screen in a 1000-node fleet. What components handle aggregation, retention, search, alerting? What does `cloud-init collect-logs` give you that ad-hoc `journalctl` doesn't?

**Read first:**
- [OpenTelemetry: Logs](https://opentelemetry.io/docs/concepts/signals/logs/), vendor-neutral model for representing log signals in an observability pipeline.
- [sapcc/redfish-exporter](https://github.com/sapcc/redfish-exporter) and [mrlhansen/idrac_exporter](https://github.com/mrlhansen/idrac_exporter), Prometheus exporter examples for BMC/iDRAC metrics.
- [Metal3: CAPM3 Metrics](https://book.metal3.io/capm3/metrics.html), metrics exposed by Cluster API Provider Metal3.

### 4. Failures invisible without physical access

What can't you see without physical access, even with a fully instrumented stack? What's the failure mode where the BMC itself is the problem and Redfish lies to you?

**Read first:**
- [OpenBMC: phosphor-host-postd](https://github.com/openbmc/phosphor-host-postd), the OpenBMC component for collecting and exposing host POST codes.
- For physical hardware, the vendor service manual for your chassis LCD/LED/IML diagnostics. Generic docs are not enough for pre-BMC failures.

### 5. Operational scale: 1,000 GPU nodes, flat L2 BMC network. What fails first, and what prevents it?

Think: broadcast domains, credential blast radius, firmware update storms, the sheer volume of synchronous Redfish polling against thousands of BMCs, `cloud-init` log volume hitting your aggregator when 200 nodes re-provision in a 10-minute window, kubelet TLS bootstrap CSR floods on a fresh cluster join. Apply the diagnostic surfaces table to a fleet, not a single node.

**Read first:**
- [Brendan Gregg: USE Method](https://www.brendangregg.com/usemethod.html), already cited in Q2; reapply it to the BMC-network case yourself.
- *Optional:* [Open Compute Project, Hardware Management](https://www.opencompute.org/projects/hardware-management), industry context for fleet-scale management network and firmware operating models.

## Exit artifact

`lab/vmetal/milestone-04-5/`:

- `notes.md`, five conceptual answers.
- `runbooks/`, one-page diagnostic runbook per failure mode (5 runbooks).
- `signals-by-layer.md`, your annotated diagnostic-surfaces table — the cheat sheet you'll consult during real tickets.
- `installer-vs-target.md`, a short note explaining how to tell from a customer's symptom whether they're SSH'd into the live-server installer or the deployed system. This is the single most common ticket-routing mistake.

When all five runbooks are written and you can route a fresh symptom to the right layer in one read, you're ready for **Milestone 5: Cluster API + Metal3**, where the diagnostic surfaces above stop being your direct interface and start being inputs to a controller.

---

Last Updated: 2026-05-01
