# Linux Storage Command Reference
> Generated from course notes | May 2026

---

## Quick Reference Table

| Command | One-liner description |
|---------|----------------------|
| `lsblk` | List block devices (disks, partitions, mount points) |
| `fdisk --list` | Show partition table and details for a disk |
| `cfdisk` | Interactive text-based partition editor |
| `mkswap` | Format a partition or file as swap space |
| `swapon` | Enable a swap partition or file |
| `swapoff` | Disable a swap partition or file |
| `dd` | Low-level data copy tool (create swap files, test I/O) |
| `mkfs.xfs` | Create an XFS filesystem on a partition |
| `mkfs.ext4` | Create an EXT4 filesystem on a partition |
| `tune2fs` | Display or modify ext2/3/4 filesystem parameters |
| `xfs_admin` | Manage XFS filesystem metadata (label, UUID) |
| `mount` | Attach a filesystem to a directory |
| `umount` | Detach a mounted filesystem |
| `findmnt` | Display mounted filesystems with options |
| `blkid` | Print block device attributes (UUID, type, label) |
| `exportfs` | Manage NFS server exports |
| `nbd-client` | Connect/disconnect Network Block Devices |
| `pvcreate` | Initialize a disk as an LVM Physical Volume |
| `pvs` | Summary list of Physical Volumes |
| `pvdisplay` | Detailed Physical Volume info |
| `pvremove` | Remove LVM label from a physical volume |
| `vgcreate` | Create a Volume Group from one or more PVs |
| `vgs` | Summary list of Volume Groups |
| `vgdisplay` | Detailed Volume Group info |
| `vgextend` | Add a PV to an existing Volume Group |
| `vgreduce` | Remove a PV from a Volume Group |
| `lvcreate` | Create a Logical Volume inside a VG |
| `lvs` | Summary list of Logical Volumes |
| `lvdisplay` | Detailed Logical Volume info |
| `lvresize` | Resize a Logical Volume (with or without filesystem) |
| `lvremove` | Delete a Logical Volume |
| `lvmdiskscan` | Scan all devices for LVM-capable disks |
| `iostat` | I/O statistics for devices and CPUs |
| `pidstat` | Per-process I/O statistics |
| `setfacl` | Set Access Control List entries on files/dirs |
| `getfacl` | Display Access Control List for a file/dir |
| `chattr` | Change file attributes (immutable, append-only) |
| `lsattr` | List file attributes |
| `mdadm` | Manage software RAID arrays |

---

## 1. Disk & Partition Management

### `lsblk`
**What it does:** Lists all block devices (disks and their partitions) in a tree view with mount points.

**Syntax:**
```bash
lsblk [OPTIONS] [DEVICE]
```

**Common flags:**
| Flag | Description |
|------|-------------|
| `-f` | Show filesystem type, label, and UUID |
| `-o NAME,TYPE,SIZE,MODEL` | Custom output columns |
| (no flag) | Default tree view with size and mount points |

**Examples:**
```bash
# List all block devices (disks and partitions)
lsblk

# Show filesystem info including UUID and label
lsblk -f

# Show specific device
lsblk /dev/vdb1

# Custom column output to identify local vs remote disks
lsblk -o NAME,TYPE,SIZE,MODEL
```

**Quick tip:** `lsblk` shows you *where* things are mounted. If a partition's MOUNTPOINTS column is empty, it's not mounted. Use `lsblk -f` to also see filesystem types — handy to confirm swap is active (`[SWAP]` appears in the MOUNTPOINTS column).

---

### `fdisk`
**What it does:** Displays disk partition tables or opens an interactive partition editor for a disk.

**Syntax:**
```bash
fdisk [OPTIONS] DEVICE
```

**Common flags:**
| Flag | Description |
|------|-------------|
| `--list` or `-l` | Display partition table (read-only, safe) |
| (no flag) | Interactive mode — can create/delete partitions |

**Examples:**
```bash
# Show partition table for a specific disk (safe, read-only)
sudo fdisk --list /dev/sda

# Show all disks and partitions on the system
sudo fdisk -l

# Verify partitions after creating them with cfdisk
sudo fdisk --list /dev/vdd
```

**Quick tip:** Always use `fdisk --list` (or `-l`) for inspection — it's read-only. Opening `fdisk` without flags starts interactive mode where accidental keystrokes can destroy data.

---

### `cfdisk`
**What it does:** A text-based, menu-driven partitioning tool — easier to use than `fdisk`. Lets you view, create, delete, resize, and write partition tables.

**Syntax:**
```bash
cfdisk DEVICE
```

**Partition table types when prompted:**
| Type | Description |
|------|-------------|
| `gpt` | GPT (GUID Partition Table) — modern, recommended for disks >2 TB |
| `dos` | MBR (Master Boot Record) — legacy, max 4 primary partitions |
| `sgi` | Silicon Graphics IRIX systems |
| `sun` | Sun Microsystems / Solaris |

**Examples:**
```bash
# Open cfdisk for disk vdd — will prompt for partition table if new disk
sudo cfdisk /dev/vdd

# After partitioning: validate with fdisk
sudo fdisk --list /dev/vdd
```

**⚠️ Note:** `cfdisk` edits entire disks. Changes only take effect when you choose **Write** from the menu. Navigate with arrow keys; selecting the wrong disk and writing can destroy all data. Always double-check the device name before opening.

---

## 🧪 Lab: Disk & Partition Management

**Objective:** Create partitions on a disk, inspect them, and delete one.

**Setup:** A fresh disk `/dev/vdd` (2 GB) with no partitions.

**Tasks:**
1. List all block devices to identify available disks → `lsblk`
2. Open `cfdisk` on `/dev/vdd` and create 3 partitions: 10 MB, 21 MB, 15 MB. Write and exit.
3. Verify all three partitions were created → `sudo fdisk --list /dev/vdd`
4. Open `cfdisk` again and delete the 10 MB partition. Write and exit.
5. (Challenge) Identify which partition hosts `/` (root filesystem) and save its name to a file.

**Solution:**
```bash
# Task 1
lsblk

# Task 2
sudo cfdisk /dev/vdd
# (Use arrow keys: New → 10M, New → 21M, New → 15M, Write, Quit)

# Task 3
sudo fdisk --list /dev/vdd

# Task 4
sudo cfdisk /dev/vdd
# (Select /dev/vdd1 → Delete → Write → Quit)

# Task 5 (Challenge)
lsblk | grep -E "part /$"
# Save partition name (e.g., vda1) to file:
sudo sh -c "lsblk | grep 'part /$' | awk '{print \$1}' | tr -d '├─└─' > /root/part"
```

---

## 2. Swap Space

### `swapon`
**What it does:** Activates a swap partition or swap file so Linux can use it as virtual memory.

**Syntax:**
```bash
swapon [OPTIONS] DEVICE_OR_FILE
```

**Common flags:**
| Flag | Description |
|------|-------------|
| `--show` | Display all currently active swap areas |
| `--verbose` | Show details during activation |
| `-a` | Activate all swap entries in `/etc/fstab` |

**Examples:**
```bash
# Check currently active swap areas
swapon --show

# Activate a swap partition with verbose output
sudo swapon --verbose /dev/vdb3

# Activate a swap file
sudo swapon /swap

# Activate all swap defined in /etc/fstab
sudo swapon -a
```

**Quick tip:** If you activate swap but don't add it to `/etc/fstab`, the swap will disappear after a reboot. Always add a persistent entry to make it survive restarts.

---

### `swapoff`
**What it does:** Deactivates a swap partition or file, stopping Linux from using it as virtual memory.

**Syntax:**
```bash
swapoff DEVICE_OR_FILE
```

**Examples:**
```bash
# Stop using a swap partition
sudo swapoff /dev/vdb3

# Stop using a swap file
sudo swapoff /swap
```

**Quick tip:** Run `swapon --show` before and after to confirm the swap was deactivated successfully.

---

### `mkswap`
**What it does:** Formats a partition or file as swap space, writing the swap header that Linux requires before `swapon` can activate it.

**Syntax:**
```bash
mkswap [OPTIONS] DEVICE_OR_FILE
```

**Common flags:**
| Flag | Description |
|------|-------------|
| `-L LABEL` | Assign a label to the swap space |

**Examples:**
```bash
# Format a partition as swap
sudo mkswap /dev/vdb3

# Format with a label
sudo mkswap -L SwapFS /dev/vdd

# Format a swap file (after creating it with dd)
sudo mkswap /swap
```

**Quick tip:** `mkswap` only *prepares* the space — you still need `swapon` to *activate* it. And before `mkswap`, you need to create the space first (partition via `cfdisk`, or file via `dd`).

---

### `dd`
**What it does:** Low-level data copying tool. Used to create swap files filled with zeros, test disk performance, or clone devices at the block level.

**Syntax:**
```bash
dd if=INPUT of=OUTPUT bs=BLOCK_SIZE count=NUM_BLOCKS [OPTIONS]
```

**Common flags:**
| Flag | Description |
|------|-------------|
| `if=/dev/zero` | Input from /dev/zero (infinite zeros) |
| `of=/path/file` | Output destination (file or device) |
| `bs=1M` | Block size (e.g., 1M = 1 megabyte per block) |
| `count=128` | Number of blocks to write |
| `status=progress` | Show progress during write |
| `oflag=dsync` | Synchronous writes — useful for I/O testing |

**Examples:**
```bash
# Create a 128 MB swap file filled with zeros
sudo dd if=/dev/zero of=/swap bs=1M count=128

# Create a 2 GB swap file with progress bar
sudo dd if=/dev/zero of=/swap bs=1M count=2048 status=progress

# Simulate 1 MB of synchronous writes (disk I/O test)
dd if=/dev/zero of=DELETEME bs=1 count=1000000 oflag=dsync &
```

**Quick tip:** After creating a swap file with `dd`, always set permissions with `chmod 600 /swap` before running `mkswap` — otherwise the kernel will refuse to use it.

---

### Complete Swap File Setup (Step-by-Step)

```bash
# Step 1: Create the swap file (128 MB)
sudo dd if=/dev/zero of=/swap bs=1M count=128 status=progress

# Step 2: Secure it — only root can read/write
sudo chmod 600 /swap

# Step 3: Format it as swap
sudo mkswap /swap

# Step 4: Activate it
sudo swapon /swap

# Step 5: Verify it's active
swapon --show

# Step 6: Make it persistent across reboots
echo '/swap   none   swap   sw   0  0' | sudo tee -a /etc/fstab
```

---

### `/etc/fstab` — Swap Entry

```
# Using device path (less reliable)
/dev/vdb3    none    swap    defaults    0    0

# Using UUID (recommended — survives cable/disk order changes)
UUID=a39c80a1-a0c6-4ba2-a4a3-8a82058d8859    none    swap    sw    0    0
```

| Field | Meaning |
|-------|---------|
| Device / UUID | What to mount |
| `none` | No mount point (swap has no directory) |
| `swap` | Filesystem type |
| `defaults` or `sw` | Mount options |
| `0 0` | No dump, no fsck (swap doesn't need checks) |

---

## 🧪 Lab: Swap Space

**Objective:** Create and activate a swap partition, then create a persistent swap file.

**Setup:** Disk `/dev/vdd` available with free space.

**Tasks:**
1. Check if any swap is currently configured → `swapon --show`
2. Format `/dev/vdd3` as swap and activate it with verbose output
3. Verify swap is active → `swapon --show`
4. Deactivate that swap → `sudo swapoff /dev/vdd3`
5. (Challenge) Create a persistent 256 MB swap file at `/swap2`, activate it, and add it to `/etc/fstab`

**Solution:**
```bash
# Task 1
swapon --show

# Task 2
sudo mkswap /dev/vdd3
sudo swapon --verbose /dev/vdd3

# Task 3
swapon --show

# Task 4
sudo swapoff /dev/vdd3

# Task 5 (Challenge)
sudo dd if=/dev/zero of=/swap2 bs=1M count=256 status=progress
sudo chmod 600 /swap2
sudo mkswap /swap2
sudo swapon /swap2
swapon --show
echo '/swap2   none   swap   sw   0  0' | sudo tee -a /etc/fstab
```

---

## 3. Filesystems

### `mkfs.xfs`
**What it does:** Creates an XFS filesystem on a partition or logical volume. XFS is the default for Red Hat / CentOS / Rocky Linux.

**Syntax:**
```bash
mkfs.xfs [OPTIONS] DEVICE
```

**Common flags:**
| Flag | Description |
|------|-------------|
| `-L LABEL` | Set a human-readable label (uppercase `-L`) |
| `-i size=512` | Set inode size to 512 bytes (default: 256) |
| `-f` | Force creation even if filesystem already exists |

**Examples:**
```bash
# Create a default XFS filesystem
sudo mkfs.xfs /dev/sdb1

# Create XFS with a label
sudo mkfs.xfs -L "BackupVolume" /dev/sdb1

# Create with larger inodes (useful for heavy metadata workloads)
sudo mkfs.xfs -i size=512 /dev/sdb1

# Force creation (overwrites existing filesystem — dangerous!)
sudo mkfs.xfs -f -i size=512 /dev/sdb1
```

**⚠️ Note:** Use uppercase `-L` for labels. Lowercase `-l` is a different option (log config) — a common mistake.

---

### `mkfs.ext4`
**What it does:** Creates an EXT4 filesystem on a partition. EXT4 is the default for Ubuntu/Debian systems.

**Syntax:**
```bash
mkfs.ext4 [OPTIONS] DEVICE
```

**Common flags:**
| Flag | Description |
|------|-------------|
| `-N COUNT` | Set the number of inodes explicitly |
| `-I SIZE` | Set inode size in bytes |
| `-L LABEL` | Set a label for the filesystem |

**Examples:**
```bash
# Create default EXT4 filesystem
sudo mkfs.ext4 /dev/sdb2

# Create with a specific number of inodes
sudo mkfs.ext4 -N 500000 /dev/sdb2

# Create with inode size of 2048 bytes and a label
sudo mkfs.ext4 -I 2048 -L "DataDisk" /dev/vde

# Verify inode count after creation
sudo dumpe2fs -h /dev/vde | grep "Inode count"
```

**Quick tip:** `mkfs.ext4` is a wrapper around `mke2fs` — both commands do the same thing. Check `man mke2fs` for the full option list.

---

### `xfs_admin`
**What it does:** Displays or changes parameters of an existing XFS filesystem, such as its label or UUID.

**Syntax:**
```bash
xfs_admin [OPTIONS] DEVICE
```

**Common flags:**
| Flag | Description |
|------|-------------|
| `-l` | Display the current label |
| `-L LABEL` | Change the label |

**Examples:**
```bash
# Show the current label of an XFS partition
sudo xfs_admin -l /dev/sdb1
# Output: label = "BackupVolume"

# Change the label
sudo xfs_admin -L "NewLabelName" /dev/sdb1
```

**Quick tip:** The device must not be mounted when changing the label, or you'll need to unmount first.

---

### `tune2fs`
**What it does:** Displays or modifies parameters of an EXT2/3/4 filesystem. Useful for inspecting UUID, checking mount count, and changing labels.

**Syntax:**
```bash
tune2fs [OPTIONS] DEVICE
```

**Common flags:**
| Flag | Description |
|------|-------------|
| `-l` | List all filesystem parameters (read-only) |
| `-L LABEL` | Change the filesystem label |

**Examples:**
```bash
# Show all filesystem info (UUID, features, inode count, etc.)
sudo tune2fs -l /dev/sdb2

# Change the label on an EXT4 partition
sudo tune2fs -L "SecondFS" /dev/sdb2
```

**Quick tip:** Use `tune2fs -l` to find a partition's UUID when building `/etc/fstab` entries — more reliable than remembering device names.

---

### XFS Utilities Reference

| Tool | Purpose |
|------|---------|
| `xfs_admin` | Change label, UUID |
| `xfs_repair` | Check and repair corrupted XFS |
| `xfs_info` | Display geometry of a mounted XFS filesystem |
| `xfs_growfs` | Expand XFS to fill more space |
| `xfs_freeze` | Freeze I/O for snapshots |
| `xfs_fsr` | Defragment files |
| `xfs_quota` | Manage disk quotas |
| `xfs_db` | Low-level debugging |

---

## 🧪 Lab: Filesystem Creation

**Objective:** Create and label filesystems on partitions.

**Setup:** Partitions `/dev/vdd` (XFS) and `/dev/vde` (EXT4) available.

**Tasks:**
1. Create an XFS filesystem with label `DataDisk` on `/dev/vdd`
2. Create an EXT4 filesystem with 2048 inodes on `/dev/vde`
3. Verify the inode count on `/dev/vde`
4. Display the label on `/dev/vdd`
5. (Challenge) Change the label on `/dev/vdd` to `BackupDisk` and verify

**Solution:**
```bash
# Task 1
sudo mkfs.xfs -L "DataDisk" /dev/vdd

# Task 2
sudo mkfs.ext4 -I 2048 /dev/vde

# Task 3
sudo dumpe2fs -h /dev/vde | grep "Inode count"

# Task 4
sudo xfs_admin -l /dev/vdd

# Task 5 (Challenge)
sudo xfs_admin -L "BackupDisk" /dev/vdd
sudo xfs_admin -l /dev/vdd
```

---

## 4. Mounting Filesystems

### `mount`
**What it does:** Attaches a filesystem (partition, device, or NFS share) to a directory in the filesystem hierarchy.

**Syntax:**
```bash
mount [OPTIONS] DEVICE MOUNTPOINT
```

**Common flags:**
| Flag | Description |
|------|-------------|
| `-o ro` | Mount read-only |
| `-o rw` | Mount read-write (default) |
| `-o noexec` | Prevent execution of binaries on this filesystem |
| `-o nosuid` | Prevent SUID/SGID bits from taking effect |
| `-o remount` | Remount with new options without unmounting |
| `-a` | Mount all filesystems listed in `/etc/fstab` |

**Examples:**
```bash
# Basic mount
sudo mount /dev/vdd /mnt

# Mount read-only
sudo mount -o ro /dev/vdb2 /mnt

# Mount with multiple security options
sudo mount -o ro,noexec,nosuid /dev/vdb2 /mnt

# Remount an already-mounted filesystem with different options
sudo mount -o remount,rw,noexec,nosuid /dev/vdb2 /mnt

# Mount an NFS share
sudo mount 127.0.0.1:/etc /mnt

# Test all fstab entries for syntax errors (safe, dry-run style)
sudo mount -a
```

**Quick tip:** After editing `/etc/fstab`, always run `sudo mount -a` to test for errors *before* rebooting. A broken fstab can prevent your system from booting.

---

### `umount`
**What it does:** Detaches a mounted filesystem. Note: the command is `umount`, not `unmount`.

**Syntax:**
```bash
umount DEVICE_OR_MOUNTPOINT
```

**Examples:**
```bash
# Unmount using the mount point
sudo umount /mnt

# Unmount using the device name
sudo umount /dev/vdd

# Verify it was unmounted
lsblk
```

**⚠️ Note:** The command is spelled `umount` (one 'n'), not `unmount`. Using `unmount` in your notes is incorrect — it won't work.

---

### `findmnt`
**What it does:** Displays all mounted filesystems with their options. More detailed than `lsblk` for mount option inspection.

**Syntax:**
```bash
findmnt [OPTIONS] [DEVICE_OR_MOUNTPOINT]
```

**Common flags:**
| Flag | Description |
|------|-------------|
| `-t TYPE` | Filter by filesystem type (e.g., `ext4,xfs`) |
| (no flag) | Show all mounts including virtual filesystems |

**Examples:**
```bash
# Show all mounted filesystems
findmnt

# Show only real disk filesystems (not virtual)
findmnt -t ext4,xfs

# Show mount options for the root filesystem
findmnt /

# Extract just the options field for scripting
findmnt / | awk '{print $4}'
# Then save to file:
sudo sh -c "findmnt / | awk '{print \$4}' > /root/moptions"
```

**Quick tip:** Use `findmnt` instead of `lsblk` when you need to see mount options. `lsblk` only shows where things are mounted; `findmnt` shows *how* they are mounted (ro, noexec, etc.).

---

### `blkid`
**What it does:** Prints the UUID, label, and filesystem type for block devices. Essential for building reliable `/etc/fstab` entries.

**Syntax:**
```bash
blkid [DEVICE]
```

**Examples:**
```bash
# Show UUID and type for a specific partition
sudo blkid /dev/vde
# Output: /dev/vde: UUID="0a6dba58-..." BLOCK_SIZE="4096" TYPE="ext4"

# Check all UUIDs on the system
ls -l /dev/disk/by-uuid/

# Alternative: lsblk shows UUID too
sudo lsblk /dev/vdb1
```

**Quick tip:** Always use UUID in `/etc/fstab` instead of device names like `/dev/sdb1`. Device names can change if you add/remove disks; UUIDs are permanent per filesystem.

---

### `/etc/fstab` Format

```
# DEVICE/UUID    MOUNTPOINT    FSTYPE    OPTIONS    DUMP    FSCK_ORDER
UUID=...         /mybackups    xfs       defaults    0       2
/dev/vda2        /boot         ext4      defaults    0       1
/dev/vdb3        none          swap      sw          0       0
127.0.0.1:/etc   /mnt          nfs       defaults    0       0
```

| Field | Values | Notes |
|-------|--------|-------|
| Device | UUID=… or /dev/sdX | UUID preferred |
| Mount point | /path or `none` | `none` for swap |
| FSType | ext4, xfs, swap, nfs | |
| Options | defaults, ro, noexec, nosuid | Comma-separated |
| Dump | 0 or 1 | 0 = no backup |
| FSCheck | 0, 1, or 2 | 1=root first, 2=others, 0=skip |

**After editing fstab:**
```bash
sudo systemctl daemon-reload   # Reload systemd unit files
sudo mount -a                  # Test all entries
```

---

## 🧪 Lab: Mounting

**Objective:** Mount filesystems manually and configure persistent boot-time mounts.

**Setup:** `/dev/vdd` (XFS), `/dev/vde` (EXT4) available and formatted.

**Tasks:**
1. Mount `/dev/vdd` to `/mnt` — verify with `lsblk`
2. Unmount `/mnt` — verify with `lsblk`
3. Mount `/dev/vdd` again with options: `ro,noexec,nosuid`
4. Remount it as read-write without unmounting
5. (Challenge) Configure `/dev/vde` to auto-mount at `/test` on boot using UUID in fstab. Create the directory, test with `mount -a`

**Solution:**
```bash
# Task 1
sudo mount /dev/vdd /mnt
lsblk

# Task 2
sudo umount /mnt
lsblk

# Task 3
sudo mount -o ro,noexec,nosuid /dev/vdd /mnt
findmnt -t xfs,ext4

# Task 4
sudo mount -o remount,rw,noexec,nosuid /dev/vdd /mnt

# Task 5 (Challenge)
sudo mkdir /test
sudo blkid /dev/vde   # Note the UUID
sudo vim /etc/fstab
# Add: UUID=<your-uuid>  /test  ext4  defaults  0  1
sudo mount -a
lsblk   # Verify /test is mounted
```

---

## 5. NFS (Network File System)

### Server Side

```bash
# Install NFS server
sudo apt install nfs-kernel-server

# Edit exports file
sudo vim /etc/exports
```

**`/etc/exports` format:**
```
/home  10.0.0.0/24(ro,sync,no_subtree_check)
/home  192.0.0.0/24(ro)  127.0.0.10(rw,no_root_squash)
```

| Option | Meaning |
|--------|---------|
| `ro` / `rw` | Read-only / Read-write |
| `sync` | Write to disk before replying (safer) |
| `no_subtree_check` | Skip subtree verification (recommended for performance) |
| `root_squash` | Map remote root → `nobody` (security) |
| `no_root_squash` | Allow remote root full access |
| `no_all_squash` | Keep real UIDs for normal users |

```bash
# Apply changes (re-export)
sudo exportfs -r

# Show active exports (verbose)
sudo exportfs -v
```

### Client Side

```bash
# Install NFS client
sudo apt install nfs-common

# Mount an NFS share manually
sudo mount 127.0.0.1:/etc /mnt
sudo mount server1:/etc /mnt

# Unmount
sudo umount /mnt

# Auto-mount at boot (/etc/fstab)
127.0.0.1:/home   /mnt   nfs   defaults   0   0
```

---

## 6. Network Block Devices (NBD)

NBD lets one Linux server use a disk that is physically on another server over TCP/IP — like NFS, but at the raw block level.

### Server Side (disk owner)

```bash
# Install NBD server
sudo apt install nbd-server

# Configure /etc/nbd-server/config
sudo vim /etc/nbd-server/config
```

```ini
[generic]
    includedir = /etc/nbd-server/conf.d
    allowlist = true

[partition2]
    exportname = /dev/sda1
```

```bash
# Restart NBD server
sudo systemctl restart nbd-server.service
```

### Client Side (remote user)

```bash
# Install NBD client
sudo apt install nbd-client

# Load the kernel module (required before connecting)
sudo modprobe nbd

# Make module load permanent at boot
echo 'nbd' | sudo tee -a /etc/modules-load.d/modules.conf

# List available exports from server
sudo nbd-client -l 127.0.0.1

# Connect to a remote disk
sudo nbd-client 127.0.0.1 -N partition2
# Or using hostname:
sudo nbd-client server2.example.com -N partition2

# Mount the remote disk
sudo mount /dev/nbd0 /mnt

# Unmount and disconnect when done
sudo umount /mnt
sudo nbd-client -d /dev/nbd0

# Verify (should show 0 size when disconnected)
lsblk | grep nbd
```

---

## 7. LVM (Logical Volume Manager)

LVM adds a virtualization layer between physical disks and the filesystems on top of them. It allows flexible resizing, pooling multiple disks, and snapshots.

**Concept flow:**  
`Physical Disk → PV (Physical Volume) → VG (Volume Group) → LV (Logical Volume) → Filesystem`

```bash
# Install LVM
sudo apt install lvm2
```

### Physical Volumes (PV)

```bash
# Scan for LVM-capable disks
sudo lvmdiskscan

# Initialize disks as physical volumes
sudo pvcreate /dev/sdc /dev/sdd

# Summary view
sudo pvs

# Detailed view
sudo pvdisplay
sudo pvdisplay /dev/vde

# Remove LVM label from a disk
sudo pvremove /dev/sde
```

### Volume Groups (VG)

```bash
# Create a VG from one or more PVs
sudo vgcreate my_volume /dev/sdc /dev/sdd

# Summary view
sudo vgs

# Detailed view
sudo vgdisplay

# Add a PV to an existing VG (extend)
sudo vgextend my_volume /dev/sde

# Remove a PV from a VG (data must be empty)
sudo vgreduce my_volume /dev/sde
```

### Logical Volumes (LV)

```bash
# Create a logical volume
sudo lvcreate --size 2G --name partition1 my_volume
sudo lvcreate --size 0.5g --name smalldata volume1

# Summary view
sudo lvs

# Detailed view (shows device path)
sudo lvdisplay

# Resize LV (no filesystem)
sudo lvresize --size 3G my_volume/partition1

# Resize LV AND its filesystem in one command
sudo lvresize --resizefs --size 3G my_volume/partition1

# Use 100% of remaining VG space for an LV
sudo lvresize --extents 100%VG my_volume/partition1

# Resize to exact size (e.g., 752 MB)
sudo lvresize -L 752M /dev/volume1/smalldata

# Remove a logical volume (unmount first if mounted)
sudo umount /dev/volume1/smalldata
sudo lvremove /dev/volume1/smalldata
# Skip confirmation prompt:
sudo lvremove -y /dev/volume1/smalldata
```

### Creating a Filesystem on LV

```bash
# Create EXT4 on logical volume
sudo mkfs.ext4 /dev/my_volume/partition1

# Create XFS on logical volume
sudo mkfs.xfs /dev/volume1/smalldata

# Mount it
sudo mount /dev/my_volume/partition1 /mydata
```

**⚠️ Note:** When resizing an LV that already has a filesystem, always use `--resizefs` flag to resize both the LV and the filesystem together. Using `lvresize` alone without `--resizefs` only changes the LV size — the filesystem won't know about it and data corruption can occur.

---

## 🧪 Lab: LVM

**Objective:** Build a full LVM stack from physical volume to mounted logical volume.

**Setup:** Two fresh disks `/dev/vdd` and `/dev/vde` available.

**Tasks:**
1. Initialize both disks as PVs → verify with `pvs`
2. Create VG `volume1` using `/dev/vdd`
3. Extend `volume1` by adding `/dev/vde`
4. Create LV `smalldata` with 500 MB in `volume1`
5. Create an XFS filesystem on `smalldata`, mount it at `/data`
6. (Challenge) Resize `smalldata` to 752 MB (resizing the filesystem too)

**Solution:**
```bash
# Task 1
sudo pvcreate /dev/vdd /dev/vde
sudo pvs

# Task 2
sudo vgcreate volume1 /dev/vdd
sudo vgs

# Task 3
sudo vgextend volume1 /dev/vde
sudo vgs

# Task 4
sudo lvcreate --size 0.5g --name smalldata volume1
sudo lvs

# Task 5
sudo mkfs.xfs /dev/volume1/smalldata
sudo mkdir /data
sudo mount /dev/volume1/smalldata /data
lsblk

# Task 6 (Challenge)
sudo lvresize --resizefs -L 752M /dev/volume1/smalldata
sudo lvs
```

---

## 8. RAID

RAID (Redundant Array of Independent Disks) combines multiple disks for redundancy or performance. Managed on Linux with `mdadm`.

| RAID Level | Also Called | Description |
|------------|-------------|-------------|
| RAID 0 | Striping | Performance — data split across disks, no redundancy |
| RAID 1 | Mirroring | Redundancy — data written to both disks identically |
| RAID 5 | Striping + Parity | Balance of performance and redundancy (min 3 disks) |
| RAID 6 | Double parity | Like RAID 5 but survives 2 disk failures (min 4 disks) |
| RAID 10 | RAID 1+0 | Mirror + stripe (min 4 disks) |

### `mdadm`

```bash
# Check current RAID status
cat /proc/mdstat

# Create a RAID 1 (mirror) array
sudo mdadm --create /dev/md0 --level=1 --raid-devices=2 /dev/vdd /dev/vde

# Save RAID config to survive reboot
sudo mdadm --detail --scan | sudo tee -a /etc/mdadm/mdadm.conf
sudo update-initramfs -u

# Create a filesystem on the RAID array
sudo mkfs.ext4 /dev/md0
sudo mkdir /mnt/raid1
sudo mount /dev/md0 /mnt/raid1

# Verify
df -h /mnt/raid1
sudo mdadm --detail /dev/md0
```

---

## 9. Storage Performance Monitoring

### `iostat`
**What it does:** Reports CPU and device I/O statistics. Part of the `sysstat` package.

```bash
# Install
sudo apt install sysstat

# Basic usage — show stats since boot
iostat

# Refresh every 1 second
iostat 1

# Device-only report (no CPU)
iostat -d

# Human-readable output (KB/MB instead of raw bytes)
iostat -h

# All partitions on all devices
iostat -p ALL

# Specific device and its partitions
iostat -p sda
```

**Output columns:**
| Column | Meaning |
|--------|---------|
| `tps` | Transactions (I/O operations) per second |
| `kB_read/s` | Kilobytes read per second |
| `kB_wrtn/s` | Kilobytes written per second |
| `%iowait` | % of CPU time waiting for I/O |
| `%idle` | % of CPU time idle |

---

### `pidstat`
**What it does:** Reports per-process I/O statistics. Helps identify which process is causing disk activity.

```bash
# Show per-process disk I/O
pidstat -d

# Update every 1 second
pidstat -d 1

# Human-readable format
pidstat -d --human

# After finding a PID consuming disk, inspect it
ps 1411
```

**Output columns:**
| Column | Meaning |
|--------|---------|
| `PID` | Process ID |
| `kB_rd/s` | Kilobytes read per second |
| `kB_wr/s` | Kilobytes written per second |
| `Command` | Process name |

**Workflow to diagnose high I/O:**
```bash
# Step 1: Find which device is busy
iostat 1

# Step 2: Find which process is writing to it
pidstat -d 1

# Step 3: Get full command details
ps <PID>

# Step 4: If it's a runaway process, kill it
kill <PID>
kill -9 <PID>   # Force kill (last resort)

# For LVM-backed devices, check dm-0 mapping
sudo dmsetup info /dev/dm-0
lsblk
```

---

## 10. ACL & File Attributes

### `setfacl`
**What it does:** Sets Access Control List (ACL) entries on files and directories, providing finer-grained permissions than standard Unix rwx.

**Syntax:**
```bash
setfacl [OPTIONS] RULE FILE
```

**Common flags:**
| Flag | Description |
|------|-------------|
| `--modify` or `-m` | Add or modify an ACL entry |
| `--remove` or `-x` | Remove a specific ACL entry |
| `--remove-all` or `-b` | Remove all ACL entries |
| `--recursive` or `-R` | Apply recursively to directory contents |

**Examples:**
```bash
# Install ACL tools
sudo apt install acl

# Grant user jeremy read+write on file3
sudo setfacl --modify user:jeremy:rw file3

# Grant group sudo read+write
sudo setfacl --modify group:sudo:rw file3

# Grant group mail read+execute
sudo setfacl --modify group:mail:rx /home/bob/specialfile

# Apply recursively to a directory (user gets rwx on dir and all contents)
sudo setfacl --recursive -m user:john:rwx /home/bob/collection

# Set mask (limits effective permissions for named users/groups)
sudo setfacl --modify mask:r file3

# Remove a specific user's ACL
sudo setfacl --remove user:jeremy file3

# Remove a specific group's ACL
sudo setfacl --remove group:sudo file3

# Remove ALL ACL entries from a file
sudo setfacl --remove-all file3

# Set permissions to none (revoke without removing entry)
sudo setfacl --modify user:john:--- file3
```

---

### `getfacl`
**What it does:** Displays the full Access Control List for a file or directory, showing all user, group, and mask entries.

**Syntax:**
```bash
getfacl FILE_OR_DIR
```

**Examples:**
```bash
# View ACL for a file
getfacl file3

# View ACL for a file with absolute path
getfacl /home/bob/archive

# View ACL for a directory
getfacl /home/bob/collection
```

**Sample output explained:**
```
# file: file3
# owner: alex
# group: staff
user::rw-          ← owner permissions
user:jeremy:rw-    ← named user ACL (#effective:r-- if mask limits it)
group::rw-         ← group permissions
group:sudo:rw-     ← named group ACL
mask::rw-          ← max effective permissions for named users/groups
other::r--         ← world permissions
```

**Quick tip:** A `+` sign at the end of `ls -l` output (`-rw-rw-r--+`) indicates a file has ACL entries. Run `getfacl` on it to see them.

---

### `chattr`
**What it does:** Changes special file attributes that go beyond standard permissions — like making files immutable or append-only, even for root.

**Syntax:**
```bash
chattr [+|-] ATTRIBUTE FILE
```

**Common attributes:**
| Attribute | Flag | Description |
|-----------|------|-------------|
| Append-only | `+a` | File can only be appended to, not overwritten or deleted |
| Immutable | `+i` | File cannot be renamed, deleted, or edited — even by root |

**Examples:**
```bash
# Make a file append-only (e.g., for log files)
sudo chattr +a /var/log/myapp.log

# Remove append-only attribute
sudo chattr -a /var/log/myapp.log

# Make a file completely immutable (cannot be changed or deleted)
sudo chattr +i /etc/resolv.conf

# Remove immutable attribute
sudo chattr -i /etc/resolv.conf

# See all available options
man chattr
```

---

### `lsattr`
**What it does:** Lists the special attributes set by `chattr` on files and directories.

**Syntax:**
```bash
lsattr [FILE_OR_DIR]
```

**Examples:**
```bash
# Check attributes on a specific file
lsattr myfile

# List attributes for all files in current directory
lsattr

# Sample output:
# ----i--------e------- /etc/resolv.conf   ← 'i' = immutable
# ----a--------e------- /var/log/myapp.log ← 'a' = append-only
```

---

## 🧪 Lab: ACL & File Attributes

**Objective:** Practice fine-grained permissions and file protection.

**Setup:** File `/home/bob/specialfile` owned by root.

**Tasks:**
1. View the current ACL on `specialfile` → `getfacl /home/bob/specialfile`
2. Grant user `john` read+write ACL on `specialfile` → verify with `getfacl`
3. Add group `mail` with read+execute on `specialfile`
4. Remove john's ACL entirely from `specialfile`
5. (Challenge) Apply `rwx` ACL for user `john` recursively on `/home/bob/collection`

**Solution:**
```bash
# Task 1
getfacl /home/bob/specialfile

# Task 2
sudo setfacl --modify user:john:rw /home/bob/specialfile
getfacl /home/bob/specialfile

# Task 3
sudo setfacl --modify group:mail:rx /home/bob/specialfile
getfacl /home/bob/specialfile

# Task 4
sudo setfacl --remove user:john /home/bob/specialfile
getfacl /home/bob/specialfile

# Task 5 (Challenge)
sudo setfacl --recursive -m user:john:rwx /home/bob/collection
getfacl /home/bob/collection
```

---

## Appendix: Common Patterns & One-Liners

```bash
# Find UUID of a device and add to fstab
sudo blkid /dev/vde
sudo vim /etc/fstab
# → UUID=<uuid>  /mountpoint  ext4  defaults  0  1

# Identify which disk holds root (/)
lsblk | grep "part /$"

# Find and save swapfile path
sudo swapon --show | tail -1 | awk '{print $1}'
sudo sh -c "swapon --show | tail -1 | awk '{print \$1}' > /root/swap"

# Get mount options for a specific mount point and save
sudo sh -c "findmnt / | awk '{print \$4}' > /root/moptions"

# Extract PV size for scripting
sudo pvs --noheadings -o pv_size /dev/vde

# Watch I/O in real time (find the busy process)
iostat 1
pidstat -d 1

# Check if a file has ACL entries (look for '+')
ls -l /path/to/file

# Combine pipe + head + awk for scripted extraction
lsblk | grep "part /$" | head -1 | awk '{print $1}'

# Full LVM setup in 4 commands
sudo pvcreate /dev/vdd
sudo vgcreate myvg /dev/vdd
sudo lvcreate --size 1G --name mylv myvg
sudo mkfs.ext4 /dev/myvg/mylv && sudo mkdir /data && sudo mount /dev/myvg/mylv /data

# Full swap partition setup in 4 commands
sudo mkswap /dev/vdb3
sudo swapon --verbose /dev/vdb3
swapon --show
echo 'UUID=$(sudo blkid -s UUID -o value /dev/vdb3)  none  swap  sw  0  0' | sudo tee -a /etc/fstab
```

---

> **Disclaimer:** This cheat sheet is generated from course notes and real lab sessions. Always consult `man <command>` for the authoritative reference.  
> For quick help: `man lsblk`, `man fdisk`, `man mount`, `man fstab`, `man setfacl`, `man lvcreate`