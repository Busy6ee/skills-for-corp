---
name: xiangshan-memory
description: Use when working on XiangShan memory subsystem — MemBlock, LoadUnit, StoreUnit, LSQueue, Sbuffer, DCache, MMU (TLB/PTW), L2/L3 cache (huancun/coupledL2), or hardware prefetchers (SMS, Berti, Stride, Stream).
---

# XiangShan Memory — Memory Subsystem and Cache

## MemBlock Overview

MemBlock is the top-level module of XiangShan's memory subsystem.

```
Backend (MemIQ)
  ↓
MemBlock (mem/MemBlock.scala, 72KB)
  ├── LoadUnit × 3 ──→ DCache (load pipe)
  ├── StoreUnit × 2 ──→ StoreQueue → Sbuffer → DCache (store pipe)
  ├── LSQWrapper
  │   ├── LoadQueue (RAR, RAW, Replay, Uncache)
  │   └── StoreQueue
  ├── Sbuffer ──→ DCache (drain)
  ├── DCache ←→ L2Cache (TileLink)
  ├── DTLB ←→ L2TLB/PTW
  └── Prefetchers (SMS, Berti, Stride, Stream)
```

### Key Files

| Module | File | Size |
|--------|------|------|
| MemBlock | `mem/MemBlock.scala` | **72KB** |
| Bundles | `mem/Bundles.scala` | 21KB |
| LSQWrapper | `mem/lsqueue/LSQWrapper.scala` | 19KB |
| LoadQueue | `mem/lsqueue/LoadQueue.scala` | 13KB |
| LoadQueueReplay | `mem/lsqueue/LoadQueueReplay.scala` | **44KB** |
| NewStoreQueue | `mem/lsqueue/NewStoreQueue.scala` | **101KB** |
| Sbuffer | `mem/sbuffer/Sbuffer.scala` | 45KB |
| SMSPrefetcher | `mem/prefetch/SMSPrefetcher.scala` | 60KB |
| DCacheWrapper | `cache/dcache/DCacheWrapper.scala` | 68KB |
| TLB | `cache/mmu/TLB.scala` | 38KB |
| L2TLB | `cache/mmu/L2TLB.scala` | 53KB |
| PageTableWalker | `cache/mmu/PageTableWalker.scala` | 62KB |
| PageTableCache | `cache/mmu/PageTableCache.scala` | **78KB** |

## Load/Store Queue (LSQ)

### LoadQueue Structure

```
LoadQueue
├── LoadQueueRAR (11KB) ── Read-After-Read dependency tracking
├── LoadQueueRAW (17KB) ── Read-After-Write dependency (store→load forwarding)
├── LoadQueueReplay (44KB) ── Replay logic (retry after cache miss, TLB miss)
└── LoadQueueUncache (24KB) ── Non-cacheable load handling
```

### Memory Ordering

```
Store execution → Save address/data in StoreQueue (speculative)
Load execution → Search StoreQueue from LoadQueue (store-to-load forwarding)
  ├── Matching store found → Forward (store data → load result)
  ├── Partial match → Replay (retry later)
  └── No match → DCache access
```

### Store-to-Load Forwarding

```scala
// Search StoreQueue for stores to the same address
// Condition: store.addr == load.addr && store precedes load in program order
// Result: Forward store.data as the load result
```

> **Note**: Partial overlaps (e.g., 4-byte store, 8-byte load) cannot be forwarded → replay.

### Load Replay

A mechanism that retries loads when they fail due to cache misses, TLB misses, bank conflicts, etc.

```
Load failure causes:
  ├── DCache miss → MSHR allocation, replay after refill
  ├── TLB miss → Replay after PTW result
  ├── Bank conflict → Replay next cycle
  ├── Forwarding not possible → Replay later
  └── Memory dependency violation → Redirect + replay
```

## Store Buffer (Sbuffer)

### Role

Temporarily holds committed stores and merges them before draining to DCache.

```
ROB commit (store)
  → StoreQueue → Sbuffer
    → Merge (combine stores to the same cache line)
      → DCache drain (write to cache)
```

### Core Operation

```scala
// Sbuffer entry
class SbufferEntry extends Bundle {
  val tag    = UInt(tagBits.W)       // Cache line tag
  val data   = Vec(CacheLineBytes, UInt(8.W))  // Byte-granularity data
  val mask   = Vec(CacheLineBytes, Bool())      // Valid byte mask
}

// Merge: Combine stores with the same tag into a single entry
// Drain: Write the oldest/fullest entry to DCache
```

## DCache (L1 Data Cache)

### Structure

```
DCache
├── Tag Array ── Stores physical tags
├── Data Array ── Stores data per bank
├── MainPipe ── Handles refill, probe, replace
├── LoadPipe ── Load access pipeline
├── StorePipe ── Store access (from Sbuffer)
├── MSHR ── Miss Status Holding Register
├── ProbeQueue ── Coherence probe handling
└── Uncache ── Non-cacheable memory access
```

### Key Characteristics

| Characteristic | Value |
|----------------|-------|
| Indexing | **VIPT** (Virtual Index, Physical Tag) |
| Size | 64KB (default: 256 sets × 8 ways × 64B line) |
| Banking | Data array banking (parallel access) |
| Replacement policy | PLRU or Random |
| Coherence | TileLink (L1 ↔ L2) |

### MSHR (Miss Status Holding Register)

```
Cache miss occurs
  → Allocate MSHR entry (store miss address, request type)
    → Send refill request to L2 (TileLink Acquire)
      → L2 response (TileLink Grant)
        → Data refill → MSHR release → Load replay
```

### Probe Handling (Coherence)

```
Receive Probe from L2 (another core requests the same line)
  → Store in ProbeQueue
    → Check line state in DCache
      → Return data if needed + downgrade state
```

## MMU (Memory Management Unit)

### TLB Hierarchy

```
ITLB (instruction) ──┐
                      ├── L2TLB ── PTW (Page Table Walker)
DTLB (data) ─────────┘
                            ↓
                     PageTableCache (78KB implementation)
                            ↓
                     Memory (page table access)
```

### DTLB Structure

```scala
// TLB entry
class TLBEntry extends Bundle {
  val tag     = UInt(vpnBits.W)     // Virtual page number
  val ppn     = UInt(ppnBits.W)     // Physical page number
  val perm    = new TLBPermission   // Access permissions (R/W/X/U/A/D)
  val level   = UInt(2.W)           // Page size (4KB/2MB/1GB)
}
```

### Page Table Walker (PTW)

```
TLB miss
  → PTW request
    → PageTableCache lookup (intermediate level caching)
      ├── Hit → Return physical address
      └── Miss → Read page table from memory
           → Sv39: 3-level walk (L2 → L1 → L0)
           → Sv48: 4-level walk
```

### Sv39/Sv48 Page Table

```
Sv39 (default):
  VPN[2] → L2 PT → PTE
  VPN[1] → L1 PT → PTE
  VPN[0] → L0 PT → PTE (leaf) → PPN

  Huge page:
  VPN[2] → L2 PT → PTE (leaf, 1GB page)
  VPN[2,1] → L1 PT → PTE (leaf, 2MB page)
```

## Cache Coherence

### Protocol Hierarchy

```
L1 (DCache/ICache) ←── TileLink ──→ L2 (coupledL2)
                                        ←── CHI ──→ L3 (openLLC)
```

### TileLink (L1 ↔ L2)

| Channel | Direction | Purpose |
|---------|-----------|---------|
| A | L1 → L2 | Acquire (cache miss request) |
| B | L2 → L1 | Probe (downgrade/invalidation request) |
| C | L1 → L2 | Release (voluntary eviction) / ProbeAck |
| D | L2 → L1 | Grant (data response) |
| E | L1 → L2 | GrantAck (acknowledgment) |

### CHI (L2 ↔ L3)

The interface between coupledL2 and openLLC uses the **AMBA CHI** protocol.

> huancun (former L2) is TileLink-based. coupledL2 is CHI-based. This depends on the project configuration.

## Hardware Prefetchers

### Prefetcher Types

| Prefetcher | File | Size | Algorithm |
|------------|------|------|-----------|
| SMS | `SMSPrefetcher.scala` | 60KB | Spatial Memory Streaming — spatial pattern learning |
| Berti | `Berti.scala` | 38KB | BERTI — timing-based adaptive prefetch |
| Stream | `L1StreamPrefetcher.scala` | 21KB | Sequential stream detection |
| Stride | `L1StridePrefetcher.scala` | 10KB | PC-based stride detection |
| FDP | `FDP.scala` | 7KB | Feedback Directed — accuracy feedback-based throttling |

### Prefetch Request Injection

```
Prefetcher → L1PrefetchComponent (41KB)
  → Inject prefetch request into DCache
    → On cache miss, send prefetch request to L2
```

### Prefetch Accuracy Tracking

```scala
// PrefetcherMonitor (14KB)
// Tracks the ratio of prefetch requests vs. actual usage
// If accuracy is low, reduce aggressiveness; if high, increase aggressiveness
```

For detailed cache parameters and coherence state transitions, see [cache-hierarchy.md](cache-hierarchy.md).

## Memory Parameters (DSE)

| Parameter | Default | Impact |
|-----------|---------|--------|
| `LoadPipelineWidth` | 3 | Number of load pipes → memory-level parallelism |
| `StorePipelineWidth` | 2 | Number of store pipes |
| `LoadQueueReplaySize` | 72 | Replay queue → retry capacity after misses |
| `StoreQueueSize` | 56 | Store queue → speculative store capacity |
| `SbufferEntries` | 16 | Store buffer → merge/coalescing capacity |
| `DCacheSets` | 256 | Number of DCache sets |
| `DCacheWays` | 8 | DCache associativity |
| `EnableL1Prefetcher` | true | Enable L1 prefetcher |

## Gotchas

| Pitfall | Description |
|---------|-------------|
| `MemBlock.scala` = 72KB monolith | Navigate via class/def search. Reading the entire file is impractical |
| `NewStoreQueue.scala` = 101KB | Largest file in the project. Store queue logic is extremely complex |
| `PageTableCache` = 78KB | Large PTW cache implementation. Caches intermediate-level PTEs |
| TileLink ↔ CHI boundary | L1-L2 uses TileLink, L2-L3 uses CHI. Protocol conversion logic exists |
| Store-to-load forwarding timing | Forwarding path is timing-critical. Partial matches trigger replays |
| DCache bank conflicts | Multiple accesses to the same bank → conflict → performance degradation |
| Prefetch aggressiveness | Overly aggressive prefetching causes cache pollution. Parameter tuning required |
| huancun vs coupledL2 | Two L2 implementations exist. Current Kunminghu uses coupledL2 (CHI) |
| VIPT aliasing | DCache uses VIPT, so aliasing issues arise when cache size > way count × page size |
| Load replay stalls | Replay queue full → load pipe stall → overall performance degradation |
