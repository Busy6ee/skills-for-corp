# XiangShan Debug Cheatsheet

## Log Grep Pattern Collection

### Difftest Related

```bash
# First mismatch point
grep "first diff at pc" sim.log

# Detailed register mismatch
grep "\[DIFF\]" sim.log

# REF vs DUT comparison
grep -A2 "\[DIFF\] pc:" sim.log

# TRAP occurrence
grep "TRAP\|HIT GOOD TRAP\|HIT BAD TRAP" sim.log
```

### Performance Analysis

```bash
# All performance counter output
grep "\[PERF \]" sim.log

# Specific counter
grep "\[PERF \].*cache_miss" sim.log

# IPC related
grep "commitInstr\|total guest instructions\|total host time" sim.log

# Histogram output
grep "\[HIST \]" sim.log
```

### Pipeline Debug

```bash
# Committed instructions
grep "commit" sim.log | grep "valid=1"

# ROB related
grep "\[ROB\]" sim.log

# Redirect occurrence
grep "redirect\|Redirect" sim.log | grep "valid"

# Branch misprediction
grep "misPred\|mispred" sim.log

# Pipeline stalls
grep "stall\|blocked\|full" sim.log
```

### Memory Subsystem

```bash
# Cache misses
grep "miss\|Miss" sim.log | grep -i "dcache\|icache\|l2"

# TLB related
grep "tlb\|TLB\|ptw\|PTW" sim.log

# Store Buffer
grep "sbuffer\|Sbuffer" sim.log

# Prefetch
grep "prefetch\|Prefetch" sim.log
```

## Difftest Error Message Decoder

| Message | Meaning | Action |
|---------|---------|--------|
| `HIT GOOD TRAP` | Normal termination (ecall with a0=0) | Normal |
| `HIT BAD TRAP` | Abnormal termination (ecall with a0!=0) | Check test failure |
| `ABORT at pc = 0x...` | Halted due to difftest mismatch | Backtrace from mismatch PC |
| `[DIFF] pc: 0x...` | State mismatch at specific PC | Analyze the instruction |
| `[DIFF] REF reg[N] = X, DUT reg[N] = Y` | Register value mismatch | Trace the last write to register N |
| `[DIFF] REF csr[X] = A, DUT csr[X] = B` | CSR value mismatch | Check CSR update logic |
| `timeout after N cycles` | N cycles elapsed without instruction commit | Suspect deadlock/livelock |

## Debugging Workflows

### 1. Functional Bugs

```
TRAP occurs
  → grep "[DIFF]" → Identify first mismatch PC/register
  → objdump -d test.elf | grep <PC> → Identify the instruction
  → Dump waveform (-b <trap_cycle-1000> -e <trap_cycle+100>)
  → Trace the instruction's pipeline path in the waveform
  → If source operand is wrong → trace the producing instruction
  → If execution result is wrong → check FU logic
```

### 2. Performance Regression

```
IPC drop detected
  → Compare performance counters (before vs after)
  → Identify counters with the largest differences
  → Enable detailed logging for the relevant module
  → Check latency distribution via histograms
  → Analyze bottleneck intervals in waveforms
```

### 3. Deadlock

```
Timeout occurs
  → Dump waveform (last N cycles)
  → Check rob_io_commits signal → identify commit stall point
  → Inspect ROB head instruction → determine what failed to complete
  → Trace the corresponding FU/memory path
  → Check for circular dependencies (e.g., LSQ <-> DCache)
```

## Debug Build Options

```bash
# Enable performance counters
make emu CONFIG=DefaultConfig PERF=1

# Enable ChiselDB
make emu WITH_CHISELDB=1

# All debug features
make emu PERF=1 WITH_CHISELDB=1

# Release build (debug disabled, maximum speed)
make emu RELEASE=1
```
