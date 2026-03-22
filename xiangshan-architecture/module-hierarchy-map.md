# XiangShan Module Hierarchy Map

## Frontend Module Tree

```
Frontend (frontend/Frontend.scala)
├── BPU (bpu/Bpu.scala, 29KB)
│   ├── Composer ── Predictor pipeline integration
│   ├── FTB (bpu/abtb/, bpu/mbtb/, bpu/ubtb/)
│   │   ├── ABTB ── Advanced BTB
│   │   ├── MBTB ── Main BTB
│   │   └── uBTB ── Micro BTB
│   ├── TAGE (bpu/tage/) ── Tagged geometric history
│   ├── SC (bpu/sc/) ── Statistical Corrector
│   ├── ITTage (bpu/ittage/) ── Indirect Target TAGE
│   ├── RAS (bpu/ras/) ── Return Address Stack
│   └── History (bpu/history/) ── Branch history management
├── FTQ (ftq/Ftq.scala, 23KB)
│   ├── EntryQueue, CommitQueue, ResolveQueue
│   ├── SpeculationQueue, CfiQueue, MetaQueue
│   └── FtqPtr, FtqPtrVec ── Queue pointer management
├── IFU (ifu/) ── Instruction fetch unit
├── ICache (icache/)
│   ├── ICacheMainPipe.scala (21KB) ── Main fetch pipe
│   ├── ICachePrefetchPipe.scala (22KB) ── Prefetch
│   ├── ICacheMissUnit.scala (16KB) ── Miss handling
│   ├── ICacheMshr.scala ── Miss Status Holding Register
│   └── ICacheReplacer.scala ── Replacement policy
├── IBuffer (ibuffer/) ── Instruction buffer
└── InstrUncache (instruncache/) ── Micro-op cache
```

## Backend Module Tree

```
Backend (backend/Backend.scala, 34KB)
├── CtrlBlock (ctrlblock/CtrlBlock.scala, 47KB)
│   ├── DecodeStage (decode/DecodeStage.scala, 17KB)
│   │   ├── DecodeUnit (decode/DecodeUnit.scala, 74KB) ── Main decoder
│   │   ├── DecodeUnitComp (decode/DecodeUnitComp.scala, 87KB) ── Complex decode
│   │   ├── FusionDecoder (decode/FusionDecoder.scala, 31KB) ── Instruction fusion
│   │   └── VecDecoder (decode/VecDecoder.scala, 62KB) ── Vector decode
│   ├── Rename (rename/Rename.scala, 49KB)
│   │   ├── RenameTable (rename/RenameTable.scala, 19KB)
│   │   ├── BusyTable (rename/BusyTable.scala, 13KB)
│   │   └── FreeList (rename/freelist/)
│   ├── Dispatch (dispatch/NewDispatch.scala, 50KB)
│   ├── ROB (rob/Rob.scala, 88KB)
│   │   ├── Rab (rob/Rab.scala, 13KB)
│   │   ├── ExceptionGen (rob/ExceptionGen.scala)
│   │   ├── VTypeBuffer (rob/VTypeBuffer.scala, 16KB)
│   │   └── RobBundles (rob/RobBundles.scala)
│   └── RedirectGenerator (ctrlblock/RedirectGenerator.scala)
├── IssueQueues (issue/IssueQueue.scala, 67KB)
│   ├── Entries (issue/Entries.scala, 38KB)
│   ├── EntryBundles (issue/EntryBundles.scala, 41KB)
│   ├── AgeDetector / NewAgeDetector
│   ├── EnqPolicy / DeqPolicy
│   └── MultiWakeupQueue
├── ExeUnits (exu/)
│   ├── ExeUnit (exu/ExeUnit.scala, 26KB) ── Execution unit wrapper
│   ├── ExuBlock (exu/ExuBlock.scala) ── Block grouping
│   └── FunctionalUnits (fu/)
│       ├── ALU, BKU (Bit/Crypto)
│       ├── MUL, SRT16Divider
│       ├── Branch, Jump, CSR, Fence
│       ├── fpu/ ── Floating point
│       ├── vector/ ── Vector (yunsuan integration)
│       └── PMA, PMP ── Memory attributes/protection
├── DataPath (datapath/DataPath.scala, 47KB)
│   ├── BypassNetwork (datapath/BypassNetwork.scala, 16KB)
│   ├── WbArbiter (datapath/WbArbiter.scala, 20KB)
│   ├── RFReadArbiter (datapath/RFReadArbiter.scala, 11KB)
│   └── WbFuBusyTable (datapath/WbFuBusyTable.scala, 11KB)
└── RegFile (regfile/Regfile.scala, 22KB)
```

## MemBlock Module Tree

```
MemBlock (mem/MemBlock.scala, 72KB)
├── LoadUnit (pipeline/) ── Load pipeline (3 instances)
├── StoreUnit (pipeline/) ── Store pipeline (2 instances)
├── LSQWrapper (lsqueue/LSQWrapper.scala, 19KB)
│   ├── LoadQueue (lsqueue/LoadQueue.scala, 13KB)
│   │   ├── LoadQueueData (9KB)
│   │   ├── LoadQueueRAR (11KB) ── Read-After-Read
│   │   ├── LoadQueueRAW (17KB) ── Read-After-Write
│   │   ├── LoadQueueReplay (44KB) ── Replay logic
│   │   └── LoadQueueUncache (24KB)
│   └── NewStoreQueue (lsqueue/NewStoreQueue.scala, 101KB)
├── Sbuffer (sbuffer/Sbuffer.scala, 45KB)
│   └── StorePrefetchBursts (11KB)
├── Prefetchers (prefetch/)
│   ├── L1PrefetchComponent (41KB)
│   ├── SMSPrefetcher (60KB) ── Spatial Memory Streaming
│   ├── Berti (38KB) ── BERTI prefetcher
│   ├── L1StreamPrefetcher (21KB)
│   ├── L1StridePrefetcher (10KB)
│   └── FDP (7KB) ── Feedback Directed Prefetch
├── DCache (cache/dcache/)
│   ├── DCacheWrapper (68KB) ── Main implementation
│   ├── MainPipe (mainpipe/)
│   ├── LoadPipe (loadpipe/)
│   ├── StorePipe (storepipe/)
│   ├── Meta (meta/) ── Metadata
│   ├── Data (data/) ── Data array
│   └── Uncache (23KB) ── Non-cacheable
├── MMU (cache/mmu/)
│   ├── TLB (TLB.scala, 38KB) ── ITLB + DTLB
│   ├── TLBStorage (17KB)
│   ├── L2TLB (L2TLB.scala, 53KB) ── Two-level TLB
│   ├── PageTableWalker (PTW, 62KB)
│   ├── PageTableCache (78KB) ── Page table cache
│   ├── MMUBundle (59KB)
│   └── Repeater (28KB)
└── Vector Memory (vector/)
    ├── VSegmentUnit (51KB)
    ├── VMergeBuffer (24KB)
    └── VSplit (25KB)
```

## Top-Level Integration

```
XSTop (top/Top.scala, 22KB)
├── XSNoCTop (top/XSNoCTop.scala, 23KB) ── NoC integration
├── XSTileWrap (xiangshan/XSTileWrap.scala, 10KB)
│   └── XSTile (xiangshan/XSTile.scala, 10KB)
│       ├── XSCore ── Processor core (see above)
│       └── L2Top (top/L2Top.scala, 17KB) ── L2 cache + L2 TLB
├── Configs (top/Configs.scala, 25KB) ── Configuration classes
├── ArgParser (top/ArgParser.scala, 12KB) ── CLI argument parsing
├── BusPerfMonitor (top/BusPerfMonitor.scala, 7KB)
└── YamlParser (top/YamlParser.scala, 8KB)
```

## Key Utils Files

| File | Size | Purpose |
|------|------|---------|
| `EnumUInt.scala` | 11KB | Enum utilities |
| `TLDump.scala` | 9KB | TileLink debug dump |
| `AXI4Lite.scala` | 4KB | AXI4-Lite protocol |
| `Duplicate.scala` | 7KB | Data duplication/distribution |
| `AddrField.scala` | 7KB | Address field management |
| `VecRotate.scala` | 4KB | Vector rotation operations |
| `BitsUtils.scala` | 2KB | Bit manipulation utilities |
| `PipeWithFlush.scala` | 1.4KB | Flushable pipeline |
| `LowPowerState.scala` | 2KB | Power management |
