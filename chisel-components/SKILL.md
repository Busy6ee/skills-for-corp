---
name: chisel-components
description: Use when implementing standard digital building blocks in Chisel — FIFOs, UARTs, counters, adders, arbiters, shift registers, encoders/decoders, or input processing (debounce, synchronizer, edge detection). Use as a pattern library for common hardware components.
---

# Chisel Components — Standard Building Block Library

## FIFO Design Patterns

### Abstract Base + Parametric IO

```scala
class FifoIO[T <: Data](private val gen: T) extends Bundle {
  val enq = Flipped(new DecoupledIO(gen))
  val deq = new DecoupledIO(gen)
}

abstract class Fifo[T <: Data](gen: T, val depth: Int) extends Module {
  val io = IO(new FifoIO(gen))
  require(depth > 0, "Number of buffer elements needs to be larger than 0")
}
```

### BubbleFifo — Simplest Implementation

```scala
class BubbleFifo[T <: Data](gen: T, depth: Int) extends Fifo(gen, depth) {
  private class Buffer extends Module {
    val io = IO(new FifoIO(gen))
    val fullReg = RegInit(false.B)
    val dataReg = Reg(gen)

    when(fullReg) {
      when(io.deq.ready) { fullReg := false.B }
    }.otherwise {
      when(io.enq.valid) {
        fullReg := true.B
        dataReg := io.enq.bits
      }
    }
    io.enq.ready := !fullReg
    io.deq.valid := fullReg
    io.deq.bits := dataReg
  }

  private val buffers = Array.fill(depth) { Module(new Buffer()) }
  for (i <- 0 until depth - 1) {
    buffers(i + 1).io.enq <> buffers(i).io.deq
  }
  io.enq <> buffers(0).io.enq
  io.deq <> buffers(depth - 1).io.deq
}
```

### RegFifo — Pointer-Based Circular Buffer

```scala
// Implemented with Reg(Vec), uses read/write pointers
// Counter helper function:
def counter(depth: Int, incr: Bool): (UInt, UInt) = {
  val cntReg = RegInit(0.U(log2Ceil(depth).W))
  val nextVal = Mux(cntReg === (depth-1).U, 0.U, cntReg + 1.U)
  when(incr) { cntReg := nextVal }
  (cntReg, nextVal)
}
```

### MemFifo — SyncReadMem-Based (Large Capacity)

Uses SyncReadMem + WriteFirst to implement large-capacity FIFOs. An output register is used to handle the 1-cycle read latency.

### FIFO Selection Guide

| FIFO Type | Throughput | Resource | Best For |
|-----------|------------|----------|----------|
| BubbleFifo | 1/2N cycles | FF | Small, simple |
| DoubleBufferFifo | 1/cycle | FF | Medium, pipelined |
| RegFifo | 1/cycle | FF | Medium, general purpose |
| MemFifo | 1/cycle | BRAM | Large |
| CombFifo | 1/cycle | BRAM+FF | Large, optimal |

## Counter Patterns

### When-Based Counter

```scala
val cntReg = RegInit(0.U(8.W))
cntReg := cntReg + 1.U
when(cntReg === (n-1).U) {
  cntReg := 0.U
}
```

### Mux-Based Counter (One-Liner)

```scala
val cntReg = RegInit(0.U(8.W))
cntReg := Mux(cntReg === (n-1).U, 0.U, cntReg + 1.U)
```

### Down Counter

```scala
val cntReg = RegInit((n-1).U(8.W))
cntReg := cntReg - 1.U
when(cntReg === 0.U) {
  cntReg := (n-1).U
}
val tick = cntReg === 0.U
```

### Function-Based Counter Generator

```scala
def genCounter(n: Int) = {
  val cntReg = RegInit(0.U(8.W))
  cntReg := Mux(cntReg === n.U, 0.U, cntReg + 1.U)
  cntReg
}
val count10 = genCounter(10)
val count99 = genCounter(99)
```

### Tick Generator (Clock Divider)

```scala
val tickReg = RegInit(0.U(32.W))
val tick = tickReg === (N-1).U
tickReg := tickReg + 1.U
when(tick) { tickReg := 0.U }

// Tick-based low-frequency counter
val lowFreqCnt = RegInit(0.U(4.W))
when(tick) {
  lowFreqCnt := lowFreqCnt + 1.U
}
```

## Input Processing Chain

### 1. Synchronization (Metastability Prevention)

```scala
val btnSync = RegNext(RegNext(btn))  // 2-stage FF synchronizer
```

### 2. Debounce

```scala
val btnDebReg = RegInit(false.B)
val cntReg = RegInit(0.U(32.W))
val tick = cntReg === (fac-1).U
cntReg := cntReg + 1.U
when(tick) {
  cntReg := 0.U
  btnDebReg := btnSync  // Sample at each tick interval
}
```

### 3. Majority Voting Filter

```scala
val shiftReg = RegInit(0.U(3.W))
when(tick) {
  shiftReg := shiftReg(1,0) ## btnDebReg
}
val btnClean = (shiftReg(2) & shiftReg(1)) |
               (shiftReg(2) & shiftReg(0)) |
               (shiftReg(1) & shiftReg(0))
```

### 4. Edge Detection

```scala
val risingEdge = btnClean & !RegNext(btnClean)
```

### Functional Input Processing (Reusable)

```scala
def sync(v: Bool) = RegNext(RegNext(v))
def rising(v: Bool) = v & !RegNext(v)

def tickGen(fac: Int) = {
  val reg = RegInit(0.U(log2Up(fac).W))
  val tick = reg === (fac-1).U
  reg := Mux(tick, 0.U, reg + 1.U)
  tick
}

def filter(v: Bool, t: Bool) = {
  val reg = RegInit(0.U(3.W))
  when(t) { reg := reg(1,0) ## v }
  (reg(2) & reg(1)) | (reg(2) & reg(0)) | (reg(1) & reg(0))
}

// Composition:
val btnSync = sync(io.btnU)
val tick = tickGen(100000000/100)
val btnDeb = RegInit(false.B)
when(tick) { btnDeb := btnSync }
val btnClean = filter(btnDeb, tick)
val pressed = rising(btnClean)
```

## Shift Registers

### Serial-In / Parallel-Out

```scala
val shiftReg = RegInit(0.U(4.W))
shiftReg := shiftReg(2, 0) ## io.din  // Left shift, new bit at LSB
val parallelOut = shiftReg
```

### Parallel-Load / Serial-Out

```scala
val shiftReg = Reg(UInt(4.W))
when(io.load) {
  shiftReg := io.parallelIn
} .otherwise {
  shiftReg := 0.U ## shiftReg(3, 1)  // Right shift
}
io.dout := shiftReg(0)
```

## Encoders / Decoders

### Decoder (1-of-N)

```scala
val decoded = 1.U << io.sel  // Only the bit at position sel is 1
```

### Encoder (Priority)

```scala
// Switch-based
val encoded = WireDefault(0.U)
switch(io.input) {
  is("b0001".U) { encoded := 0.U }
  is("b0010".U) { encoded := 1.U }
  is("b0100".U) { encoded := 2.U }
  is("b1000".U) { encoded := 3.U }
}
```

## Gotchas

| Pitfall | Description |
|---------|-------------|
| **BubbleFifo throughput** | 2 cycles/element per buffer stage — not pipeline throughput |
| **2FF synchronizer** | `RegNext(RegNext(btn))` is required — a single FF does not resolve metastability |
| **Debounce sampling period** | Divide the system clock appropriately. E.g., 100 MHz / 100 = 1 MHz |
| **MemFifo WriteFirst** | `SyncReadMem.WriteFirst` is required for SyncReadMem — otherwise R/W hazard occurs |
| **Down counter tick position** | `RegInit(N)` counts down to 0 — tick fires at 0 |
| **Decoder width** | Result width of `1.U << sel` depends on sel width — explicit width truncation may be needed |
| **FIFO depth** | BubbleFifo depth = number of buffers. RegFifo/MemFifo depth = number of entries |
