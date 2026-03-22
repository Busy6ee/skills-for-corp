---
name: xiangshan-architecture
description: Use when navigating the XiangShan (香山) RISC-V processor codebase, understanding module hierarchy, locating source files, or tracing signal flow between Frontend/Backend/MemBlock/Cache subsystems. Also use when onboarding to XiangShan development.
---

# XiangShan Architecture — Module Hierarchy & Code Navigation

## Generations

| Generation | Name | Branch | Notes |
|------------|------|--------|-------|
| 1st | Yanqihu (雁栖湖) | master (legacy) | First stable version (2020) |
| 2nd | Nanhu (南湖) | nanhu | MICRO 2022 paper |
| 3rd | Kunminghu (昆明湖) | **kunminghu-v3** | **Active development** |

> The current default branch is `kunminghu-v3` (Kunminghu V3 generation). Earlier generation code resides on separate branches.

## Repository Layout

```
XiangShan/
├── src/main/scala/
│   ├── xiangshan/           # Core RTL
│   │   ├── frontend/        # BPU, IFU, ICache, FTQ, IBuffer
│   │   ├── backend/         # Decode, Rename, Dispatch, IQ, ROB, ExeUnit
│   │   │   ├── ctrlblock/   # Control block
│   │   │   ├── decode/      # Instruction decoding
│   │   │   ├── dispatch/    # Dispatch logic
│   │   │   ├── rename/      # Register renaming
│   │   │   ├── issue/       # Issue queues
│   │   │   ├── exu/         # Execution units
│   │   │   ├── fu/          # Functional units (ALU, FPU, etc.)
│   │   │   ├── rob/         # Reorder Buffer
│   │   │   ├── datapath/    # Datapath, bypass
│   │   │   └── regfile/     # Register file
│   │   ├── mem/             # MemBlock, LSQ, Sbuffer, prefetchers
│   │   │   ├── lsqueue/     # Load-Store Queue
│   │   │   ├── sbuffer/     # Store Buffer
│   │   │   ├── prefetch/    # Hardware prefetchers
│   │   │   ├── pipeline/    # Memory pipeline
│   │   │   └── vector/      # Vector memory operations
│   │   ├── cache/           # DCache, MMU (TLB, PTW)
│   │   │   ├── dcache/      # Data cache
│   │   │   ├── mmu/         # Memory management unit
│   │   │   └── wpu/         # Write/Prefetch Unit
│   │   └── transforms/      # Chisel transformation utilities
│   ├── top/                 # SoC top-level, Config definitions
│   └── utils/               # Common utilities (24+ files)
├── build.mill               # Mill build configuration
├── Makefile                 # Simulation build wrapper
└── [submodules]             # External dependencies
```

## Submodules

| Submodule | Role | Location |
|-----------|------|----------|
| rocket-chip | RISC-V infrastructure, Diplomacy, TileLink | `rocket-chip/` |
| huancun | L2/L3 cache (TileLink-based) | `huancun/` |
| coupledL2 | Non-inclusive L2 cache (CHI protocol) | `coupledL2/` |
| openLLC | Last-Level Cache | `openLLC/` |
| yunsuan | Vector/floating-point arithmetic library | `yunsuan/` |
| difftest | ISA differential testing framework (NEMU integration) | `difftest/` |
| utility | Common hardware utilities | `utility/` |
| ChiselAIA | Advanced Interrupt Architecture | `ChiselAIA/` |

## Module Hierarchy

```
XSTop (src/main/scala/top/Top.scala)
└── XSTileWrap (XSTileWrap.scala)
    └── XSTile (XSTile.scala)
        ├── XSCore (XSCore.scala) ─────────────── Processor core
        │   ├── Frontend (frontend/Frontend.scala)
        │   │   ├── BPU ──── Branch prediction
        │   │   ├── FTQ ──── Fetch Target Queue
        │   │   ├── IFU ──── Instruction fetch
        │   │   ├── ICache ─ Instruction cache
        │   │   └── IBuffer ─ Instruction buffer
        │   ├── Backend (backend/Backend.scala)
        │   │   ├── CtrlBlock ── Decode/Rename/Dispatch/ROB
        │   │   ├── IssueQueue ─ Multiple issue queues (Int/FP/Mem/Vec)
        │   │   ├── ExeUnit ──── Execution units
        │   │   ├── DataPath ─── Datapath/bypass
        │   │   └── RegFile ──── Physical register file
        │   └── MemBlock (mem/MemBlock.scala)
        │       ├── LoadUnit ─── Load pipeline
        │       ├── StoreUnit ── Store pipeline
        │       ├── LSQueue ──── Load-Store Queue
        │       ├── Sbuffer ──── Store Buffer
        │       ├── DCache ───── L1 data cache
        │       ├── TLB ──────── DTLB + ITLB
        │       └── PTW ──────── Page Table Walker
        └── L2Top (L2Top.scala)
            ├── L2Cache ─── coupledL2 / huancun
            └── L2TLB ──── Two-level TLB
```

## Signal Flow Overview

### Frontend → Backend
```
BPU → FTQ → IFU → ICache → IBuffer → Decode
                                         ↓
                              Rename → Dispatch → IssueQueues
```
- FTQ **decouples** the BPU (prediction) from the IFU (fetch)
- IBuffer serves as a buffer between Frontend and Backend Decode
- When a redirect occurs in the Backend → it propagates back to Frontend (via FTQ)

### Backend → MemBlock
```
IssueQueue(Mem) → LoadUnit / StoreUnit
                       ↓
                  LSQueue ↔ DCache ↔ L2Cache
                       ↓
                    Sbuffer → DCache (drain)
```
- Memory instructions are dispatched from MemIQ
- On store commit from ROB → Sbuffer → DCache drain

### Cache Hierarchy
```
ICache (L1I) ──┐
               ├── L2Cache ── L3Cache (openLLC)
DCache (L1D) ──┘
    ↑
   TLB ← PTW (L2TLB)
```
- L1 ↔ L2: **TileLink** protocol
- L2 ↔ L3: **CHI** protocol (when using coupledL2)

## Key Entry-Point Files

| File | Role | Size |
|------|------|------|
| `XSCore.scala` | Instantiation and interconnection of Frontend + Backend + MemBlock | ~13KB |
| `XSTile.scala` | Core + L2 tile composition | ~10KB |
| `Top.scala` | SoC top-level module, multi-core composition | ~22KB |
| `Parameters.scala` | Complete parameter system (150+ fields) | ~33KB |
| `Bundle.scala` | Cross-module bundle definitions | ~26KB |
| `Configs.scala` | Config classes (Default, Minimal, etc.) | ~25KB |
| `package.scala` | Package-level utilities, implicits | ~39KB |

## Subsystem Quick Reference

| Subsystem | Main File | Bundles | Parameters | Key Subdirectories |
|-----------|----------|---------|------------|-------------------|
| Frontend | `frontend/Frontend.scala` (15KB) | `frontend/Bundles.scala` (19KB) | `frontend/FrontendParameters.scala` (4KB) | `bpu/`, `ftq/`, `ifu/`, `icache/`, `ibuffer/` |
| Backend | `backend/Backend.scala` (34KB) | `backend/Bundles.scala` (80KB) | `backend/BackendParams.scala` (27KB) | `decode/`, `rename/`, `dispatch/`, `issue/`, `exu/`, `fu/`, `rob/`, `datapath/` |
| MemBlock | `mem/MemBlock.scala` (72KB) | `mem/Bundles.scala` (21KB) | (in Parameters.scala) | `lsqueue/`, `sbuffer/`, `prefetch/`, `pipeline/`, `vector/` |
| Cache | `cache/L1Cache.scala` (4KB) | (in xiangshan Bundle.scala) | (in Parameters.scala) | `dcache/`, `mmu/`, `wpu/` |

For the detailed module tree, see [module-hierarchy-map.md](module-hierarchy-map.md).

## Gotchas

| Pitfall | Description |
|---------|-------------|
| `backend/Bundles.scala` = 80KB | All Backend bundle definitions in a single file. Grep by class name when searching |
| `mem/MemBlock.scala` = 72KB | Monolithic file. Contains both load/store pipelines and cache connections |
| `rob/Rob.scala` = 87KB | ROB is also an oversized single file |
| `xiangshan/` vs `top/` | Core RTL lives in `xiangshan/`, SoC integration and Configs live in `top/` — avoid confusion |
| Submodule versions | Submodules are pinned to specific commits. `git submodule update --init` is required |
| `package.scala` = 39KB | Contains many implicit conversions and utility functions. Loaded entirely via `import xiangshan._` |
| Mixed generations | Default branch is kunminghu-v3, but some Nanhu legacy code may still exist |
| Chisel version | Uses **Chisel 7.3.0** + Scala 2.13.17. There may be API differences compared to the chisel-book skill (6.5.0) |
| LazyModule pattern | Frontend/Backend/MemBlock are all **LazyModule**-based — Diplomacy connections are required |
