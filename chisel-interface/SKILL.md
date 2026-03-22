---
name: chisel-interface
description: Use when connecting Chisel modules together, using DecoupledIO (ready/valid handshaking), Flipped ports, bulk connections (<>), or designing bus interfaces and inter-module communication protocols.
---

# Chisel Interface — 모듈 연결 및 프로토콜

## DecoupledIO — Ready/Valid 핸드셰이킹

### 구조

```scala
// chisel3.util.DecoupledIO 내부 구조 (참고용)
class DecoupledIO[T <: Data](gen: T) extends Bundle {
  val ready = Input(Bool())    // 소비자 → 생산자
  val valid = Output(Bool())   // 생산자 → 소비자
  val bits  = Output(gen)      // 데이터
}
```

### 생산자 측 (Output)

```scala
val io = IO(new Bundle {
  val out = new DecoupledIO(UInt(8.W))  // 기본: valid/bits = Output, ready = Input
})
```

### 소비자 측 (Input) — Flipped

```scala
val io = IO(new Bundle {
  val in = Flipped(new DecoupledIO(UInt(8.W)))  // 방향 반전: valid/bits = Input, ready = Output
})
```

### Ready/Valid Buffer 예제

```scala
class ReadyValidBuffer extends Module {
  val io = IO(new Bundle {
    val in  = Flipped(new DecoupledIO(UInt(8.W)))
    val out = new DecoupledIO(UInt(8.W))
  })

  val dataReg  = Reg(UInt(8.W))
  val emptyReg = RegInit(true.B)

  io.in.ready  := emptyReg
  io.out.valid := !emptyReg
  io.out.bits  := dataReg

  when(emptyReg & io.in.valid) {
    dataReg  := io.in.bits
    emptyReg := false.B
  }

  when(!emptyReg & io.out.ready) {
    emptyReg := true.B
  }
}
```

### fire 신호

```scala
// 데이터 전송 성공 조건
val transferred = io.out.fire  // === io.out.valid && io.out.ready
```

## Flipped — 방향 반전

```scala
// 기본 Bundle
class Channel extends Bundle {
  val data  = Output(UInt(32.W))
  val valid = Output(Bool())
  val ready = Input(Bool())
}

// 생산자 IO
val producer = IO(new Channel())       // data/valid = Output, ready = Input

// 소비자 IO
val consumer = IO(Flipped(new Channel()))  // data/valid = Input, ready = Output
```

## Bulk Connection `<>`

같은 이름의 필드를 자동으로 연결:

```scala
// 모듈 간 연결
val producer = Module(new Producer())
val consumer = Module(new Consumer())
producer.io.out <> consumer.io.in  // 이름이 같은 필드 자동 매칭

// FIFO 체이닝
val buffers = Array.fill(depth) { Module(new Buffer()) }
for (i <- 0 until depth - 1) {
  buffers(i + 1).io.enq <> buffers(i).io.deq
}
io.enq <> buffers(0).io.enq
io.deq <> buffers(depth - 1).io.deq
```

## 파라메트릭 IO Bundle

```scala
class FifoIO[T <: Data](private val gen: T) extends Bundle {
  val enq = Flipped(new DecoupledIO(gen))
  val deq = new DecoupledIO(gen)
}

// 사용
class MyFifo[T <: Data](gen: T, depth: Int) extends Module {
  val io = IO(new FifoIO(gen))
  // ...
}
```

## 커스텀 프로토콜 Bundle

### 메모리 매핑 인터페이스

```scala
class MemoryMappedIO extends Bundle {
  val address = Input(UInt(16.W))
  val rd      = Input(Bool())
  val wr      = Input(Bool())
  val rdData  = Output(UInt(32.W))
  val wrData  = Input(UInt(32.W))
}
```

## 모듈 계층 구조

### 서브모듈 인스턴스화

```scala
class Top extends Module {
  val io = IO(new Bundle {
    val in  = Input(UInt(8.W))
    val out = Output(UInt(8.W))
  })

  // 서브모듈 생성
  val stage1 = Module(new Stage1())
  val stage2 = Module(new Stage2())

  // 수동 연결
  stage1.io.in := io.in
  stage2.io.in := stage1.io.out
  io.out := stage2.io.out
}
```

### Bulk connection으로 연결

```scala
class Pipeline extends Module {
  val fetch  = Module(new Fetch())
  val decode = Module(new Decode())

  fetch.io <> decode.io  // 이름 매칭 자동 연결
}
```

## 연결 연산자 비교

| 연산자 | 방향 | 동작 |
|--------|------|------|
| `:=` | 단방향 | LHS의 모든 필드에 RHS 할당 (마지막 연결 우선) |
| `<>` | 양방향 | 이름 매칭으로 자동 연결 (Input↔Output) |

## Gotchas

| 함정 | 설명 |
|------|------|
| **`<>` 이름 불일치** | 매칭되지 않는 필드는 **무시** (에러 아님) — 의도치 않은 미연결 주의 |
| **`Flipped()` 재귀** | Bundle 내 모든 방향을 재귀적으로 반전 — 혼합 방향 Bundle에서 주의 |
| **`ready` 미구동** | `DecoupledIO` 소비자는 반드시 `ready` 신호를 구동해야 함 (미구동 = 합성 에러) |
| **`fire` 사용 권장** | `io.out.valid && io.out.ready` 대신 `io.out.fire` 사용 |
| **`Module(new X())` 괄호** | `new X()` 주위 괄호 필수 — `Module(new X)` 도 동작하지만 인자 있을 때 혼동 |
| **`:=` 마지막 우선** | 같은 Wire에 여러 `:=` 시 마지막만 유효 |
| **`<>`는 마지막 우선 아님** | `<>`는 양방향 — 같은 포트에 여러 `<>` 시 에러 |
| **DecoupledIO 기본 방향** | `new DecoupledIO(gen)` → valid/bits=Output, ready=Input. 입력 측은 반드시 `Flipped()` |
| **Bundle 필드 방향** | IO에서 직접 `Input`/`Output` 쓰는 것과 `Flipped` 조합 주의 — 의도한 방향 확인 |
