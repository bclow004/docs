# VergeOS 12-Node / 2-Cluster Design: Cisco HX240c M5 All Flash

This document presents a reference architecture for deploying VergeOS across two sites (Production DC and DR) using existing Cisco HyperFlex HX240c M5 All Flash hardware. It covers node roles, storage and compute capacity, network design, and all identified capacity gaps and hardware incompatibilities.

---

## Executive Summary

| | Production (DC) Cluster | DR Cluster |
|---|---|---|
| **Nodes (current BOM)** | 3 | 3 |
| **Nodes (required for 12-node target)** | **6** | **6** |
| **Additional nodes needed** | **3** | **3** |
| **CPU per node** | 2× Intel Xeon Gold 6148 (20C) | 2× Intel Xeon Gold 6248R (24C) |
| **RAM per node** | 1 TB | 2 TB |
| **Raw storage per node** | 22.8 TB SATA SSD + 1.6 TB NVMe | 60.8 TB SATA SSD + 1.6 TB NVMe |
| **GPU** | None | 2× NVIDIA T4 16 GB |
| **Switching** | 2× UCS FI 6454 (present) | **None — missing from BOM** |

> **Critical gap:** Both sites require 3 additional nodes each to reach the 12-node target. The DR site has no switching infrastructure.

---

## Hardware Inventory

### Production (DC) — 3× Cisco HX240c M5 All Flash

| Component | Part | Qty | Per Node |
|---|---|---|---|
| Server chassis | HXAF240C-M5SX | 3 | 1 |
| CPU | HX-CPU-6148 — Xeon Gold 6148 2.4 GHz / 20C / 150 W | 6 | 2 |
| RAM | HX-ML-X64G4RS-H — 64 GB DDR4-2666 LRDIMM | 48 | 16 |
| SATA SSD (capacity) | HX-SD38T61X-EV — 3.8 TB 2.5″ SATA SSD | 18 | 6 |
| NVMe (cache / tier 1) | HX-NVMEHW-H1600 — 1.6 TB U.2 NVMe | 3 | 1 |
| OS SSD | HX-SD240GM1X-EV — 240 GB SATA SSD | 3 | 1 |
| OS M.2 | HX-M2-240GB — 240 GB M.2 SATA | 3 | 1 |
| NIC | HX-MLOM-C25Q-04 — Cisco UCS VIC 1457 4× 10/25G SFP28 | 3 | 1 |
| SAS HBA | HX-SAS-M5HD — Cisco 12G Modular SAS HBA | 3 | 1 |
| PSU | HX-PSU1-1600W — 1600 W AC PSU | 6 | 2 (redundant) |
| Fabric switch | HX-FI-6454 — UCS Fabric Interconnect 6454 | 2 | shared |

**DC cluster raw capacity:**

| Resource | Per Node | 3-Node Cluster Total |
|---|---|---|
| Physical CPU cores | 40 (2× 20C) | 120 |
| RAM | 1 TB (16× 64 GB) | 3 TB |
| SATA SSD (raw) | 22.8 TB (6× 3.8 TB) | 68.4 TB |
| NVMe (raw) | 1.6 TB | 4.8 TB |

---

### DR Site — 3× Cisco HX240c M5 All Flash

| Component | Part | Qty | Per Node |
|---|---|---|---|
| Server chassis | HXAF240C-M5SX | 3 | 1 |
| CPU | HX-CPU-I6248R — Xeon Gold 6248R 3.0 GHz / 24C / 205 W | 6 | 2 |
| RAM | HX-ML-128G4RT-H — 128 GB DDR4-2933 LRDIMM | 48 | 16 |
| SATA SSD (capacity) | HX-SD38T61X-EV — 3.8 TB 2.5″ SATA SSD | 48 | 16 |
| NVMe (cache / tier 1) | HX-NVMEHW-H1600 — 1.6 TB U.2 NVMe | 3 | 1 |
| OS SSD | HX-SD240GM1X-EV — 240 GB SATA SSD | 3 | 1 |
| OS M.2 | HX-M2-240GB — 240 GB M.2 SATA | 3 | 1 |
| NIC | HX-MLOM-C25Q-04 — Cisco UCS VIC 1457 4× 10/25G SFP28 | 3 | 1 |
| SAS HBA | HX-SAS-M5HD — Cisco 12G Modular SAS HBA | 3 | 1 |
| GPU | HX-GPU-T4-16 — NVIDIA Tesla T4 16 GB PCIe 75 W | 6 | 2 |
| GPU software | HX-NV-GRVAS-3YR — NVIDIA GRID VDI Apps 1 CCU 3-yr | 6 | — |
| PSU | HX-PSU1-1050W — 1050 W AC PSU | 6 | 2 (redundant) |
| **Fabric switch** | **— none listed —** | **0** | **MISSING** |

**DR cluster raw capacity:**

| Resource | Per Node | 3-Node Cluster Total |
|---|---|---|
| Physical CPU cores | 48 (2× 24C) | 144 |
| RAM | 2 TB (16× 128 GB) | 6 TB |
| SATA SSD (raw) | 60.8 TB (16× 3.8 TB) | 182.4 TB |
| NVMe (raw) | 1.6 TB | 4.8 TB |
| GPU VRAM | 32 GB (2× 16 GB) | 96 GB |

---

## VergeOS Solution Design

### Cluster Architecture Overview

```
┌──────────────────────────────────────────────────────────────────┐
│  PRODUCTION SITE (DC)                                            │
│  VergeOS Cluster 1 — 6 nodes (3 current + 3 to procure)         │
│                                                                  │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐                         │
│  │ Node 1   │ │ Node 2   │ │ Node 3   │  ← existing BOM         │
│  │Controller│ │Controller│ │ Compute/ │                         │
│  │(Primary) │ │(Backup)  │ │ Storage  │                         │
│  └──────────┘ └──────────┘ └──────────┘                         │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐                         │
│  │ Node 4   │ │ Node 5   │ │ Node 6   │  ← to procure           │
│  │ Compute/ │ │ Compute/ │ │ Compute/ │                         │
│  │ Storage  │ │ Storage  │ │ Storage  │                         │
│  └──────────┘ └──────────┘ └──────────┘                         │
│  Switching: 2× UCS FI 6454 (reconfigure to standard Ethernet)   │
│                     │  Site Sync (VergeOS)  │                   │
└─────────────────────┼───────────────────────┘                   │
                      │                                            │
┌─────────────────────┼───────────────────────┐                   │
│  DR SITE                                    │                   │
│  VergeOS Cluster 2 — 6 nodes (3 current + 3 to procure)         │
│                                             │                   │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐    │                   │
│  │ Node 1   │ │ Node 2   │ │ Node 3   │  ← existing BOM         │
│  │Controller│ │Controller│ │ Compute/ │                         │
│  │(Primary) │ │(Backup)  │ │ Storage +│                         │
│  │          │ │          │ │ GPU (T4) │                         │
│  └──────────┘ └──────────┘ └──────────┘                         │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐                         │
│  │ Node 4   │ │ Node 5   │ │ Node 6   │  ← to procure           │
│  │ Compute/ │ │ Compute/ │ │ Compute/ │                         │
│  │ Storage  │ │ Storage  │ │ Storage  │                         │
│  └──────────┘ └──────────┘ └──────────┘                         │
│  Switching: PROCURE 2× ToR switches (25G)  ← MISSING            │
└──────────────────────────────────────────────────────────────────┘
```

### Node Roles

| Role | Count per Cluster | Description |
|---|---|---|
| **Controller (Primary)** | 1 | Hosts VergeOS UI, API, and cluster metadata |
| **Controller (Backup)** | 1 | Hot standby controller; takes over automatically on failure |
| **Compute/Storage** | 4 | Runs VMs and contributes storage to the vSAN |

> VergeOS requires a minimum of 3 nodes per cluster. A 6-node cluster provides N+1 redundancy for both compute and storage.

---

### Storage Design

VergeOS vSAN distributes and redundantly stores data across all nodes. With 6 nodes, **3-way redundancy** (three copies of each block) is available, allowing up to 2 simultaneous node failures without data loss. The table below uses **2-copy** (standard) and **3-copy** (high redundancy) scenarios.

#### Production Cluster (6 nodes — current 3 + 3 procured, matching spec)

| Tier | Drive | Raw per node | Cluster raw | Usable 2-copy | Usable 3-copy |
|---|---|---|---|---|---|
| Tier 1 — NVMe | 1× 1.6 TB U.2 | 1.6 TB | 9.6 TB | ~4.8 TB | ~3.2 TB |
| Tier 2 — SATA SSD | 6× 3.8 TB | 22.8 TB | 136.8 TB | ~68.4 TB | ~45.6 TB |
| Boot (VergeOS OS) | 240 GB M.2 | dedicated — not in vSAN pool | | | |

#### DR Cluster (6 nodes — current 3 + 3 procured, matching spec)

| Tier | Drive | Raw per node | Cluster raw | Usable 2-copy | Usable 3-copy |
|---|---|---|---|---|---|
| Tier 1 — NVMe | 1× 1.6 TB U.2 | 1.6 TB | 9.6 TB | ~4.8 TB | ~3.2 TB |
| Tier 2 — SATA SSD | 16× 3.8 TB | 60.8 TB | 364.8 TB | ~182.4 TB | ~121.6 TB |
| Boot (VergeOS OS) | 240 GB M.2 | dedicated — not in vSAN pool | | | |

> **Recommendation:** Use the 240 GB M.2 SATA drive on each node as the VergeOS OS boot device. The 240 GB SATA SSD (`HX-SD240GM1X-EV`) can be added as a third vSAN tier or kept as a hot spare.

---

### Compute Capacity (6-node target per cluster)

| Metric | DC Cluster (6 nodes) | DR Cluster (6 nodes) |
|---|---|---|
| Physical cores | 240 (6× 2× 20C) | 288 (6× 2× 24C) |
| vCPUs (2:1 ratio) | ~480 | ~576 |
| Total RAM | 6 TB | 12 TB |
| GPU (T4 16 GB) | 0 | 24 (6 nodes × 4 T4) — see note |

> **GPU note:** Current DR BOM has 2× T4 per node across the 3 existing nodes = 6 T4 cards total. If procured nodes also carry 2× T4 each, the DR cluster reaches 12 T4 cards across 6 nodes. The BOM only shows 6 GRID VDI licenses (1 CCU each) — additional licenses will be needed to match expanded GPU capacity.

---

### Network Design

VergeOS uses two mandatory physical networks:

| Network | Purpose | Recommended Speed |
|---|---|---|
| **Core** | Internal vSAN traffic, VM live migration, cluster heartbeat | 25 GbE (minimum 10 GbE) |
| **External** | VM guest traffic, management UI, inter-site replication | 10–25 GbE |

The Cisco UCS VIC 1457 provides 4× 25G SFP28 ports. Recommended port assignment per node:

```
VIC 1457 Port 1 ──► Core Network (VLAN trunk) ──► Switch A
VIC 1457 Port 2 ──► Core Network (bond/LACP)  ──► Switch B  (redundancy)
VIC 1457 Port 3 ──► External Network           ──► Switch A
VIC 1457 Port 4 ──► External Network (bond)    ──► Switch B  (redundancy)
```

**DC Site switching (UCS FI 6454):**

The UCS FI 6454 must be reconfigured from *HyperFlex / UCS end-host mode* to standard **Ethernet switching mode** to work with VergeOS. In this mode the FI acts as a standard ToR switch with VLAN support. The 48× 10/25G server-facing ports and 8× 40/100G uplink ports remain fully usable.

**DR Site switching:** No switches are included in the DR BOM. See [Capacity Gaps and Incompatibilities](#capacity-gaps-and-incompatibilities).

---

### VergeOS Features Enabled by This Hardware

| Feature | Cluster | Notes |
|---|---|---|
| **vSAN (built-in HCI storage)** | Both | Tiered: NVMe (tier 1) + SATA SSD (tier 2) |
| **VM live migration** | Both | Core network; no shared storage required |
| **Cloud Snapshots** | Both | Point-in-time VM and tenant snapshots |
| **Site Sync** | Both → Both | Async replication DC ↔ DR via External network |
| **Repair Server (ioGuardian)** | Both | Auto-retrieves missing blocks from peer site |
| **NVIDIA vGPU / GPU passthrough** | DR | T4 cards support NVIDIA vGPU; GRID licenses transfer to VergeOS |
| **VDI** | DR | GPU-accelerated virtual desktops via VergeOS VDI feature |
| **Tenants** | Both | Multi-tenancy; DR can host isolated tenant workloads |
| **WireGuard / IPSec VPN** | Both | Site-to-site VPN for inter-cluster replication if no dedicated link |
| **Global Inline Deduplication** | Both | Reduces storage consumed by replicated snapshots |

---

## Capacity Gaps and Incompatibilities

The following issues must be resolved before or during deployment.

---

### GAP 1 — Node Shortfall (Critical)

**Both sites are 3 nodes short of the 12-node target.**

| Site | Nodes in BOM | Nodes Required | Shortfall |
|---|---|---|---|
| Production (DC) | 3 | 6 | **3 nodes** |
| DR | 3 | 6 | **3 nodes** |

A 3-node cluster is the absolute VergeOS minimum and provides no compute N+1 headroom — a single node failure consumes the entire redundancy budget. **For production workloads, 6 nodes per cluster is the recommended minimum.**

**Resolution:** Procure 3 additional HX240c M5 nodes for each site with matching specifications:

- DC additions: match CPU (6148), RAM (64 GB DIMMs × 16), and drive (6× 3.8 TB SATA + 1× 1.6 TB NVMe) configuration.
- DR additions: match CPU (6248R), RAM (128 GB DIMMs × 16), and drive (16× 3.8 TB SATA + 1× 1.6 TB NVMe) configuration. Consider adding 2× T4 per node to maintain GPU density.

---

### GAP 2 — Missing DR Switching Infrastructure (Critical)

**The DR BOM contains zero network switches.** VergeOS requires at minimum two redundant ToR (Top-of-Rack) switches to provide Core and External networks with HA path redundancy.

**Resolution:** Procure for the DR site:

- 2× ToR switches with ≥25 GbE server-facing ports and sufficient capacity for 6 nodes (12 uplinks per switch minimum: 6 nodes × 2 redundant uplinks).
- Suitable options: Cisco Nexus 93180YC-FX, Arista 7050CX3, or similar 25G-capable switches.
- Alternatively, 2× UCS FI 6454 (matching the DC site) if Cisco UCS tooling is desired for out-of-band management.

---

### INCOMPATIBILITY 1 — Cisco UCS VIC 1457 Standalone Behavior (Important)

The Cisco UCS VIC 1457 is a Converged Network Adapter (CNA) designed to work with UCS Manager and Fabric Interconnects. When used **outside** the UCS managed domain (as it will be in VergeOS), the VIC presents via the open-source `enic` Linux kernel driver in **standalone / passthrough mode**.

| Concern | Detail |
|---|---|
| Driver support | Linux `enic` driver is included in the mainline kernel and is compatible with VergeOS |
| Loss of UCS Manager features | Port profiles, FCoE, VNIC templates, and blade-side switching are unavailable outside UCS |
| 25G speed in standalone mode | VIC 1457 standalone mode defaults to **10G** on some firmware versions; 25G requires explicit firmware and BIOS configuration |
| SR-IOV | SR-IOV is supported by the enic driver but requires careful configuration |

**Resolution:**

1. Validate that the installed VIC 1457 firmware version supports 25G in standalone mode before deployment.
2. Consider adding a dedicated Intel X710 (4× 10G) or XXV710 (2× 25G) PCIe NIC to each node for the VergeOS Core network to guarantee line-rate 25G performance and simplify driver support. Riser slots are available (HX-PCI-1-C240M5 / HX-PCI-2B-240M5 in DC; HX-RIS-1B / HX-RIS-2B in DR).
3. If the VIC 1457 is used in its current state, test throughput and confirm Core network achieves ≥10 Gbps per node before production cutover.

---

### INCOMPATIBILITY 2 — UCS Fabric Interconnect Reconfiguration Required (Important)

The DC UCS FI 6454 units are configured in **UCS Manager end-host mode** for HyperFlex. This mode is incompatible with VergeOS, which expects standard 802.1Q Ethernet switching with VLAN trunks.

**Required changes:**

1. Factory-reset or reconfigure the FI 6454 into **Ethernet switching mode** (also called NX-OS switching mode).
2. Configure VLANs for Core and External VergeOS networks.
3. Configure LACP/port-channel for bonded node uplinks.
4. Decommission UCS Manager — it will no longer manage compute nodes.

> This reconfiguration is destructive and irreversible without a factory reset. Plan a maintenance window and ensure no production workloads are running on the FI before proceeding.

---

### INCOMPATIBILITY 3 — DR Power Budget with 1050 W PSUs (Medium)

The DR nodes use **1050 W PSUs** in an N+1 redundant configuration (one active, one standby). At full load, a single PSU must power the entire node.

**Estimated maximum DR node power draw:**

| Component | Power |
|---|---|
| 2× Intel Xeon 6248R (205 W each) | 410 W |
| 2× NVIDIA T4 (75 W each) | 150 W |
| 16× 128 GB LRDIMM (~8 W each) | 128 W |
| 16× 3.8 TB SATA SSD (~3 W each) | 48 W |
| 1× 1.6 TB NVMe | 10 W |
| Motherboard / fans / misc | ~100 W |
| **Total estimated** | **~846 W** |

The 1050 W PSU leaves only ~204 W headroom (≈19%) under PSU-failure conditions. Intel recommends ≥20% headroom for sustained workloads. GPU-intensive VDI workloads or memory-bandwidth-heavy jobs may transiently exceed safe limits.

**Resolution:** Replace DR node PSUs with **HX-PSU1-1600W** (matching DC configuration) to provide adequate headroom. Order 6× 1600 W PSUs for the 3 DR nodes (2 per node), plus 6 more for the 3 nodes to be procured.

---

### INCOMPATIBILITY 4 — CPU Microarchitecture Mismatch Between Clusters (Low — Design Note)

| Cluster | CPU | Microarchitecture | AVX-512 variant |
|---|---|---|---|
| DC | Xeon Gold 6148 | Skylake-SP (2017) | AVX-512F/BW/VL/CD/DQ |
| DR | Xeon Gold 6248R | Cascade Lake-R (2020) | Adds VNNI, IFMA, VBMI |

Because these are **separate VergeOS clusters**, VMs do not live-migrate between DC and DR. The CPU differences are therefore not operationally problematic. However:

- VMs migrated between sites via **Site Sync + failover** will restart (not live-migrate) on the DR cluster. Ensure guest OS drivers do not hard-depend on Cascade Lake-specific instructions if the VM may need to run on the DC cluster.
- Do not attempt to join DC and DR nodes into a single cluster — the CPU generation mismatch would cause instability for VMs with passthrough CPU features.

---

### INCOMPATIBILITY 5 — Legacy VMware and HyperFlex Licenses Are Unusable (Cost / Planning)

The BOM includes significant VMware and Cisco HyperFlex software:

| License | DC Qty | DR Qty | Usable with VergeOS |
|---|---|---|---|
| VMware vSphere 6.5 / 6.7 (per CPU) | 6 CPU | 6 CPU | **No** — VergeOS replaces ESXi |
| Cisco HXDP Enterprise / Datacenter Premier | 3 nodes | 3 nodes | **No** — HyperFlex stack is removed |
| Cisco UCS Director / Intersight | various | — | **No** — replaced by VergeOS UI |

**NVIDIA GRID VDI App licenses** (6× 1 CCU, DR) are tied to the physical T4 GPUs and are **hypervisor-agnostic** — they remain valid and usable with VergeOS NVIDIA vGPU support.

---

### INCOMPATIBILITY 6 — DC Heatsinks at Thermal Limit (Low — Risk Monitoring)

DC nodes use **UCSC-HS-C240M5** heatsinks, which are rated for CPUs up to **150 W TDP**. The Intel Xeon Gold 6148 has a TDP of exactly **150 W**, leaving no thermal headroom.

- Ensure datacenter ambient inlet temperature is maintained at or below **25 °C**.
- Monitor CPU junction temperatures in VergeOS System → Nodes → IPMI / SEL.
- Under sustained full-core workloads, consider configuring BIOS power capping to 145 W per CPU as a protective measure.

---

## Procurement Summary

The following items must be procured to complete the 12-node / 2-cluster design:

| Item | Qty | Reason |
|---|---|---|
| Cisco HX240c M5 node (DC spec: 2× 6148, 16× 64 GB, 6× 3.8 TB SATA, 1× 1.6 TB NVMe, VIC 1457) | 3 | Node shortfall — DC Cluster |
| Cisco HX240c M5 node (DR spec: 2× 6248R, 16× 128 GB, 16× 3.8 TB SATA, 1× 1.6 TB NVMe, VIC 1457) | 3 | Node shortfall — DR Cluster |
| 25G-capable ToR switch (≥24× 25G ports) | 2 | DR site switching — completely absent |
| 25G DAC / AOC cables for DR ToR switches | as needed | DR inter-switch and node cabling |
| HX-PSU1-1600W (or equivalent ≥1600 W) | 12 | Replace 1050 W PSUs in all 6 DR nodes (3 existing + 3 new) |
| Intel X710/XXV710 PCIe NIC (optional but recommended) | 12 | Dedicated VergeOS Core network NIC — one per node across both clusters |
| NVIDIA GRID VDI App CCU licenses (additional) | as needed | Expand GPU-licensed users if additional T4 cards are added to new DR nodes |

---

## VergeOS Deployment Checklist

- [ ] Confirm VIC 1457 firmware supports 25G in standalone mode on all 6 existing nodes
- [ ] Reconfigure DC UCS FI 6454 units to Ethernet switching mode (plan maintenance window)
- [ ] Procure and rack 3× DC nodes and 3× DR nodes
- [ ] Procure and install 2× ToR switches for DR site with proper cabling
- [ ] Replace DR node PSUs with 1600 W units
- [ ] Install VergeOS on boot M.2 drives; configure Core and External networks per VLAN design
- [ ] Initialize DC cluster (node 1 as primary controller, node 2 as backup controller)
- [ ] Initialize DR cluster independently
- [ ] Configure VergeOS Sites dashboard to link DC ↔ DR
- [ ] Configure Site Sync profiles for cloud snapshot replication from DC → DR
- [ ] Enable NVIDIA vGPU on DR cluster nodes with T4 cards
- [ ] Validate GRID licenses are applied and functional
- [ ] Verify storage tier configuration: NVMe as Tier 1, SATA SSD as Tier 2
- [ ] Set vSAN redundancy level: 3-copy recommended with 6 nodes
- [ ] Decommission VMware vSphere and Cisco HyperFlex software stacks

---

*Reference Architecture | VergeOS | Cisco HX240c M5 All Flash | 12-Node / 2-Cluster*
