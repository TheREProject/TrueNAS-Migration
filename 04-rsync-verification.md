# KB: Verifying rsync Copies on TrueNAS Using Checksum Comparison

## Overview
This guide explains how to verify that files copied via rsync from a Linux system 
to a TrueNAS dataset are intact and identical in content, even after the transfer 
has already completed.

This method validates data integrity, not file permissions or ownership.

## Key Concept: What You Are Verifying

When validating a copy, differences fall into two categories:

### Critical (Data Integrity)
- File contents
- Verified using `--checksum`
- Indicated in rsync output by: `c`
- Any `c` = actual data mismatch ❌

### Non-Critical (Metadata)
- Permissions (`p`)
- Group (`g`)
- Owner (`o`)
- Timestamps (`t`)

These differences are common in OpenZFS systems and do **NOT** indicate data corruption.

## Recommended Verification Method

### Step 1: Run rsync in verification mode
```bash
rsync -avnic --checksum /source/path/ /destination/path/
```

**Flags Explained**
- `-a` = archive mode (preserves structure)
- `-v` = verbose output
- `-n` = dry run (no changes made)
- `-i` = itemized differences
- `-c` = checksum-based file comparison

### Step 2: Interpret Output

#### A. No Output or Minimal Output
✔ Files match (or only harmless metadata differences)

#### B. Metadata Differences Only (SAFE)
.f.....g...
.d.....g...

- `f` = file
- `d` = directory
- `g` = group differs

✔ Data is intact — only ownership/group mismatch

#### C. Permission / Timestamp Differences (SAFE)
.f...p....t.

- `p` = permissions differ
- `t` = timestamps differ

✔ Data is still intact

#### D. Critical Issue (DATA MISMATCH)
fc........ file.txt

- `c` = checksum/content mismatch

❌ File contents are different

## Common Causes of Metadata Differences in TrueNAS
- Different UID/GID mappings between systems
- SMB or NFS permission translation
- Dataset ACL configuration differences
- Default ownership behavior in TrueNAS datasets

## Content-Only Verification (Ignore Metadata)

Use this if you only care about file integrity:

```bash
rsync -rnic --checksum \
  --no-perms --no-owner --no-group --no-times \
  /source/path/ /destination/path/
```

✔ No output = identical file contents

## Post-Verification (Recommended)

Run a scrub on the storage pool:

```bash
zpool scrub poolname
zpool status
```

This verifies on-disk integrity using OpenZFS.

## Summary
- `rsync --checksum` validates file contents
- Metadata differences are expected in TrueNAS environments
- Only `c` indicates real data mismatch
- ZFS scrub provides additional block-level protection
- Your data is considered safe if no `c` differences appear
