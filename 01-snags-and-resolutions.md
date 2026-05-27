# Snags and their resolutions.


## Backup Type
- During Proxmox VMs Backup, the original backup attempted to backup the OS AND Data disks.
  - Under normal circumstances, this is fine. However, there are not enough resources for a full offsite backup at this time. You must disable backups for the data disks in their advanced options under the VM’s hardware section.
  
- During the Jellyfin backup, the backup directory was inaccessible. This is a permissions issue. The actual owner is not root or a linux user. Rather, the jellyfin admin is the owner (different from the Linux admins.)
  - This was solved by changing ownership of the directory to the current user and copying the contents out of the folder. Original permissions were reassigned.
 
## Hardware Snag    
-  After constructing the machine, it would not power on. 
  - This was resolved by using the other 4 pin CPU connector on the motherboard.

## Network Snag
- Proxmox successfully booted. But there was no network activity.
  - This is because we migrated the hardware to another motherboard, and by extension, another NIC. This needs to be configured by confirming the NIC is enabled in the BIOS and Proxmox.
  - After a motherboard swap, the NIC identity changes. Proxmox loses the previous network configuration — requiring manual interface reassignment.
  - Once that is complete, you can assign IP Addressing, either keeping the current setup, or adjusting for a new subnet if applicable.
  - Important commands to remember
    - ip a (shows all network interfaces and their LAN IPs)
    - ifup/ifdown *interface name* (Manual “reboot” of selected interface)
    - nano /etc/network/interfaces (configure interfaces)

## Adding and initializing disk to Proxmox VM (TrueNAS Scale)
### Disk uniqueness
- After attempting to add disk 1 to the current TrueNAS Pool, we received a message.

*“Warning: There are 1 disks available that have non-unique serial numbers. Non-unique serial numbers can be caused by a cabling issue and adding such disks can cause data loss.*

- Proxmox does not pass through unique serial numbers to virtual machines by default—even if the physical disks have unique serials. 
  - To resolve this, for each disk line (e.g., scsi0:, virtio0:, or sata0:), append a unique serial:

  - (e.g)   scsi0: /dev/disk/by-id/ata-WDC-XXXX,serial=UNIQUE123
  - (e.g 2) scsi1: /dev/disk/by-id/ata-WDC-YYYY,serial=UNIQUE456   

## Unable to add disk to TrueNAS Pool
- With the serial key issues resolved, another issue arose. Disk 1 could not be added to the current Pool. This is because Scale does not accept Pools with white space in the name (Big Pool) The previous name was grandfathered in. 

  - To resolve this, we need to rename the pool (which may prevent future issues)
  - **NOTE:** If your system-dataset is on the pool, its a good idea to move it back to a different pool, or perhaps the boot pool. 
  1. Firstly, you need to export your pool that you want to rename from the GUI. **Note its current name.**
    - System > Advanced Settings > Storage > System Dataset Pool. Configure this and choose a different pool or boot pool.

- Once that is finished fire up a terminal and import the pool into the CLI with the new name.
  - zpool import original_name new_name
    -(replace original_name and new_name, respectively)

  
- You can check your handy work with zpool status new_name
- Then export the pool again
  - zpool export new_name
- Then import it again in the GUI
- Pools -> Add, import existing
- Job Done.

## Unlocking Encrypted Drive
- Yet another snag arrived when attempting to unlock the encrypted datasets after the renaming. The likely cause may be tied to the key file containing the previous pool name. 
  - This is resolved by manually opening the .JSON file in notepad, copying the key, and pasting it in the option to enter the key manually.
 
## Do not add disks as stripe to a RAID 0 pool    
Yet another snag. Adding a disk with striped data (RAID) to the current pool was not a good idea.

While the pool now has more space, there is no way to properly arrange the data. As both disks are treated quite literally as one pool. Because data is spread across both disks, failure/removal of one will cause the pool to be inaccessible. 

Had to backup all critical data before removing disk 1 and create a unique pool for it.

Even after that removal, the original pool needed to be deleted and recreated too. Which was ok because all important data was backed up anyway.

**In the future, unless another disk is used for pool redundancy, DO NOT ADD A DISK TO A CURRENT POOL. CREATE A NEW POOL AND ADD THE DISK TO THAT NEW POOL.**

# Data transfer speeds issue
Noticed during the backup that transfer speeds from NAS to PCs were very slow. Usually around 18 MB/s.

The resolution is covered in another document.
