---
date: 2026-06-09
title: Rescuing a Linux based Boot-From-Volume Instance in OpenStack
authors:
  - chrisbreu
description: >
  A production-ready guide to analyzing storage layout and repairing a corrupted filesystem on an OpenStack Linux based Boot-From-Volume instance.  Written with assistance by xerotier.ai
categories:
  - openstack
  - compute
  - cinder
  - recovery
---

# Rescuing a Linux based Boot-From-Volume Instance in OpenStack

When an OpenStack instance backed by a persistent Cinder volume (Boot-From-Volume, or BFV) gets corrupted, misconfigured, or suffers a broken bootloader, recovery can be challenging.  However, OpenStack control planes (introduced in Ussuri) support Stable Device Instance Rescue for volume-backed instances.


This guide provides a comprehensive guide to help analyze the storage layout and some basic commands to reapir a filesystem using nativ Linux tools.

## Part 1: Architecture & Control Plane Execution

### The Workflow Overview

The automated rescue workflow uses OpenStack's Stable Device Instance Rescue to attach a rescue image to the instance without altering the original storage layout:

- Nova initiates a soft shutdown of the corrupted VM.
- Nova boots the lightweight rescue image on a new secondary device (e.g., /dev/vdc), leaving the original boot volume (e.g., /dev/vda) untouched.
- You mount the original disk via the console, perform your repairs, and exit.

### 1: Download a Lightweight Rescue OS

Cirros is a minimal, fast-booting Linux image ideal for an emergency recovery medium because it downloads instantly and initializes in seconds.

```bash
# Download the stable Cirros image (approx. 16MB)
curl -L -o cirros-rescue.img http://download.cirros-cloud.net/0.6.2/cirros-0.6.2-x86_64-disk.img
```

### 2: Upload to Glance with Stable Disk Metadata

To force Nova out of its legacy code path (which triggers the 400 error), you must tag the image with specific hardware properties.

While some systems accept virtual optical drives (cdrom/scsi), utilizing standard virtual disk mappings (disk/virtio) ensures that guest Linux kernels recognize the block devices instantly without needing a manual SCSI bus rescan.

```bash
openstack --os-cloud dfw-prod-me image create \
  --disk-format qcow2 \
  --container-format bare \
  --file ./cirros-rescue.img \
  --property hw_rescue_device=disk \
  --property hw_rescue_bus=virtio \
  --private \
  "cirros-rescue-image"
```

### 3: Trigger the Rescue

```bash
openstack --os-cloud dfw-prod-me server rescue \
  --image "cirros-rescue-image" \
7209d6ec-9ee8-4b25-959e-875c477fc812
```

## Part 2: Dissecting the Guest Storage Layout

Once the instance shifts into a RESCUE state, open your browser console viewer to log into the recovery interface:

```bash
openstack --os-cloud dfw-prod-me console url show 7209d6ec-9ee8-4b25-959e-875c477fc812
```

### The Authentication Credentials

Log into the Cirros terminal environment using the built-in defaults:

- Username: cirros
- Password: gocubsgo

### Mapping the Block Devices

Often, the hypervisor maps the virtual disks in an unexpected order. You cannot blindly assume /dev/vda is your broken production disk. To map this out accurately, run lsblk with explicit columns tracking size and active OS mount points:
Bash

```bash
lsblk -o NAME,SIZE,TYPE,MOUNTPOINT
```

Real-World Output Scenario:

```
NAME      SIZE TYPE MOUNTPOINT
vda       100G disk                      
├─vda1     99G part                     
├─vda14     4M part                     
├─vda15  106M part                     
└─vda16  913M part                     
vdb         1G disk                     
vdc       112M disk                      
├─vdc1    103M part /                  
└─vdc15     8M part                     
```

### How to Parse This Specific Layout:

- **Locate the Rescue Root (/):** Look immediately at the MOUNTPOINT column. In the output above, `/dev/vdc1` is mounted to `/`. At a total disk size of only 112M, `/dev/vdc` is definitively our lightweight Cirros rescue OS footprint.

- **Identify the Production Storage (Target):** Look for your target volume capacity. Here, `/dev/vda` stands out at 100G. It features a standard cloud-image partition table architecture—including the small `vda14` boot flag, `vda15` EFI partition, and `/dev/vda1` (99G) as the primary root filesystem partition.

- **Disregard Metadata Side-Channels:** `/dev/vdb` at a flat 1G has no mount point or partitions. This is typically an ephemeral configuration drive (config-drive) injected by the Nova hypervisor to pass metadata to the instance. You can ignore it.

!!! tip "SSH Alternative"

    If your instance has a floating IP assigned, you can use SSH to access the rescue session as an alternative to the browser console. This can be more convenient for editing files or running commands:

    ```bash
    ssh cirros@<floating-ip>
    ```

## Part 3: Step-by-Step Linux Recovery Procedure

#### 1: Elevate Privileges and Check System Integrity

Drop into a root shell wrapper inside your rescue interface to ensure you have complete administrative clearance over the block storage devices:

```bash
sudo su -
```

#### 2: Identify the Filesystem

Before running any repair tools, identify the filesystem type on the partition to ensure you use the correct command:

```bash
blkid /dev/vda1
```

Example output for ext4:
```
/dev/vda1: UUID="..." TYPE="ext4"
```

Example output for XFS:
```
/dev/vda1: UUID="..." TYPE="xfs"
```

If `blkid` returns no type, check the partition table with `fdisk`:

```bash
fdisk -l /dev/vda
```

This will show the partition layout and help confirm which partition holds your root filesystem.

Then proceed with the appropriate repair command for your filesystem type:

- **For XFS Filesystems** (Common on RHEL, CentOS, Rocky Linux):

```bash
xfs_repair /dev/vda1
```

- **For Ext4 Filesystems** (Common on Ubuntu, Debian):

```bash
e2fsck -f /dev/vda1
```

#### 3: Mount the Production Partition

Create a staging mount directory within the rescue memory space and attach the primary production data partition (`/dev/vda1` based on our mapping analysis):

```bash
mkdir -p /mnt/recovery
mount /dev/vda1 /mnt/recovery
```

#### Fast-Path Corrections (Direct Text Edits)

If your server instance failed to boot due to a basic syntax error, an invalid network map, or a hanging secondary storage mount, you can modify those configuration target files directly:

- **Fix Broken System Mounts:** If a missing or deprecated block attachment is stalling the boot layout:

```bash
vi /mnt/recovery/etc/fstab
```

- **Fix Network Configurations or Read Log Files Directly:**

```bash
vi /mnt/recovery/etc/netplan/50-cloud-init.yaml
tail -n 100 /mnt/recovery/var/log/syslog
```

## Part 4: Advanced Recovery via Chroot Jail

If your remediation requires running tasks as if you were natively logged directly into the original operating system host—such as regenerating kernel flags, altering locked user accounts, or invoking automated system package managers to fix a broken bootloader—you must build a chroot environment jail.

#### 1: Bind Essential Kernel Subsystems

You must mirror the host kernel virtual filesystems down into the mounted production disk path so applications inside the jail can interact with the hypervisor virtual hardware:

```bash
mount --bind /dev /mnt/recovery/dev
mount --bind /proc /mnt/recovery/proc
mount --bind /sys /mnt/recovery/sys
mount --bind /run /mnt/recovery/run
```

#### 2: Enter the Chroot Environment

```bash
chroot /mnt/recovery /bin/bash
```

> **Operational Note:** Your terminal is now chrooted into the production OS. Commands you run here will use the repaired system's packages, libraries, and configuration rather than the minimal Cirros rescue environment.

#### Common Repair Tasks

Choose from the following based on the issue you are addressing:

- **Regenerate an initramfs kernel image** — Fixes kernel/module mismatches after package updates or kernel panics:

```bash
update-initramfs -u -k all
```

- **Rebuild the GRUB Bootloader Configuration** — Repairs bootloader entries when GRUB is missing or misconfigured:

```bash
mkdir -p /boot/grub
update-grub
```

- **Reset the root password** — Resolves locked or forgotten administrative access:

```bash
passwd root
```

## Part 5: Clean Tear-Down and Return to Production

When your remediation modifications are finished, you must step back out of the structural layers cleanly. This flushes any pending block cache writes to the underlying storage fabric and avoids leaving filesystems in a dirty state.

```bash
# 1. Step out of the chroot jail
exit

# 2. Convert and recursively unmount all bound kernel directories and the root disk cleanly
umount -R /mnt/recovery
```

With the filesystem safely unmounted, close your console viewer session. Run the final deployment cleanup command from your native terminal interface to tell OpenStack to restore the original storage priorities:

```bash
openstack --os-cloud dfw-prod-me server unrescue 7209d6ec-9ee8-4b25-959e-875c477fc812
```

Nova will automatically terminate the temporary rescue image loop, slide your 100G volume back into the primary boot channel mapping slot, and spin your server directly back into production.

## Part 6: Alternative Recovery Methods

The approaches described below are outside the scope of this guide's focus on Nova's Stable Device Instance Rescue.

### Alternative: Recover by Detaching and Attaching to Another Instance

If the rescue workflow is not an option, you can recover by terminating the broken VM, detaching the Cinder volume, and attaching it to a new helper VM. You can then mount and repair the volume from there using the same repair steps described in Parts 3–5 of this guide.

!!! warning "Critical: Delete Policy on the Original Volume"

    This approach **only works if the original Boot-From-Volume was created with the `delete_on_termination = false` option** (the default for Cinder volumes in many deployments). If the VM was created with `delete_on_termination = true`, **deleting the corrupted server will also delete the underlying volume and all of its data permanently**.

    !!! danger "GeneStack and Rackspace Public Cloud Note"

        On **GeneStack** and **Rackspace Public Cloud**, Boot-From-Volume instances are created with `delete_on_termination = true` by default. **This means that `delete_server` will permanently destroy the volume and all data it contains.** For these platforms, **this alternative recovery method will NOT work** and you must use the Nova rescue workflow described in Part 1 or the snapshot-based approach described below.

    Before attempting this alternative:

    ```bash
    openstack server show 7209d6ec-9ee8-4b25-959e-875c477fc812 -f values -c volumes_attached
    ```

    Verify the volume's `delete_on_termination` flag is set to `false`. If uncertain, do not delete the server — use the rescue workflow instead.

1. **Find the Cinder volume ID attached to the broken instance:**

    ```bash
    openstack server show 7209d6ec-9ee8-4b25-959e-875c477fc812 -f value -c config_drive
    openstack volume list
    ```

2. **Shut down and delete the broken server** (note this will **not** delete the volume if `delete_on_termination = false`):

    ```bash
    openstack server stop 7209d6ec-9ee8-4b25-959e-875c477fc812
    openstack server delete 7209d6ec-9ee8-4b25-959e-875c477fc812
    ```

3. **Detach the volume from the (now deleted) server and verify it is available:**

    ```bash
    openstack volume list
    ```

    Confirm the volume status is `available` (not `in-use`, `deleting`, or `error`).

4. **Create a helper VM** (any simple flavor with network access is sufficient):

    ```bash
    openstack server create --flavor <helper-flavor> --image <cirros-or-any-linux-image> --network <network-name> helper-instance
    ```

5. **Attach the recovered volume to the helper VM** as a secondary (non-root) volume:

    ```bash
    openstack server add volume helper-instance <recovered-volume-id>
    ```

6. **SSH into the helper VM and follow the repair steps** in Parts 3–5 of this guide (identify filesystem, run `xfs_repair` or `e2fsck`, mount, edit configs, optionally chroot).

### Alternative: Snapshot, Rebuild, and Re-attach

If neither the rescue workflow nor direct detach/attach is possible (e.g., the control plane does not support Stale Device Instance Rescue in your region), you can recover by creating a snapshot of the broken boot volume, spinning up a helper volume from that snapshot on a test VM, repairing it, and then booting a new instance from the fixed volume.

1. **Create a snapshot of the boot volume:**

    ```bash
    openstack volume snapshot create --volume <broken-volume-id> broken-vol-snapshot
    ```

2. **Create a new volume from the snapshot:**

    ```bash
    openstack volume create --snapshot broken-vol-snapshot --size <size-in-gb> fixed-vol
    ```

3. **Attach this new volume to a test VM** and follow the repair steps in Parts 3–5 of this guide.

4. **Boot a new instance from the fixed volume:**

    ```bash
    openstack server create --block-device source=volume,volume-id=<fixed-vol>,shutdown=preserve,destination=root,bootindex=0 --flavor <flavor> <new-server-name>
    ```

!!! note "Considerations"

    - This approach results in a **new server UUID** and new network identity. Floating IPs, DNS records, and any hardcoded server references will need to be updated.
    - Ensure the snapshot and new volume are in the **same region** as your environment.
    - This method provides a clean slate but requires additional downtime for the rebuild and re-attach process.
