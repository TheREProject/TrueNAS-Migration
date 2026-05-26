# KB: Troubleshooting rsync to TrueNAS SCALE SMB Shares from Debian

## Summary
When copying files from Debian to a TrueNAS SCALE SMB share using rsync, 
users may encounter:
- `Operation not permitted (1)`
- `No such file or directory (2)`
- Unexpected folder structure behavior
- SMB mount confusion between GUI and CLI methods

This KB documents a working migration workflow and the troubleshooting 
process used to resolve these issues.

## Environment
- **Source:** Debian Linux desktop
- **Destination:** TrueNAS SCALE SMB share
- **Transfer Tool:** rsync

## Symptoms

### Symptom 1 — Operation not permitted (1)
rsync: [receiver] mkstemp "...": Operation not permitted (1)

- **Initial Assumption:** ACL or ZFS permissions issue on TrueNAS
- **Actual Cause:** rsync was attempting metadata operations unsupported or restricted over SMB

### Symptom 2 — No such file or directory (2)
link_stat "/home/user" --no-owner" failed: No such file or directory (2)

**Cause:** Malformed rsync syntax caused an option to be interpreted as 
part of the source path.

Broken syntax:
```bash
rsync ... "/home/user"--no-owner ...
```
Correct syntax:
```bash
rsync ... "/home/user" --no-owner ...
```

### Symptom 3 — Folder contents copied instead of folder itself
**Cause:** Trailing slash behavior in rsync.

## Root Cause Analysis

The issue was **not** caused by:
- TrueNAS ACLs
- ZFS readonly flags
- Dataset corruption
- SMB permissions

The primary issues were:
1. Incorrect rsync options for SMB
2. Path syntax errors
3. SMB mount confusion
4. rsync trailing-slash semantics

## Recommended SMB Workflow

### Why SMB Requires Different rsync Flags
SMB is not a native POSIX filesystem from Linux's perspective. Avoid preserving:
- Ownership
- ACLs
- xattrs
- Permissions

These frequently cause `Operation not permitted` and inconsistent metadata.

### Transfer Command
```bash
rsync -rltD --info=progress2 \
--no-owner --no-group --no-perms \
"/home/user/MyFolder" \
"/path/to/smb/share/"
```

### Verification Command
```bash
rsync -rltD --checksum --dry-run \
--no-owner --no-group --no-perms \
"/home/user/MyFolder" \
"/path/to/smb/share/"
```
**Expected Result:** No output = source and destination match

## SMB Mount Methods

### Recommended for Interactive Use: GUI Mount (GNOME Files / Nautilus)
1. Open Files
2. Select Other Locations
3. Connect to: `smb://TRUENAS/SHARE`
4. Authenticate
5. Bookmark the share if desired

### Important Note About GUI SMB Mounts
GUI SMB mounts use GVFS. Actual filesystem path typically becomes:
/run/user/1000/gvfs/smb-share:server=truenas,share=media

This is the path rsync must use.

### How to Find the Actual GVFS Path
```bash
mount | grep gvfs
```
or:
```bash
df -h | grep gvfs
```

## rsync Trailing Slash Behavior

| Command | Result |
|---|---|
| `rsync ... /source/folder /destination/` | Copies folder AND contents → `/destination/folder/` |
| `rsync ... /source/folder/ /destination/` | Copies ONLY contents → `/destination/<contents>` |

## Final Working Configuration

### Transfer
```bash
rsync -rltD --info=progress2 \
--no-owner --no-group --no-perms \
"/home/user/MyFolder" \
"/run/user/1000/gvfs/smb-share:server=truenas,share=media/"
```

### Verification
```bash
rsync -rltD --checksum --dry-run \
--no-owner --no-group --no-perms \
"/home/user/MyFolder" \
"/run/user/1000/gvfs/smb-share:server=truenas,share=media/"
```

## Lessons Learned

### SMB ≠ POSIX Filesystem
For SMB targets, simpler rsync flags are more reliable. 
Metadata preservation often causes failures.

### GUI Mounts Are Acceptable
- For one-time migrations: GVFS GUI mounts are simpler and sufficient
- For automation: kernel CIFS mounts are preferable

## Best Practices

**Recommended:**
- Use simplified rsync flags over SMB
- Use checksum verification after migration
- Avoid ACL/xattr preservation over SMB
- Use GUI mounts for interactive workflows

**Avoid:**
- `-A` flag
- `-X` flag
- Preserving Linux ownership over SMB
- Malformed quoted paths
- Accidental trailing slash mistakes

## Final Outcome
Migration from Debian → TrueNAS SCALE SMB share completed successfully using:
- GUI-mounted SMB share
- Simplified rsync options
- Checksum verification pass
