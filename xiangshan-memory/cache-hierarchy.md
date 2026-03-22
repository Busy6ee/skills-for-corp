# XiangShan Cache Hierarchy Details

## DCache Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| `DCacheSets` | 256 | Number of sets |
| `DCacheWays` | 8 | Associativity |
| `DCacheLineBytes` | 64 | Cache line size (bytes) |
| **Total capacity** | **128KB** | 256 × 8 × 64B |
| Indexing | VIPT | Virtual Index, Physical Tag |
| Data bank count | Variable | For parallel access |
| MSHR count | Variable | Number of concurrent miss handlers |

## ICache Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| `ICacheSets` | 256 | Number of sets |
| `ICacheWays` | 8 | Associativity |
| `ICacheLineBytes` | 64 | Cache line size |
| **Total capacity** | **128KB** | 256 × 8 × 64B |
| Pipeline | MainPipe + PrefetchPipe | 2 pipes |

## L2 Cache (coupledL2)

| Parameter | Default (DefaultConfig) | Description |
|-----------|------------------------|-------------|
| `L2NSets` | 1024 | Number of sets |
| `L2NWays` | 8 | Associativity |
| Line size | 64B | Cache line |
| **Total capacity** | **512KB** | 1024 × 8 × 64B |
| Protocol (upstream) | TileLink | L1 ↔ L2 |
| Protocol (downstream) | CHI | L2 ↔ L3 |
| Policy | Non-inclusive | |

## L3 Cache (openLLC)

| Parameter | Default | Description |
|-----------|---------|-------------|
| Total capacity | 4MB (DefaultConfig) | |
| Protocol | CHI | |
| Policy | Non-inclusive | |

## MinimalConfig Cache Sizes

| Cache | DefaultConfig | MinimalConfig |
|-------|---------------|---------------|
| L1D | 128KB | 64KB |
| L1I | 128KB | 64KB |
| L2 | 512KB | 64KB |
| L3 | 4MB | 256KB |

> MinimalConfig is optimized for simulation speed. Caches are extremely small → IPC is unrealistic.

## TileLink Coherence State Transitions

### MOESI-like States (TileLink-based)

```
State    | Description
---------|---------------------------
None     | Not present in cache
Branch   | Clean, read-only (shared)
Trunk    | Clean, exclusive
Tip      | Dirty, exclusive (modified)
```

### L1 → L2 Transition Examples

```
[Load Miss]
None → Acquire(NtoB) → L2 Grant → Branch

[Store Miss]
None → Acquire(NtoT) → L2 Grant → Tip

[Store to Branch line]
Branch → Acquire(BtoT) → L2 Grant → Tip

[Probe from L2]
Tip → ProbeAck(data) → Branch or None
Branch → ProbeAck → None
```

### CHI Coherence (L2 ↔ L3)

The CHI protocol uses a more fine-grained state machine than TileLink.

```
Key states:
  I  (Invalid)
  SC (Shared Clean)
  SD (Shared Dirty)
  UC (Unique Clean)
  UD (Unique Dirty)

Key transactions:
  ReadShared    → SC
  ReadUnique    → UC/UD
  CleanUnique   → UC (SC→UC upgrade)
  WriteBackFull → I (dirty data eviction)
  Evict         → I (clean data eviction)
  SnpShared     → SC (downgrade)
  SnpUnique     → I (invalidation)
```

## Prefetcher Configuration Parameters

### SMS (Spatial Memory Streaming)

| Parameter | Impact |
|-----------|--------|
| Active Generation Table size | Number of spatial patterns that can be tracked concurrently |
| Pattern History Table size | Storage capacity for learned patterns |
| Spatial region size | Prefetch range (number of cache lines) |
| Confidence threshold | Condition for issuing prefetches |

### Berti

| Parameter | Impact |
|-----------|--------|
| History Table size | Per-PC access history storage |
| Timing Table size | Access timing pattern storage |
| Degree (aggressiveness) | Number of concurrent prefetch requests |

### Stream

| Parameter | Impact |
|-----------|--------|
| Stream detection window | Range for detecting sequential accesses |
| Prefetch distance | Prefetch distance relative to current access |
| Max active streams | Number of streams that can be tracked concurrently |

### Stride

| Parameter | Impact |
|-----------|--------|
| RPT (Reference Prediction Table) size | Per-PC stride storage |
| Confidence threshold | Start prefetching after stride confirmation |

## Memory Access Latency (Approximate)

| Access | Latency (cycles) |
|--------|-------------------|
| L1 DCache hit | 3-4 |
| L1 DCache miss → L2 hit | 10-15 |
| L2 miss → L3 hit | 30-50 |
| L3 miss → DRAM | 100-200+ |
| TLB hit | 0 (included in pipeline) |
| TLB miss → L2TLB hit | 5-10 |
| L2TLB miss → PTW | 20-100+ (multi-level walk) |

> Latencies vary depending on the Config and simulation environment. More accurate with DRAMSim3.
