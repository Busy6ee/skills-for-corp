---
name: xiangshan-frontend
description: Use when working on XiangShan frontend — BPU (TAGE, ITTage, SC, RAS, FTB), IFU, ICache, FTQ, or IBuffer. Use when modifying branch prediction, fetch pipeline, or instruction buffer logic.
---

# XiangShan Frontend — Branch Prediction and Instruction Fetch

## Frontend Pipeline Overview

```
                    ┌─────────── redirect (from Backend) ───────────┐
                    ↓                                                │
BPU ──→ FTQ ──→ IFU ──→ ICache ──→ IBuffer ──→ Decode (Backend)
 ↑        │       │
 └────────┘       │
 (BPU update)     └── TLB (ITLB)
```

### Key Files

| Module | File | Size | Role |
|--------|------|------|------|
| Frontend | `frontend/Frontend.scala` | 15KB | Top-level integration |
| BPU | `frontend/bpu/Bpu.scala` | 29KB | Branch prediction main |
| FTQ | `frontend/ftq/Ftq.scala` | 23KB | Fetch Target Queue |
| IFU | `frontend/ifu/` | - | Instruction Fetch Unit |
| ICache | `frontend/icache/ICacheMainPipe.scala` | 21KB | Instruction cache main pipe |
| IBuffer | `frontend/ibuffer/` | - | Instruction buffer |
| Bundles | `frontend/Bundles.scala` | 19KB | Frontend shared bundles |
| Params | `frontend/FrontendParameters.scala` | 4KB | Frontend parameters |

## BPU (Branch Prediction Unit)

### 3-Stage Pipeline Structure

```
s1 (1st cycle)     s2 (2nd cycle)     s3 (3rd cycle)
 │                  │                  │
 uBTB/FTB lookup   TAGE prediction    SC correction
 Generate base      ITTage lookup      Finalize prediction
 prediction         RAS operation
```

- **s1**: Fast prediction (FauFTB, FTB lookup) — generates a target every cycle
- **s2**: Precise prediction (TAGE, ITTage) — can override s1 prediction
- **s3**: Statistical correction (SC) — can override s2 prediction

### Predictor Components

| Predictor | Role | Table Structure |
|-----------|------|-----------------|
| **FTB** (Fetch Target Buffer) | Stores branch info per fetch block | 2048 entries, 4-way |
| **FauFTB** (Fast uFTB) | Fast 1-cycle prediction (acts as uBTB) | Direct-mapped, small table |
| **TAGE** | Conditional branch direction prediction | Tagged geometric history, multiple tables |
| **SC** (Statistical Corrector) | Corrects TAGE predictions | Auxiliary tables for TAGE |
| **ITTage** (Indirect Target TAGE) | Indirect branch target prediction | Similar to TAGE, stores target addresses |
| **RAS** (Return Address Stack) | Function return address prediction | Stack structure (16 + 32 speculative) |

### FTB (Fetch Target Buffer) Entry

```scala
class FTBEntry(implicit p: Parameters) extends XSBundle {
  val valid       = Bool()
  val brSlots     = Vec(numBrSlot, new FTBSlot)  // branch slots
  val tailSlot    = new FTBSlot                    // last slot (jmp/br)
  val pftAddr     = UInt(...)                      // partial fall-through address
  val carry       = Bool()                         // carry bit
  val isCall      = Bool()                         // whether it is a call instruction
  val isRet       = Bool()                         // whether it is a ret instruction
  val isJalr      = Bool()                         // whether it is a jalr instruction
  val last_may_be_rvi_call = Bool()
}

class FTBSlot extends Bundle {
  val offset    = UInt(...)   // offset within fetch block
  val lower     = UInt(...)   // lower bits of target address
  val tarStat   = UInt(...)   // target status
  val sharing   = Bool()      // sharing flag
  val valid     = Bool()
}
```

> The FTB operates at the **fetch block** granularity, not per individual branch. A single FTB entry describes multiple branches within one fetch block.

### BPU Update Path

```
Backend (redirect/commit)
  → FTQ (collects update information)
    → BPU (updates each predictor table)
```

For detailed TAGE/SC/ITTage/RAS structures, see [bpu-components.md](bpu-components.md).

## FTQ (Fetch Target Queue)

The key queue that **decouples** the BPU (prediction) from the IFU (fetch).

### Roles

1. Stores BPU prediction results in the queue
2. IFU dequeues entries one at a time to generate fetch requests
3. Flushes the queue upon receiving a redirect from the Backend
4. Collects BPU update information at commit time and forwards it to the BPU

### Internal Queue Structure

```
FTQ
├── EntryQueue ──── Stores BPU prediction entries
├── CommitQueue ─── Entries awaiting commit
├── ResolveQueue ── Entries awaiting branch resolution
├── SpeculationQueue ── Manages speculative state
├── CfiQueue ───── Control flow instruction information
└── MetaQueue ──── BPU metadata (for updates)
```

### FtqPtr

```scala
class FtqPtr extends CircularQueuePtr[FtqPtr] {
  // Pointer indicating a position within the FTQ
  // Circular queue pointer, similar to robIdx
}
```

## IFU (Instruction Fetch Unit)

### Fetch Request Flow

```
FTQ → IFU fetch request (pc, prediction info)
  → ITLB lookup (virtual → physical address translation)
    → ICache access (cache lookup using physical address)
      → Fetch result → IBuffer
```

### ICache Structure

- **VIPT** (Virtual Index, Physical Tag): Indexed by virtual address, compared by physical tag
- **Multi-pipe**: MainPipe (fetch) + PrefetchPipe (prefetch)
- **Banked structure**: Banking for parallel access
- **MSHR**: Miss Status Holding Registers — manages L2 requests

```
ICache
├── ICacheMainPipe (21KB) ── Main fetch pipeline
├── ICachePrefetchPipe (22KB) ── Prefetch pipeline
├── ICacheMissUnit (16KB) ── Miss handling
├── ICacheMshr ── MSHR management
├── ICacheReplacer ── Replacement policy
├── Data Array ── Data storage per bank
└── Meta Array ── Tag + valid bits
```

## IBuffer (Instruction Buffer)

A FIFO buffer between the Frontend and Backend (Decode).

### Roles
- **Absorbs speed differences**: Frontend fetch rate ≠ Backend decode rate
- **Flush handling**: Removes speculative instructions from the buffer on redirect
- **Width conversion**: FetchWidth → DecodeWidth (usually the same, but provides flexibility)

### Flow Control

```scala
// IBuffer → Decode (Backend)
val io = IO(new Bundle {
  val in  = Flipped(DecoupledIO(new FetchPacket))  // from Frontend
  val out = Vec(DecodeWidth, DecoupledIO(new CtrlFlow))  // to Decode
  val redirect = Flipped(ValidIO(new Redirect))     // flush signal
})
```

## Frontend Parameters (DSE-Related)

| Parameter | Default | DSE Impact |
|-----------|---------|------------|
| `FetchWidth` | 8 | Fetch width per cycle → Frontend throughput |
| `PredictWidth` | 8 | Prediction width |
| `FtbSize` | 2048 | Number of FTB entries → Branch prediction coverage |
| `FtbWays` | 4 | FTB associativity → Reduces conflict misses |
| `FtqSize` | 64 | FTQ depth → Degree of BPU-IFU decoupling |
| `RasSize` | 16 | RAS depth → Supported function call depth |
| TAGE table sizes | (variable) | Critical for branch prediction accuracy |

## Gotchas

| Pitfall | Description |
|---------|-------------|
| BPU is speculative | BPU predictions can always be wrong. All BPU results can be invalidated by a redirect |
| FTB ≠ BTB | FTB operates at fetch block granularity (contains multiple branches), BTB operates per individual branch. Do not confuse them |
| TAGE tables = DSE critical | TAGE table size, count, and history lengths have a major impact on IPC |
| ICache VIPT | Virtual indexing → potential aliasing issues (when cache size > page size × number of ways) |
| Redirect priority | Backend redirect > Frontend internal redirect. Backend takes priority on simultaneous occurrence |
| FTQ full | FTQ full → BPU prediction stalls → Frontend stall. FtqSize affects performance |
| BPU update latency | BPU table updates occur at commit time. Long latency until learning takes effect |
| ICache prefetch | Prefetch pipe can contend for resources with the main pipe |
