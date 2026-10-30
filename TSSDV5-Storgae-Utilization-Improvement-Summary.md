# TSSDV5 Storage Utilization‑Improvement Project Phase Summary

## 1. Project Background
The overall storage utilization of the TSSD V3 platform is only around 30 %, which imposes significant cost pressure on business teams.
Operations tasks including splitting of full small‑tables, data migration triggered by server crash or disk failure all require manual intervention from O&M engineers, resulting in heavy operational workload.

TSSD V5 improves storage utilization through storage‑engine optimization, development of the automatic `scheduler` module, and operational enhancements including a shared resource pool and V5 health‑status monitoring. The target business storage utilization is set to 75 %, while overall platform automation is also elevated.

## 2. Analysis of TSSD V5 Storage Utilization Improvement Principles
### 2.1 Storage Capacity Breakdown of TSSD V3

| Item | Dump Area | Failure‑and‑Growth Buffer | Fragmentation Ratio | Business Free Capacity | SSD Reserved Area | Storage Utilization |
|---|---:|---:|---:|---:|---:|---:|
| Value | 3.40 % | 13.21 % | 20.00 % | 25.73 % | 7.96 % | 29.70 % |

> Figure 2‑1: TSSD V3 Storage Capacity Breakdown

Analysis of TSSD‑V3 storage components:
1. **Dump Area**: Stores index data and write‑ahead logs.
2. **Failure‑and‑Growth Buffer**: Each V3 warehouse reserves roughly 13 % capacity for failure recovery and business‑data growth.
3. **Fragmentation Ratio**: The bitcast‑based storage engine requires regular garbage collection. The 20 % fragmentation threshold is configured mainly to preserve SSD write endurance.
4. **SSD Reserved Area**: Reduces write amplification at the SSD hardware layer.
5. **Business Free Capacity**: Capacity allocated to businesses but not yet consumed. The high proportion of free space originates from V3’s expansion logic: physical capacity doubles upon scaling while actual data volume stays unchanged, dragging down overall utilization.
6. **Storage Utilization**: Percentage of physical capacity occupied by real business data.

### 2.2 Methods to Improve Storage Utilization in TSSD V5
1. **Reduce failure‑and‑growth buffer ratio**:
Based on real‑world disk‑failure statistics and operational experience, the failure buffer is reduced to 1.5 % and the growth buffer to 3.5 %. A **shared resource pool** centrally manages failure‑ and growth‑buffers across all warehouses.

2. **Lower fragmentation ratio**:
Raise the block‑reclamation threshold for cells. Disk fragmentation is reduced while still guaranteeing a four‑year service lifetime for SSDs.

3. **Remove the SSD reserved area**:
Production statistics show SSDs can still sustain a four‑year lifetime after eliminating the dedicated SSD reserved area.

4. **Cut down business free capacity**:
Optimize the cell storage engine. New hardware provisioning is no longer required for small‑table splitting; capacity expansion no longer degrades storage utilization.

> Figure 2‑2: TSSD V5 Storage Capacity Breakdown

| Item | Fragmentation | Storage Utilization | Failure‑and‑Growth Buffer | Dump Area |
|---|---:|---:|---:|---:|
| Value | 15.00 % | 76.60 % | 5.00 % | 3.40 % |

## 3. Operational Challenges & Mitigation Solutions for Higher TSSD‑V5 Utilization

Challenges introduced by higher storage‑utilization:
1. Dense mixed‑use of disk space raises the risk of full disks. Once a disk becomes full, all small‑tables residing on that disk are impacted. In TSSD‑V3, a single full small‑table would not affect other tables on the same disk. TSSD‑V5 requires timely detection of full‑disk events and automatic scheduling actions.
2. Shared failure buffer: Server crashes and disk failures must be detected promptly. New servers need to be provisioned for data migration, and faulty hardware must be reclaimed.
3. Shared growth buffer: Detect warehouses approaching capacity limits, trigger expansion with new machine provisioning; perform scale‑in for under‑utilized warehouses and reclaim surplus hardware to sustain high platform‑wide utilization.
4. End‑to‑end management of failure‑buffer and free‑buffer resources for the whole platform.

### Proposed Solutions
#### 1. Shared Resource‑Pool Module
Core capabilities:
1. Manage failure‑buffer and growth‑buffer resources for each region.
2. Periodically collect warehouse capacity and health metrics. Provision new machines for warehouses hit by server crashes, disk failures or capacity exhaustion. Trigger data‑migration via the automatic scheduler and reclaim faulty hardware.
3. Support scale‑in operations: reclaim excess machines for warehouses with low storage utilization.
4. Continuously audit resource‑pool saturation; apply for additional hardware from the department‑level resource pool when resources run low.

> Diagram: TSSD V5 Global Shared‑Resource‑Pool Management
> Workflow: Department‑level resource pool ↔(apply / return hardware)↔ TSSD shared resource pool ↔(resource in / resource out)↔ TSSD‑V5 Warehouses (warehouse 1 … warehouse n)

#### 2. Scheduler Resource‑Scheduling Module — Reduce O&M Burden & Safeguard Utilization Gains
Scheduler is a newly‑added component for TSSD V5. Key features:
1. **Automated load balancing**: Monitor abnormal load conditions such as full disks or full small‑tables. Automatically trigger data migration to evenly distribute large‑tables across cell nodes.
2. **Automated fault handling**: Perform data migration for crashed nodes and failed disks.

## 4. Project Progress & Business Benefits
TSSD‑V5 has been rolled‑out for Qzone core applications, Feed‑type workloads and file‑index services, covering roughly 50 % of total data volume across the entire TSSD platform. Full platform migration to TSSD‑V5 is expected to complete in the second half of the year.

Current overall utilization of the V5 cluster reaches 66 %, with more than **1 400 storage servers saved in aggregate**.

The shared‑resource‑pool module has been gray‑released across 11 warehouses supporting Qzone core services. Operations including crash‑and‑failure migration, hardware provisioning / decommissioning, capacity expansion and buffer‑resource management are fully handled by the shared resource pool. Both the resource‑pool and scheduler components are operating largely as expected.

Scale‑in operations have been executed for seven warehouses located in Shenzhen for Qzone core services. After scale‑in, storage‑utilization for these Shenzhen‑region warehouses exceeds 75 %.
