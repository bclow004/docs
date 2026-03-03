---
title: Azure VM Migration - VM Drops to GRUB Prompt and Will Not Boot
slug: azure-vm-migration-grub-boot-failure
description: Troubleshooting and resolving GRUB boot failures when importing or migrating virtual machines from Microsoft Azure into VergeOS.
author: Support
draft: false
date: 2026-03-03T00:00:00.000Z
tags:
  - azure
  - import
  - migrate
  - migration
  - grub
  - boot
  - not booting
  - not bootable
  - vm wont boot
  - wont start
  - troubleshooting
  - linux
  - hyper-v
  - initramfs
  - dracut
  - virtio
categories:
  - Migration
  - Troubleshooting
editor: markdown
dateCreated: 2026-03-03T00:00:00.000Z
---

# Azure VM Migration - VM Drops to GRUB Prompt and Will Not Boot

## Overview

!!! info "Key Points"
    - Azure VMs run on Microsoft Hyper-V and carry Hyper-V-specific drivers in their `initramfs`.
    - VergeOS uses KVM/QEMU virtualization; the VirtIO drivers required are absent in Azure-built images.
    - A disk interface mismatch between Azure (SCSI/`/dev/sda`) and VergeOS (VirtIO/`/dev/vda`) causes GRUB to fail locating the root filesystem.
    - The firmware type (BIOS/UEFI) of the VergeOS VM must match the Azure VM generation (Gen1 = BIOS, Gen2 = UEFI).

When a virtual machine is migrated or imported from Microsoft Azure into VergeOS, it may fail to start and instead drop to an interactive **GRUB command prompt** (`grub>`), as shown below:

```
GNU GRUB  version 2.06

Minimal BASH-like line editing is supported. For the first word, TAB
lists possible command completions. Anywhere else TAB lists possible
device or file completions.

grub>
```

!!! note "GRUB prompt vs. GRUB rescue prompt"
    The `grub>` prompt means GRUB loaded its modules successfully but cannot find or parse `grub.cfg` or the kernel. The `grub rescue>` prompt is a more severe failure where GRUB cannot find its own modules. This article covers the `grub>` case common to Azure migrations.

## Root Causes

Azure virtual machines have several platform-specific characteristics that conflict with VergeOS defaults after migration:

| Cause | Azure Configuration | VergeOS Default | Result |
|-------|---------------------|-----------------|--------|
| Disk driver | Hyper-V SCSI (`hv_storvsc`) | VirtIO (`virtio_blk` / `virtio-scsi`) | GRUB/kernel cannot find root disk |
| Disk device name | `/dev/sda` | `/dev/vda` | Device paths in `grub.cfg` / `fstab` resolve to nothing |
| initramfs drivers | Hyper-V modules only | VirtIO modules needed | Kernel panics or hangs after GRUB |
| VM firmware | Gen1 = BIOS/MBR, Gen2 = UEFI | Must be set manually | GRUB stage 1/2 mismatch |
| Network driver | Hyper-V NIC (`hv_netvsc`) | VirtIO NIC (`virtio_net`) | No network after boot (secondary issue) |

## Prerequisites

- Access to the VergeOS UI with permission to edit VM settings.
- Console access to the VM (via VergeOS UI VM console).
- A Linux rescue ISO or the ability to add a temporary drive to the VM (for recovery method 2).

## Resolution

There are two paths to resolution depending on whether the VM reaches the `grub>` prompt or fails earlier.

---

### Step 1 — Verify VM Firmware Type in VergeOS

Azure **Generation 1** VMs use BIOS/MBR boot. Azure **Generation 2** VMs use UEFI.

1. In the VergeOS UI, navigate to the VM's settings.
2. Check the **Machine Type** / **BIOS** setting:
    - If the Azure VM was **Generation 1** → ensure VergeOS VM uses **BIOS** (SeaBIOS), not UEFI.
    - If the Azure VM was **Generation 2** → enable **UEFI Boot** in VergeOS VM settings.
3. If you changed the firmware type, power off and power on the VM again before proceeding.

!!! tip "How to identify Azure VM generation"
    In the Azure Portal, go to the VM → **Properties** → **VM generation**. Alternatively, Gen2 VMs use a GPT disk with an EFI System Partition; Gen1 VMs use MBR.

---

### Step 2 — Set Disk Interface to IDE or SATA (Immediate Compatibility Fix)

Changing the disk interface to **IDE** or **SATA** avoids the VirtIO driver requirement entirely and allows the Azure-built OS to boot without driver changes. This is the quickest path to first boot.

1. Power off the VM in VergeOS.
2. Navigate to the VM's **Drives** settings.
3. For each drive, change the **Interface** from `virtio` or `virtio-scsi` to **`ide`**.
4. Ensure the boot drive is assigned **ID 0** in the boot order.
5. Power on the VM.

!!! warning
    IDE mode has lower performance than VirtIO. After completing the driver fix (Step 4), switch the interface back to `virtio-scsi` for production use.

---

### Step 3 — Emergency Boot from the GRUB Prompt (if VM Still Drops to `grub>`)

If the VM still reaches the GRUB prompt after the firmware and interface adjustments, use the following commands in the `grub>` console to manually identify and boot the OS.

**3a. List available devices and partitions:**
```
grub> ls
```
Example output: `(hd0) (hd0,msdos1) (hd0,msdos2)`

**3b. Locate the boot partition (look for `/boot` or kernel files):**
```
grub> ls (hd0,msdos1)/
grub> ls (hd0,msdos1)/boot/
```

**3c. Find the GRUB configuration file:**
```
grub> ls (hd0,msdos1)/boot/grub/
grub> ls (hd0,msdos1)/boot/grub2/
```

**3d. Load the configuration and attempt automatic boot:**
```
grub> set root=(hd0,msdos1)
grub> configfile /boot/grub/grub.cfg
```

**3e. If `configfile` fails, boot the kernel manually:**

First, identify the exact kernel and initrd filenames:
```
grub> ls (hd0,msdos1)/boot/
```

Then boot using those filenames (substitute your actual version):
```
grub> set root=(hd0,msdos1)
grub> linux /boot/vmlinuz-6.1.0-27-amd64 root=/dev/sda1 ro
grub> initrd /boot/initrd.img-6.1.0-27-amd64
grub> boot
```

!!! tip "UEFI / GPT disks"
    For UEFI VMs with GPT partition tables, partitions appear as `(hd0,gpt1)`, `(hd0,gpt2)`, etc. The EFI System Partition is typically `gpt1`; the root/boot partition is usually `gpt2` or `gpt3`.

---

### Step 4 — Rebuild initramfs with VirtIO Drivers (Permanent Fix)

Once the VM has booted (via IDE interface or manual GRUB boot), rebuild the `initramfs` to include VirtIO drivers so the VM boots normally with `virtio-scsi` drives.

#### Debian / Ubuntu

```bash
# Add VirtIO modules to initramfs and rebuild
sudo update-initramfs -u -k all
```

To verify VirtIO modules are included:
```bash
lsinitramfs /boot/initrd.img-$(uname -r) | grep virtio
```

#### Red Hat / CentOS / AlmaLinux / Rocky Linux

```bash
# Rebuild initramfs with VirtIO block and network drivers
sudo dracut -f --add-drivers "virtio_blk virtio_net virtio_scsi" --regenerate-all
```

#### SUSE / openSUSE

```bash
sudo mkinitrd
```

---

### Step 5 — Update GRUB Configuration

After rebuilding initramfs, regenerate the GRUB configuration to ensure device references are correct:

#### Debian / Ubuntu
```bash
sudo update-grub
```

#### Red Hat / CentOS / AlmaLinux / Rocky Linux
```bash
# For BIOS systems:
sudo grub2-mkconfig -o /boot/grub2/grub.cfg

# For UEFI systems:
sudo grub2-mkconfig -o /boot/efi/EFI/redhat/grub.cfg
```

If GRUB itself is corrupted or was not installed to the correct device, reinstall it:
```bash
# BIOS/MBR
sudo grub-install /dev/sda

# UEFI (Debian/Ubuntu)
sudo grub-install --target=x86_64-efi --efi-directory=/boot/efi
```

---

### Step 6 — Verify `/etc/fstab` Device References

Azure disk names (`/dev/sda`, `/dev/sdb`) may differ from VergeOS disk names (`/dev/vda`, `/dev/sda` with IDE). The safest configuration uses UUID-based references.

```bash
# Check current fstab
cat /etc/fstab

# Find UUIDs of your partitions
blkid
```

Replace any `/dev/sdX` device paths in `/etc/fstab` with their corresponding `UUID=` values to make the configuration hypervisor-agnostic.

---

### Step 7 — Switch Disk Interface Back to VirtIO-SCSI

Once the VM boots successfully with IDE and the initramfs has been rebuilt with VirtIO drivers:

1. Power off the VM.
2. In the VergeOS UI, change all drive interfaces back to **`virtio-scsi`**.
3. Set the NICs to **`virtio`** for best network performance.
4. Power on the VM and confirm it boots correctly.

---

## Troubleshooting

!!! warning "Common Issues"

    - **Problem**: `grub>` prompt appears even after changing interface to IDE.
      - **Solution**: Verify the firmware type (BIOS vs UEFI) matches the original Azure VM generation. Check that the OS drive is boot ID 0.

    - **Problem**: Kernel boots but panics with "unable to mount root fs".
      - **Solution**: The VirtIO/SCSI drivers are still missing from initramfs. Complete Step 4 by booting via IDE first, then rebuilding initramfs.

    - **Problem**: VM boots but has no network connectivity.
      - **Solution**: The Hyper-V NIC driver (`hv_netvsc`) is loaded but no `virtio_net` driver is present. Ensure the NIC in VergeOS is set to `virtio` and the initramfs rebuild (Step 4) included `virtio_net`.

    - **Problem**: Azure WALinuxAgent (waagent) causes errors at boot.
      - **Solution**: The Azure Linux Agent is not needed outside Azure. It can be disabled: `sudo systemctl disable waagent` or removed entirely after migration.

    - **Problem**: `grub rescue>` prompt appears instead of `grub>`.
      - **Solution**: This indicates GRUB cannot find its own modules. Boot from a Linux Live ISO/rescue media, chroot into the OS, and run `grub-install` and `update-grub`.

## Additional Resources

- [Importing a Physical/Virtual Machine into VergeOS](/knowledge-base/importing-a-physicalvirtual-machine-into-vergeio/)
- [How to Import a RedHat / RHEL / CentOS based VM](/knowledge-base/import-rhel-centos-vm/)
- [UEFI Tweaks for Imported VMs](/knowledge-base/uefi-tweaks-for-imported-vms/)
- [Importing VMs from Media](/product-guide/virtual-machines/import-from-upload/)

## Feedback

!!! question "Need Help?"
    If you need further assistance or have any questions about this article, please don't hesitate to reach out to our support team.

---

!!! note "Document Information"
    - Last Updated: 2026-03-03
    - vergeOS Version: 4.12.6
