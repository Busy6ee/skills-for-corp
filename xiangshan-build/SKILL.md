---
name: xiangshan-build
description: Use when building XiangShan RTL, running simulations (emu/VCS/xsim), generating Verilog, configuring build options, or setting up the XiangShan development environment. Also use when troubleshooting build or simulation errors.
---

# XiangShan Build — Build System and Simulation

## Build System Overview

```
build.mill (Mill)           ← Scala build (RTL generation)
    ↓                        Chisel 7.3.0 + Scala 2.13.17
Makefile                    ← Simulation build wrapper
    ↓
emu / simv / xsim           ← Simulator binaries
```

> **Mill ≠ sbt**. XiangShan uses `build.mill`, not `build.sbt`.
> Uses **Chisel 7.3.0** — differs from chisel-book skill's Chisel 6.5.0.

## Environment Setup

### Required Tools

| Tool | Version Requirement | Notes |
|------|---------------------|-------|
| Java (JDK) | 11+ | Required to run Mill |
| Mill | 0.12.3 | Scala build tool |
| Verilator | 5.008+ (recommended) | `make emu` — version-sensitive |
| GCC/G++ | 11+ | Verilator C++ compilation |
| GNU Make | 4.0+ | Makefile build |
| NEMU | Commit-matched to XiangShan | difftest reference model |
| RISC-V Toolchain | GCC 12+ | For compiling workloads |

### Initial Setup

```bash
# 1. Clone (including submodules)
git clone --recursive https://github.com/OpenXiangShan/XiangShan.git
cd XiangShan

# 2. Initialize submodules (skip if --recursive was used above)
git submodule update --init --recursive

# 3. Verify Mill installation
./mill version   # or: mill version

# 4. Build NEMU (for difftest)
cd ../NEMU
make riscv64-xs-ref_defconfig
make -j$(nproc)
# → Produces build/riscv64-nemu-interpreter-so
```

## Verilog Generation

```bash
# Generate Verilog (default output: SystemVerilog)
make verilog

# Generate with a specific Config
make verilog CONFIG=MinimalConfig

# Default CHISEL_TARGET is systemverilog
# ISSUE=E.b (CHI issue version)

# Output location: build/ directory
# Internally calls: mill xiangshan.runMain top.TopMain
```

## Simulation Build

### Verilator (emu)

```bash
# Default build (single-threaded)
make emu

# Multi-threaded build (caution: may be non-deterministic)
make emu EMU_THREADS=4

# Specific Config
make emu CONFIG=MinimalConfig

# Include DRAMSim3 memory model
make emu WITH_DRAMSIM3=1

# Combined build options
make emu CONFIG=MinimalConfig EMU_THREADS=2 MFC=1

# Build output
# build/emu
```

### VCS (Synopsys)

```bash
make simv              # Build VCS simulator
# Build output: build/simv
```

### Xilinx xsim

```bash
make xsim              # Build xsim simulator
```

## Running Simulations

### Basic Execution

```bash
# Run a workload
./build/emu -i <workload.bin>

# Example: CoreMark
./build/emu -i ready-to-run/coremark-2-iteration.bin
```

### Key Runtime Flags

| Flag | Description | Example |
|------|-------------|---------|
| `-i <file>` | Workload binary | `-i test.bin` |
| `-b <n>` | Start cycle (begin log/wave recording) | `-b 10000` |
| `-e <n>` | End cycle (stop log/wave recording) | `-e 50000` |
| `--diff <so>` | Path to difftest reference model SO | `--diff ../NEMU/build/riscv64-nemu-interpreter-so` |
| `--dump-wave` | Enable waveform dump | `--dump-wave` |
| `-I <n>` | Terminate after max instruction count | `-I 1000000` |
| `--seed <n>` | Specify random seed (for reproducibility) | `--seed 42` |

### Execution Examples

```bash
# Run with difftest enabled
./build/emu -i test.bin --diff ../NEMU/build/riscv64-nemu-interpreter-so

# Dump waveform for a specific interval
./build/emu -i test.bin --dump-wave -b 5000 -e 10000

# Run with instruction count limit
./build/emu -i test.bin -I 500000 --diff $NEMU_HOME/build/riscv64-nemu-interpreter-so
```

## Using Mill Directly

```bash
# Compile RTL (Chisel only, no Verilog generation)
mill -i XiangShan.compile

# Run tests
mill -i XiangShan.test

# Run a specific test
mill -i XiangShan.test.testOnly <TestClassName>

# List available targets
mill resolve XiangShan._
```

## Submodule Management

```bash
# Update all submodules
git submodule update --init --recursive

# Update a specific submodule only
git submodule update --init huancun
git submodule update --init difftest

# Check submodule status
git submodule status
```

> Submodules are pinned to specific commits. Manually checking out a different commit instead of using `git submodule update` will cause build failures.

## Config Selection Guide

| Config | Use Case | Build Time | Notes |
|--------|----------|------------|-------|
| `DefaultConfig` | Full-featured, benchmarking | 30 min+ | Default |
| `MinimalConfig` | Fast iteration, functional verification | 10~15 min | Reduced caches, unrealistic IPC |
| Custom Config | DSE, specific experiments | Varies | Add to `Configs.scala` |

### Creating a Custom Config

```scala
// Add to src/main/scala/top/Configs.scala
class MyExperimentConfig extends Config(
  new WithNKBL2(256) ++      // L2 256KB
  new DefaultConfig
)
```

```bash
make emu CONFIG=MyExperimentConfig
```

## Build Troubleshooting

### Common Patterns

```bash
# Clean build (delete build cache)
make clean
make emu

# Delete Mill cache only
rm -rf out/

# If Java runs out of heap space
export JAVA_OPTS="-Xmx16g -Xss256m"
make emu
```

## Gotchas

| Pitfall | Description |
|---------|-------------|
| Mill ≠ sbt | Not `sbt compile`. Use `mill -i XiangShan.compile` |
| Verilator version | Build succeeds only with specific Verilator versions. Version mismatch causes compilation errors |
| NEMU commit matching | XiangShan/NEMU version mismatch causes difftest failures. Always use the correct NEMU branch/commit |
| `make emu` time | First build takes 30+ min (Verilator C++ compilation). Use `MinimalConfig` to reduce build time |
| `EMU_THREADS > 1` | Multi-threaded simulation is **non-deterministic**. Use `EMU_THREADS=1` for reproducible debugging |
| Submodule mismatch | Forgetting `git submodule update` after `git pull` is the **#1 cause of build failures** |
| Java heap | Large Config builds may trigger OOM. Set `JAVA_OPTS="-Xmx16g"` |
| `build/` directory | Location of build artifacts. Included in `.gitignore`. Safe to delete manually |
| `MFC=1` | Uses MLIR FIRRTL Compiler for faster FIRRTL processing. Default is legacy FIRRTL |
| Docker | XiangShan provides a Dockerfile for consistent environment setup |
