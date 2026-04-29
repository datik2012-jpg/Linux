Here is a structured summary of the Linux storage concepts, commands, and lab
exercises you provided.

🗄️ Linux Storage Management Summary

This guide covers how to view disks, create partitions, format file systems,
manage swap space, and ensure your drives mount automatically when the system
boots.

1. Viewing Disks and Partitioning

To run different operating systems (like Windows using NTFS and Linux using
EXT4) on the same disk, you must divide the physical disk into separate
partitions.

Key Commands to View Disks:

  - lsblk – Lists all block devices (disks and partitions) and their mount
    points. (e.g., sda is the disk, sda1 is a partition).
  - sudo fdisk --list /dev/sda – Shows a detailed list of partitions on a
    specific disk.

Partitioning with cfdisk:

  - sudo cfdisk /dev/sdb – Opens a text-based, arrow-key-driven interface to
    manage partitions.
  - Partition Tables: If a disk is brand new, cfdisk will ask you to create a
    partition table.
      - gpt (Modern, recommended)
      - dos / MBR (Older, legacy systems)
  - ⚠️ Warning: Viewing layouts is safe, but creating, deleting, or writing
    changes can destroy existing data if done on the wrong disk.

2. Swap Space Configuration

Swap is a dedicated area on a drive acting as "virtual memory." Linux moves
temporary data here from RAM when physical memory is full. You can create Swap
using either a partition or a file.

Using a Partition for Swap:

1.  Format it: sudo mkswap /dev/vdb3
2.  Enable it: sudo swapon /dev/vdb3 (Add --verbose for detailed output).
3.  Disable it: sudo swapoff /dev/vdb3
4.  Check Swap Status: swapon --show

Creating a Swap File (Manual Method): If you don't have a spare partition, you
can create a file filled with zeros to act as swap.

1.  Create file: sudo dd if=/dev/zero of=/swap bs=1M count=128 status=progress
    (Creates a 128MB file).
2.  Secure it: sudo chmod 600 /swap (Only root can read/write).
3.  Format it: sudo mkswap /swap
4.  Enable it: sudo swapon /swap

3. Creating and Managing File Systems (Formatting)

Once a partition is created, it must be formatted with a file system.

  - EXT4: Default for Ubuntu/Debian.
  - XFS: Default for RedHat/CentOS.

Formatting Commands:

  - sudo mkfs.ext4 /dev/sdb2 (or mke2fs) – Formats as EXT4.
  - sudo mkfs.xfs /dev/sdb1 – Formats as XFS.

Useful Formatting Flags:

  - -L "LabelName" – Assigns a human-readable name to the volume.
  - -i size=512 (XFS) or -N 500000 (EXT4) – Modifies inodes. (Inodes are data
    structures that store file metadata. Modifying them is useful for systems
    with heavy metadata usage).
  - -f – Forces formatting (bypasses safety checks, erases existing data).

Management Utilities:

  - EXT4 (tune2fs): sudo tune2fs -l /dev/sdb2 (view details/health), sudo
    tune2fs -L "NewLabel" /dev/sdb2 (change label).
  - XFS (xfs_admin): sudo xfs_admin -l /dev/sdb1 (view label), sudo xfs_admin -L
    "NewLabel" /dev/sdb1 (change label). XFS also has many other tools like
    xfs_repair, xfs_growfs, and xfs_info.

4. Mounting & Persistence (/etc/fstab)

To use a formatted partition, it must be attached (mounted) to a directory in
the Linux file tree.

Temporary Mounting (Lost on Reboot):

  - Mount: sudo mount /dev/vdb1 /mnt/
  - Unmount: sudo umount /mnt/ (Note: the correct command is umount, without the
    first 'n').

Permanent Mounting (Survives Reboot): To make mounts permanent, you must add
them to the File System Table configuration file:

  - sudo vim /etc/fstab

fstab Syntax Example:

# Device/UUID       Mount Point   File System   Options    Dump  Pass
/dev/vdb1           /mybackups    xfs           defaults   0     2
/dev/vdb3           none          swap          defaults   0     0

Best Practice: Using UUIDs instead of Device Names Device names (/dev/vda1) can
change if cables are moved or hardware is added. UUIDs (Universally Unique
Identifiers) never change.

  - Find UUIDs: Look inside /dev/disk/by-uuid/ or run lsblk with advanced flags.
  - fstab Example with UUID: UUID=a51d7731-b033-4c07-b171-628ae951ea01 /boot
    ext4 defaults 0 1

Applying fstab Changes Safely:

1.  Reload daemons: sudo systemctl daemon-reload
2.  Test the configuration: sudo mount -a (This verifies there are no syntax
    errors in fstab before you reboot and accidentally break your system).
