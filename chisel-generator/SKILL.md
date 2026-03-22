---
name: chisel-generator
description: Use when creating parametric or reusable hardware generators in Chisel, using Scala generics [T <: Data], constructor parameters, abstract classes, or functional programming patterns (reduce, reduceTree, map, zip) for hardware generation.
---

# Chisel Generator — Parametric/Reusable Hardware Generators

## Constructor Parameters

```scala
class WhenCounter(n: Int) extends Module {
  val io = IO(new Bundle {
    val cnt  = Output(UInt(8.W))
    val tick = Output(Bool())
  })

  val N = (n - 1).U
  val cntReg = RegInit(0.U(8.W))
  cntReg := cntReg + 1.U
  when(cntReg === N) {
    cntReg := 0.U
  }
  io.tick := cntReg === N
  io.cnt := cntReg
}

// Usage: Module(new WhenCounter(10))
```

### Parameter Validation with require()

```scala
class CarryRippleAdder(w: Int = 32) extends Module {
  require(w > 0, "adders should have a positive width")
  // ...
}
```

## Abstract Classes — Module Families

```scala
// Abstract base
abstract class Counter(n: Int) extends Module {
  val io = IO(new Bundle {
    val cnt  = Output(UInt(8.W))
    val tick = Output(Bool())
  })
}

// Concrete implementations
class WhenCounter(n: Int) extends Counter(n) {
  val cntReg = RegInit(0.U(8.W))
  cntReg := cntReg + 1.U
  when(cntReg === (n-1).U) { cntReg := 0.U }
  io.tick := cntReg === (n-1).U
  io.cnt := cntReg
}

class MuxCounter(n: Int) extends Counter(n) {
  val cntReg = RegInit(0.U(8.W))
  cntReg := Mux(cntReg === (n-1).U, 0.U, cntReg + 1.U)
  io.tick := cntReg === (n-1).U
  io.cnt := cntReg
}
```

### Generic FIFO Pattern

```scala
abstract class Fifo[T <: Data](gen: T, val depth: Int) extends Module {
  val io = IO(new FifoIO(gen))
  require(depth > 0, "Number of buffer elements needs to be larger than 0")
}

class BubbleFifo[T <: Data](gen: T, depth: Int) extends Fifo(gen, depth) {
  // implementation...
}

// Usage: Module(new BubbleFifo(UInt(8.W), 4))
// Or:    Module(new BubbleFifo(new MyBundle(), 8))
```

## Generic Type Parameters `[T <: Data]`

```scala
// Generic Mux function
def myMux[T <: Data](sel: Bool, tVal: T, fVal: T): T = {
  Mux(sel, tVal, fVal)
}
```

### Manifest Requirement (for Vec/Module Creation)

```scala
class ArbiterTree[T <: Data: Manifest](gen: T, n: Int) extends Module {
  // [T <: Data: Manifest] — Manifest required for Vec(n, gen) creation
  val io = IO(new Bundle {
    val in  = Input(Vec(n, gen))
    val out = Output(gen)
  })
  // ...
}
```

## Function-Based Hardware Generators

### Basics: Function Call = Hardware Instance

```scala
// Each call generates new hardware
def adder(x: UInt, y: UInt) = x + y

val sum1 = adder(a, b)  // Creates adder 1
val sum2 = adder(c, d)  // Creates adder 2 (separate hardware)
```

### Pipeline Delay

```scala
def delay(x: UInt) = RegNext(x)

// 2-stage pipeline
val delayed2 = delay(delay(din))
```

### Tuple Return Values

```scala
def compare(a: UInt, b: UInt) = {
  val equ = a === b
  val gt = a > b
  (equ, gt)
}

// Option 1: Tuple access
val cmp = compare(inA, inB)
val equResult = cmp._1
val gtResult = cmp._2

// Option 2: Destructuring
val (equ, gt) = compare(inA, inB)
```

### Counter Generator Function

```scala
def genCounter(n: Int) = {
  val cntReg = RegInit(0.U(8.W))
  cntReg := Mux(cntReg === n.U, 0.U, cntReg + 1.U)
  cntReg
}

// Easily create multiple independent counters
val count10 = genCounter(10)
val count99 = genCounter(99)
```

## Functional Programming Patterns

### reduce — Linear Chain

```scala
// Linear chain summing all Vec elements
val sum = vec.reduce(_ + _)

// Equivalent to: vec(0) + vec(1) + vec(2) + ...
// Depth: O(n) — suboptimal for timing
```

### reduceTree — Balanced Tree

```scala
// Balanced tree summing all Vec elements
val sum = vec.reduceTree(_ + _)

// Depth: O(log n) — optimal for timing
```

### Finding the Minimum Value

```scala
val min = vec.reduceTree((x, y) => Mux(x < y, x, y))
```

### Minimum Value + Index (Bundle + reduceTree)

```scala
class ValIdx extends Bundle {
  val v = UInt(w.W)
  val idx = UInt(8.W)
}

val vecTwo = Wire(Vec(n, new ValIdx()))
for (i <- 0 until n) {
  vecTwo(i).v := vec(i)
  vecTwo(i).idx := i.U
}

val result = vecTwo.reduceTree((x, y) => Mux(x.v < y.v, x, y))
// result.v = minimum value, result.idx = index
```

### zipWithIndex + map + reduce

```scala
val (minVal, minIdx) = vec.zipWithIndex
  .map(x => (x._1, x._2.U))
  .reduce((x, y) => (
    Mux(x._1 < y._1, x._1, y._1),
    Mux(x._1 < y._1, x._2, y._2)
  ))
```

## Optional IO Ports

```scala
class RegisterFile(debug: Boolean = false) extends Module {
  val io = IO(new Bundle {
    val rdData = Output(UInt(32.W))
    val debugPort = if (debug) Some(Output(Vec(32, UInt(32.W)))) else None
  })
  // Using the debug port
  if (debug) {
    io.debugPort.get := regFile
  }
}
```

## Gotchas

| Pitfall | Description |
|---------|-------------|
| **Function call = new hardware** | Calling `adder(x,y)` twice creates **2** separate adders. Hardware cannot be shared |
| **`reduce` vs `reduceTree`** | `reduce` = O(n) depth (linear), `reduceTree` = O(log n) (balanced). Always use `reduceTree` for timing-critical paths |
| **`[T <: Data: Manifest]`** | `: Manifest` context bound is required when using generic types inside Vec/Module |
| **Provide defaults** | Reusable modules should have sensible defaults, e.g., `class Foo(w: Int = 32)` |
| **`require()` placement** | Place at the top of the class body to catch invalid parameters before elaboration |
| **abstract class vs trait** | Use abstract class when constructor parameters are needed. Use trait for mixins |
| **`WireDefault` vs `cloneType`** | For type cloning, use `Wire(gen.cloneType)` or `WireDefault(gen)` |
| **`isPow2` utility** | `chisel3.util.isPow2(w)` — use for power-of-2 validation |
