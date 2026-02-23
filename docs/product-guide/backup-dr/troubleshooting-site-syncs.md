# Troubleshooting Site Sync Failures

## Overview

Site syncs can fail or behave unexpectedly due to network, configuration, or scheduling issues. This guide covers the most common causes of sync failures and how to resolve them.

## Reading the Sync Logs

Sync logs appear at the bottom of both the **Incoming Sync** and **Outgoing Sync** dashboards. Each sync attempt produces at minimum two log entries: one when the sync starts and one when it completes (or fails).

A completed sync entry looks like:

```
Sync for 'Hourly for 3 hours_YYYYMMDD_HH' complete (Xs)
  checked [XB] scanned [XB] sent [XB] sent net [XB] dirs [X] files [X] updated [X]
```

Key fields to check:

| Field | What it means |
|-------|--------------|
| `checked` | Total size of files compared |
| `scanned` | Total data blocks compared |
| `sent` | Data identified as changed and queued to send |
| `sent net` | Actual bytes transferred (post-compression) |
| `updated` | Number of files updated on the destination |

- **`sent [0B]` with a large `scanned` value** is normal when no data has changed since the last sync.
- **A sync entry that starts but never completes** indicates the connection was interrupted mid-transfer.

---

## Common Causes and Resolutions

### 1. Port 14201 Not Reachable

Sync traffic uses **TCP port 14201** between the sending and receiving systems. If this port is blocked, syncs will fail to connect.

**Symptoms:** Sync shows an error or never starts; no "complete" log entry follows the "start" entry.

**Resolution:**

- Verify the **PAT rule for port 14201** exists on the receiving system's Core Network and the appropriate External Network. See [Configuring a Site Sync](/product-guide/backup-dr/sync-configuration) for the exact rule setup.
- Confirm firewall rules between the two sites allow TCP 14201.
- If the vSAN address differs from the system's public address, set the **VSAN host/ip override** on the receiving system: **System > Settings > Advanced Settings > VSAN host/ip override (used for incoming site syncs)**.

---

### 2. Invalid or Expired Registration Code

The outgoing sync must be registered with the receiving system using a valid registration key. A misconfigured or reused key causes registration to fail.

**Symptoms:** Outgoing sync shows "Registration failed" or stays in an unregistered state; the incoming sync dashboard shows the key as unused or does not show a registered date.

**Resolution:**

1. On the **receiving system**, navigate to **Backup/DR > Incoming Syncs** and open the incoming sync dashboard.
2. Verify the registration code is present. If it shows `--Used--` with a registered date, the pairing succeeded.
3. If registration is broken, delete both the incoming and outgoing syncs, recreate the incoming sync to generate a new key, then recreate the outgoing sync using the new key.

!!! warning "Deleting and recreating syncs"
    Deleting an incoming or outgoing sync while snapshots are in-flight will interrupt those syncs. Any snapshot currently being transferred must be re-queued after the new sync is configured.

---

### 3. Sync Deleted and Recreated Mid-Operation

As visible in sync logs, events such as **"Deleted Outgoing Sync"** and **"Deleted Incoming Sync"** followed immediately by recreation indicate the sync pair was torn down and re-established. Any snapshot transfers in progress at that moment will fail and must be re-queued.

**Symptoms:** Log shows a deletion event followed by re-creation; previously queued snapshots no longer appear in the queue.

**Resolution:**

- After recreating the sync, verify the **Auto Sync Configuration** on the outgoing sync is still correctly defined (snapshot periods and remote retention).
- Manually re-queue any snapshots that were in flight at the time of the deletion using **Add Cloud Snapshot to Queue** on the outgoing sync dashboard. See [Manual Site Syncs](/product-guide/backup-dr/manual-site-syncs).

---

### 4. Snapshot Expires Before It Can Be Sent

If a snapshot's local retention period is shorter than the time needed to sync it, the snapshot may expire and be deleted locally before the transfer completes. This is especially common for high-frequency, short-retention snapshot periods over slow WAN links.

**Symptoms:** A snapshot appears in the queue but disappears before completing; log may show the snapshot was removed.

**Resolution:**

- In the **Auto Sync Configuration**, enable the **Do not expire snapshot** option for affected periods. This prevents local expiration until the sync completes.
- Consider increasing the local retention for the affected snapshot period if bandwidth constraints cause syncs to take longer than the retention window.

---

### 5. Target Storage Tier Does Not Exist on the Destination

By default, sync sends data to the same storage tier on the destination as the source. If that tier does not exist on the destination vSAN, the sync may fail or redirect to an unintended tier.

**Symptoms:** Sync completes but data lands on the wrong tier; or sync errors reference tier availability.

**Resolution:**

- On the outgoing sync, use **Override Destination Storage Tier** to explicitly direct all sync data to a tier that exists on the destination.
- Alternatively, configure a **Force Tier** on the incoming sync on the receiving system to redirect all incoming data to a specific tier.

---

### 6. Incorrect Remote URL

If the **Remote URL** in the outgoing sync configuration does not match the actual address reachable from the sending system, the sync cannot connect.

**Symptoms:** Sync status shows "offline" or connection errors; no sync activity occurs.

**Resolution:**

1. On the **sending system**, open the outgoing sync dashboard and click **Edit**.
2. Verify the **Remote URL** matches the address of the receiving system as reachable from the sending side.
3. Confirm DNS resolution or IP routing is working from the sending system to that address.

---

## Verifying Sync Health

Use the following checks regularly to confirm syncs are operating correctly:

1. **Outgoing Sync Dashboard**: Confirm the **Sync Status** is not stuck in "error" state.
2. **Snapshot Queue**: Verify snapshots are moving through the queue and not accumulating.
3. **Remote Snapshots count**: Confirm the count is increasing as expected with each sync interval.
4. **Subscriptions**: Set up an **On-demand** subscription with the **Site Sync Received** profile on the receiving system to get notified each time a sync completes. See [Monitoring Site Syncs](/product-guide/backup-dr/monitoring-site-syncs).

---

## Additional Resources

- [Configuring a Site Sync](/product-guide/backup-dr/sync-configuration)
- [Monitoring Site Syncs](/product-guide/backup-dr/monitoring-site-syncs)
- [Manual Site Syncs](/product-guide/backup-dr/manual-site-syncs)
- [Repair Server](/product-guide/backup-dr/repair-server)
