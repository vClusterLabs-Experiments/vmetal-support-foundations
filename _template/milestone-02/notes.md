# Milestone 2: Notes

Brief: [`../../milestone-02-netboot.md`](../../milestone-02-netboot.md)

## Files to produce

- [ ] `notes.md` (this file)
- [ ] `boot-override-trace.md`, observed behavior of `BootSourceOverrideEnabled = Once` vs. `Continuous` (deferred from M1, exercised here)
- [ ] `boot-chain-trace.md`, annotated `tcpdump` of the DHCP exchange
- [ ] `ipxe-script.ipxe`, your custom iPXE script with comments
- [ ] `failure-modes.md`, what happened when you broke TFTP vs. DHCP
- [ ] `scaffolding/dnsmasq.conf`
- [ ] `scaffolding/http-server-notes.md`

## Anchor question

> What is the smallest amount of state a server needs to know about itself before it can boot?

(your answer)

## Conceptual questions

### 1. Why does iPXE do its own DHCP after being loaded by PXE?

(your answer, what info does iPXE need that the firmware DHCP didn't provide? why not reuse the lease?)

### 2. Why the two-step (PXE → iPXE → kernel) instead of having firmware load the kernel directly?

(your answer)

### 3. The node has no disk and no persistent config. What guarantees deterministic boot behavior across power cycles? What would you change to make it vary per-node?

(your answer)

### 4. DHCP is broadcast and trusts the first reply. What does that imply for security on a shared L2? How is this prevented in production?

(your answer, DHCP snooping, isolated provisioning VLANs)

### 5. 500 nodes PXE-boot simultaneously. What melts first?

(your answer, TFTP is UDP, single-threaded per transfer, slow)

### 6. When the BMC says `boot=Pxe`, where does that flag live? UEFI, BIOS, or both?

(your answer)

## Observations

(anything surprising)