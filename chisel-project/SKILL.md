---
name: chisel-project
description: Use when setting up a new Chisel project, configuring build.sbt, generating Verilog/SystemVerilog from Chisel, or integrating BlackBox Verilog modules. Also use when troubleshooting sbt dependency or version mismatch errors.
---

# Chisel Project Setup & Build

## Chisel 6.x build.sbt (Current Standard)

```scala
scalaVersion := "2.13.14"
val chiselVersion = "6.5.0"
addCompilerPlugin("org.chipsalliance" % "chisel-plugin" % chiselVersion cross CrossVersion.full)
libraryDependencies += "org.chipsalliance" %% "chisel" % chiselVersion
libraryDependencies += "edu.berkeley.cs" %% "chiseltest" % "6.0.0"
```

**Recommended scalacOptions:**
```scala
scalacOptions ++= Seq(
  "-deprecation",
  "-feature",
  "-unchecked",
  "-language:reflectiveCalls",
)
```

## Version History (Note the org Change)

| Version | org | Package Name | Notes |
|---------|-----|--------------|-------|
| 3.x | `edu.berkeley.cs` | `chisel3` | Legacy |
| 5.x | `org.chipsalliance` | `chisel` | org changed |
| 6.x | `org.chipsalliance` | `chisel` | Current standard |
| 7.x | `org.chipsalliance` | `chisel` | `chiseltest` → `scalatest` |

## Chisel 7.x build.sbt (Future)

```scala
scalaVersion := "2.13.18"
libraryDependencies += "org.chipsalliance" %% "chisel" % "7.5.0"
libraryDependencies += "org.scalatest" %% "scalatest" % "3.2.18" % Test
addCompilerPlugin("org.chipsalliance" % "chisel-plugin" % "7.5.0" cross CrossVersion.full)
Test / fork := true
Test / parallelExecution := false
```

## Verilog Generation

```scala
import chisel3._

// Method 1: Generate file (default path)
object MyDesign extends App {
  emitVerilog(new MyModule())
}

// Method 2: Specify output directory
object MyDesign extends App {
  emitVerilog(new MyModule(), Array("--target-dir", "generated"))
}

// Method 3: Return as string (for testing/debugging)
object MyDesign extends App {
  val verilog = getVerilogString(new MyModule())
  println(verilog)
}

// Method 4: Generate SystemVerilog
object MyDesign extends App {
  emitSystemVerilog(new MyModule())
}
```

## BlackBox — External Verilog Integration

### Three Approaches

**1. Inline — Write Verilog code directly in Scala:**
```scala
class InlineBlackBoxAdder extends HasBlackBoxInline {
  val io = IO(new BlackBoxAdderIO)
  setInline("InlineBlackBoxAdder.v",
    s"""
       |module InlineBlackBoxAdder(a, b, cin, c, cout);
       |input  [31:0] a, b;
       |input  cin;
       |output [31:0] c;
       |output cout;
       |wire   [32:0] sum;
       |assign sum  = a + b + {31'b0, cin};
       |assign c    = sum[31:0];
       |assign cout = sum[32];
       |endmodule
    """.stripMargin)
}
```

**2. Path — Reference a Verilog file within the project:**
```scala
class PathBlackBoxAdder extends HasBlackBoxPath {
  val io = IO(new BlackBoxAdderIO)
  addPath("./src/main/resources/PathBlackBoxAdder.v")
}
```

**3. Resource — Load from src/main/resources:**
```scala
class ResourceBlackBoxAdder extends HasBlackBoxResource {
  val io = IO(new BlackBoxAdderIO)
  addResource("/ResourceBlackBoxAdder.v")
}
```

### BlackBox vs ExtModule

```scala
// BlackBox: Module definition included in generated Verilog
class BUFGCE extends BlackBox(Map("SIM_DEVICE" -> "7SERIES")) {
  val io = IO(new Bundle {
    val I = Input(Clock())
    val CE = Input(Bool())
    val O = Output(Clock())
  })
}

// ExtModule: Module definition NOT included in generated Verilog (for external libraries)
class alt_inbuf extends ExtModule(
  Map("io_standard" -> "1.0 V",
    "location" -> "IOBANK_1")) {
  val io = IO(new Bundle {
    val i = Input(Bool())
    val o = Output(Bool())
  })
}
```

## Gotchas

| Gotcha | Description |
|--------|-------------|
| **Class name = Verilog module name** | The module name in `setInline` or inside the file referenced by `addPath` must exactly match the Scala class name |
| **IO port name case sensitivity** | Verilog is case-sensitive — BlackBox IO field names must exactly match the Verilog port names |
| **`addResource` path** | Must start with `/` — relative to `src/main/resources/` |
| **`addPath` path** | Relative to the project root |
| **BlackBox `io.` prefix** | BlackBox/ExtModule uses `val io = IO(...)` — port names do NOT get the `io_` prefix |
| **Parameter passing** | Pass via constructor using `Map("KEY" -> "VALUE")` format |
| **Simulation constraints** | Inline/Path/Resource BlackBoxes can only be simulated with Verilator/VCS (not Treadle) |
| **Chisel 3.x → 6.x migration** | org `edu.berkeley.cs` → `org.chipsalliance`, package name `chisel3` → `chisel` |

## sbt Commands

```bash
sbt test                          # Run all tests
sbt "testOnly *MyTestClass"       # Run specific test only
sbt run                           # Run App (generate Verilog)
sbt "runMain MyDesign"            # Run specific App
```
