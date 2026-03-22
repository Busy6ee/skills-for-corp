---
name: xiangshan-parameters
description: Use when configuring XiangShan core parameters, creating parameter configs, understanding the XSCoreParameters/HasXSParameter system, or performing Design Space Exploration (DSE) by varying ROB size, issue queue width, cache sizes, or BPU table dimensions.
---

# XiangShan Parameters — Parameter System and DSE

## Parameter Architecture

XiangShan builds its own parameter hierarchy on top of rocket-chip's `Parameters` system.

```
rocket-chip Parameters (implicit p: Parameters)
    └── XSCoreParameters (case class, 150+ fields)
        ├── Frontend parameters (FetchWidth, BPU tables, etc.)
        ├── Backend parameters (RobSize, IQ configs, etc.)
        ├── Memory parameters (LSQ sizes, pipeline counts, etc.)
        └── Cache parameters (DCache, ICache configs)
```

### Key Files

| File | Role |
|------|------|
| `src/main/scala/xiangshan/Parameters.scala` (~33KB) | `XSCoreParameters` case class, `HasXSParameter` trait |
| `src/main/scala/top/Configs.scala` (~25KB) | Config classes such as `DefaultConfig`, `MinimalConfig` |
| `src/main/scala/xiangshan/backend/BackendParams.scala` (~27KB) | Backend-specific parameters |
| `src/main/scala/xiangshan/frontend/FrontendParameters.scala` (~4KB) | Frontend-specific parameters |

## XSCoreParameters Key Fields

### Frontend

```scala
case class XSCoreParameters(
  // Fetch
  FetchWidth:         Int = 8,      // Instructions fetched per cycle
  PredictWidth:       Int = 8,      // Prediction width

  // BPU
  EnableBPU:          Boolean = true,
  FtbSize:            Int = 2048,   // FTB entry count (separate when using FauFTB)
  FtbWays:            Int = 4,      // FTB associativity
  FtqSize:            Int = 64,     // FTQ size
  RasSize:            Int = 16,     // RAS stack depth
  RasSpecSize:        Int = 32,     // Speculative RAS size

  // TAGE (default: Seq((4096,8,8), (4096,13,8), (4096,32,8), (4096,119,8)))
  TageTableInfos:     Seq[Tuple3[Int,Int,Int]], // (nRows, histLen, tagLen)
  SCTableInfos:       Seq[Tuple3[Int,Int,Int]], // SC tables
  // ITTage (default: Seq((256,4,9), (256,8,9), (512,13,9), (512,16,9), (512,32,9)))
  ITTageTableInfos:   Seq[Tuple3[Int,Int,Int]], // ITTage tables

  // ICache
  ICacheECCForceError: Boolean = false,
  // ...
)
```

### Backend

```scala
  // ROB
  RobSize:            Int = 160,    // ROB entry count
  RabSize:            Int = 256,    // Rename Allocation Buffer
  CommitWidth:        Int = 8,      // Commit width per cycle
  RenameWidth:        Int = 6,      // Rename width per cycle

  // Physical registers (PregParams-based)
  intPreg:            IntPregParams = IntPregParams(numEntries = 224),
  fpPreg:             FpPregParams = FpPregParams(numEntries = 192),
  vfPreg:             VfPregParams = VfPregParams(numEntries = 128),
  v0Preg:             V0PregParams = V0PregParams(numEntries = 22),
  vlPreg:             VlPregParams = VlPregParams(numEntries = 32),

  // Execution units
  IntExuNum:          Int,          // Number of integer execution units
  FpExuNum:           Int,          // Number of FP execution units
  // ...
```

### Memory

```scala
  // Load/Store pipelines
  LoadPipelineWidth:  Int = 3,      // Number of load pipes
  StorePipelineWidth: Int = 2,      // Number of store pipes

  // LSQ
  LoadQueueRARSize:   Int = 72,     // RAR queue size
  LoadQueueRAWSize:   Int = 64,     // RAW queue size
  LoadQueueReplaySize: Int = 72,    // Replay queue size
  StoreQueueSize:     Int = 56,     // Store queue size

  // Sbuffer
  SbufferEntries:     Int = 16,     // Store buffer entries

  // Prefetch
  EnableL1Prefetcher: Boolean = true,
  // ...
```

### Cache

```scala
  // DCache
  DCacheSets:         Int = 256,    // Number of DCache sets
  DCacheWays:         Int = 8,      // DCache associativity
  DCacheLineBytes:    Int = 64,     // Cache line size

  // L2
  L2NWays:            Int = 8,
  L2NSets:            Int = 1024,
  // ...
```

## HasXSParameter Trait

The standard way to access parameters from any XiangShan module:

```scala
// Definition (Parameters.scala)
trait HasXSParameter {
  implicit val p: Parameters
  val coreParams = p(XSCoreParamsKey)

  // Derived parameters
  val PAddrBits = coreParams.PAddrBits
  val VAddrBits = coreParams.VAddrBits
  val XLEN = coreParams.XLEN  // 64
  // ... dozens of derived vals
}

// Usage
class MyModule(implicit val p: Parameters) extends Module with HasXSParameter {
  // Direct access to PAddrBits, VAddrBits, XLEN, etc.
  val io = IO(new Bundle {
    val addr = Input(UInt(PAddrBits.W))
  })
}
```

### XSModule / XSBundle Base Classes

```scala
// Base Module with built-in parameters
abstract class XSModule(implicit val p: Parameters) extends Module
  with HasXSParameter

// Base Bundle with built-in parameters
abstract class XSBundle(implicit val p: Parameters) extends Bundle
  with HasXSParameter
```

## Config System

### Config Definition (`top/Configs.scala`)

```scala
// Default config — full feature set
class DefaultConfig(n: Int = 1) extends Config(
  L3CacheConfig("16MB", inclusive = false, banks = 4, ways = 16) ++
  L2CacheConfig("1MB", inclusive = true, banks = 4) ++
  WithNKBL1D(64, ways = 4) ++
  new BaseConfig(n)
)

// Minimal config — for fast simulation iteration
// RobSize=48, IssueQueueSize=10, FtqSize=8, reduced caches
class MinimalConfig(n: Int = 1) extends Config(...)

// Kunminghu V2 specific (CHI protocol)
class KunminghuV2Config(n: Int = 1) extends Config(
  new WithCHI ++ DefaultConfig(n)  // Enable CHI protocol
)
```

### Config Composition Pattern

```scala
// Config fragment
class WithNKBL2(n: Int) extends Config((site, here, up) => {
  case L2ParamKey => up(L2ParamKey).copy(sets = n * 1024 / 8 / 64)
})

// Composition: chain configs with the ++ operator
new WithCustomROB(256) ++ new DefaultConfig
// → Applied right-to-left (DefaultConfig first, WithCustomROB overrides)
```

### Selecting a Config at Build Time

```bash
# Select via the CONFIG variable in the Makefile
make emu CONFIG=DefaultConfig    # Default
make emu CONFIG=MinimalConfig    # Minimal (fast build)
```

## DSE (Design Space Exploration) Key Parameters

### IPC Exploration

| Parameter | Range | Impact |
|-----------|-------|--------|
| `RobSize` | 64–320 | Instruction window size → ILP utilization |
| `FtbSize` / `FtbWays` | 512–4096 / 2–8 | Branch prediction accuracy |
| `LoadQueueReplaySize` | 32–128 | Memory-level parallelism |
| `IntPhyRegs` / `FpPhyRegs` | 96–384 | Available rename registers |
| Issue queue entry count | 8–32 per IQ | Scheduling window |

### Area Exploration

| Parameter | Impact |
|-----------|--------|
| `DCacheSets` × `DCacheWays` | L1D capacity → area |
| `L2NSets` × `L2NWays` | L2 capacity → dominant area contributor |
| `RobSize` | ROB storage |
| BPU table sizes | TAGE/FTB SRAM area |

### Power Exploration

| Parameter | Impact |
|-----------|--------|
| `FetchWidth` / `PredictWidth` | Frontend concurrent activation width |
| `CommitWidth` | Commit logic width |
| `LoadPipelineWidth` / `StorePipelineWidth` | Number of memory pipes |

## Procedure for Adding New Parameters

1. Add a field to `XSCoreParameters` in `Parameters.scala` (default value required)
2. If needed, add a derived val in `HasXSParameter`
3. Access from the target module via `coreParams.newParam` or the derived val
4. Optionally add a Config fragment for the parameter in `Configs.scala`
5. Verify reasonable default values in both `MinimalConfig` and `DefaultConfig`

## Gotchas

| Pitfall | Description |
|---------|-------------|
| PhyRegs constraint | `IntPhyRegs` must be ≥ logical registers (32) + RobSize. Insufficient count causes rename stalls |
| FetchWidth cascading effect | Changing FetchWidth affects IBuffer, Decode, and Rename widths |
| Config application order | `++` composition applies **right-to-left**. The left side overrides the right side |
| Diplomacy vs. XS parameters | rocket-chip Diplomacy parameters (`LazyModule`) and XSCoreParameters are separate systems |
| implicit Parameters | Missing `implicit val p: Parameters` scope causes compile errors. Must be propagated to all modules |
| MinimalConfig limitations | MinimalConfig uses extremely small caches, producing unrealistic IPC on real workloads |
| Derived parameter dependencies | Some derived vals depend on other parameters in complex ways. Changing one parameter requires verifying the entire derived value chain |
| BackendParams | Backend-specific parameters are defined separately in `BackendParams.scala` — looking only at `Parameters.scala` is incomplete |
