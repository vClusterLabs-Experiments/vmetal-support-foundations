# Milestone 1: Notes

Brief: [`../../milestone-01-bmc.md`](../../milestone-01-bmc.md)

## Pre-flight (one-time scaffolding)

- [ ] Lima VM running Ubuntu (`limactl list` shows it `Running`)
- [ ] libvirt + QEMU installed inside the VM (`virsh list --all` works)
- [ ] Isolated libvirt network + `node01` domain defined (`virsh net-list`, `virsh dumpxml node01`)
- [ ] sushy-tools bound to `node01` (`curl /redfish/v1/Systems/` returns the System)

## Files to produce

- [ ] `notes.md` (this file), answers to conceptual questions
- [ ] `redfish-power.sh`, sequence of HTTP calls satisfying the four success criteria
- [ ] `scaffolding/node01.xml`, the libvirt domain definition
- [ ] `scaffolding/provisioning-net.xml`, the libvirt network definition
- [ ] `scaffolding/sushy-config.conf`, the sushy-tools config

## Conceptual questions

### 1. What does a BMC fundamentally provide that the host OS cannot, and why must it be a separate compute element?

(your answer, think: host is off, host is wedged, host's NIC driver crashed, first OS install)

**Lab checkpoint:** `GET /redfish/v1/Systems/<id>` and confirm `PowerState` reflects what `virsh list` shows.

### 2. What state does the BMC own vs. the host?

(your answer, where does "boot device = PXE" actually live? what persists across host power cycle? across BMC reset?)

**Lab checkpoint:** PATCH a boot-device value, restart sushy-tools, GET it back. Power-cycle the libvirt domain, GET it back. Note what survived which.

### 3. Power actions are asynchronous. How should a consumer handle that?

(your answer, what shape of caller does an async API force? Hint: loop, not RPC.)

**Lab checkpoint:** POST `ComputerSystem.Reset = On`, immediately poll `PowerState` every 1s, log timestamps until convergence. Repeat from On → ForceOff. Record the convergence delays.

## Observations

(anything surprising, anything you want to remember, anything to come back to)
