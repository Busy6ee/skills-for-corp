# XiangShan Backend Key Interfaces

## DynInst Major Fields

```scala
class DynInst(implicit p: Parameters) extends XSBundle {
  // === Instruction Information ===
  val instr       = UInt(32.W)          // Original RISC-V instruction
  val pc          = UInt(VAddrBits.W)   // Program counter
  val foldpc      = UInt(MemPredPCWidth.W) // Folded PC for memory dependency prediction

  // === Decode Results ===
  val fuType      = FuType()            // Functional unit type (ALU, MUL, FPU, MEM, etc.)
  val fuOpType    = FuOpType()          // Detailed operation type
  val srcType     = Vec(MaxSrcNum, SrcType()) // Source type (reg/imm/fp/vec)
  val lsrc        = Vec(MaxSrcNum, UInt(LogicRegsWidth.W)) // Logical source registers
  val ldest       = UInt(LogicRegsWidth.W)  // Logical destination register

  // === Rename Results ===
  val robIdx      = new RobPtr          // ROB index
  val psrc        = Vec(MaxSrcNum, UInt(PhyRegIdxWidth.W)) // Physical source registers
  val pdest       = UInt(PhyRegIdxWidth.W)  // Physical destination register
  val old_pdest   = UInt(PhyRegIdxWidth.W)  // Previous physical destination (for freeing)

  // === Control Flags ===
  val rfWen       = Bool()              // Integer RF write enable
  val fpWen       = Bool()              // FP RF write enable
  val vecWen      = Bool()              // Vector RF write enable
  val isMove      = Bool()              // Move elimination candidate

  // === Branch/Jump ===
  val ftqPtr      = new FtqPtr          // FTQ pointer
  val ftqOffset   = UInt(log2Up(PredictWidth).W)
  val pred_taken  = Bool()              // BPU prediction result

  // === Exceptions ===
  val exceptionVec = ExceptionVec()     // Exception vector
  val trigger     = new TriggerCf       // Trigger information

  // === Vector Extension ===
  val vpu         = new VPUCtrlSignals  // Vector control signals
  val numUops     = UInt(...)           // Number of vector uops
}
```

## Backend IO

```scala
class BackendIO(implicit p: Parameters) extends XSBundle {
  // Frontend ↔ Backend
  val fromFrontend = Flipped(new FrontendToBackendIO)
  val toFrontend   = new BackendToFrontendIO

  // Backend ↔ MemBlock
  val fromMemBlock = Flipped(new MemBlockToBackendIO)
  val toMemBlock   = new BackendToMemBlockIO
}
```

### FrontendToBackendIO

```scala
class FrontendToBackendIO extends Bundle {
  val cfVec = Vec(DecodeWidth, DecoupledIO(new CtrlFlow))  // Instruction delivery
  val fromFtq = new FtqToBackendIO                         // FTQ information
}
```

### BackendToFrontendIO

```scala
class BackendToFrontendIO extends Bundle {
  val redirect  = ValidIO(new Redirect)     // Redirect command
  val toFtq     = new BackendToFtqIO        // BPU update information
}
```

## ROB Commit Interface

```scala
class RobCommitIO(implicit p: Parameters) extends XSBundle {
  val isCommit  = Bool()                           // Commit occurring
  val commitValid = Vec(CommitWidth, Bool())        // Per-port commit valid
  val info      = Vec(CommitWidth, new RobCommitInfo)  // Commit information

  val isWalk    = Bool()                           // Walk-back mode
  val walkValid = Vec(CommitWidth, Bool())
}

class RobCommitInfo extends Bundle {
  val pc        = UInt(VAddrBits.W)
  val robIdx    = new RobPtr
  val pdest     = UInt(PhyRegIdxWidth.W)
  val old_pdest = UInt(PhyRegIdxWidth.W)
  val rfWen     = Bool()
  val fpWen     = Bool()
  val vecWen    = Bool()
  val isMove    = Bool()
  val commitType = CommitType()
}
```

## IssueQueue Wakeup

```scala
class WakeupInfo(implicit p: Parameters) extends XSBundle {
  val pdest   = UInt(PhyRegIdxWidth.W)  // Physical register to wake up
  val fpWen   = Bool()
  val rfWen   = Bool()
  val vecWen  = Bool()
}
```

## ExeUnit IO

```scala
class ExuInput(implicit p: Parameters) extends XSBundle {
  val uop    = new DynInst              // Instruction information
  val src    = Vec(MaxSrcNum, UInt(XLEN.W))  // Source operand data
}

class ExuOutput(implicit p: Parameters) extends XSBundle {
  val uop    = new DynInst              // Instruction information (forwarded)
  val data   = UInt(XLEN.W)            // Execution result
  val fflags = UInt(5.W)               // FP exception flags
  val redirectValid = Bool()            // Branch redirect occurred
  val redirect = new Redirect           // Redirect information
}
```

## Redirect

```scala
class Redirect(implicit p: Parameters) extends XSBundle {
  val robIdx      = new RobPtr
  val ftqIdx      = new FtqPtr
  val ftqOffset   = UInt(log2Up(PredictWidth).W)
  val level       = RedirectLevel()     // flushAfter, flush, ...
  val interrupt    = Bool()
  val cfiUpdate   = new CfiUpdateInfo   // For BPU update
  // Memory-related redirect
  val stFtqIdx    = new FtqPtr
  val stFtqOffset = UInt(...)
  val debug_runahead_checkpoint_id = UInt(...)
}
```

## FuType / SrcType Enumerations

```scala
// Functional unit type
object FuType {
  val alu  = ...  // ALU
  val mul  = ...  // Multiplier
  val div  = ...  // Divider
  val jmp  = ...  // Jump
  val brh  = ...  // Branch
  val csr  = ...  // CSR
  val fmac = ...  // FP multiply-accumulate
  val fmisc = ... // FP miscellaneous
  val fDivSqrt = ... // FP division/square root
  val ldu  = ...  // Load
  val stu  = ...  // Store
  val mou  = ...  // Memory ordering unit
  val valu = ...  // Vector ALU
  val vfp  = ...  // Vector FP
  // ...
}

// Source operand type
object SrcType {
  val reg = ...  // Integer register
  val fp  = ...  // FP register
  val vec = ...  // Vector register
  val imm = ...  // Immediate
  val pc  = ...  // PC
  val no  = ...  // No source
}
```
