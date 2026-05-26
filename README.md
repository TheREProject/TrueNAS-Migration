# TrueNAS-Migration
This project documents a full storage infrastructure migration from TrueNAS Core to TrueNAS Scale, including hardware migration, network troubleshooting, storage configuration, and data integrity validation.

What started as a straightforward platform upgrade turned into a multi-phase troubleshooting journey covering virtualization, VLAN networking, ZFS storage, and SMB performance. Every snag was documented as it happened — not cleaned up after the fact — so the repository reflects the real process rather than an idealized walkthrough.
What this project covers:

- Proxmox VE deployment and NIC reconfiguration following a hardware migration.
- TrueNAS Scale VM deployment with raw disk passthrough and ZFS pool configuration.
- A nine-phase network performance investigation that traced severe inter-VLAN throughput degradation to disabled VLAN hardware offloading on a Ubiquiti EdgeRouter X.
- ZFS recordsize tuning and SMB multichannel validation for large sequential workloads.
- Cross-platform data migration from Debian Linux to TrueNAS Scale SMB shares using rsync.
- Checksum-based data integrity verification and ZFS scrub validation post-migration.
