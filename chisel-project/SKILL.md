---
name: chisel-project
description: Use when setting up a new Chisel project, configuring build.sbt, generating Verilog/SystemVerilog from Chisel, or integrating BlackBox Verilog modules. Also use when troubleshooting sbt dependency or version mismatch errors.
---

# Chisel Project Setup & Build

## Chisel 6.x build.sbt (현재 표준)

```scala
scalaVersion := "2.13.14"
val chiselVersion = "6.5.0"
addCompilerPlugin("org.chipsalliance" % "chisel-plugin" % chiselVersion cross CrossVersion.full)
libraryDependencies += "org.chipsalliance" %% "chisel" % chiselVersion
libraryDependencies += "edu.berkeley.cs" %% "chiseltest" % "6.0.0"
```

**scalacOptions 권장:**
```scala
scalacOptions ++= Seq(
  "-deprecation",
  "-feature",
  "-unchecked",
  "-language:reflectiveCalls",
)
```

## 버전 이력 (org 변경 주의)

| 버전 | org | 패키지명 | 비고 |
|------|-----|---------|------|
| 3.x | `edu.berkeley.cs` | `chisel3` | 레거시 |
| 5.x | `org.chipsalliance` | `chisel` | org 변경 |
| 6.x | `org.chipsalliance` | `chisel` | 현재 표준 |
| 7.x | `org.chipsalliance` | `chisel` | `chiseltest` → `scalatest` |

## Chisel 7.x build.sbt (향후)

```scala
scalaVersion := "2.13.18"
libraryDependencies += "org.chipsalliance" %% "chisel" % "7.5.0"
libraryDependencies += "org.scalatest" %% "scalatest" % "3.2.18" % Test
addCompilerPlugin("org.chipsalliance" % "chisel-plugin" % "7.5.0" cross CrossVersion.full)
Test / fork := true
Test / parallelExecution := false
```

## Verilog 생성

```scala
import chisel3._

// 방법 1: 파일 생성 (기본 경로)
object MyDesign extends App {
  emitVerilog(new MyModule())
}

// 방법 2: 출력 디렉토리 지정
object MyDesign extends App {
  emitVerilog(new MyModule(), Array("--target-dir", "generated"))
}

// 방법 3: 문자열로 반환 (테스트/디버깅용)
object MyDesign extends App {
  val verilog = getVerilogString(new MyModule())
  println(verilog)
}

// 방법 4: SystemVerilog 생성
object MyDesign extends App {
  emitSystemVerilog(new MyModule())
}
```

## BlackBox — 외부 Verilog 통합

### 3가지 방식

**1. Inline — Verilog 코드를 Scala 내 직접 작성:**
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

**2. Path — 프로젝트 내 Verilog 파일 참조:**
```scala
class PathBlackBoxAdder extends HasBlackBoxPath {
  val io = IO(new BlackBoxAdderIO)
  addPath("./src/main/resources/PathBlackBoxAdder.v")
}
```

**3. Resource — src/main/resources 에서 로드:**
```scala
class ResourceBlackBoxAdder extends HasBlackBoxResource {
  val io = IO(new BlackBoxAdderIO)
  addResource("/ResourceBlackBoxAdder.v")
}
```

### BlackBox vs ExtModule

```scala
// BlackBox: 생성된 Verilog에 모듈 정의 포함
class BUFGCE extends BlackBox(Map("SIM_DEVICE" -> "7SERIES")) {
  val io = IO(new Bundle {
    val I = Input(Clock())
    val CE = Input(Bool())
    val O = Output(Clock())
  })
}

// ExtModule: 생성된 Verilog에 모듈 정의 미포함 (외부 라이브러리용)
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

| 함정 | 설명 |
|------|------|
| **클래스명 = Verilog 모듈명** | `setInline`의 모듈명, `addPath`의 파일 내 모듈명이 Scala 클래스명과 정확히 일치해야 함 |
| **IO 포트명 대소문자** | Verilog는 case-sensitive — BlackBox IO 필드명이 Verilog 포트명과 정확히 일치해야 함 |
| **`addResource` 경로** | `/`로 시작 — `src/main/resources/` 기준 상대경로 |
| **`addPath` 경로** | 프로젝트 루트 기준 상대경로 |
| **BlackBox에 `io.` 프리픽스** | BlackBox/ExtModule은 `val io = IO(...)` 사용 — 포트명에 `io_` 프리픽스가 붙지 않음 |
| **파라미터 전달** | `Map("KEY" -> "VALUE")` 형식으로 생성자에 전달 |
| **시뮬레이션 제약** | Inline/Path/Resource BlackBox는 Verilator/VCS에서만 시뮬 가능 (Treadle 불가) |
| **Chisel 3.x → 6.x 마이그레이션** | org `edu.berkeley.cs` → `org.chipsalliance`, 패키지명 `chisel3` → `chisel` |

## sbt 명령어

```bash
sbt test                          # 전체 테스트
sbt "testOnly *MyTestClass"       # 특정 테스트만
sbt run                           # App 실행 (Verilog 생성)
sbt "runMain MyDesign"            # 특정 App 실행
```
