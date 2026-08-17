# 📝 Storage & Permission Documentation: Unprivileged Media Server (Ext4 Only)
## 🎯 Architecture Overview
This document covers the complete storage permission blueprint for an unprivileged Proxmox LXC container running on an Intel N100 iGPU server.

* Host Storage Layer: Uses standard Linux POSIX ACLs (Access Control Lists) on Ext4 mount points to create a high-range shared entry bridge (GID 110000).
* Container Layer: Uses a custom hardware-level UID/GID map inside the unprivileged LXC configuration files. This maps internal user 1001 directly to host user 110 (`jellyfin`) and host group 44 (`video`) to eliminate data access friction.

------------------------------
## 💾 Section 1: Host Filesystem Storage & POSIX ACL Setup
To ensure unprivileged LXC containers can read and write to your host storage pools, initialize a shared system group on the Proxmox host using POSIX ACLs.
## 1. Create Paths and Apply ACL Inheritance Rules
Run these commands on the Proxmox Host CLI to provision your directories and enforce the 110000 high-group permissions recursively:

# 1. Install standard ACL tool utilities
```
apt-get install acl -y
```
# 2.System directories on the Ext4 drive
```
/mnt/pve/data-no-backup
```
# 3. Apply the 110000 LXC group permissions recursively
```
setfacl -R -m g:110000:rwx /mnt/pve/data-no-backup
```
# 4. Enforce Default Inheritance Rules (forces all new files to inherit group 110000)
```
setfacl -R -d -m g:110000:rwx /mnt/pve/data-no-backup
```
# 5. Lock directory group inheritance using the Linux SetGID bit
```
find /mnt/pve/data-no-backup/videos -type d -exec chmod g+s {} +
```

## 🔍 Verification: Checking Host ACLs
To check if the ACL rules are active on a directory, execute:
```
getfacl /mnt/pve/data-no-backup/videos/
```
Expected Output snippet:
```
group:110000:rwx
default:group:110000:rwx
```
------------------------------
## 🔗 Section 2: Custom UID/GID Mapping for the Arr LXC
This setup overrides the default Proxmox 100000 shift rule for the arr application user (UID/GID 1001) so it writes out to disk directly as UID 110 (`jellyfin`) and GID 44 (`video`).
## 1. Whitelist Host IDs on Proxmox
Run these commands on the Proxmox Host CLI to allow the container namespace to interact with low-number system accounts:

# Allow root to map host UID 110 (`jellyfin`)
```
echo "root:110:1" >> /etc/subuid
```
# Allow root to map host GID 44 (`video`)
```
echo "root:44:1" >> /etc/subgid
```
## 2. Inject the Custom ID Mapping Table
Open your container configuration file on the Proxmox Host CLI (replace `[VMID]` with your container ID, e.g., 101):
```
nano /etc/pve/lxc/[VMID].conf
```
Append this mathematically exact translation table to the bottom of the configuration file:
```
# --- UID Mapping (User) ---
# 1. Map container UIDs 0 through 1000 (total of 1001 IDs) -> host 100000
lxc.idmap: u 0 100000 1001
# 2. Map container UID 1001 directly to host UID 110 (jellyfin)
lxc.idmap: u 1001 110 1
# 3. Map container UIDs 1002 through 65535 (total of 64534 IDs) -> host 101002
lxc.idmap: u 1002 101002 64534

# --- GID Mapping (Group) ---
# 1. Map container GIDs 0 through 1000 (total of 1001 IDs) -> host 100000
lxc.idmap: g 0 100000 1001
# 2. Map container GID 1001 directly to host GID 44 (video)
lxc.idmap: g 1001 44 1
# 3. Map container GIDs 1002 through 65535 (total of 64534 IDs) -> host 101002
lxc.idmap: g 1002 101002 64534
```
## 3. Fixing the Permission Denied / `nobody:1002` Home Directory Error
Changing user mapping models on a pre-existing container breaks its internal home folder properties because the internal files are physically stamped with old, unmapped translation IDs. This must be repaired from the host filesystem level while the container is locked down.
Run these repair commands on the Proxmox Host CLI:

# 1. Stop the container to cleanly lock storage files
```
pct stop [VMID]
```
# 2. Mount the container file system directly onto the host terminal
```
pct mount [VMID]
```
# 3. Force correct ownership using translated host credentials (110:44)# This forces the physical folder inside the rootfs to realign with the new translation bridge
```
chown -R 110:44 /var/lib/lxc/[VMID]/rootfs/home/arr
```
# 4. Unmount and spin the container back online
```pct unmount [VMID]
pct start [VMID]
```
------------------------------
## 🧪 Section 3: Final Verification Loop## 1. Test from inside the Container:
Log into your LXC console and execute a write action inside your mapped storage path:

# Ensure home folder ownership displays cleanly as `arr:arr`
```
ls -la /home/
```
# Switch user and execute write command to the mount point
```
su - arr
touch /data/mapping_test.txt
```
## 2. Confirm on the Host Filesystem:
On your Proxmox Host, run an explicit attribute lookup on the newly generated item to verify the loop is closed:
```
ls -lh /mnt/pve/data-no-backup/videos/mapping_test.txt
```
Expected Result: The file will display `jellyfin` as its owner and `video` as its group on the host, proving that your storage configurations are complete and the arr home directory is perfectly functional.
