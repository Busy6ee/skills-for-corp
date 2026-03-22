---
name: chisel-memory
description: Use when implementing SRAM, register files, or any memory structure in Chisel. Use when dealing with SyncReadMem, read-during-write behavior, memory forwarding, dual-port memory, or multi-port register file access patterns.
---

# Chisel Memory — Memory and Register Files

## SyncReadMem — Basic SRAM

```scala
class Memory extends Module {
  val io = IO(new Bundle {
    val rdAddr = Input(UInt(10.W))
    val rdData = Output(UInt(8.W))
    val wrAddr = Input(UInt(10.W))
    val wrData = Input(UInt(8.W))
    val wrEna  = Input(Bool())
  })

  val mem = SyncReadMem(1024, UInt(8.W))

  io.rdData := mem.read(io.rdAddr)

  when(io.wrEna) {
    mem.write(io.wrAddr, io.wrData)
  }
}
```

## Read-During-Write Forwarding

Pattern for returning the latest data when reading and writing the same address simultaneously:

```scala
class ForwardingMemory extends Module {
  val io = IO(new Bundle {
    val rdAddr = Input(UInt(10.W))
    val rdData = Output(UInt(8.W))
    val wrAddr = Input(UInt(10.W))
    val wrData = Input(UInt(8.W))
    val wrEna  = Input(Bool())
  })

  val mem = SyncReadMem(1024, UInt(8.W))

  // Delay forwarding condition and data by 1 cycle
  val wrDataReg = RegNext(io.wrData)
  val doForwardReg = RegNext(io.wrAddr === io.rdAddr && io.wrEna)

  val memData = mem.read(io.rdAddr)

  when(io.wrEna) {
    mem.write(io.wrAddr, io.wrData)
  }

  // Select forwarded or memory data
  io.rdData := Mux(doForwardReg, wrDataReg, memData)
}
```

## WriteFirst Mode

```scala
// Specify WriteFirst on SyncReadMem
val mem = SyncReadMem(1024, UInt(8.W), SyncReadMem.WriteFirst)
```

WriteFirst: when the same address is accessed simultaneously, write data is reflected in the read. No manual forwarding needed.

## True Dual-Port Memory

```scala
class TrueDualPortMemory extends Module {
  val io = IO(new Bundle {
    val addrA   = Input(UInt(10.W))
    val rdDataA = Output(UInt(8.W))
    val wrEnaA  = Input(Bool())
    val wrDataA = Input(UInt(8.W))
    val addrB   = Input(UInt(10.W))
    val rdDataB = Output(UInt(8.W))
    val wrEnaB  = Input(Bool())
    val wrDataB = Input(UInt(8.W))
  })

  val mem = SyncReadMem(1024, UInt(8.W))

  // Port A
  io.rdDataA := mem.read(io.addrA)
  when(io.wrEnaA) {
    mem.write(io.addrA, io.wrDataA)
  }

  // Port B
  io.rdDataB := mem.read(io.addrB)
  when(io.wrEnaB) {
    mem.write(io.addrB, io.wrDataB)
  }
}
```

## Register File — Reg(Vec) Based

```scala
// Basic register file (no reset)
val registerFile = Reg(Vec(32, UInt(32.W)))
registerFile(wrIdx) := wrData
val rdData = registerFile(rdIdx)

// Resettable register file
val resetRegFile = RegInit(VecInit(Seq.fill(32)(0.U(32.W))))
```

### Optional Debug Port

```scala
class RegisterFile(debug: Boolean = false) extends Module {
  val io = IO(new Bundle {
    val rdAddr = Input(UInt(5.W))
    val rdData = Output(UInt(32.W))
    val wrAddr = Input(UInt(5.W))
    val wrData = Input(UInt(32.W))
    val wrEna  = Input(Bool())
    // Conditional port
    val debugPort = if (debug) Some(Output(Vec(32, UInt(32.W)))) else None
  })

  val regFile = Reg(Vec(32, UInt(32.W)))

  io.rdData := regFile(io.rdAddr)
  when(io.wrEna) {
    regFile(io.wrAddr) := io.wrData
  }

  if (debug) {
    io.debugPort.get := regFile
  }
}
```

## File Initialization

```scala
import chisel3.util.experimental.loadMemoryFromFileInline

val mem = SyncReadMem(1024, UInt(8.W))
loadMemoryFromFileInline(
  mem,
  "./src/main/resources/init.hex",
  firrtl.annotations.MemoryLoadFileType.Hex
)
```

### Dynamically Generating Hex Files in Scala

```scala
val hello = "Hello, World!"
val helloHex = hello.map(_.toInt.toHexString).mkString("\n")
val file = new java.io.PrintWriter("hello.hex")
file.write(helloHex)
file.close()

val mem = SyncReadMem(1024, UInt(8.W))
loadMemoryFromFileInline(mem, "hello.hex", firrtl.annotations.MemoryLoadFileType.Hex)
```

## Multi-Clock Memory

```scala
class MultiClockMemory extends Module {
  val io = IO(new Bundle {
    val clkB    = Input(Bool())  // External clock as Bool input
    val rdAddr  = Input(UInt(10.W))
    val rdData  = Output(UInt(8.W))
    val wrAddr  = Input(UInt(10.W))
    val wrData  = Input(UInt(8.W))
    val wrEna   = Input(Bool())
  })

  val mem = SyncReadMem(1024, UInt(8.W))

  // Read in the default clock domain
  io.rdData := mem.read(io.rdAddr)

  // Write in a different clock domain
  withClock(io.clkB.asClock) {
    when(io.wrEna) {
      mem.write(io.wrAddr, io.wrData)
    }
  }
}
```

## SyncReadMem vs Reg(Vec) Selection Guide

| | SyncReadMem | Reg(Vec) |
|--|------------|----------|
| **Synthesis result** | BRAM/Block RAM | FF/Distributed RAM |
| **Read latency** | 1 cycle (synchronous read) | 0 cycles (combinational read) |
| **Suitable size** | Large memory (>64 entries) | Small memory (<64 entries) |
| **Number of ports** | Limited (depends on FPGA resources) | Flexible (FF-based) |
| **Reset** | Not supported (file init only) | `RegInit(VecInit(...))` |
| **read-during-write** | Requires explicit handling | Automatic (combinational read) |

## Gotchas

| Pitfall | Description |
|------|------|
| **SyncReadMem 1-cycle latency** | `mem.read(addr)` result is valid on the **next** clock edge. Cannot read in the same cycle |
| **Default read-during-write is undefined** | Simultaneous R/W to the same address yields undefined results. Must implement forwarding or use `WriteFirst` |
| **Dual-port synthesis limitations** | `SyncReadMem` dual-port does not always synthesize to Block RAM on all FPGAs (e.g., synthesized to FFs on Cyclone V) |
| **`loadMemoryFromFileInline` backend** | Based on FIRRTL annotations — may not be supported by all synthesis backends |
| **RegFile read is combinational** | `Reg(Vec)` read is same-cycle (combinational), `SyncReadMem` is next-cycle — consider this difference in pipeline design |
| **Memory size and resources** | 32 entries or fewer: `Reg(Vec)` recommended. Larger: use `SyncReadMem` to leverage BRAM |
| **`.asClock` safety** | `Bool.asClock` does not handle CDC (Clock Domain Crossing) — separate synchronization logic is required |
