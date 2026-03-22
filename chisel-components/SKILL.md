---
name: chisel-components
description: Use when implementing standard digital building blocks in Chisel — FIFOs, UARTs, counters, adders, arbiters, shift registers, encoders/decoders, or input processing (debounce, synchronizer, edge detection). Use as a pattern library for common hardware components.
---

# Chisel Components — 표준 빌딩블록 라이브러리

## FIFO 설계 패턴

### 추상 베이스 + 파라메트릭 IO

```scala
class FifoIO[T <: Data](private val gen: T) extends Bundle {
  val enq = Flipped(new DecoupledIO(gen))
  val deq = new DecoupledIO(gen)
}

abstract class Fifo[T <: Data](gen: T, val depth: Int) extends Module {
  val io = IO(new FifoIO(gen))
  require(depth > 0, "Number of buffer elements needs to be larger than 0")
}
```

### BubbleFifo — 가장 단순

```scala
class BubbleFifo[T <: Data](gen: T, depth: Int) extends Fifo(gen, depth) {
  private class Buffer extends Module {
    val io = IO(new FifoIO(gen))
    val fullReg = RegInit(false.B)
    val dataReg = Reg(gen)

    when(fullReg) {
      when(io.deq.ready) { fullReg := false.B }
    }.otherwise {
      when(io.enq.valid) {
        fullReg := true.B
        dataReg := io.enq.bits
      }
    }
    io.enq.ready := !fullReg
    io.deq.valid := fullReg
    io.deq.bits := dataReg
  }

  private val buffers = Array.fill(depth) { Module(new Buffer()) }
  for (i <- 0 until depth - 1) {
    buffers(i + 1).io.enq <> buffers(i).io.deq
  }
  io.enq <> buffers(0).io.enq
  io.deq <> buffers(depth - 1).io.deq
}
```

### RegFifo — 포인터 기반 순환 버퍼

```scala
// Reg(Vec)로 구현, read/write 포인터 사용
// counter 헬퍼 함수:
def counter(depth: Int, incr: Bool): (UInt, UInt) = {
  val cntReg = RegInit(0.U(log2Ceil(depth).W))
  val nextVal = Mux(cntReg === (depth-1).U, 0.U, cntReg + 1.U)
  when(incr) { cntReg := nextVal }
  (cntReg, nextVal)
}
```

### MemFifo — SyncReadMem 기반 (대용량)

SyncReadMem + WriteFirst로 대용량 FIFO 구현. 1사이클 읽기 레이턴시 처리를 위해 출력 레지스터 사용.

### FIFO 선택 가이드

| FIFO 타입 | 처리량 | 리소스 | 적합한 용도 |
|-----------|--------|--------|------------|
| BubbleFifo | 1/2N cycles | FF | 소형, 간단 |
| DoubleBufferFifo | 1/cycle | FF | 중형, 파이프라인 |
| RegFifo | 1/cycle | FF | 중형, 범용 |
| MemFifo | 1/cycle | BRAM | 대형 |
| CombFifo | 1/cycle | BRAM+FF | 대형, 최적 |

## 카운터 패턴

### When 기반 카운터

```scala
val cntReg = RegInit(0.U(8.W))
cntReg := cntReg + 1.U
when(cntReg === (n-1).U) {
  cntReg := 0.U
}
```

### Mux 기반 카운터 (1줄)

```scala
val cntReg = RegInit(0.U(8.W))
cntReg := Mux(cntReg === (n-1).U, 0.U, cntReg + 1.U)
```

### 다운 카운터

```scala
val cntReg = RegInit((n-1).U(8.W))
cntReg := cntReg - 1.U
when(cntReg === 0.U) {
  cntReg := (n-1).U
}
val tick = cntReg === 0.U
```

### 함수 기반 카운터 생성기

```scala
def genCounter(n: Int) = {
  val cntReg = RegInit(0.U(8.W))
  cntReg := Mux(cntReg === n.U, 0.U, cntReg + 1.U)
  cntReg
}
val count10 = genCounter(10)
val count99 = genCounter(99)
```

### Tick 생성기 (클럭 분주)

```scala
val tickReg = RegInit(0.U(32.W))
val tick = tickReg === (N-1).U
tickReg := tickReg + 1.U
when(tick) { tickReg := 0.U }

// tick 기반 저주파 카운터
val lowFreqCnt = RegInit(0.U(4.W))
when(tick) {
  lowFreqCnt := lowFreqCnt + 1.U
}
```

## 입력 처리 체인

### 1. 동기화 (메타스태빌리티 방지)

```scala
val btnSync = RegNext(RegNext(btn))  // 2단 FF 동기화
```

### 2. 디바운스

```scala
val btnDebReg = RegInit(false.B)
val cntReg = RegInit(0.U(32.W))
val tick = cntReg === (fac-1).U
cntReg := cntReg + 1.U
when(tick) {
  cntReg := 0.U
  btnDebReg := btnSync  // tick 주기마다 샘플링
}
```

### 3. 다수결 필터링

```scala
val shiftReg = RegInit(0.U(3.W))
when(tick) {
  shiftReg := shiftReg(1,0) ## btnDebReg
}
val btnClean = (shiftReg(2) & shiftReg(1)) |
               (shiftReg(2) & shiftReg(0)) |
               (shiftReg(1) & shiftReg(0))
```

### 4. 엣지 검출

```scala
val risingEdge = btnClean & !RegNext(btnClean)
```

### 함수형 입력 처리 (재사용)

```scala
def sync(v: Bool) = RegNext(RegNext(v))
def rising(v: Bool) = v & !RegNext(v)

def tickGen(fac: Int) = {
  val reg = RegInit(0.U(log2Up(fac).W))
  val tick = reg === (fac-1).U
  reg := Mux(tick, 0.U, reg + 1.U)
  tick
}

def filter(v: Bool, t: Bool) = {
  val reg = RegInit(0.U(3.W))
  when(t) { reg := reg(1,0) ## v }
  (reg(2) & reg(1)) | (reg(2) & reg(0)) | (reg(1) & reg(0))
}

// 조합:
val btnSync = sync(io.btnU)
val tick = tickGen(100000000/100)
val btnDeb = RegInit(false.B)
when(tick) { btnDeb := btnSync }
val btnClean = filter(btnDeb, tick)
val pressed = rising(btnClean)
```

## 시프트 레지스터

### Serial-In / Parallel-Out

```scala
val shiftReg = RegInit(0.U(4.W))
shiftReg := shiftReg(2, 0) ## io.din  // 왼쪽 시프트, 새 비트 LSB
val parallelOut = shiftReg
```

### Parallel-Load / Serial-Out

```scala
val shiftReg = Reg(UInt(4.W))
when(io.load) {
  shiftReg := io.parallelIn
} .otherwise {
  shiftReg := 0.U ## shiftReg(3, 1)  // 오른쪽 시프트
}
io.dout := shiftReg(0)
```

## 인코더 / 디코더

### 디코더 (1-of-N)

```scala
val decoded = 1.U << io.sel  // sel 비트 위치만 1
```

### 인코더 (Priority)

```scala
// switch 기반
val encoded = WireDefault(0.U)
switch(io.input) {
  is("b0001".U) { encoded := 0.U }
  is("b0010".U) { encoded := 1.U }
  is("b0100".U) { encoded := 2.U }
  is("b1000".U) { encoded := 3.U }
}
```

## Gotchas

| 함정 | 설명 |
|------|------|
| **BubbleFifo 처리량** | 버퍼 단계당 2사이클/원소 — 파이프라인 처리량이 아님 |
| **동기화 2FF** | `RegNext(RegNext(btn))` 필수 — 1FF는 메타스태빌리티 미해결 |
| **디바운스 샘플링 주기** | 시스템 클럭을 적절히 분주. 예: 100MHz/100 = 1MHz |
| **MemFifo WriteFirst** | SyncReadMem에 `SyncReadMem.WriteFirst` 필수 — 없으면 R/W 해저드 |
| **다운카운터 tick 위치** | `RegInit(N)` → 0까지 카운트 → tick은 0에서 발생 |
| **디코더 폭** | `1.U << sel` 결과 폭이 sel 폭에 의존 — 명시적 폭 절단 필요할 수 있음 |
| **FIFO depth** | BubbleFifo는 depth = 버퍼 수. RegFifo/MemFifo는 depth = 엔트리 수 |
