---
name: chisel-interface
description: Use when connecting Chisel modules together, using DecoupledIO (ready/valid handshaking), Flipped ports, bulk connections (<>), or designing bus interfaces and inter-module communication protocols.
---

# Chisel Interface — Module Connection and Protocols

## DecoupledIO — Ready/Valid Handshaking

### Structure

```scala
// chisel3.util.DecoupledIO internal structure (reference)
class DecoupledIO[T <: Data](gen: T) extends Bundle {
  val ready = Input(Bool())    // Consumer → Producer
  val valid = Output(Bool())   // Producer → Consumer
  val bits  = Output(gen)      // Data
}
```

### Producer Side (Output)

```scala
val io = IO(new Bundle {
  val out = new DecoupledIO(UInt(8.W))  // Default: valid/bits = Output, ready = Input
})
```

### Consumer Side (Input) — Flipped

```scala
val io = IO(new Bundle {
  val in = Flipped(new DecoupledIO(UInt(8.W)))  // Direction reversed: valid/bits = Input, ready = Output
})
```

### Ready/Valid Buffer Example

```scala
class ReadyValidBuffer extends Module {
  val io = IO(new Bundle {
    val in  = Flipped(new DecoupledIO(UInt(8.W)))
    val out = new DecoupledIO(UInt(8.W))
  })

  val dataReg  = Reg(UInt(8.W))
  val emptyReg = RegInit(true.B)

  io.in.ready  := emptyReg
  io.out.valid := !emptyReg
  io.out.bits  := dataReg

  when(emptyReg & io.in.valid) {
    dataReg  := io.in.bits
    emptyReg := false.B
  }

  when(!emptyReg & io.out.ready) {
    emptyReg := true.B
  }
}
```

### fire Signal

```scala
// Condition for successful data transfer
val transferred = io.out.fire  // === io.out.valid && io.out.ready
```

## Flipped — Direction Reversal

```scala
// Base Bundle
class Channel extends Bundle {
  val data  = Output(UInt(32.W))
  val valid = Output(Bool())
  val ready = Input(Bool())
}

// Producer IO
val producer = IO(new Channel())       // data/valid = Output, ready = Input

// Consumer IO
val consumer = IO(Flipped(new Channel()))  // data/valid = Input, ready = Output
```

## Bulk Connection `<>`

Automatically connects fields with matching names:

```scala
// Inter-module connection
val producer = Module(new Producer())
val consumer = Module(new Consumer())
producer.io.out <> consumer.io.in  // Automatically matches fields with the same name

// FIFO chaining
val buffers = Array.fill(depth) { Module(new Buffer()) }
for (i <- 0 until depth - 1) {
  buffers(i + 1).io.enq <> buffers(i).io.deq
}
io.enq <> buffers(0).io.enq
io.deq <> buffers(depth - 1).io.deq
```

## Parameterized IO Bundle

```scala
class FifoIO[T <: Data](private val gen: T) extends Bundle {
  val enq = Flipped(new DecoupledIO(gen))
  val deq = new DecoupledIO(gen)
}

// Usage
class MyFifo[T <: Data](gen: T, depth: Int) extends Module {
  val io = IO(new FifoIO(gen))
  // ...
}
```

## Custom Protocol Bundle

### Memory-Mapped Interface

```scala
class MemoryMappedIO extends Bundle {
  val address = Input(UInt(16.W))
  val rd      = Input(Bool())
  val wr      = Input(Bool())
  val rdData  = Output(UInt(32.W))
  val wrData  = Input(UInt(32.W))
}
```

## Module Hierarchy

### Submodule Instantiation

```scala
class Top extends Module {
  val io = IO(new Bundle {
    val in  = Input(UInt(8.W))
    val out = Output(UInt(8.W))
  })

  // Create submodules
  val stage1 = Module(new Stage1())
  val stage2 = Module(new Stage2())

  // Manual connection
  stage1.io.in := io.in
  stage2.io.in := stage1.io.out
  io.out := stage2.io.out
}
```

### Connection via Bulk Connection

```scala
class Pipeline extends Module {
  val fetch  = Module(new Fetch())
  val decode = Module(new Decode())

  fetch.io <> decode.io  // Automatic connection by name matching
}
```

## Connection Operator Comparison

| Operator | Direction | Behavior |
|----------|-----------|----------|
| `:=` | Unidirectional | Assigns RHS to all fields of LHS (last connection wins) |
| `<>` | Bidirectional | Automatic connection by name matching (Input↔Output) |

## Gotchas

| Pitfall | Description |
|---------|-------------|
| **`<>` name mismatch** | Unmatched fields are **silently ignored** (not an error) — watch for unintended unconnected signals |
| **`Flipped()` is recursive** | Recursively reverses all directions within a Bundle — use caution with mixed-direction Bundles |
| **Undriven `ready`** | `DecoupledIO` consumers must always drive the `ready` signal (undriven = synthesis error) |
| **Prefer `fire`** | Use `io.out.fire` instead of `io.out.valid && io.out.ready` |
| **`Module(new X())` parentheses** | Parentheses around `new X()` are required — `Module(new X)` works but causes confusion when arguments are needed |
| **`:=` last connection wins** | When multiple `:=` target the same Wire, only the last one takes effect |
| **`<>` does NOT have last-connection semantics** | `<>` is bidirectional — multiple `<>` on the same port causes an error |
| **DecoupledIO default direction** | `new DecoupledIO(gen)` → valid/bits=Output, ready=Input. The input side must use `Flipped()` |
| **Bundle field directions** | Be careful when combining explicit `Input`/`Output` in IO with `Flipped` — verify the intended directions |
