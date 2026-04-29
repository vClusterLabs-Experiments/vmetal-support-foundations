# Milestone 4.5: Observability of the Boot Chain

**Required for M5.** M5's reconciliation exercise (kill a provisioned node and watch the controller react) is unreadable without the diagnostic surfaces this milestone teaches.

**Goal:** Build the diagnostic muscle that distinguishes a support engineer from someone who just installs things. When 1 of 500 nodes fails to provision, where do you look, in what order, and what do you look for?

## What you're actually learning

- **The boot chain has many layers; a failure surfaces at one and points back to another.** Symptom location ≠ root cause location.
- **The signals available at each layer** and which one you reach for first.
- **Pattern recognition**: each failure mode has a fingerprint. After this milestone you should be able to look at a symptom and predict the layer.

## Anchor question

> Given only a customer report of "node won't provision," what's the minimum-cost diagnostic path that lands on the right layer?

## Lab exercise

Induce four failures. For each, record:
- The exact symptom (what does the customer see?)
- The layer where the failure actually originates
- The first three things you'd check, in order, and why

| # | Failure to induce | Hint |
|---|---|---|
| 1 | BMC unreachable | Stop sushy-tools, attempt power-on |
| 2 | DHCP misconfiguration | Wrong `next-server`, observe firmware behavior |
| 3 | Bad iPXE script | Typo in URL, observe where the chain breaks |
| 4 | Bad machine config | Invalid YAML, OS boots but never converges |

## Diagnostic surfaces to learn

| Layer | Where the signal lives | How to read it |
|---|---|---|
| BMC | sushy-tools logs / Redfish event log on real hw | `journalctl`, `/redfish/v1/Systems/{id}/LogServices/` |
| DHCP/TFTP | dnsmasq logs, `tcpdump` on the bridge | `journalctl -u dnsmasq`, `tcpdump -i virbrN port 67 or 69` |
| HTTP (iPXE script + kernel/initrd + machine-config fetch) | HTTP server access log, `tcpdump` for the fetch | `journalctl -u nginx` / `python -m http.server` stdout, `tcpdump -i virbrN port 80 or port 443`, most M3+ failures land here |
| iPXE | Serial console of the booting node | QEMU `-serial`, real hw IPMI SOL |
| OS first-boot | Same serial console + `talosctl dmesg` | Talos exposes a dmesg API even when broken |
| Talos node liveness | `talosctl health`, `talosctl service` | runs against the Talos API; works pre-K8s and tells you which Talos services are up |
| Cluster join | API server audit log, kubelet logs | `kubectl logs -n kube-system`, kubelet `journalctl` |

## Reading list

| Topic | Source |
|---|---|
| Serial-over-LAN concept | [OpenBMC: Host serial console](https://github.com/openbmc/docs/blob/master/console.md) and [OpenBMC: IPMITOOL cheatsheet](https://github.com/openbmc/docs/blob/master/IPMITOOL-cheatsheet.md) |
| Talos dashboard / dmesg | [Talos: CLI reference](https://docs.siderolabs.com/talos/v1.12/reference/cli), focus on `dmesg`, `logs`, `read`, and `service` |
| Reading DHCP traces | [Wireshark: DHCP](https://wiki.wireshark.org/DHCP), `tcpdump -vv port 67 or port 68`, and [iPXE: Using ISC dhcpd](https://ipxe.org/howto/dhcpd) |

## Success criteria

1. Successfully induce all four failures.
2. For each, produce a one-page failure runbook: symptom → diagnostic path → fix.
3. Demonstrate you can identify the layer from symptom alone, before touching anything, on at least 3 of 4.

## Conceptual questions

Each question has a **Read first** pointer to a source that teaches the underlying concept.

### 1. Why is serial console the gold standard for early-boot debugging?

What can it see that the OS-level logs can't?

**Read first:**
- [OpenBMC: Host serial console](https://github.com/openbmc/docs/blob/master/console.md), documents how to connect to the host UART console through OpenBMC and how UART data is shared with console clients.
- [OpenBMC: IPMITOOL cheatsheet](https://github.com/openbmc/docs/blob/master/IPMITOOL-cheatsheet.md), lists command-level examples for remote BMC operations.

### 2. Diagnostic ordering: where to look first

The same symptom ("node never appears") can come from any of five layers. What's a heuristic ordering, likelihood × cost-to-check?

**Read first:**
- [Google SRE Book: Effective Troubleshooting](https://sre.google/sre-book/effective-troubleshooting/), the general framing of "check the cheap, common things first" and a systematic ordering for narrowing the failure layer.
- [Brendan Gregg: USE Method](https://www.brendangregg.com/usemethod.html), a reusable methodology for narrowing down where in a stack a problem lives.

### 3. Observability at AI Cloud scale

What replaces "SSH to the lab box and read journalctl" when you have thousands of nodes?

**Read first:**
- [OpenTelemetry: Logs](https://opentelemetry.io/docs/concepts/signals/logs/), vendor-neutral model for representing log signals in an observability pipeline.
- [sapcc/redfish-exporter](https://github.com/sapcc/redfish-exporter) and [mrlhansen/idrac_exporter](https://github.com/mrlhansen/idrac_exporter), Prometheus exporter examples; inspect each README for the exact BMC or iDRAC metrics exposed.
- [Metal3: CAPM3 Metrics](https://book.metal3.io/capm3/metrics.html), documents metrics exposed by Cluster API Provider Metal3.

### 4. Failures invisible without physical access

What can't you see without physical access, even with a fully instrumented stack?

**Read first:**
- [OpenBMC: phosphor-host-postd](https://github.com/openbmc/phosphor-host-postd), OpenBMC component for collecting and exposing host POST codes.
- For physical hardware, use the exact vendor service manual for your chassis LCD/LED/IML diagnostics. Generic docs are not enough for pre-BMC failures.

### 5. Operational scale: 1,000 GPU nodes, flat L2 BMC network. What fails first, and what prevents it?

Think: broadcast domains, credential blast radius, firmware update storms, the sheer volume of synchronous Redfish polling against thousands of BMCs.

This question is here, not in M1, because you now have first-hand experience with what the BMC + boot chain produces under stress (the four induced failures above). Apply the diagnostic surfaces table to a fleet, not a single node.

**Read first:**
- [Brendan Gregg: USE Method](https://www.brendangregg.com/usemethod.html), already cited in Q2 above; reapply it to the BMC-network case yourself.
- *Optional:* [Open Compute Project, Hardware Management](https://www.opencompute.org/projects/hardware-management), industry context for fleet-scale management network and firmware operating models.

## Exit artifact

`lab/vmetal/milestone-04-5/`:

- `notes.md`, five conceptual answers.
- `runbooks/`, one-page diagnostic runbook per failure mode.
- `signals-by-layer.md`, your annotated table of where to look for what.

---

Last Updated: 2026-04-29
