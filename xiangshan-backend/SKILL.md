---
name: xiangshan-backend
description: Use when working on XiangShan backend — Decode, Rename, Dispatch, IssueQueue, ExeUnit (ALU/MDU/FPU/Vector), ROB, DataPath, or BypassNetwork. Use when modifying instruction scheduling, execution, or retirement logic.
---

# XiangShan Backend — Out-of-Order Execution Engine

## Backend Pipeline Overview

```
IBuffer (Frontend)
  ↓
Decode ──→ Rename ──→ Dispatch ──→ IssueQueue ──→ ExeUnit ──→ Writeback
                                       ↑              ↓            ↓
                                   Wakeup ←──── DataPath ──→ BypassNetwork
                                                               ↓
                                                             ROB (commit/redirect)
```

### Core Data Types

- **`DynInst`**: Instruction bundle flowing through the entire pipeline (Kunminghu generation)
- **`RobPtr`**: ROB index (circular queue pointer, used for flush determination)
- **`FtqPtr`**: FTQ index (references branch prediction information)

### Key Files

| Module | File | Size |
|--------|------|------|
| Backend | `backend/Backend.scala` | 34KB |
| BackendParams | `backend/BackendParams.scala` | 27KB |
| Bundles | `backend/Bundles.scala` | **80KB** |
| CtrlBlock | `backend/ctrlblock/CtrlBlock.scala` | 47KB |
| DecodeUnit | `backend/decode/DecodeUnit.scala` | **74KB** |
| Rename | `backend/rename/Rename.scala` | 49KB |
| Dispatch | `backend/dispatch/NewDispatch.scala` | 50KB |
| IssueQueue | `backend/issue/IssueQueue.scala` | **67KB** |
| ROB | `backend/rob/Rob.scala` | **88KB** |
| DataPath | `backend/datapath/DataPath.scala` | 47KB |

## Decode Stage

### DecodeUnit

Converts RISC-V instructions into internal micro-operations (uops).

```scala
class DecodeUnit(implicit p: Parameters) extends XSModule {
  val io = IO(new Bundle {
    val in  = Input(new CtrlFlow)     // Received from IBuffer
    val out = Output(new DynInst)     // Sent to Rename
  })

  // Instruction field extraction
  val instr  = io.in.instr
  val opcode = instr(6, 0)
  val funct3 = instr(14, 12)
  val funct7 = instr(31, 25)

  // Decode table matching → sets fuType, fuOpType, srcType, etc.
}
```

### Related Decoders

| Decoder | File | Role |
|---------|------|------|
| `DecodeUnit` | 74KB | Main RISC-V decoder |
| `DecodeUnitComp` | 87KB | Complex instruction decode |
| `FusionDecoder` | 31KB | Instruction fusion (2 instrs → 1 uop) |
| `VecDecoder` | 62KB | Vector extension decode |

> Instruction fusion: merges two consecutive instructions into a single uop to maximize backend width utilization.

## Rename Stage

### Role

1. **Register renaming**: Logical register → physical register mapping
2. **WAR/WAW hazard elimination**: Removes false dependencies through physical register usage
3. **Free list management**: Allocation and deallocation of available physical registers

### Core Structure

```scala
class Rename(implicit p: Parameters) extends XSModule {
  // RenameTable: logical → physical mapping table
  val intRenameTable = Module(new RenameTable(IntPhyRegs))
  val fpRenameTable  = Module(new RenameTable(FpPhyRegs))

  // BusyTable: tracks physical register readiness
  val intBusyTable = Module(new BusyTable(IntPhyRegs))
  val fpBusyTable  = Module(new BusyTable(FpPhyRegs))

  // FreeList: manages available physical registers
  val intFreeList = Module(new FreeList(IntPhyRegs))
  val fpFreeList  = Module(new FreeList(FpPhyRegs))
}
```

### Redirect Handling

```
Redirect occurs (misprediction/exception)
  → Restore RenameTable to committed state
  → Restore FreeList (return incorrectly allocated physical registers)
  → Restore BusyTable
```

## Dispatch Stage

Distributes renamed instructions to the appropriate Issue Queues.

### Backend Region Structure

The Backend consists of 3 **Regions**:

```scala
// Inside BackendInlinedImp
private val intRegion = Module(new Region(params.intSchdParams.get))
private val fpRegion = Module(new Region(params.fpSchdParams.get))
private val vecRegion = Module(new Region(params.vecSchdParams.get))
```

Each Region contains its own IssueQueue + ExeUnit + DataPath.

### Dispatch Targets

| Region / IQ Type | Instructions Handled | Execution Units |
|------------------|---------------------|-----------------|
| intRegion (IntIQ) | Integer arithmetic, branch/jump | ALU×4, BJU×4 |
| fpRegion (FpIQ) | Floating-point arithmetic | FMA×3, FDIV×2 |
| vecRegion (VecIQ) | Vector operations | VFMA, VIALU, VFALU, VFDIV |
| memSchdParams (MemIQ) | Load/store/vector memory | LDU×3, STA×2, VLSU×2, STD×2 |

### Dispatch Conditions

```scala
// All conditions must be met for dispatch to proceed
val canDispatch =
  iqNotFull &&          // IQ has free entries
  robNotFull &&         // ROB has free entries
  srcReady              // (optional) source operands are ready
```

> **Dispatch stall** back-propagates to the Frontend. ROB/IQ full → Dispatch stall → Rename stall → Decode stall → IBuffer full → Frontend stall.

## Issue Queue Architecture

### Wakeup Mechanism

```
ExeUnit completes execution
  → Broadcasts wakeup signal (pdest, fuType)
    → Compares against source operands of all IQ entries
      → On match, sets srcReady
        → All srcs ready → issue candidate
```

### Issue Selection

```scala
// Age-based selection (oldest-first)
// 1. Select the oldest among all ready entries
// 2. Multi-issue: issue multiple entries simultaneously (depending on IQ width)
```

### IssueQueue IO

```scala
class IssueQueueIO(implicit p: Parameters) extends XSBundle {
  val enq     = Vec(RenameWidth, Flipped(DecoupledIO(new DynInst)))  // Receives from dispatch
  val deq     = Vec(issueWidth, DecoupledIO(new DynInst))            // Issues to ExeUnit

  val wakeup  = Vec(wakeupWidth, Flipped(ValidIO(new WakeupInfo)))   // Receives wakeup signals
  val redirect = Flipped(ValidIO(new Redirect))                       // flush
}
```

For detailed interface definitions, see [backend-interfaces.md](backend-interfaces.md).

## ROB (Reorder Buffer)

### Role

1. **Order restoration**: Commits out-of-order execution results in program order
2. **Exception handling**: On exception, commits only instructions preceding the faulting one
3. **Redirect generation**: Issues redirect on branch misprediction or exception
4. **Store commit**: Commits store instructions → notifies MemBlock

### ROB Entry

```scala
class RobEntry(implicit p: Parameters) extends XSBundle {
  val valid      = Bool()
  val committed  = Bool()
  val pc         = UInt(VAddrBits.W)
  val fuType     = FuType()
  val fpWen      = Bool()
  val rfWen      = Bool()
  val pdest      = UInt(PhyRegIdxWidth.W)
  val exceptionVec = ExceptionVec()
  val flushPipe  = Bool()
  // ...
}
```

### Commit Flow

```
Attempts to commit CommitWidth instructions simultaneously from ROB head
  ├── Normal commit → Update RenameTable architectural state, free old physical registers
  ├── Exception found → Generate redirect, flush subsequent ROB entries
  └── Store commit → Send store commit signal to MemBlock (allows Sbuffer drain)
```

### Walk-back (Recovery After Redirect)

```
Redirect occurs
  → ROB invalidates entries after the redirect point
  → RenameTable walk-back (restore to state at redirect point)
  → FreeList restoration
  → Flush speculative entries in IQ/Pipeline
```

## DataPath & BypassNetwork

### DataPath

Reads operands from the register file or bypass network for instructions dequeued from the IssueQueue, then delivers them to the ExeUnit.

```
IQ deq → Read RF / Bypass → ExeUnit input
ExeUnit output → WB Arbiter → Write RF + Wakeup broadcast
```

### BypassNetwork

Forwards execution results directly to subsequent instructions without writing to the register file.

```scala
// Bypass source: output port of each ExeUnit
// Bypass sink: source operands of each IQ dequeue
// Matching: pdest == psrc → select bypass data
```

> BypassNetwork timing is on the critical path. Number of bypass stages = performance vs. timing trade-off.

## Execution Units

| ExeUnit | Functional Unit | Latency |
|---------|----------------|---------|
| ALU | Integer arithmetic/logic | 1 cycle |
| BKU | Bit manipulation, crypto | 1-2 cycles |
| MUL | Integer multiplication | 2-3 cycles |
| DIV (SRT16) | Integer division | Variable (multi-cycle) |
| Branch/Jump | Branch/jump execution | 1 cycle |
| CSR | Control status register | Variable |
| FPU | Floating-point | 3-5 cycles |
| Vector | Vector operations (yunsuan) | Variable |

### ExeUnit Wrapper

```scala
class ExeUnit(exuParams: ExeUnitParams)(implicit p: Parameters) extends XSModule {
  val io = IO(new Bundle {
    val in  = Flipped(DecoupledIO(new ExuInput))
    val out = DecoupledIO(new ExuOutput)
  })
  // Instantiates FUs according to parameters
}
```

## Backend Parameters (DSE)

| Parameter | Default | IPC Impact |
|-----------|---------|------------|
| `RobSize` | 160 | Instruction window → ILP exploitation |
| `RenameWidth` | 6 | Rename width → Frontend→Backend bandwidth |
| `CommitWidth` | 8 | Commit width → throughput upper bound |
| `IntPhyRegs` | 224 | Integer physical registers → rename availability |
| `FpPhyRegs` | 192 | FP physical registers |
| IQ entry count | Variable | Scheduling window |
| Issue width per IQ | Variable | Simultaneous issues per IQ |

## Gotchas

| Pitfall | Description |
|---------|-------------|
| ROB = 88KB single file | Navigate by section comments. Search by class/def recommended |
| `DynInst` massive Bundle | Dozens of fields. Verify impact across entire pipeline when modifying |
| `backend/Bundles.scala` = 80KB | Contains all backend-specific bundles. Use grep to locate specific Bundles |
| IQ parameter dependency | IQ behavior differs for `IntIQ`/`FpIQ`/`MemIQ`/`VecIQ`. Type-specific config in `BackendParams` |
| BypassNetwork timing | Adding/removing bypass stages → critical path change. Timing analysis required |
| Dispatch stall back-propagation | ROB/IQ full → Dispatch → Rename → Decode → Frontend full stall chain |
| Speculative wakeup | Wakeup can fire before execution completes (speculative wakeup). Replay needed on cancel |
| DecodeUnit 74KB | Decode table is massive. Watch for pattern matching locations when adding new instructions |
| Instruction fusion | `FusionDecoder` manages fusion patterns. Add new fusion patterns here |
