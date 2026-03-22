---
name: chisel-fsm
description: Use when designing finite state machines in Chisel, including Mealy/Moore machines, multi-state controllers, or FSMs communicating with datapaths (timers, counters). Use when the design requires state machine, FSM, or sequential control logic.
---

# Chisel FSM — Finite State Machine Design

## Basic FSM Pattern

```scala
import chisel3._
import chisel3.util._

class SimpleFsm extends Module {
  val io = IO(new Bundle {
    val badEvent = Input(Bool())
    val clear = Input(Bool())
    val ringBell = Output(Bool())
  })

  // 1. State definition (ChiselEnum)
  object State extends ChiselEnum {
    val green, orange, red = Value
  }
  import State._

  // 2. State register (with initial state)
  val stateReg = RegInit(green)

  // 3. Next-state logic
  switch (stateReg) {
    is (green) {
      when(io.badEvent) {
        stateReg := orange
      }
    }
    is (orange) {
      when(io.badEvent) {
        stateReg := red
      } .elsewhen(io.clear) {
        stateReg := green
      }
    }
    is (red) {
      when(io.clear) {
        stateReg := green
      }
    }
  }

  // 4. Output logic (Moore: depends only on state)
  io.ringBell := stateReg === red
}
```

## Moore vs Mealy

### Moore Machine — Output depends only on state
```scala
// Output logic is outside the switch block
switch (stateReg) {
  is (zero) {
    when(io.din) { stateReg := one }
  }
  is (one) {
    when(!io.din) { stateReg := zero }
  }
}
// Moore output: determined by state alone
io.risingEdge := stateReg === one
```

### Mealy Machine — Output depends on state + input
```scala
// Set default output value (important!)
io.risingEdge := false.B

switch (stateReg) {
  is (zero) {
    when(io.din) {
      stateReg := one
      io.risingEdge := true.B  // Mealy: output immediately on transition
    }
  }
  is (one) {
    when(!io.din) {
      stateReg := zero
    }
  }
}
```

**Key differences:**
| | Moore | Mealy |
|--|-------|-------|
| Output | Depends only on state | Depends on state + input |
| Timing | 1-cycle delay | Immediate response |
| Glitches | None | Possible on input changes |
| State count | More states | Fewer states |

## FSM + Datapath Integration

Pattern for combining FSMs with timers or counters:

```scala
class Flasher extends Module {
  val io = IO(new Bundle {
    val start = Input(Bool())
    val light = Output(Bool())
  })

  object State extends ChiselEnum {
    val off, flash1, space1, flash2, space2, flash3 = Value
  }
  import State._

  val stateReg = RegInit(off)

  // Timer (datapath)
  val timerReg = RegInit(0.U(16.W))
  val timerDone = timerReg === 0.U

  // FSM -> Datapath: control signals
  val timerLoad = WireDefault(false.B)
  val timerSelect = WireDefault(0.U(1.W))

  // Timer logic
  val timerVal = Mux(timerSelect === 0.U, 999.U, 499.U)
  when(timerLoad) {
    timerReg := timerVal
  } .elsewhen(timerReg =/= 0.U) {
    timerReg := timerReg - 1.U
  }

  // Default output
  io.light := false.B

  // FSM logic
  switch(stateReg) {
    is(off) {
      when(io.start) {
        stateReg := flash1
        timerLoad := true.B
      }
    }
    is(flash1) {
      io.light := true.B
      when(timerDone) {
        stateReg := space1
        timerLoad := true.B
        timerSelect := 1.U
      }
    }
    // ... remaining states
  }
}
```

## Counter-Based State Reduction

Pattern for reducing state count with a counter when states follow a repetitive pattern:

```scala
object State extends ChiselEnum {
  val off, flash, space = Value
}

val stateReg = RegInit(off)
val cntReg = RegInit(0.U(2.W))

switch(stateReg) {
  is(flash) {
    io.light := true.B
    when(timerDone) {
      stateReg := space
      timerLoad := true.B
    }
  }
  is(space) {
    when(timerDone) {
      when(cntReg === 2.U) {
        stateReg := off
        cntReg := 0.U
      } .otherwise {
        stateReg := flash
        cntReg := cntReg + 1.U
        timerLoad := true.B
      }
    }
  }
}
```

## Communicating FSMs — Connecting FSMs Together

```scala
// Timer FSM (reusable timer)
class TimerFsm(maxCount: Int) extends Module {
  val io = IO(new Bundle {
    val start = Input(Bool())
    val done = Output(Bool())
  })

  val cntReg = RegInit(0.U(log2Ceil(maxCount + 1).W))
  io.done := cntReg === 0.U

  when(io.start) {
    cntReg := maxCount.U
  } .elsewhen(cntReg =/= 0.U) {
    cntReg := cntReg - 1.U
  }
}

// Main FSM — uses TimerFsm
class MainController extends Module {
  val timer = Module(new TimerFsm(1000))

  // FSM -> Timer: control signal
  timer.io.start := false.B

  // Timer -> FSM: status signal
  when(timer.io.done) {
    stateReg := nextState
  }
}
```

## Gotchas

| Pitfall | Description |
|---------|-------------|
| **ChiselEnum placement** | Define inside the Module class or its companion object. Package-level definitions may cause issues |
| **Missing `import State._`** | You must `import State._` after the enum definition to use state names directly |
| **Missing default output value** | Mealy outputs require a default value before the `switch` block. Otherwise you get a "not fully initialized" error or latch inference |
| **Mealy glitches** | Mealy outputs are on combinational paths -- input glitches propagate to outputs. Use caution in asynchronous contexts |
| **No otherwise in switch** | `switch/is` does not support `otherwise` -- unmatched states retain their current value (register) |
| **when vs switch** | `switch` is for matching `ChiselEnum`/`UInt`. Use `when` for conditional branching within a state |
| **State encoding** | ChiselEnum uses binary encoding by default. One-hot encoding requires separate handling |
| **Timer off-by-one** | When detecting done with `timerReg === 0.U`, a load value of N means done fires after N+1 cycles |
| **WireDefault for control signals** | Declare FSM control outputs with `WireDefault(false.B)` -- only assign `true.B` inside the switch when needed |
