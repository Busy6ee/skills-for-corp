---
name: xiangshan-debug
description: Use when debugging XiangShan simulations, adding performance counters (XSPerfAccumulate), using ChiselDB trace points, difftest ISA verification, or analyzing simulation logs and IPC regressions.
---

# XiangShan Debug — Debugging and Performance Analysis

## Difftest Framework

XiangShan verifies functional correctness through differential testing (difftest) against the **NEMU** (New EMUlator) reference model.

### How It Works

```
XiangShan RTL (emu)     NEMU (reference model)
       │                       │
  Instruction commit ──────→ Execute same instruction
       │                       │
  Architectural state ←──── Compare architectural state
       │                       │
  Mismatch detected → TRAP → Halt simulation + print error
```

### Core Bundles

```scala
// difftest/src/main/scala/DifftestBundle.scala
class DifftestArchIntRegState extends Bundle {
  val gpr = Vec(32, UInt(64.W))   // Integer registers x0~x31
}

class DifftestCSRState extends Bundle {
  val mstatus  = UInt(64.W)
  val mcause   = UInt(64.W)
  val mepc     = UInt(64.W)
  // ... all CSRs
}

class DifftestInstrCommit extends Bundle {
  val valid   = Bool()
  val pc      = UInt(64.W)
  val instr   = UInt(32.W)
  val wdest   = UInt(8.W)
  val wdata   = UInt(64.W)
  val skip    = Bool()    // Skip difftest comparison (e.g., MMIO)
}
```

### Running Difftest

```bash
# Run with difftest enabled
./build/emu -i test.bin --diff $NEMU_HOME/build/riscv64-nemu-interpreter-so

# Run with difftest disabled (performance measurement only)
./build/emu -i test.bin
```

## Performance Counters (PerfCounter)

### XSPerfAccumulate

Counts the number of event occurrences. (Defined in the `utility` submodule)

```scala
import utility._

class MyModule(implicit p: Parameters) extends XSModule {
  // Count cycles where condition is true
  XSPerfAccumulate("load_hit", io.loadResp.valid && io.loadResp.bits.hit)
  XSPerfAccumulate("load_miss", io.loadResp.valid && !io.loadResp.bits.hit)
  XSPerfAccumulate("store_blocked", io.storeReq.valid && !io.storeReq.ready)

  // Named counter sequence (overloaded variant)
  XSPerfAccumulate("events", valid, Seq(("typeA", condA), ("typeB", condB)))
}
```

### XSPerfHistogram

Collects value distributions as a histogram.

```scala
// (name, value, enable condition, start, end, step)
XSPerfHistogram("rob_occupancy", robOccupancy, true.B, 0, RobSize, 10)
XSPerfHistogram("iq_delay", issueDelay, io.issue.fire, 0, 64, 4)
```

### XSPerfRolling

Computes a rolling average over time intervals.

```scala
XSPerfRolling("ipc_rolling", commitCount, 1.U, clock, reset)
```

### Viewing Counter Output

```bash
# Automatically printed at simulation end
# Output format:
# [PERF ][time=xxxxx] module_name.counter_name, xxx
grep "\[PERF \]" sim.log | grep "load_hit"
```

## ChiselDB

Records microarchitectural events to a SQLite database during simulation.

```scala
import utility.ChiselDB

class MyModule extends XSModule {
  // 1. Define table
  val table = ChiselDB.createTable("MyEvents", new Bundle {
    val pc    = UInt(64.W)
    val event = UInt(8.W)
    val cycle = UInt(64.W)
  })

  // 2. Insert records
  when (eventOccurred) {
    table.log(
      data = Wire(new Bundle { ... }).asInstanceOf[...],
      en   = eventOccurred,
      site = "MyModule",
      clock = clock,
      reset = reset
    )
  }
}

// 3. Query SQLite after simulation
// sqlite3 chiseldb.db "SELECT * FROM MyEvents WHERE event = 1"
```

## Log Macros

### Level-Based Macros

```scala
import utils._

class MyModule extends XSModule {
  XSDebug(condition, "debug message: data=%x\n", data)   // Verbose debug
  XSInfo(condition, "info message: state=%d\n", state)    // Informational
  XSWarn(condition, "warning: queue full\n")               // Warning
  XSError(condition, "error: illegal state=%d\n", state)   // Error (may halt simulation)
}
```

### Log Output Format

```
[DEBUG][time=12345][MyModule] debug message: data=deadbeef
[INFO ][time=12345][MyModule] info message: state=3
[WARN ][time=12345][MyModule] warning: queue full
[ERROR][time=12345][MyModule] error: illegal state=7
```

### Log Filtering

```bash
# Filter by specific module
grep "\[MyModule\]" sim.log

# Filter by time range
awk '/\[time=1[0-9]{4}\]/' sim.log

# Errors only
grep "\[ERROR\]" sim.log
```

## Simulation Log Analysis

### TRAP Detection

TRAP messages printed when a difftest mismatch occurs:

```
[DIFF] pc: 0x80000100, instr: 0x00a00513
[DIFF] REF  reg[10] = 0x0000000000000005
[DIFF] DUT  reg[10] = 0x0000000000000003
[DIFF] first diff at pc = 0x80000100
```

### Analysis Procedure

1. **Identify TRAP PC**: Find the first mismatch PC from the `[DIFF] first diff at pc = ` line
2. **Compare registers**: Check REF (NEMU) vs DUT (XiangShan) value differences
3. **Backtrace**: Trace from the mismatched PC's instruction to source operands to producing instructions
4. **Inspect waveforms**: Examine pipeline state at cycles near the mismatch PC

### Useful grep Patterns

```bash
# Difftest mismatches
grep "\[DIFF\]" sim.log

# Trace committed instructions
grep "commitInstr" sim.log

# IPC calculation (total committed instructions / total cycles)
grep "total guest instructions" sim.log

# Trace execution of a specific PC
grep "pc=0x80001234" sim.log
```

For detailed grep patterns and error decoder, see [debug-cheatsheet.md](debug-cheatsheet.md).

## Waveform Debugging

### Generating Waveforms

```bash
# Full waveform (caution: generates very large files)
./build/emu -i test.bin --dump-wave

# Specific range only (recommended)
./build/emu -i test.bin --dump-wave -b 10000 -e 20000

# Output: build/*.vcd or build/*.fst
```

### Signal Hierarchy in Waveform Viewer

```
TOP
└── XSTop
    └── XSTile
        └── XSCore
            ├── frontend
            │   ├── bpu
            │   ├── ftq
            │   └── ifu
            ├── backend
            │   ├── ctrlBlock
            │   ├── issueQueues
            │   └── exeUnits
            └── memBlock
                ├── loadUnits
                ├── storeUnits
                └── dcache
```

### Key Debugging Signals

| Signal | Location | Purpose |
|--------|----------|---------|
| `io_redirect_valid` | Backend | When a redirect occurs |
| `rob_io_commits_*_valid` | ROB | Instruction commit confirmation |
| `ftq_io_enq_*` | FTQ | Fetch target enqueue |
| `dcache_io_lsu_*` | DCache | Cache access |
| `issueQueue_io_deq_*_valid` | IQ | Instruction issue |

## Gotchas

| Pitfall | Description |
|---------|-------------|
| `EnablePerfDebug` | XSPerfAccumulate is controlled via `PerfCounterOptionsKey`. Disable with `DISABLE_PERF=1` |
| `utility` submodule | XSPerfAccumulate/Histogram/Rolling are defined in the `utility/` submodule, not in the XiangShan main repo |
| ChiselDB activation | ChiselDB also requires a build flag. When disabled, `createTable` calls become no-ops |
| NEMU commit matching | NEMU version != XiangShan version can cause difftest false positives/negatives |
| Log size | `XSDebug` is `printf`-based. Large simulations generate **GB-scale logs** |
| `XSDebug` performance | printf slows simulation by 10x or more. Disable for benchmarking |
| Waveform file size | Full waveform dumps can be tens to hundreds of GB. Use `-b`/`-e` to limit the range |
| `skip` flag | MMIO and certain CSR accesses set `skip=true` to bypass difftest comparison. Related bugs are not caught by difftest |
| `EMU_THREADS > 1` | Multi-threaded simulation logs are **interleaved**. Reproduce with single thread for debugging |
