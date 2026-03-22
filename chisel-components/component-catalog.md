# Component Catalog — Detailed Reference

## UART Design

### Tx (Transmitter)

```
Structure: Idle → Start bit → 8 data bits (LSB first) → Stop bit → Idle
Clock: Divide system clock to baud rate
Baud rate calculation: val divisor = (frequency + baudRate/2) / baudRate - 1
```

FSM states: `idle`, `start`, `data`, `stop`
- `idle`: txd = 1 (mark state), transitions to start on valid input
- `start`: txd = 0, transitions to data after 1-bit time
- `data`: txd = shiftReg(0), transitions to stop after 8 bits transmitted
- `stop`: txd = 1, transitions to idle after 1-bit time

### Rx (Receiver)

```
Start bit detection → Sample at bit center → Receive 8 bits → Verify stop bit
```

Bit-center sampling: starts at `cntReg === (divisor/2)`, then samples at `divisor` intervals thereafter

## Adder Family

| Type | Depth | Area | Use Case |
|------|-------|------|----------|
| CarryRipple | O(n) | O(n) | Small, low frequency |
| CarrySelect | O(sqrt(n)) | O(2n) | Medium |
| CarrySkip | O(sqrt(n)) | O(n) | Medium, area efficient |
| CarryLookahead | O(log n) | O(n log n) | High speed |
| PrefixAdder (Brent-Kung) | O(log n) | O(n log n) | Maximum speed |

```scala
// Abstract base
abstract class AbstractAdder(w: Int) extends Module {
  require(w > 0, "adders should have a positive width")
  val io = IO(new Bundle {
    val a   = Input(UInt(w.W))
    val b   = Input(UInt(w.W))
    val cin = Input(Bool())
    val sum = Output(UInt(w.W))
    val cout = Output(Bool())
  })
}
```

## Arbiter Tree

Fair arbiter combining `reduceTree` with a 2-way arbiter:

```scala
class ArbiterTree[T <: Data: Manifest](gen: T, n: Int) extends Module {
  val io = IO(new Bundle {
    val in  = Vec(n, Flipped(new DecoupledIO(gen)))
    val out = new DecoupledIO(gen)
  })
  // Builds a tree of 2-way arbiters using reduceTree
  // Tracks last-served via FSM to ensure fairness
}
```

## ALU Pattern

```scala
val alu = WireDefault(0.U(16.W))
switch(io.fn) {
  is(0.U) { alu := io.a + io.b }
  is(1.U) { alu := io.a - io.b }
  is(2.U) { alu := io.a | io.b }
  is(3.U) { alu := io.a & io.b }
}
io.result := alu
```
