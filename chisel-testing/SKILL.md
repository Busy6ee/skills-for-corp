---
name: chisel-testing
description: Use when writing ChiselTest test benches, using poke/expect/step/peek operations, or debugging Chisel module behavior through simulation. Use when test compilation or simulation errors occur in Chisel designs.
---

# Chisel Testing — ChiselTest Patterns

## Basic Structure (Chisel 6.x + chiseltest 6.0.0)

```scala
import chisel3._
import chiseltest._
import org.scalatest.flatspec.AnyFlatSpec

class MyModuleTest extends AnyFlatSpec with ChiselScalatestTester {
  "MyModule" should "produce correct output" in {
    test(new MyModule()) { dut =>
      dut.io.in.poke(5.U)
      dut.clock.step()
      dut.io.out.expect(5.U)
    }
  }
}
```

## Core API

### Setting Inputs
```scala
dut.io.a.poke(3.U)          // UInt input
dut.io.enable.poke(true.B)  // Bool input
dut.io.data.poke(-5.S)      // SInt input
```

### Verifying Outputs
```scala
dut.io.out.expect(7.U)           // Value assertion (errors on mismatch)
dut.io.valid.expect(true.B)      // Bool assertion
```

### Reading Outputs (for Scala-side computation)
```scala
val value = dut.io.out.peekInt()    // Returns BigInt
val flag = dut.io.valid.peekBoolean() // Returns Boolean
```

### Clock Control
```scala
dut.clock.step()     // Advance 1 cycle
dut.clock.step(10)   // Advance 10 cycles
```

## Combinational Logic Testing

```scala
test(new CombModule()) { dut =>
  dut.io.a.poke(3.U)
  dut.io.b.poke(4.U)
  dut.io.out.expect(7.U)  // No step() needed — combinational logic updates immediately
}
```

## Sequential Logic Testing

```scala
test(new Counter(10)) { dut =>
  for (i <- 0 until 10) {
    dut.io.cnt.expect(i.U)
    dut.clock.step()
  }
  dut.io.cnt.expect(0.U)  // Wrap-around at 10
}
```

## FSM Testing

```scala
test(new SimpleFsm()) { dut =>
  // Verify initial state
  dut.io.ringBell.expect(false.B)

  // green → orange transition
  dut.io.badEvent.poke(true.B)
  dut.clock.step()
  dut.io.ringBell.expect(false.B)

  // orange → red transition
  dut.clock.step()
  dut.io.ringBell.expect(true.B)

  // red → green (clear)
  dut.io.badEvent.poke(false.B)
  dut.io.clear.poke(true.B)
  dut.clock.step()
  dut.io.ringBell.expect(false.B)
}
```

## Parametric Testing

```scala
class CounterTest extends AnyFlatSpec with ChiselScalatestTester {
  // Repeat the same test with multiple configurations
  for (n <- Seq(4, 7, 8, 13)) {
    s"Counter($n)" should s"count to ${n-1}" in {
      test(new WhenCounter(n)) { dut =>
        for (i <- 0 until 3 * n) {
          dut.io.cnt.expect((i % n).U)
          dut.clock.step()
        }
      }
    }
  }
}
```

## Trait-Based Test Reuse

```scala
trait CountTest {
  def testFn(dut: Counter): Unit = {
    val n = dut.io.cnt.getWidth  // Access parameter
    for (i <- 0 until 100) {
      dut.clock.step()
    }
  }
}

class WhenCounterTest extends AnyFlatSpec with ChiselScalatestTester with CountTest {
  "WhenCounter" should "count" in {
    test(new WhenCounter(10)) { dut => testFn(dut) }
  }
}
```

## printf Debugging

```scala
class MyDebugModule extends Module {
  val io = IO(new Bundle {
    val in = Input(UInt(8.W))
    val out = Output(UInt(8.W))
  })

  val reg = RegInit(0.U(8.W))
  reg := io.in
  io.out := reg

  // Prints every cycle during simulation
  printf("cycle: reg=%d, in=%d\n", reg, io.in)
}
```

## Waveform Generation (VCD)

```scala
test(new MyModule()).withAnnotations(Seq(WriteVcdAnnotation)) { dut =>
  // Test code...
  dut.clock.step(100)
}
// Generates .vcd file in test_run_dir/<test-name>/
```

## Fork — Parallel Thread Testing

```scala
test(new MyModule()) { dut =>
  // Forked thread: send data
  fork {
    for (i <- 0 until 10) {
      dut.io.in.poke(i.U)
      dut.clock.step()
    }
  }
  // Main thread: verify received data
  dut.clock.step(2)  // Pipeline delay
  for (i <- 0 until 10) {
    dut.io.out.expect(i.U)
    dut.clock.step()
  }
}
```

## Running Tests

```bash
sbt test                          # Run all tests
sbt "testOnly *MyModuleTest"      # Run a specific test class
sbt "testOnly *MyModuleTest -- -z \"should count\"" # Run a specific test case
```

## Gotchas

| Pitfall | Explanation |
|---------|-------------|
| `poke(3)` | Must use `poke(3.U)` — Chisel literals required |
| Immediate expect on sequential logic | Call `step()` before `expect` — registers update on the next cycle |
| `peekInt()` return type | Returns `BigInt`, not `Int` |
| Unnecessary step on combinational logic | Combinational logic reflects results immediately without `step()` |
| `println` vs `printf` in tests | `println` = Scala-side (host), `printf` = hardware-side (simulation) |
| Chisel 7.x API changes | `chiseltest` → `scalatest` + `ChiselSim`, significant API changes |
| VCD file location | Generated in the `test_run_dir/<test-name>/` directory |
| `fork` timing | Forked threads start at the same simulation time |
