---
name: chisel-basics
description: Use when writing Chisel module definitions, IO ports, data types (UInt, SInt, Bool, Bundle, Vec), Wire/Reg declarations, or combinational logic (when/switch/Mux). Use as the primary Chisel syntax reference for code generation.
---

# Chisel Basics — Modules, Types, Combinational Logic

## Import

```scala
import chisel3._
import chisel3.util._  // DecoupledIO, switch/is, log2Ceil, etc.
```

## Module Definition Pattern

```scala
class MyModule extends Module {
  val io = IO(new Bundle {
    val in  = Input(UInt(8.W))
    val out = Output(UInt(8.W))
  })

  io.out := io.in
}
```

## Data Types

### Primitive Types
```scala
UInt(8.W)    // 8-bit unsigned integer
SInt(10.W)   // 10-bit signed integer
Bool()       // 1-bit boolean
Bits(8.W)    // 8-bit (no arithmetic ops, rarely used)
```

### Constant Literals
```scala
0.U           // Width-inferred UInt 0
3.U(4.W)      // 4-bit UInt 3
-3.S          // Width-inferred SInt -3
true.B        // Bool true
false.B       // Bool false
"hff".U       // Hexadecimal 255
"o377".U      // Octal 255
"b1111_1111".U // Binary 255 (underscores for readability)
'A'.U         // Character to UInt (65)
```

### Width Specification
```scala
val n = 10
n.W            // Width object
UInt(n.W)      // Parameterized width
```

## Wire and Reg

### Wire — Combinational Signal
```scala
val w = Wire(UInt(8.W))     // Uninitialized Wire (must be assigned)
w := someValue

val wd = WireDefault(0.U)   // Wire with default value (safer)
when (cond) {
  wd := 3.U
}
```

### Reg — Register (Sequential Logic)
```scala
val reg = Reg(UInt(4.W))         // Register without reset
reg := inputVal

val regInit = RegInit(0.U(4.W))  // Register with reset value
regInit := inputVal

val regNext = RegNext(inputVal)  // 1-cycle delay (most concise)
val regNextInit = RegNext(inputVal, 0.U)  // With reset value

// Enable register
val enableReg = RegEnable(inputVal, enable)               // No reset
val enableRegInit = RegEnable(inputVal, 0.U(4.W), enable) // With reset
```

## Bundle — Composite Type

```scala
// Definition
class Channel extends Bundle {
  val data = UInt(32.W)
  val valid = Bool()
}

// Usage
val ch = Wire(new Channel())
ch.data := 123.U
ch.valid := true.B

// Bundle as IO
val io = IO(new Bundle {
  val tx = Output(new Channel())
  val rx = Input(new Channel())
})
```

### Bundle Register Initialization
```scala
// Create initial value with Wire, then pass to RegInit
val initVal = Wire(new Channel())
initVal.data := 0.U
initVal.valid := false.B
val channelReg = RegInit(initVal)
```

## Vec — Vector/Array

```scala
// Wire Vec
val v = Wire(Vec(3, UInt(4.W)))
v(0) := 1.U
v(1) := 3.U
v(2) := 5.U
val elem = v(index)  // Dynamic indexing

// Reg Vec (register file)
val regFile = Reg(Vec(32, UInt(32.W)))
regFile(wrIdx) := wrData
val rdData = regFile(rdIdx)

// VecInit — Create with initial values
val initVec = VecInit(1.U(3.W), 2.U, 3.U)
val fromSignals = VecInit(sigA, sigB, sigC)  // Vec from existing signals

// RegInit + VecInit — Resettable Vec register
val resetRegFile = RegInit(VecInit(Seq.fill(32)(0.U(32.W))))

// Vec of Bundle
val vecBundle = Wire(Vec(8, new Channel()))

// Bundle containing Vec
class BundleWithVec extends Bundle {
  val field = UInt(8.W)
  val vector = Vec(4, UInt(8.W))
}
```

## Combinational Logic

### when / elsewhen / otherwise
```scala
val w = Wire(UInt(4.W))
w := 0.U  // Default value (always assign first)

when (condA) {
  w := 1.U
} .elsewhen (condB) {
  w := 2.U
} .otherwise {
  w := 3.U
}
```

### Mux
```scala
val result = Mux(sel, trueVal, falseVal)
```

### switch / is (ChiselEnum or UInt matching)
```scala
import chisel3.util._

switch (selector) {
  is (0.U) { out := a }
  is (1.U) { out := b }
  is (2.U) { out := c }
}
```

## Bitwise Operations

```scala
// Bit extraction
val sign = x(31)         // Single bit
val lowByte = x(7, 0)    // Range extraction

// Bit concatenation
val word = highByte ## lowByte  // Cat operator

// Logical operations
val and = a & b    // bitwise AND
val or  = a | b    // bitwise OR
val xor = a ^ b    // bitwise XOR
val inv = ~a       // bitwise NOT

// Arithmetic operations
val sum  = a + b
val diff = a - b
val neg  = -a
val prod = a * b
val quot = a / b
val rem  = a % b
```

## Type Conversion

```scala
val uint = sint.asUInt    // SInt -> UInt
val sint = uint.asSInt    // UInt -> SInt
val bool = uint.asBool    // UInt(1.W) -> Bool
val uint = vecBool.asUInt // Vec[Bool] -> UInt
val bundle = uint.asTypeOf(new MyBundle())  // UInt -> Bundle (bit reinterpretation)
```

## Comparison Operations

```scala
val eq  = a === b   // Hardware equality comparison
val neq = a =/= b   // Hardware inequality comparison
val gt  = a > b
val gte = a >= b
val lt  = a < b
val lte = a <= b
```

## Partial Bit Assignment (Workaround)

Chisel does not support partial assignment of the form `x(7,0) := y`.

**Workaround 1: Use a Bundle**
```scala
class Split extends Bundle {
  val high = UInt(8.W)
  val low = UInt(8.W)
}
val split = Wire(new Split())
split.low := lowByte
split.high := highByte
val combined = split.asUInt
```

**Workaround 2: Use Vec[Bool]**
```scala
val bits = Wire(Vec(4, Bool()))
bits(0) := data(0)
bits(1) := data(1)
bits(2) := data(2)
bits(3) := data(3)
val result = bits.asUInt
```

## Edge Detection

```scala
val risingEdge  = din & !RegNext(din)    // Rising edge
val fallingEdge = !din & RegNext(din)    // Falling edge
```

## Gotchas

| Pitfall | Correct Usage |
|---------|---------------|
| `UInt(8)` | `UInt(8.W)` — `.W` is required |
| `RegInit(0.U)` = 1-bit register | `RegInit(0.U(32.W))` — specify width explicitly |
| `==` (Scala comparison) | `===` (hardware comparison) |
| `!=` (Scala) | `=/=` (hardware) |
| Scala `if/else` for hardware | Use `when/otherwise` or `Mux` |
| `Wire(UInt())` left unassigned | Must assign on all paths or use `WireDefault` |
| `x(7,0) := y` partial assignment | Requires Bundle or Vec[Bool] workaround |
| `:=` last-connect semantics | Multiple assignments to the same Wire — only the last one takes effect |
| `VecInit` width inference | Specify explicit width on the first element: `VecInit(1.U(3.W), 2.U, 3.U)` |
| Bundle register initialization | Create initial value with `Wire`, then use `RegInit(wire)` |
| `Reg(Vec(...))` has no reset | Use `RegInit(VecInit(Seq.fill(n)(0.U)))` when reset is needed |
