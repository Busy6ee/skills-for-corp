---
name: chisel-memory
description: Use when implementing SRAM, register files, or any memory structure in Chisel. Use when dealing with SyncReadMem, read-during-write behavior, memory forwarding, dual-port memory, or multi-port register file access patterns.
---

# Chisel Memory — 메모리 및 레지스터 파일

## SyncReadMem — 기본 SRAM

```scala
class Memory extends Module {
  val io = IO(new Bundle {
    val rdAddr = Input(UInt(10.W))
    val rdData = Output(UInt(8.W))
    val wrAddr = Input(UInt(10.W))
    val wrData = Input(UInt(8.W))
    val wrEna  = Input(Bool())
  })

  val mem = SyncReadMem(1024, UInt(8.W))

  io.rdData := mem.read(io.rdAddr)

  when(io.wrEna) {
    mem.write(io.wrAddr, io.wrData)
  }
}
```

## Read-During-Write 포워딩

동일 주소에 동시 읽기/쓰기 시 최신 데이터를 반환하는 패턴:

```scala
class ForwardingMemory extends Module {
  val io = IO(new Bundle {
    val rdAddr = Input(UInt(10.W))
    val rdData = Output(UInt(8.W))
    val wrAddr = Input(UInt(10.W))
    val wrData = Input(UInt(8.W))
    val wrEna  = Input(Bool())
  })

  val mem = SyncReadMem(1024, UInt(8.W))

  // 포워딩 조건과 데이터를 1사이클 지연
  val wrDataReg = RegNext(io.wrData)
  val doForwardReg = RegNext(io.wrAddr === io.rdAddr && io.wrEna)

  val memData = mem.read(io.rdAddr)

  when(io.wrEna) {
    mem.write(io.wrAddr, io.wrData)
  }

  // 포워딩 or 메모리 데이터 선택
  io.rdData := Mux(doForwardReg, wrDataReg, memData)
}
```

## WriteFirst 모드

```scala
// SyncReadMem에 WriteFirst 명시
val mem = SyncReadMem(1024, UInt(8.W), SyncReadMem.WriteFirst)
```

WriteFirst: 동일 주소 동시 접근 시 쓰기 데이터가 읽기에 반영. 수동 포워딩 불필요.

## True Dual-Port Memory

```scala
class TrueDualPortMemory extends Module {
  val io = IO(new Bundle {
    val addrA   = Input(UInt(10.W))
    val rdDataA = Output(UInt(8.W))
    val wrEnaA  = Input(Bool())
    val wrDataA = Input(UInt(8.W))
    val addrB   = Input(UInt(10.W))
    val rdDataB = Output(UInt(8.W))
    val wrEnaB  = Input(Bool())
    val wrDataB = Input(UInt(8.W))
  })

  val mem = SyncReadMem(1024, UInt(8.W))

  // Port A
  io.rdDataA := mem.read(io.addrA)
  when(io.wrEnaA) {
    mem.write(io.addrA, io.wrDataA)
  }

  // Port B
  io.rdDataB := mem.read(io.addrB)
  when(io.wrEnaB) {
    mem.write(io.addrB, io.wrDataB)
  }
}
```

## Register File — Reg(Vec) 기반

```scala
// 기본 레지스터 파일 (리셋 없음)
val registerFile = Reg(Vec(32, UInt(32.W)))
registerFile(wrIdx) := wrData
val rdData = registerFile(rdIdx)

// 리셋 가능 레지스터 파일
val resetRegFile = RegInit(VecInit(Seq.fill(32)(0.U(32.W))))
```

### Optional 디버그 포트

```scala
class RegisterFile(debug: Boolean = false) extends Module {
  val io = IO(new Bundle {
    val rdAddr = Input(UInt(5.W))
    val rdData = Output(UInt(32.W))
    val wrAddr = Input(UInt(5.W))
    val wrData = Input(UInt(32.W))
    val wrEna  = Input(Bool())
    // 조건부 포트
    val debugPort = if (debug) Some(Output(Vec(32, UInt(32.W)))) else None
  })

  val regFile = Reg(Vec(32, UInt(32.W)))

  io.rdData := regFile(io.rdAddr)
  when(io.wrEna) {
    regFile(io.wrAddr) := io.wrData
  }

  if (debug) {
    io.debugPort.get := regFile
  }
}
```

## 파일 초기화

```scala
import chisel3.util.experimental.loadMemoryFromFileInline

val mem = SyncReadMem(1024, UInt(8.W))
loadMemoryFromFileInline(
  mem,
  "./src/main/resources/init.hex",
  firrtl.annotations.MemoryLoadFileType.Hex
)
```

### Scala에서 hex 파일 동적 생성

```scala
val hello = "Hello, World!"
val helloHex = hello.map(_.toInt.toHexString).mkString("\n")
val file = new java.io.PrintWriter("hello.hex")
file.write(helloHex)
file.close()

val mem = SyncReadMem(1024, UInt(8.W))
loadMemoryFromFileInline(mem, "hello.hex", firrtl.annotations.MemoryLoadFileType.Hex)
```

## 멀티클럭 메모리

```scala
class MultiClockMemory extends Module {
  val io = IO(new Bundle {
    val clkB    = Input(Bool())  // 외부 클럭을 Bool로 입력
    val rdAddr  = Input(UInt(10.W))
    val rdData  = Output(UInt(8.W))
    val wrAddr  = Input(UInt(10.W))
    val wrData  = Input(UInt(8.W))
    val wrEna   = Input(Bool())
  })

  val mem = SyncReadMem(1024, UInt(8.W))

  // 기본 클럭 도메인에서 읽기
  io.rdData := mem.read(io.rdAddr)

  // 다른 클럭 도메인에서 쓰기
  withClock(io.clkB.asClock) {
    when(io.wrEna) {
      mem.write(io.wrAddr, io.wrData)
    }
  }
}
```

## SyncReadMem vs Reg(Vec) 선택 가이드

| | SyncReadMem | Reg(Vec) |
|--|------------|----------|
| **합성 결과** | BRAM/Block RAM | FF/분산 RAM |
| **읽기 레이턴시** | 1사이클 (동기 읽기) | 0사이클 (조합 읽기) |
| **적합한 크기** | 큰 메모리 (>64 entries) | 작은 메모리 (<64 entries) |
| **포트 수** | 제한적 (FPGA 리소스 의존) | 자유로움 (FF 기반) |
| **리셋** | 지원 안함 (file init만) | `RegInit(VecInit(...))` |
| **read-during-write** | 명시적 처리 필요 | 자동 (조합 읽기) |

## Gotchas

| 함정 | 설명 |
|------|------|
| **SyncReadMem 1사이클 레이턴시** | `mem.read(addr)` 결과는 **다음** 클럭 엣지에서 유효. 같은 사이클에서 읽기 불가 |
| **기본 read-during-write 미정의** | 동일 주소 동시 R/W 시 결과 미정의. 반드시 포워딩 구현 또는 `WriteFirst` 사용 |
| **듀얼포트 합성 제한** | `SyncReadMem` 듀얼포트가 모든 FPGA에서 Block RAM으로 합성되지 않음 (Cyclone V에서 FF로 합성된 사례) |
| **`loadMemoryFromFileInline` 백엔드** | FIRRTL annotation 기반 — 모든 합성 백엔드에서 지원되지 않을 수 있음 |
| **RegFile 읽기는 조합** | `Reg(Vec)` 읽기는 같은 사이클 (조합), `SyncReadMem`은 다음 사이클 — 파이프라인 설계 시 차이 고려 |
| **메모리 크기 → 리소스** | 32 entry 이하: `Reg(Vec)` 권장. 그 이상: `SyncReadMem`으로 BRAM 활용 |
| **`.asClock` 안전성** | `Bool.asClock`는 CDC(Clock Domain Crossing) 처리 없음 — 동기화 로직 별도 필요 |
