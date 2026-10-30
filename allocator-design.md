# Allocator Detailed Design V1.6.0

## Table of Contents
1. [Allocator Introduction](#1-allocator-introduction)
   - 1.1 [Allocator Interaction Modules](#11-allocator-interaction-modules)
   - 1.2 [Design Constraints](#12-design-constraints)
   - 1.3 [External Interfaces](#13-external-interfaces)
2. [Resource Management](#2-resource-management)
   - 2.1 [Data Structure Definition](#21-data-structure-definition)
   - 2.2 [Resource Fetching — ResourceSyncer](#22-resource-fetching--resourcesyncer)
     - 2.2.1 [Fetching at Process Startup](#221-fetching-at-process-startup)
     - 2.2.2 [Periodic Fetching](#222-periodic-fetching)
     - 2.2.3 [External-Triggered Fetching](#223-external-triggered-fetching)
   - 2.3 [Resource Persistence](#23-resource-persistence)
     - 2.3.1 [Dump File Format](#231-dump-file-format)
     - 2.3.2 [Dump File Generation — ResourceManager::Dump()](#232-dump-file-generation--resourcemanagerdump)
     - 2.3.3 [Dump File Loading — ResourceManager::Load()](#233-dump-file-loading--resourcemanagerload)
3. [Upload Space Allocation](#3-upload-space-allocation)
   - 3.1 [Routing Hierarchy](#31-routing-hierarchy)
   - 3.2 [Upload Space Management](#32-upload-space-management)
     - 3.2.1 [Allocatable Space Management — ClassSpaceDicer](#321-allocatable-space-management--classspacedicer)
     - 3.2.2 [Allocatable Volume Management under Space — SpaceVolumeDicer](#322-allocatable-volume-management-under-space--spacevolumedicer)
     - 3.2.3 [Weight-based Space Selection and Multi-level Allocation Queue Algorithm](#323-weight-based-space-selection-and-multi-level-allocation-queue-algorithm)
   - 3.3 [Upload Space Allocation — Handler::AllocBlob()](#33-upload-space-allocation--handlerallocblob)
4. [Pull Volume Information from BlobNode — VolumeAllocStatusSyncer](#4-pull-volume-information-from-blobnode--volumeallocstatussyncer)
   - 4.1 [Volume Update Scenarios](#41-volume-update-scenarios)
   - 4.2 [Pending-update Queue Generation and Update](#42-pending-update-queue-generation-and-update)
   - 4.3 [Scheduling Interval for Pending-update Queue](#43-scheduling-interval-for-pending-update-queue)
   - 4.4 [Single-Volume Fetch Workflow](#44-single-volume-fetch-workflow)
   - 4.5 [Exception Handling](#45-exception-handling)
5. [Appendix](#5-appendix)
   - 5.1 [Memory Usage Estimation](#51-memory-usage-estimation)
   - 5.2 [Time-cost Estimation for Pulling max_vbid and free from vlet](#52-time-cost-estimation-for-pulling-max_vbid-and-free-from-vlet)
   - 5.3 [Ratio-Random Algorithm](#53-ratio-random-algorithm)
   - 5.4 [Analysis of Volume Non-allocable Scenarios](#54-analysis-of-volume-non-allocable-scenarios)
   - 5.5 [Volume Health-level Classification](#55-volume-health-level-classification)
   - 5.6 [Volume Partitioning for Allocator](#56-volume-partitioning-for-allocator)

---

# 1. Allocator Introduction

Allocator is responsible for assigning `bid` for newly-uploaded blobs. Within BlobStorage, multiple Allocator instances handle upload-space allocation, and each Allocator manages a distinct list of volumes.

## 1.1 Allocator Interaction Modules

- **Interface**: BlobStorage access module. Upon receiving a PutBlob request, it applies to Allocator for storage space.
- **Cayman**: Maintains the full list of Allocator hosts. The cayman-agent co-located with Interface periodically communicates with Cayman to fetch the latest Allocator host list and caches it locally. Interface retrieves the Allocator host list from cayman-agent.
- **VolumeService**: Supplies volume / space metadata to Allocator when local metadata is empty.
- **BlobNode**: Allocator periodically queries BlobNode for vlet metrics including free capacity and max_vbid.
- **Center**: Notifies Allocator of newly-online volumes. It periodically collects cluster-wide capacity statistics from Allocator to evaluate capacity-expansion requirements.

## 1.2 Design Constraints

1) `vid` generation rule under each space: `vid` is of type `uint32`. Its high-order 14 bits encode the space ID; its low-order 18 bits are auto-incremental starting from zero.

## 1.3 External Interfaces

The external interfaces exposed by Allocator are listed below:

| Interface | Function | Caller | Scenario |
|---|---|---|---|
| AllocBlob | Allocate upload space | Interface | Blob upload |
| StockVolumes | Add volumes into Allocator | Center | Center pushes newly-added volumes |
| ComplainVolume | Notify Allocator of vlet exceptions during blob upload | Interface | Upload failures against volumes allocated by Allocator |
| ListVolumeAllocStatus | Fetch volumes under specified filters | OSS / peripheral modules | Retrieve volume metadata such as max_vbid and free capacity |
| ListSpaceAllocInfo | Fetch hash-bucket list of a target space | OSS / Center | Query hash-bucket list belonging to a specific space |

### Interface Detailed Definitions

#### AllocBlob

```protobuf
message AllocBlobRequest {
  uint32 blob_size;
  uint32 storage_class;
}

message AllocBlobResponse {
  int32 code;
  uint32 blob_version;
  uint64 bid;
  uint32 timestamp;
}
```

#### StockVolumes

```protobuf
message StockVolumesRequest {
  uint64 free; // free capacity of newly-provisioned volume
  uint32 max_vbid; // max_vbid of newly-provisioned volume
  repeated uint32 vid;
}

message StockVolumesResponse {
  int32 code;
  int32 success_count;
}
```

Processing logic:
Volumes already existing locally in Allocator are skipped. For newly-added volumes, trigger the workflow to pull max_vbid and free capacity.

#### ComplainVolume

```protobuf
message ComplainVolumeRequest {
  uint32 vid;
  uint32 reason;
  uint32 total_count;
  uint32 failed_count;
}

message ComplainVolumeResponse {
  int32 code;
}

enum ComplainVolumeReason {
  kVolumeVbidError,      // Gap between allocated vbid and vlet max_vbid >50, or vbid already exists
  kVolumeFull,
  kSpaceUnavailable,     // Space exception: space offline / write-disabled
  kVletUploadTimeout,
  kPutQuorumError,       // Number of vlets with identical MV is less than put_quorum
  kVolumeUnavailable     // Volume exception e.g. volume offline
}
```

Processing logic:
1) Skip processing if the target volume is marked non-allocable.
2) For allocable volumes: apply corresponding handling according to `reason` and `failed_count` in the request:
    a. `kSpaceUnavailable`: Pull latest space metadata from VolumeService and update local records. If the space is write-disabled or marked for deletion, remove this space from allocation candidate list.
    b. `kVolumeVbidError`, `kVolumeFull`, `kVolumeUnavailable`: Remove volume from allocation list and update its status.
    c. `kVletUploadTimeout`: Remove volume from allocation list if blob upload fails; update volume health value if upload succeeds but partial vlets time out.

> **Note**: When multiple vlet failures with different error codes occur, return the error code with highest severity. Severity ranking:
> `kSpaceUnavailable` > `kPutQuorumError` > `kVolumeFull` > `kVolumeVbidError` > `kVletUploadTimeout`.

#### ListVolumeAllocStatus

```protobuf
message VolumeAllocStatus {
  uint32 vid;
  uint32 disk_id;
  uint64 free;
  uint32 max_vbid;
  uint32 last_sync_time;
  uint32 alloc_count;
  uint8 healthy_value;
}

message ListVolumeAllocStatusRequest {
  oneof list_id {
    uint8 storage_spec;
    uint32 space;
    uint64 vid;
  }
  uint32 marker; // Pagination marker: fetch next batch excluding this marker; total payload ≤ 1MB
}

message ListVolumeAllocStatusResponse {
  int32 code;
  uint32 marker; // Last vid of current batch if partial result; zero if finished
  repeated VolumeAllocStatus alloc_status;
}
```

#### ListSpaceAllocInfo

```protobuf
message ListSpaceAllocInfoRequest {
  oneof list_id {
    uint8 storage_spec;
    uint32 space_id;
  }
}

message FreeInfo {
  uint64 min_free;
  uint64 max_free;
  uint32 volume_count;
  uint64 free_total;
}

message SpaceAllocInfo {
  uint32 space_id;
  uint8 storage_spec;
  repeated uint16 bin_id_list;
  uint64 free;
  uint32 volume_count;
  repeated FreeInfo free_info_list;
}

message ListSpaceAllocInfoResponse {
  int32 code;
  repeated SpaceAllocInfo space_info_list;
}
```

---

# 2. Resource Management

Allocator persists volume and space metadata to perform upload-space allocation. Resource management covers metadata fetching, periodic refresh and persistence. This chapter describes metadata fetch workflows and persistence logic for volumes and spaces.

## 2.1 Data Structure Definition

`VolumeAllocStatus` stores metadata for each volume including max_vbid, free capacity and runtime status. Each volume in BlobStorage has a globally-unique ID.

```protobuf
enum VolumeStatus {
  kAllocStatusUnknown,     // Newly-provisioned volume or newly-fetched on process startup
  kAllocStatusNormal,      // Normal, ready for allocation
  kAllocStatusFull,        // Free capacity < 1 MB
  kAllocStatusPutQuorumError, // Count of vlets with maximum MV < put_quorum
  kAllocStatusStale,       // Time since last fetch of max_vbid / free from vlet exceeds threshold (1 day)
  kAllocStatusVbidError    // Local max_vbid deviates from vlet's value by >50, or inconsistency in max_vbid
}

struct VolumeAllocStatus {
  uint32 vid;
  uint16 space_id;
  uint64 free;
  uint32 max_vbid;
  uint32 last_sync_time;
  uint32 alloc_count; // Total allocation times against this volume
  uint8 status;
  uint8 healthy_value; // Health metric for volume
}

struct SpaceAllocStatus {
  uint32 space_id;
  uint64 free;
  CodingSchema code_schema;
  uint8 status;
}
```

## 2.2 Resource Fetching — ResourceSyncer

Three trigger scenarios for resource fetch:
1) Empty local volume / space metadata on process startup
2) Periodic scheduled fetch
3) External-event-driven fetch

### 2.2.1 Fetching at Process Startup

If dump file is missing or restoration from dump fails, Allocator pulls metadata from VolumeService:
1. Fetch all space entries from VolumeService and filter spaces managed by current Allocator.
2. Enumerate volumes under each target space:
    a. Retrieve volume list by `max_vid` of space.
    b. Filter volumes assigned to current Allocator according to space hash-bucket rules.
    c. Persist new volumes, mark them as `kAllocStatusUnknown`, add them to pending queue for max_vbid & free refresh.

Exception handling: Log ERROR and retry when fetching space list fails. Raise fatal alert after 3 consecutive retries.

> **Note**: Allocator only needs `vid` from VolumeService. `max_vbid` and `free` are fetched from vlet on BlobNode.

### 2.2.2 Periodic Fetching

Allocator periodically synchronizes metadata from VolumeService and updates local state:
1. Fetch space metadata from VolumeService.
2. Process state transitions of spaces:
    a. No state change: skip.
    b. Write-enabled → write-disabled: set allocation weight of space to zero.
    c. Write-disabled → write-enabled: recalculate allocation weight and re-join allocation pool.
    d. Write-disabled → marked-offline: no immediate action.
    e. Marked-offline → offline: peripheral modules configure deferred cleanup interval (default: one month). Allocator deletes space / volume records upon expiry.
3. Detect missing volumes:
    a. Identify newly-added volumes by comparing remote `max_vid` and local `max_vid`.
    b. Filter volumes assigned to current Allocator by space hash-bucket rules.
    c. Mark new volumes as `kAllocStatusUnknown` and enqueue for metadata refresh.

Exception handling:
1. Illegal state transitions (write-enabled → marked-offline, write-enabled → offline, offline → write-enabled, offline → write-disabled): trigger fatal alert for manual intervention.
2. Remote `max_vid` < local `max_vid`: fatal alert.

> Open question: What is the periodic fetch interval? How to achieve fast awareness for transition from write-disabled back to write-enabled.

### 2.2.3 External-Triggered Fetching

When Interface reports space anomalies, Allocator actively pulls space metadata and acts accordingly:
1. Write-disabled: set allocation weight to zero.
2. Marked-offline / offline: set allocation weight to zero.
3. Write-enabled: recalculate allocation weight.

When Center pushes new volumes whose parent space is unknown locally: Allocator fetches space metadata:
1. Write-enabled: compute weight and join allocation pool.
2. Write-disabled: weight set to zero.
3. Marked-offline / offline: weight set to zero.

## 2.3 Resource Persistence

Resource persistence enables fast metadata restoration and service resumption after process restart.

### 2.3.1 Dump File Format

Allocator persists below information into dump files:
1. Space basic info (`SpaceAllocStatus`), space allocation weight, parent `storage_spec` ID.
2. Volume records (`VolumeAllocStatus`).
3. Allocation weights for `storage_spec`.

Dump file consists of **Header section** and **Body section**.
- Header: marks file completeness, total space count and dump timestamp.
- Body: serializes each space and its subordinate volumes.

Header fields:
1. crc: 1-byte CRC checksum for header block
2. magic: 4-byte flag for file completeness; set to `0` on dump start; set to `0x7f454c46` on dump completion
3. space_count: 4-byte, total number of spaces persisted
4. dump_time: 4-byte, timestamp when dump is generated

Body section contains serialized space-volume entries.
Each space entry:
1. crc: 1-byte CRC for space block
2. spaceAllocStatus: core metadata of space
3. weight: 4-byte allocation weight
4. volume_count: 4-byte total volumes under this space
5. spec_id: 4-byte parent storage-spec identifier

Each volume entry is prefixed with a CRC byte for integrity validation, followed by `VolumeAllocStatus`.

### 2.3.2 Dump File Generation — ResourceManager::Dump()

Allocator catches termination signals to trigger dump workflow:
1. Stop accepting new `AllocBlob` requests; drain in-flight requests.
2. Write header block into dump file:
    a. crc = 0
    b. magic = 0
    c. space_count = total spaces managed
    d. dump_time = current timestamp
3. Serialize all space-volume records and write into body block.
4. Update header's crc and magic fields.
5. Force-flush file contents to disk.

Exception handling: Raise alert and exit process on disk-write failure.

### 2.3.3 Dump File Loading — ResourceManager::Load()

On startup, Allocator loads dump file to restore metadata quickly:
1. Parse header. Delete corrupted incomplete dump file and fallback to VolumeService fetch if integrity check fails.
2. Deserialize space-volume entries from body block. Add volumes with `kAllocStatusNormal` status into allocation queues.
3. Delete dump file after metadata reconstruction completes.
4. Evaluate ratio of allocable volumes:
    - If ≥10%: serve traffic directly.
    - If <10%: first trigger refresh of free / max_vbid before serving traffic.

Exception handling: Detect corruption by CRC checksum. Raise fatal alert, delete corrupted dump and reload metadata from VolumeService.

> **Note**: Step 3 deletes dump file to avoid stale metadata reuse after unexpected power-off or crash without dump completion.

---

# 3. Upload Space Allocation

This chapter describes management of allocable volumes and allocation workflow. Two core objectives:
1. Efficiently locate eligible volumes for allocation.
2. Dynamically adjust allocation weight according to remaining free capacity to avoid partial-volume saturation degrading cluster throughput.

First the BlobStorage routing hierarchy is introduced, followed by space management and allocation logic.

## 3.1 Routing Hierarchy

`storage_class` represents a data storage type (e.g. Standard, Standard-IA). Each `storage_class` contains multiple `storage_spec`. Each `storage_spec` defines erasure-coding rules (e.g. K9N18, K9N15) and allocation weight. Each `storage_spec` contains a set of spaces sharing identical encoding schema. A `space` is a namespace for volumes with homogeneous attributes, conceptually similar to a table in tabular storage systems.

Hierarchy:
`storage_class` → multiple `storage_spec` → multiple `storage_space` → multiple `volume`

## 3.2 Upload Space Management

Upload-space management has two layers: allocatable-space management under `storage_class`, and allocatable-volume management under each `space`.
When Interface requests upload space:
1. Pick one eligible space from candidates matching target `storage_class`.
2. Select one upload-ready volume from within that space.

### 3.2.1 Allocatable Space Management — ClassSpaceDicer

Maintains candidate spaces for each `storage_class` and their selection weights. Three factors determine space weight:
1. Allocation weight inherited from parent `storage_spec`.
2. Count of allocatable volumes within space.
3. Aggregate free capacity of space.

> Formula descriptions omitted (mathematical expressions in original document).

### 3.2.2 Allocatable Volume Management under Space — SpaceVolumeDicer

Volumes with free capacity larger than shard size are grouped into **18 multi-level allocation queues partitioned by free-capacity ranges**. Within each capacity queue, volumes are further divided into **4 health-level sub-queues**. Volumes with higher health value get higher allocation priority.

Partitioning rules for multi-level queues:
a. Volumes with free < 1 MB are excluded from allocation pool.
b. For free ∈ [1 MB, 8 MB): queues partitioned in 1 MB granularity.
c. For free ≥ 8 MB: queues partitioned by powers-of-two capacity boundaries.
d. Each capacity queue is split into four health-based sub-queues; higher-health sub-queues are prioritized.
e. Within a single health sub-queue, volumes are selected in round-robin fashion.

Allocation weight for each capacity queue depends on two inputs:
1. Upper-bound free capacity of queue
2. Volume count inside queue

Different weight formulas apply for queues with upper-bound ≤ 8 MB and > 8 MB respectively.
This weighting ensures volumes with larger remaining free space receive higher allocation probability and prevent early saturation of small-free volumes.

### 3.2.3 Weight-based Space Selection and Multi-level Allocation Queue Algorithm

Allocator maintains mapping `storage_class → space` and `space → multi-level queues`. Weights for spaces and queues are updated periodically.
After weights are set for candidate spaces and sub-queues, the **ratio-random algorithm** pre-processes selection probability distribution.

## 3.3 Upload Space Allocation — Handler::AllocBlob()

Allocation workflow:
1. Validate existence of requested `storage_class`. Return failure if invalid.
2. Apply ratio-random algorithm to pick one eligible space from candidates. Return failure if no candidate available.
3. Derive shard size and candidate queue set according to `blob_size` and space encoding schema. Candidate queues must have lower-bound free ≥ shard size. Sum total weight `sum_ratio`. Generate random number within specified range and select target capacity queue.
4. Inside selected capacity queue: pick volume from highest-health sub-queue; fall back to next lower health level if no available volume.
5. Generate `bid = (vid << 32) + max_vbid` from volume id and max_vbid. Generate random value as `blob_version`.
6. Update volume's free and max_vbid. If free capacity drops below lower-bound of current queue, remove volume and re-insert it into correct matching queue.

> **Note**: `precision` denotes probability precision parameter used in ratio-random algorithm.

---

# 4. Pull Volume Information from BlobNode — VolumeAllocStatusSyncer

Allocator periodically fetches `free` and `max_vbid` from BlobNode to avoid inconsistency between Allocator metadata and real vlet runtime state, which would trigger upload failures. This chapter covers trigger conditions, scheduling and per-volume fetch workflow.

## 4.1 Volume Update Scenarios

1. Volume status equals `kAllocStatusUnknown`, `kAllocStatusFull`, `kAllocStatusPutQuorumError` or `kAllocStatusVbidError`. Refer to analysis of non-allocable volumes.
2. Volume status is `kAllocStatusNormal`, and interval since `last_sync_time` exceeds threshold (default: 12 hours).

## 4.2 Pending-update Queue Generation and Update

On startup, Allocator scans all volumes and enqueues them into corresponding pending-update queues according to status. Queues are refreshed via periodic scanning and Interface feedback events.

Events that trigger queue update:
1. New volume added.
2. Interface invokes `ComplainVolume` reporting volume anomalies.
3. Remaining free of allocated volume drops below 1 MB.
4. `last_sync_time` exceeds threshold (default: two days). Volumes with smaller free capacity are prioritized; pending-update volume count capped at 10% of total allocable volumes.
5. Volumes that previously failed max_vbid / free fetch from BlobNode. Failure causes include: count of same-MV vlets < put_quorum; gap between Allocator max_vbid and vlet max_vbid >50.

## 4.3 Scheduling Interval for Pending-update Queue

| Status | Default Scheduling Interval | Adjusted Interval |
|---|---|---|
| kAllocStatusUnknown | 3 min | Immediately synchronize after volume import on fresh startup |
| kAllocStatusFull | 60 min | 30 min when full-volume ratio ∈ [20%, 50%]; 10 min when >50% |
| kAllocStatusPutQuorumError | 30 min | 10 min when error-volume ratio >10% |
| kAllocStatusVbidError | 30 min | 10 min when error-volume ratio >10% |
| kAllocStatusStale | 60 min | 20 min when stale-normal-volume ratio >30% |

## 4.4 Single-Volume Fetch Workflow

1. Query volume metadata from VolumeService.
2. Send `GetVletAllocStatus` to all vlets belonging to this volume.
3. Aggregate results:
    - Volume-level free = minimum free value returned among all vlets.
    - Volume-level max_vbid = maximum max_vbid returned among all vlets.
    - If Allocator local max_vbid > vlet max_vbid: keep Allocator value. Raise alert if gap >50. Volume becomes non-allocable and requires manual intervention.
4. After successful sync: if volume free >1 MB, add it back to allocation candidate list.

> **Open question**: How to guarantee accuracy of fetched free / max_vbid while Interface is actively uploading data?
> Mitigation: Mark volumes as stale and freeze allocation for ten minutes before fetching metadata from BlobNode.
> Impact of inaccurate fetch: Over-estimated free may cause later upload failure, which will be reported back via ComplainVolume callback.

## 4.5 Exception Handling

1. Query volume metadata from VolumeService fails
    a. Volume not found: delete volume record locally.
    b. Timeout: retry up to three times. Raise alert if still failing.
2. Count of vlets sharing identical MV < put_quorum (possible MV change not fully synced across vlets)
    a. Retry three times. Return failure if condition persists.
3. Partial vlets fail to respond: perform update only if successful responses satisfy criteria:
    a. At least `put_quorum` vlets with identical MV among successful responses.
    b. At least `put_quorum` distinct indexes among same-MV vlets in successful responses.

> **Note**: Behavior for vlets that fail metadata fetch during upload:
> a. vlet max_vbid == Allocator max_vbid: upload proceeds normally.
> b. vlet max_vbid > Allocator max_vbid: upload proceeds normally.
> c. vlet max_vbid < Allocator max_vbid with gap >50: local tinkerman on BlobNode negotiates max_vbid. Vlet decides whether to accept upload. Upload failure triggers ComplainVolume callback from Interface.

---

# 5. Appendix

All estimations assume maximum scale of one BlobStorage cluster:
- 50,000 storage nodes; 100 Allocator instances; TSC180 hardware (8T × 18); K9N18 erasure-coding; each vlet = 16 GB.

## 5.1 Memory Usage Estimation

- Volume memory: total cluster-wide volumes ≈25.6 million. Assume 40 bytes per volume. Each Allocator holds 256,000 volumes: memory footprint = 256,000 × 40 B = 10 MB.
- Space memory: maximum space count = 2¹⁴. Assume 100 bytes per space: total = 2¹⁴ × 100 B = 1600 KB.

## 5.2 Time-cost Estimation for Pulling max_vbid and free from vlet

With 100 Allocators each managing 256,000 volumes, full sync within 0.5 h requires ~8532 volumes synced per minute per Allocator.
Total QPS toward BlobNode: 8532 × 18 × 100 = 15,357,600 requests / min. Average load per BlobNode ≈ 306 requests / min.
Since free_space and max_vbid reside in memory on BlobNode, no overload risk is introduced.

## 5.3 Ratio-Random Algorithm

Inputs:
1. `value_array` of length n
2. `weight_array` of length n, selection probability for each entry in value_array; sum of weights equals 1
3. precision: probability resolution

Output:
`ratio_array` of length `1/precision`. Each entry stores index pointing to `value_array`. Probability of selecting element k matches its weight.

Algorithm steps:
1. Pre-compute probability distribution and populate ratio_array.

```
ratio_array[1/precision] = {0};
curr_index = 0
for i in [0, n):
    ratio_count = weight[i] / precision
    for j in [0, ratio_count):
        ratio_array[curr_index] = i
        ++curr_index
```

Selection method: generate random integer `k = random() % (1/precision)`. Selected element = `value_array[ratio_array[k]]`.

## 5.4 Analysis of Volume Non-allocable Scenarios

1. Newly-added volumes
    a. Volumes imported from VolumeService on cold startup; must fetch max_vbid / free from BlobNode vlets.
    b. New volumes pushed by Center.
    c. Missing volumes discovered via periodic comparison.
2. Volume capacity saturated.
3. Volume vbid / blob_version inconsistency
    a. blob_version returned by AllocBlob mismatches value on vlet.
    b. bid returned by AllocBlob already exists on vlet.
    c. vbid (lower-32-bit of bid) exceeds vlet max_vbid with gap >50.
4. Count of vlets sharing same MV on volume < put_quorum.

## 5.5 Volume Health-level Classification

Four health grades defined according to count of unhealthy vlet indexes (0-3). Unhealthy vlets include upload-failure reported by Interface, metadata-fetch failures from vlet. Health degrades as bad-vlet count increases. Volume becomes non-allocable once bad-vlet count exceeds three.

## 5.6 Volume Partitioning for Allocator

Two core principles for volume partitioning among Allocator instances:
1. Good horizontal scalability: allow dynamic expansion of Allocator count as cluster grows.
2. Keep volumes belonging to one space concentrated on a small subset of Allocators, avoiding excessive fragmentation which harms allocation performance.

---

## Glossary

| Chinese | English |
|---|---|
| 分配器 | Allocator |
| 卷 | volume |
| vlet | vlet (retain original identifier) |
| bid | blob identifier |
| vid | volume identifier |
| shard | shard |
| 存储规格 | storage_spec |
| 存储类型 | storage_class |
| 空间 | space |
| 副本组(MV) | MV (multi-version group) |
| 写仲裁 | put_quorum |
| 哈希桶 | hash-bucket |
| 持久化转储 | dump |
| 健康度 | healthy_value |
| 比例随机算法 | ratio-random algorithm |
| 多级别分配队列 | multi-level allocation queue |
| 离线/下架 | offline |
| 关写 | write-disabled |
| 开写 | write-enabled |
| 致命告警 | fatal alert |
| 编码模式 | coding schema / erasure-coding schema |
| 待更新队列 | pending-update queue |

> **Note**: All internal project identifiers (Allocator, VolumeService, BlobNode, Cayman, Center, OSS, Tinkerman, etc.) are preserved as original identifiers; protobuf-style message definitions keep original naming and data types; mathematical formula descriptions in original document are retained in text description form.
