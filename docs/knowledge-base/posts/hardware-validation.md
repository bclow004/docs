---
title: Hardware Validation for VergeOS Deployment
slug: hardware-validation
description: A guide for validating server hardware compatibility and readiness before deploying VergeOS.
author: VergeOS Documentation Team
draft: false
date: 2026-02-24T00:00:00.000Z
tags:
  - installation
  - hardware
  - validation
  - pre-installation
  - checklist
categories:
  - Installation
editor: markdown
dateCreated: 2026-02-24T00:00:00.000Z
---

# Hardware Validation for VergeOS Deployment

## Overview

Before deploying VergeOS, it is critical to validate that your server hardware meets compatibility and performance requirements. This guide walks through the key hardware areas to inspect and validate, helping you identify potential issues before installation begins.

!!! note "Reference Requirements"
    For a full list of minimum and recommended specifications, see the [Node Sizing](../../implementation-guide/sizing.md) page in the Implementation Guide.

---

## CPU Validation

VergeOS requires a 64-bit x86 processor with hardware virtualization support.

### Steps

1. **Confirm processor architecture**: The CPU must be Intel or AMD x86\_64.
2. **Enable hardware virtualization**: Verify that VT-x (Intel) or AMD-V (AMD) is enabled in the BIOS/UEFI.
3. **Enable all cores**: Ensure all CPU cores are enabled in the BIOS.
4. **Enable hyperthreading**: Confirm hyperthreading (Intel HT) or SMT (AMD) is enabled in the BIOS.
5. **Check clock speed**: For controller and storage nodes, a clock speed of 3.0 GHz or higher is recommended.

### Validation Checklist

- [ ] CPU is Intel or AMD x86\_64
- [ ] Hardware virtualization (VT-x / AMD-V) enabled in BIOS
- [ ] All CPU cores enabled
- [ ] Hyperthreading / SMT enabled
- [ ] Clock speed meets or exceeds 3.0 GHz (recommended)

---

## Memory Validation

### Minimum RAM Requirements by Node Type

| Node Type              | Minimum RAM                                              |
|------------------------|----------------------------------------------------------|
| All nodes              | 16 GB dedicated to VergeOS                               |
| Controller nodes       | 1 GB per 1 TB of storage managed                        |
| Storage / HCI nodes    | 1 GB per 1 TB of raw storage (1.5 GB recommended)       |
| Compute-only nodes     | Sized for workload requirements                          |

### Steps

1. Confirm total installed RAM meets the minimum (16 GB) plus your workload overhead.
2. Verify all memory slots are populated correctly and recognized by the BIOS.
3. Check for ECC memory support, which is strongly recommended for production deployments.

### Validation Checklist

- [ ] Total RAM ≥ 16 GB (plus workload overhead)
- [ ] RAM correctly seated and recognized by BIOS
- [ ] ECC memory in use (recommended for production)
- [ ] RAM calculated at 1 GB per 1 TB raw storage (storage/controller nodes)

---

## Storage Validation

VergeOS uses a vSAN architecture and requires drives presented as individual devices (JBOD). RAID configurations are **not supported**.

### Steps

1. **Configure JBOD / IT Mode**: Ensure the storage controller is set to JBOD or IT mode. Hardware RAID must be disabled.
2. **NVMe for Tier 0 (metadata)**: Controller nodes require at least one NVMe SSD for vSAN metadata, with a minimum of 3 Drive Writes Per Day (DWPD). Two NVMe drives are recommended.
3. **Tier 0 capacity sizing**: Allocate at least 5 GB (10 GB recommended) of Tier 0 NVMe space per 1 TB of usable vSAN capacity.
4. **HBA controller**: A dedicated HBA (Host Bus Adapter) in IT mode is preferred over RAID controllers in JBOD mode.
5. **Drive size limit**: Individual drives larger than 8 TB are not recommended in non-archive environments due to extended rebuild times. The maximum supported drive size is 64 TB.
6. **SSD for workload storage**: Each storage node must have at least one NVMe or SATA/SAS SSD for guest workload storage. Two or more SSDs per node are recommended.
7. **CPU-to-disk ratio**: Plan for at least 1 CPU core per storage disk (recommended).

### Validation Checklist

- [ ] Storage controller set to JBOD or IT mode (no RAID)
- [ ] HBA controller confirmed (preferred over RAID controller)
- [ ] At least 1 NVMe SSD for Tier 0 metadata per controller node (2 recommended)
- [ ] NVMe drives rated ≥ 3 DWPD
- [ ] Tier 0 capacity: ≥ 5 GB per 1 TB usable (10 GB recommended)
- [ ] At least 1 NVMe or SATA/SAS SSD per storage node for workload data
- [ ] Drives ≤ 8 TB for non-archive workloads (recommended)
- [ ] Minimum of 2 nodes with identical disk configuration (for vSAN redundancy)

---

## Network Interface Validation

Network connectivity is critical to VergeOS operation. Each node requires at minimum two NICs: one for the Core Fabric network and one for the External network.

### Supported NIC Vendors

VergeOS supports NICs from the following vendors:

- Intel
- Mellanox (NVIDIA)
- Broadcom

### Minimum vs. Recommended NIC Configuration

| Configuration  | External NIC         | Core Fabric NIC             |
|----------------|----------------------|-----------------------------|
| Minimum        | 1 × 1 GbE            | 1 × 10 GbE                  |
| Recommended    | 2 × 25/40/100 GbE    | 2 × 10/25/40/100 GbE        |

### Steps

1. Confirm at least two physical NICs are present and functional.
2. Identify which NIC(s) will be used for the Core Fabric network and which for External.
3. Verify NIC vendor is Intel, Mellanox, or Broadcom.
4. If using fiber/SFP modules, confirm they are supported by the NIC vendor.
5. Confirm Core Fabric switches support jumbo frames (MTU ≥ 9216).

### Validation Checklist

- [ ] Minimum 2 NICs per node
- [ ] NIC vendor is Intel, Mellanox, or Broadcom
- [ ] 1 NIC designated for Core Fabric (10 GbE minimum)
- [ ] 1 NIC designated for External network (1 GbE minimum)
- [ ] SFP modules are vendor-supported (if applicable)
- [ ] Core Fabric switches support jumbo frames (MTU ≥ 9216)

---

## IPMI / Out-of-Band Management Validation

VergeOS requires out-of-band management (IPMI, iDRAC, iLO, or equivalent) for node management and recovery operations.

### Steps

1. Confirm the server has a dedicated IPMI/BMC port.
2. Assign a static IP address to the IPMI interface.
3. Test remote console access via the IPMI interface.
4. Update IPMI firmware to the latest available version.

### Validation Checklist

- [ ] IPMI / iDRAC / iLO port present and patched
- [ ] IPMI IP address assigned and accessible
- [ ] Remote console access tested and functional
- [ ] Latest IPMI firmware installed (recommended)

---

## BIOS / UEFI Settings Validation

### Steps

1. Set the BIOS boot mode to the appropriate setting:
   - **UEFI** is required if all storage drives are NVMe.
   - Legacy or Dual mode may be used for SATA/SAS environments.
2. Enable hardware-assisted virtualization (VT-x or AMD-V).
3. Enable hyperthreading / SMT.
4. Enable all CPU cores.
5. Set system clocks to the correct time and timezone.
6. Disable any C-state or power management features that may throttle CPU performance.

### Validation Checklist

- [ ] BIOS boot mode set correctly (UEFI required for all-NVMe configurations)
- [ ] Hardware virtualization enabled
- [ ] Hyperthreading / SMT enabled
- [ ] All CPU cores enabled
- [ ] System clock set to correct time
- [ ] Power management / C-states reviewed (disable aggressive throttling)

---

## Power Supply Validation

### Validation Checklist

- [ ] Redundant power supplies (PSUs) installed and connected to separate power circuits (recommended)
- [ ] PSU capacity sufficient for all installed components under load

---

## Hardware Burn-In

Before deploying VergeOS in a production environment, it is strongly recommended to run hardware burn-in testing to validate stability under load.

Burn-in testing should stress:

- CPU (e.g., Prime95, stress-ng)
- Memory (e.g., Memtest86+)
- Storage (e.g., fio, badblocks)
- Network interfaces

### Validation Checklist

- [ ] CPU stress test completed without errors
- [ ] Memory test completed without errors
- [ ] Storage burn-in completed (no drive failures or reallocated sectors)
- [ ] Network interface throughput validated

---

## Summary Validation Table

Use the table below as a final sign-off checklist before beginning installation:

| Category              | Status |
|-----------------------|--------|
| CPU requirements met  | ☐      |
| RAM requirements met  | ☐      |
| Storage configured (JBOD, NVMe Tier 0) | ☐ |
| NICs validated        | ☐      |
| IPMI configured       | ☐      |
| BIOS settings correct | ☐      |
| Power supplies redundant | ☐   |
| Burn-in testing complete | ☐   |

---

## Additional Resources

- [Node Sizing Requirements](../../implementation-guide/sizing.md)
- [Pre-Installation Checklist](../../implementation-guide/pre-installation.md)
- [Network Design Guide](../../implementation-guide/network-design.md)
- [Installation Guide](../../implementation-guide/installation-guide.md)

---

!!! question "Need Help?"
    If you need assistance validating your hardware or have questions about compatibility, contact the [VergeOS Support Team](../../support.md).

!!! note "Document Information"
    - Last Updated: 2026-02-24
    - VergeOS Version: 4.13
