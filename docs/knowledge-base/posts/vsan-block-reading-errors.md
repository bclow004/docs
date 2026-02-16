---
title: Troubleshooting vSAN Block Reading Errors
slug: troubleshooting-vsan-block-reading-errors
description: How to diagnose and resolve repeating ybsan FillPage and block reading errors in the VergeOS dashboard logs.
draft: false
date: 2026-02-16T00:00:00.000Z
tags:
  - vSAN
  - troubleshooting
  - block errors
  - ybsan
  - diagnostics
categories:
  - Troubleshooting
  - vSAN
editor: markdown
dateCreated: 2026-02-16T00:00:00.000Z
---

# Troubleshooting vSAN Block Reading Errors

## Overview

This article covers how to diagnose and resolve repeating `ybsan` errors that appear in the VergeOS dashboard logs, specifically:

- `ybsan: FillPage error for block <block_number> ino <inode> tier <tier> hashrc 0: (2) No such file or directory`
- `ybsan: Error reading block <hash> from all sources tier <tier> in <time>ms: (2) No such file or directory`

These errors indicate the vSAN is unable to locate one or more data blocks on any available source (primary or redundant copy) within the specified storage tier. The errors will repeat continuously as the system retries the read operations.

## Common Causes

1. **Drive failure or degradation** - A physical drive has failed or developed read errors, causing blocks stored on it to become inaccessible.
2. **Multiple simultaneous drive failures** - Failures across more than one node on the same tier have exceeded the vSAN's redundancy tolerance.
3. **Incomplete repair operation** - A previous drive replacement or repair did not finish, leaving some blocks without a valid redundant copy.
4. **Block corruption** - Data blocks have become corrupted and fail hash validation.

## Diagnostic Steps

### Step 1: Check Drive Health

1. From the **Main Dashboard**, select **System** from the left menu.
2. Click the **Drives** count box to view the full list of drives.
3. Look for any drives with a **Warning** or **Error** status.
4. Double-click any flagged drive to view its dashboard for more detail (SMART data, error counts, etc.).

If a drive shows errors, follow the [Drive Replacement](/product-guide/system/drive-replacement) procedure.

### Step 2: Check Node Status

1. From the **Main Dashboard**, select **System** from the left menu, then select **Nodes**.
2. Confirm all nodes are **Running**. If any node is offline, the blocks assigned to that node will be unavailable.
3. Review the node identified in the error messages (e.g., if errors reference a specific node, check that node first).

### Step 3: Check vSAN Tier Status

1. From the **Main Dashboard**, click the **vSAN Tiers** count box.
2. Select **vSAN Diagnostics** from the left menu.
3. Run the **Get Tier Status** command to check redundancy status and health metrics for the affected tier.

!!! warning "Non-Redundant State"
    If a tier reports a non-redundant state, data protection is degraded. Prioritize resolving this condition before performing any other maintenance.

### Step 4: Check Repair Status

1. In **vSAN Diagnostics**, run the **Get Repair Status** command.
2. If a repair is in progress, allow it to complete. The block reading errors may resolve once the repair finishes restoring all blocks.

!!! warning
    **Do NOT restart, reset, or power off any nodes** while a repair operation is in progress.

### Step 5: Identify the Affected File (Optional)

The error messages contain an inode number (the `ino` value). You can use this to identify what file or VM disk is affected:

1. In **vSAN Diagnostics**, run the **Find Inode** command with the inode number from the error message.
2. Alternatively, run the **Get Path from Inode** command to resolve the inode to a filesystem path.

This helps determine whether the affected blocks belong to a specific VM, volume, or system file.

### Step 6: Run an Integrity Check

If no drive failures are found but errors persist, run an integrity check on the affected tier:

1. In **vSAN Diagnostics**, select **Integrity Check**.
2. Set the **Path** to the affected volume or use `/` for a full check.
3. Enable the **Recursive** option.

!!! danger "Fix Mode"
    The **Fix** option zeros out bad blocks and is **destructive**. Only use Fix mode under direct guidance from VergeOS support.

## Resolution

### Drive Failure Detected

If a failed drive is identified, follow the [Drive Replacement](/product-guide/system/drive-replacement) guide to replace and repair. The block reading errors should stop once the repair process restores all blocks from redundant copies.

### Multiple Drive Failures (Beyond Redundancy)

If failures exceed redundancy tolerance (e.g., simultaneous failures on the same tier across multiple nodes):

1. Check if a **Repair Server (ioGuardian)** is configured under **Backup/DR > Repair Servers**.
2. If configured, the system will automatically attempt to pull missing blocks from the remote system.
3. If no repair server exists, a rollback to a previous snapshot may be necessary. Contact [VergeOS Support](/support) for guidance.

See [Repair Server](/product-guide/backup-dr/repair-server) for details on configuring a repair server.

### No Hardware Failure Found

If all drives and nodes appear healthy but errors continue:

1. Run a **Get Tier Status** diagnostic to verify tier health.
2. Run an **Integrity Check** (without Fix mode) to identify the scope of corruption.
3. Contact [VergeOS Support](/support) with the diagnostic output for further analysis.

## Prevention

- **Configure a Repair Server** from an outgoing sync destination to enable automatic block recovery. See [Repair Server](/product-guide/backup-dr/repair-server).
- **Set up subscriptions** with *target type=system Dashboard* to receive timely alerts about drive issues. See [Creating Subscriptions](/product-guide/system/subscriptions-overview).
- **Monitor drive health** regularly through the System Dashboard, paying attention to SMART warnings for wear level, reallocated sectors, and pending sectors.
- **Avoid powering off or restarting nodes** during active repair operations.

## Related Articles

- [Drive Replacement](/product-guide/system/drive-replacement)
- [vSAN Diagnostics Guide](/knowledge-base/vsan-diagnostics-guide)
- [Repair Server (ioGuardian)](/product-guide/backup-dr/repair-server)
- [Identifying a Failed Disk Drive](/knowledge-base/identifying-a-failed-disk-drive)
- [Storage Tiers in VergeOS vSAN](/product-guide/vsan/storage-tiers)

---

!!! note "Document Information"
    - Last Updated: 2026-02-16
    - VergeOS Version: 4.13
