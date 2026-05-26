# Knowledge Base (KB)

# Case Study: Sustained Throughput Optimization After Resolving Network Bottlenecks

---

# Overview

After resolving major inter-VLAN and routing bottlenecks in a Proxmox + TrueNAS environment, additional testing revealed sustained SMB transfer speeds plateauing around 70–80 MB/s despite healthy gigabit networking.

This document covers:

* the remaining performance investigation,
* robocopy tuning experiments,
* ZFS tuning,
* SMB observations,
* HDD behavior analysis,
* and the final architectural conclusions.

The goal was to determine whether additional throughput could be achieved without replacing hardware.

---

# Environment

## Hypervisor

* Proxmox VE

## Storage VM

* TrueNAS SCALE VM
* Raw disk passthrough
* VirtIO SCSI

## Storage Layout

* Two independent ZFS pools
* One HDD per pool
* WD Red CMR drives

## Networking

* Dual Intel NICs
* SMB multichannel confirmed
* Inter-VLAN routing fixed previously via EdgeRouter X hardware offload
* Ubiquiti EdgeSwitch 24 250W

## Client

* Windows PC
* HDD-backed local storage

---

# Initial Symptoms

Even after resolving routing bottlenecks:

* transfers began quickly,
* burst throughput exceeded 1 Gbps aggregate,
* but sustained transfers eventually dropped into:

```text
30–80 MB/s
```

Observed behavior:

* initial bursts of 1.6–1.8 Gbps across both NICs,
* eventual stabilization around lower sustained throughput.

---

# Phase 1 — Storage Path Validation

---

# Disk Throughput Testing

## Proxmox / Linux Disk Test

Command:

```bash
dd if=/dev/zero of=/mnt/pool/testfile bs=1M count=20000 oflag=direct status=progress
```

Results:

```text
~1.7 GB/s
```

## Conclusion

The underlying disk path and passthrough mechanism were functioning properly.

This disproved:

* catastrophic passthrough issues,
* severe virtualization bottlenecks,
* or major ZFS malfunction.

---

# Phase 2 — Client Storage Investigation

A local Windows disk-to-disk copy operation only achieved:

```text
~60 MB/s
```

This suggested:

# client-side storage behavior was contributing significantly.

---

# WinSAT Benchmark

Command:

```powershell
winsat disk
```

Key Results:

| Metric           | Result    |
| ---------------- | --------- |
| Sequential Read  | ~133 MB/s |
| Sequential Write | ~194 MB/s |

## Interpretation

The HDD itself was healthy.

However:

# same-disk copies caused heavy seek contention.

This explained why:

* local copies behaved poorly,
* while synthetic sequential tests looked much faster.

---

# Phase 3 — robocopy Testing

Windows Explorer proved unreliable for benchmarking.

Testing shifted to:

# robocopy.

---

# Initial robocopy Testing

## Command

```powershell
robocopy source destination /J /MT:16
```

## Results

```text
~75 MB/s sustained
```

---

# Discovery — Excessive Threading Hurt HDD Performance

Reducing thread count improved throughput.

---

# Test Results

| Configuration | Result                    |
| ------------- | ------------------------- |
| /MT:16        | ~75 MB/s                  |
| /MT:4         | ~78 MB/s                  |
| /MT:1         | ~81 MB/s                  |
| No /MT        | Similar sustained runtime |

---

# Key Finding

The environment strongly preferred:

# long uninterrupted sequential writes.

Excessive robocopy threading caused:

* HDD seek thrashing,
* fragmented write ordering,
* additional metadata contention,
* less efficient ZFS transaction grouping.

---

# Architectural Conclusion

The remaining bottleneck was:

# normal spinning-disk behavior.

NOT:

* networking,
* VLAN routing,
* Proxmox,
* SMB multichannel,
* or virtualization.

---

# Phase 4 — TrueNAS / ZFS Tuning

---

# SMB Multichannel Verification

Command:

```powershell
Get-SmbMultichannelConnection
```

Result:

```text
SMB multichannel confirmed active.
```

This validated:

* multiple SMB flows,
* multi-NIC utilization,
* and proper SMB scaling behavior.

---

# Compression Verification

Checked:

```bash
zfs get compression
```

Result:

```text
compression=lz4
```

This was already optimal.

---

# Recordsize Optimization

Original setting:

```text
128K
```

Changed to:

```bash
zfs set recordsize=1M pool/dataset
```

Reasoning:

* workload consisted primarily of large sequential files,
* larger record sizes reduce metadata overhead,
* larger blocks improve sequential efficiency.

---

# Important Note

ZFS recordsize changes:

# only affect newly written data.

Existing files retain previous block layouts.

---

# SMB AIO Investigation

Command:

```bash
testparm -s | grep -i aio
```

Result:

```text
No explicit AIO settings configured.
```

Interpretation:
Modern Samba defaults were likely already sufficient.

Optional tuning considered:

```text
aio read size = 1
aio write size = 1
```

However:

* expected gains were minimal,
* and system behavior was already healthy.

---

# Sync Tuning Consideration

Potential tuning:

```bash
zfs set sync=disabled pool/dataset
```

This was intentionally NOT implemented.

Reason:

# preserving data integrity was prioritized over benchmark gains.

---

# Final robocopy Command Recommendation

For large-file migrations:

```powershell
robocopy "SOURCE" "\\NAS\SHARE" /TEE /S /E /DCOPY:DA /COPY:DAT /J /MT:1 /R:1 /W:1
```

---

# Parameter Breakdown

| Parameter | Purpose                               |
| --------- | ------------------------------------- |
| /J        | Unbuffered I/O                        |
| /MT:1     | Single-threaded HDD-friendly behavior |
| /R:1      | Minimal retries                       |
| /W:1      | Minimal retry delay                   |
| /TEE      | Console + log output                  |
| /COPY:DAT | Preserve data, attributes, timestamps |
| /DCOPY:DA | Preserve directory metadata           |

---

# Final Sustained Throughput

Observed stable sustained throughput:

```text
~80–82 MB/s
```

Equivalent to:

```text
~640–660 Mbps sustained
```

This was considered:

# healthy and expected

for:

* single-HDD ZFS pools,
* SMB,
* virtualization,
* routed VLANs,
* and spinning-disk storage.

---

# Key Lessons Learned

---

# 1. Benchmark Tools Matter

Windows Explorer copy graphs are unreliable.

Preferred tools:

* robocopy
* iperf3
* dd
* winsat

---

# 2. More Threads ≠ More Speed

On spinning disks:

# excessive concurrency hurts sequential throughput.

HDDs strongly prefer:

* long uninterrupted writes,
* low seek pressure,
* predictable queue behavior.

---

# 3. Burst Throughput ≠ Sustained Throughput

RAM caching and SMB buffering can temporarily exceed:

* actual storage write capability.

Sustained performance reflects:

* physical storage behavior,
* transaction grouping,
* and seek efficiency.

---

# 4. Healthy Systems Plateau Predictably

Once catastrophic bottlenecks were resolved:

* tuning changes produced smaller gains,
* behavior became consistent,
* and performance stabilized.

This is characteristic of:

# a healthy infrastructure stack.

---

# 5. Architecture Matters More Than Micro-Tuning

The next major performance leap would come from:

* SSD pools,
* mirrored vdevs,
* larger ARC/RAM,
* faster client disks,
* or multi-gig networking.

NOT from endlessly tuning robocopy parameters.

---

# Final Environment Status

| Component          | Status                           |
| ------------------ | -------------------------------- |
| Network            | healthy                          |
| VLAN routing       | healthy                          |
| SMB multichannel   | active                           |
| Proxmox            | healthy                          |
| TrueNAS VM         | healthy                          |
| ZFS pools          | healthy                          |
| Dual NIC setup     | functioning                      |
| Storage limitation | spinning-disk sustained behavior |

---

# Final Conclusion

The environment evolved from:

```text
"The NAS/network appears broken"
```

to:

```text
A stable, predictable, correctly functioning virtualized NAS environment.
```

The remaining limitations were determined to be:

# normal storage-mechanical realities

rather than:

* configuration failures,
* networking faults,
* or virtualization issues.

---

# Recommended Future Upgrade Path

When future upgrades become desirable:

| Upgrade           | Expected Benefit             |
| ----------------- | ---------------------------- |
| SSD client drives | smoother sustained transfers |
| SSD ZFS pool      | major throughput increase    |
| Mirrored vdevs    | better IOPS + concurrency    |
| Additional RAM    | larger ARC cache             |
| 2.5GbE / 10GbE    | beneficial once SSD-backed   |

---

# Final Outcome

The environment now operates:

* reliably,
* predictably,
* safely,
* and within expected architectural limits.

The troubleshooting process successfully isolated:

* networking bottlenecks,
* routing bottlenecks,
* client-storage behavior,
* and HDD queue characteristics,

while avoiding unnecessary hardware replacement or risky configuration changes.
