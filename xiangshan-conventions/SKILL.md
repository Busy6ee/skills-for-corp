---
name: xiangshan-conventions
description: Use when writing or modifying XiangShan Chisel code to follow project naming conventions, pipeline stage patterns, Bundle/IO definitions, ValidIO/DecoupledIO usage, redirect/flush propagation, and code organization rules.
---

# XiangShan Conventions — Coding Conventions and Design Patterns

## Naming Rules

| Target | Rule | Example |
|--------|------|---------|
| Module class | UpperCamelCase | `DecodeUnit`, `IssueQueue`, `LoadQueueReplay` |
| IO bundle class | `*IO` suffix | `FrontendIO`, `BackendIO`, `MemBlockIO` |
| Data bundle | `*Bundle` suffix | `DifftestBundle`, `RedirectBundle` |
| Signal/variable | lowerCamelCase | `robIdx`, `ftqPtr`, `loadValid` |
| Constant/enum | UpperCamelCase | `FuType.alu`, `SrcType.reg` |
| Parameter field | UpperCamelCase | `RobSize`, `FetchWidth`, `IntPhyRegs` |
| Filename | CamelCase.scala | `DecodeUnit.scala`, `IssueQueue.scala` |

### Cross-Module IO Naming

```scala
class BackendIO extends Bundle {
  val fromFrontend = Flipped(new FrontendToBackendIO)  // receive port
  val toFrontend   = new BackendToFrontendIO           // send port
  val fromMemBlock = Flipped(new MemBlockToBackendIO)
  val toMemBlock   = new BackendToMemBlockIO
}
```

> Pattern: `fromXxx` = receive (Flipped), `toXxx` = send

## Module Base Classes

```scala
// Base XiangShan module — automatically includes HasXSParameter
abstract class XSModule(implicit val p: Parameters) extends Module
  with HasXSParameter

// Base XiangShan bundle
abstract class XSBundle(implicit val p: Parameters) extends Bundle
  with HasXSParameter

// Usage example
class MyNewModule(implicit p: Parameters) extends XSModule {
  // PAddrBits, XLEN, RobSize, etc. are directly accessible
}
```

### Note: Choosing the Right Base Class

| Class | Purpose |
|-------|---------|
| `XSModule` | General module that requires XiangShan parameters |
| `XSCoreBase` extends `LazyModule` | Core-level modules (Frontend, Backend, MemBlock) — for Diplomacy connections |
| `LazyModule` | Modules requiring TileLink/Diplomacy nodes |
| `Module` (plain) | Pure Chisel modules that do not need parameters, utilities |

### LazyModule Pattern (Diplomacy)

Top-level subsystems are wrapped in `LazyModule` to perform TileLink connections.

```scala
// Definition pattern
class Backend(val params: BackendParams)(implicit p: Parameters) extends LazyModule
  with HasXSParameter {
  val inner = LazyModule(new BackendInlined(params))
  lazy val module = new BackendImp(this)
}

class BackendImp(wrapper: Backend)(implicit p: Parameters) extends LazyModuleImp(wrapper) {
  val io = IO(new BackendIO)
  io <> wrapper.inner.module.io
}

// Connection pattern (inside XSCoreImp)
frontend.io.backend <> backend.io.frontend
frontend.io.sfence <> backend.io.frontendSfence
backend.io.mem.lsqEnqIO <> memBlock.io.ooo_to_mem.enqLsq
memBlock.io.redirect := backend.io.mem.redirect
```

## Core Data Type: DynInst

`DynInst` is the **core instruction bundle** that flows through the entire pipeline.

```scala
class DynInst(implicit p: Parameters) extends XSBundle {
  // Instruction information
  val instr     = UInt(32.W)         // original instruction
  val pc        = UInt(VAddrBits.W)  // PC
  val foldpc    = UInt(MemPredPCWidth.W)

  // Decode results
  val fuType    = FuType()           // functional unit type
  val fuOpType  = FuOpType()         // operation type
  val srcType   = Vec(MaxSrcNum, SrcType())  // source operand types

  // Rename results
  val robIdx    = new RobPtr         // ROB index
  val psrc      = Vec(MaxSrcNum, UInt(PhyRegIdxWidth.W))  // physical source registers
  val pdest     = UInt(PhyRegIdxWidth.W)  // physical destination register

  // Control flags
  val rfWen     = Bool()             // integer register write
  val fpWen     = Bool()             // FP register write
  val vecWen    = Bool()             // vector register write

  // Branch/jump
  val ftqPtr    = new FtqPtr         // FTQ pointer
  val ftqOffset = UInt(log2Up(PredictWidth).W)

  // Exception
  val exceptionVec = ExceptionVec()
  // ... many additional fields
}
```

> In Kunminghu, `MicroOp` was replaced by `DynInst`. Legacy code may still contain `MicroOp`.

## Pipeline Stage Patterns

### Basic Stage Registers

```scala
// Data transfer between stages
val s0_data = Wire(new StageData)
val s1_data = RegNext(s0_data)      // s0 → s1 register
val s2_data = RegNext(s1_data)      // s1 → s2 register

// Valid propagation
val s0_valid = Wire(Bool())
val s1_valid = RegNext(s0_valid, false.B)
val s2_valid = RegNext(s1_valid, false.B)
```

### fire/valid/ready Pattern

```scala
// DecoupledIO-based pipeline
val io = IO(new Bundle {
  val in  = Flipped(DecoupledIO(new StageData))
  val out = DecoupledIO(new StageData)
})

// fire = valid && ready (handshake complete)
val s0_fire = io.in.fire   // === io.in.valid && io.in.ready
val s1_fire = io.out.fire

// Backpressure
io.in.ready := io.out.ready || !s1_valid
```

### Flush Handling Pattern

```scala
// Pipeline flush triggered by Redirect
val io = IO(new Bundle {
  val redirect = Flipped(ValidIO(new Redirect))
  // ...
})

// Check flush at each stage
val s1_flush = io.redirect.valid && s1_data.robIdx.needFlush(io.redirect.bits)
when (s1_flush) {
  s1_valid := false.B
}
```

## Redirect & Flush Mechanism

### Redirect Bundle

```scala
class Redirect(implicit p: Parameters) extends XSBundle {
  val robIdx     = new RobPtr          // ROB index of the instruction that triggered the redirect
  val ftqIdx     = new FtqPtr          // FTQ index
  val ftqOffset  = UInt(log2Up(PredictWidth).W)
  val level      = RedirectLevel()     // flush level
  val interrupt   = Bool()
  val cfiUpdate  = new CfiUpdateInfo   // BPU update information
  val stFtqIdx   = new FtqPtr          // for store redirect
  val stFtqOffset = UInt(log2Up(PredictWidth).W)
  // ...
}
```

### Flush Propagation Path

```
ROB (exception/misprediction detection)
  → RedirectGenerator
    → Backend.io.redirect (ValidIO(new Redirect))
      → Frontend (FTQ flush + BPU redirect)
      → IssueQueues (speculative entry removal)
      → MemBlock (LSQ flush)
```

### robIdx.needFlush Pattern

```scala
class RobPtr extends CircularQueuePtr[RobPtr] {
  // Determines whether this instruction should be flushed when a redirect occurs
  def needFlush(redirect: Redirect): Bool = {
    // Flush if this.robIdx is younger than redirect.robIdx
    this > redirect.robIdx || this === redirect.robIdx
  }
}
```

## Valid / Decoupled Usage Rules

| Interface | When to Use | Example |
|-----------|-------------|---------|
| `ValidIO` | Unidirectional broadcast, consumer can always accept | `redirect`, `commit`, `wakeup` |
| `DecoupledIO` | Flow control needed, producer-consumer handshake | `dispatch → IQ`, `IFU → IBuffer` |
| `Vec[ValidIO]` | Multi-port simultaneous broadcast | `ROB commit ports`, `wakeup ports` |
| `Vec[DecoupledIO]` | Multi-port flow control | `IQ → ExeUnit` |

```scala
// ValidIO — redirect (always accepted)
val redirect = ValidIO(new Redirect)

// DecoupledIO — dispatch (backpressure possible)
val dispatch = Vec(RenameWidth, DecoupledIO(new DynInst))
```

## Utility Import Scope

```scala
import xiangshan._       // XSModule, XSBundle, HasXSParameter, DynInst, etc.
import xiangshan.backend._ // Backend-specific bundles, parameters
import utils._           // XSPerfAccumulate, XSDebug, shared utilities
import chisel3._         // Chisel basics
import chisel3.util._    // DecoupledIO, log2Ceil, etc.
import org.chipsalliance.cde.config.Parameters  // implicit Parameters
```

## File Organization Rules

| Rule | Description |
|------|-------------|
| 1 module = 1 file | Major modules get their own file. Filename = class name |
| Bundles.scala | Shared bundles for a subsystem are consolidated in `Bundles.scala` |
| Parameters file | Subsystem-specific parameters go in a dedicated `*Parameters.scala` |
| Subdirectory | Related module groups are placed in subdirectories (e.g., `bpu/`, `rob/`, `dcache/`) |

## Gotchas

| Pitfall | Description |
|---------|-------------|
| `DynInst` vs `MicroOp` | Replaced by `DynInst` in Kunminghu. Code using `MicroOp` is legacy |
| `:=` last-connection wins | When multiple assignments target the same signal, the last one takes effect. Watch redirect priority |
| `XSModule` vs `Module` | Use `XSModule` when parameters are needed, plain `Module` otherwise |
| `Bundles.scala` size | Backend's `Bundles.scala` is ~80KB. Full loading is slow — use grep by class name |
| `package.scala` implicits | `import xiangshan._` loads all implicits from `package.scala`. Name collisions possible |
| Redirect valid check | Forgetting `io.redirect.valid` causes permanent flush. Always gate with valid |
| `fromFoo`/`toBar` direction | `fromFoo` must always be `Flipped`. Direction mistakes cause connection errors |
| Vector extension | Vec/FP-related bundle fields are numerous. Even integer-only changes should verify impact on Vec fields |
