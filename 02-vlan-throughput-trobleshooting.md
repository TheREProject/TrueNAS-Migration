# Knowledge Base: Diagnosing Severe Inter-VLAN Throughput Bottlenecks in a Proxmox + TrueNAS Environment

## Overview

This document details a full troubleshooting journey involving:

- Proxmox VE
- TrueNAS SCALE VM
- Intel Gigabit NICs
- Managed VLAN networking
- Ubiquiti EdgeSwitch + EdgeRouter X
- Severe asymmetric throughput bottlenecks

The issue initially appeared to be:

- a Proxmox bridge problem,
- virtualization overhead,
- weak NAS hardware,
- ZFS limitations,
- NIC driver issues,
- PCIe bandwidth problems,
- or aging motherboard/chipset constraints.

However, the true root cause was eventually isolated to:

# Disabled VLAN Hardware Offloading on the EdgeRouter X

This KB documents the full methodology, tests performed, false leads, and final resolution.

---

# Environment

## Original Server Hardware

- AMD FM1 platform
- AMD A75 chipset motherboard
- DDR3 memory
- Intel Gigabit NIC (Intel I210-class behavior)
- Proxmox VE host
- TrueNAS SCALE running as a VM
- ZFS storage pool

## Network Infrastructure

- Ubiquiti EdgeSwitch 24 250W
- Ubiquiti EdgeRouter X
- VLAN segmented network
- Inter-VLAN routing performed by ER-X

## Symptoms

### Initial Observed Throughput

Typical SMB and iperf3 throughput:

- ~200–500 Mbps
- Strong asymmetry between directions
- Uploads faster than downloads
- Multiple streams improved performance
- Same-subnet traffic often faster than inter-VLAN traffic

The network consistently negotiated at:

- 1000 Mbps Full Duplex

Yet real-world TCP throughput was substantially lower.

---

# Phase 1 — Initial Hardware Investigation

## NIC Validation

The Intel NIC was checked using:

```bash
ethtool enp2s0
```

Confirmed:

- 1000Mb/s
- Full duplex
- Link detected
- No CRC or carrier errors

NIC statistics were then reviewed:

```bash
ethtool -S enp2s0
```

Observed:

- No significant packet drops
- No CRC errors
- No TX/RX hardware faults
- MSI-X queues functioning

Conclusion:

# Physical link quality and NIC hardware appeared healthy.

---

# Phase 2 — Offloading Investigation

Several NIC offloading features were disabled to test for driver/virtualization overhead:

```bash
ethtool -K enp2s0 \
  tso off gso off gro off \
  tx off rx off sg off
```

Changes included:

- tx-checksum-ip-generic: off
- tx-generic-segmentation: off
- rx-gro: off
- tx-tcp-segmentation: off
- tx-tcp6-segmentation: off
- tx-checksum-sctp: off
- rx-checksum: off

Results:

- Minimal improvement
- Bottleneck persisted

Conclusion:

# NIC offloading was not the primary bottleneck.

---

# Phase 3 — Proxmox Bridge / Virtualization Investigation

## Theories Considered

Possible suspects included:

- Linux bridge overhead
- VirtIO bottlenecks
- TrueNAS VM overhead
- ZFS read performance
- PCIe slot bandwidth

## iperf3 Testing

Multiple iperf3 tests were run:

```bash
iperf3 -c <host> -P 4 -t 10
```

and:

```bash
iperf3 -c <host> -P 8 -t 20
```

Observed:

- Single-stream throughput poor
- Multi-stream throughput significantly better
- Typical totals:
  - 450–550 Mbps
  - occasionally 900 Mbps in one direction

This suggested:

- not a hard PHY limitation,
- but likely CPU or routing-path saturation.

---

# Phase 4 — PCIe / Motherboard Investigation

## PCIe Slot Concerns

The Intel NIC was initially installed in a standard PCIe x1 slot.

Theories:

- insufficient PCIe bandwidth,
- chipset lane limitations,
- FM1/A75 architectural ceiling.

NIC was moved between slots.

Result:

- No meaningful throughput change.

Conclusion:

# PCIe slot bandwidth was not the root cause.

---

# Phase 5 — Interrupt Saturation Investigation

Interrupt distribution was checked:

```bash
cat /proc/interrupts
```

During iperf3 testing:

```bash
iperf3 -P 8 -t 10
```

Interrupts showed:

- MSI-X queues distributed across multiple CPUs
- no single-core 100% saturation
- balanced NIC queue handling

Conclusion:

# Interrupt handling was functioning correctly.

This weakened the theory that the FM1 platform itself was fundamentally incapable of gigabit networking.

---

# Phase 6 — Direct Host Isolation Tests

## Critical Observation

A second Proxmox host (HP ProDesk 600 G2) was introduced.

### Same Subnet Tests

Results:

- Nearly full gigabit throughput both directions
- ~940 Mbps achievable

This immediately proved:

- switching fabric healthy
- NICs healthy
- Proxmox healthy
- server stack healthy

## Inter-VLAN Tests

Once traffic crossed VLAN boundaries:

Results collapsed to:

- ~652 Mbps one direction
- ~270 Mbps reverse

This was the breakthrough.

Conclusion:

# The bottleneck existed specifically on the routed inter-VLAN path.

---

# Phase 7 — Identifying the Router Bottleneck

## Network Topology Discovery

Environment:

```text
PC VLAN
   ↓
EdgeSwitch
   ↓
EdgeRouter X
   ↓
EdgeSwitch
   ↓
NAS VLAN
```

Inter-VLAN traffic was routed through the:

# Ubiquiti EdgeRouter X

The ER-X relies heavily on:

# Hardware Offloading

for near-gigabit routing.

Without offloading:

- routing becomes CPU-bound
- throughput collapses
- TCP asymmetry appears
- SMB performance degrades heavily

---

# Phase 8 — Verifying Hardware Offload Status

Command used:

```bash
show ubnt offload
```

Observed:

```text
IPv4 forwarding: enabled
IPv4 vlan: disabled
```

This was the smoking gun.

Meaning:

# VLAN traffic was being software-routed by the ER-X CPU.

This perfectly explained:

- same-VLAN fast
- inter-VLAN slow
- asymmetry
- multistream improvements
- UDP vs TCP differences

---

# Phase 9 — Enabling VLAN Hardware Offload

Entered configuration mode:

```bash
configure
```

Enabled VLAN hardware offload:

```bash
set system offload ipv4 vlan enable
```

Applied configuration:

```bash
commit
save
exit
```

Verified:

```bash
show ubnt offload
```

Expected result:

```text
IPv4 vlan: enabled
```

---

# Final Results

Immediately after enabling VLAN offload:

## Inter-VLAN iperf3

Achieved:

- Near gigabit throughput both directions
- ~930–940 Mbps
- Symmetric performance

## Real-World SMB Transfers

Observed:

- 100+ MB/s transfers
- Healthy NAS throughput
- Stable performance

Conclusion:

# The server was never the true bottleneck.

The root cause was:

# Disabled VLAN Hardware Offloading on the EdgeRouter X

---

# Important Technical Lessons

## 1. Same-VLAN vs Inter-VLAN Testing Matters

Always compare:

- same-subnet throughput
- routed inter-VLAN throughput

This instantly differentiates:

- switching problems
- routing problems
- endpoint problems

---

## 2. UDP vs TCP Testing Matters

UDP near line-rate while TCP suffers often indicates:

- CPU routing bottlenecks
- software forwarding
- ACK handling overhead
- firewall/offload issues

---

## 3. Multi-Stream Improvement Is a Clue

If:

```text
-P 8 is much faster than single-stream
```

it often points toward:

- scheduling bottlenecks
- routing overhead
- CPU packet processing constraints

rather than physical link issues.

---

## 4. Hardware Offloading Is Critical on EdgeRouter X

ER-X performance depends heavily on offloading.

Without VLAN offload:

- inter-VLAN routing becomes CPU-bound
- throughput can collapse dramatically

---

# Features That May Disable Offloading

Be careful enabling:

| Feature | Impact |
|---|---|
| Smart Queue QoS | disables offload |
| DPI / Traffic Analysis | may disable offload |
| IDS/IPS | disables offload |
| Certain bridge configs | may disable offload |
| Some advanced firewall features | may disable offload |

Always verify after changes:

```bash
show ubnt offload
```

Desired:

```text
IPv4 forwarding: enabled
IPv4 vlan: enabled
```

---

# Recommended Troubleshooting Flow

## Step 1 — Validate Physical Link

```bash
ethtool <interface>
```

Check:

- 1000Mb/s
- Full duplex
- no errors

---

## Step 2 — Check NIC Statistics

```bash
ethtool -S <interface>
```

Look for:

- drops
- CRC errors
- queue failures

---

## Step 3 — Compare Same-VLAN vs Inter-VLAN

If same-VLAN fast but inter-VLAN slow:

# investigate router/offload path immediately.

---

## Step 4 — Verify Router Hardware Offload

On ER-X:

```bash
show ubnt offload
```

Ensure:

```text
IPv4 vlan: enabled
```

---

## Step 5 — Test with iperf3

Single stream:

```bash
iperf3 -c <host>
```

Multi-stream:

```bash
iperf3 -c <host> -P 8
```

Reverse direction:

```bash
iperf3 -c <host> -R
```

UDP:

```bash
iperf3 -u -b 1G -c <host>
```

---

# Final Conclusions

## What Was NOT the Problem

- Proxmox VE
- Linux bridge
- VirtIO
- TrueNAS SCALE
- ZFS
- Intel NIC
- PCIe slot bandwidth
- FM1 motherboard
- DDR3 memory

## What WAS the Problem

# EdgeRouter X VLAN hardware offloading disabled

leading to:

# CPU software routing of inter-VLAN traffic.

---

# Final Outcome

After enabling VLAN offload:

- Near-gigabit routed throughput achieved
- SMB transfers normalized
- Server stack validated
- Future hardware upgrades became optional rather than mandatory

---

# Commands Reference

## Verify Link

```bash
ethtool enp2s0
```

## NIC Statistics

```bash
ethtool -S enp2s0
```

## Disable Offloads (Testing Only)

```bash
ethtool -K enp2s0 tso off gso off gro off tx off rx off sg off
```

## Interrupt Analysis

```bash
cat /proc/interrupts
```

## iperf3 Tests

```bash
iperf3 -c <host>
```

```bash
iperf3 -c <host> -P 8
```

```bash
iperf3 -c <host> -R
```

```bash
iperf3 -u -b 1G -c <host>
```

## ER-X Offload Verification

```bash
show ubnt offload
```

## Enable VLAN Offload

```bash
configure
set system offload ipv4 vlan enable
commit
save
exit
```

---

# Closing Notes

This troubleshooting journey demonstrated the importance of:

- systematic isolation,
- L2 vs L3 differentiation,
- validating assumptions,
- understanding hardware offload paths,
- and avoiding premature hardware replacement.

The final fix required:

# one configuration change.

But discovering the correct configuration required:

- layered testing,
- topology analysis,
- protocol comparison,
- and methodical elimination of false leads.

The dragon was slain.
