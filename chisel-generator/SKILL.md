---
name: chisel-generator
description: Use when creating parametric or reusable hardware generators in Chisel, using Scala generics [T <: Data], constructor parameters, abstract classes, or functional programming patterns (reduce, reduceTree, map, zip) for hardware generation.
---

# Chisel Generator — 파라메트릭/재사용 하드웨어 생성기

## 생성자 파라미터

```scala
class WhenCounter(n: Int) extends Module {
  val io = IO(new Bundle {
    val cnt  = Output(UInt(8.W))
    val tick = Output(Bool())
  })

  val N = (n - 1).U
  val cntReg = RegInit(0.U(8.W))
  cntReg := cntReg + 1.U
  when(cntReg === N) {
    cntReg := 0.U
  }
  io.tick := cntReg === N
  io.cnt := cntReg
}

// 사용: Module(new WhenCounter(10))
```

### require()로 파라미터 검증

```scala
class CarryRippleAdder(w: Int = 32) extends Module {
  require(w > 0, "adders should have a positive width")
  // ...
}
```

## 추상 클래스 — 모듈 패밀리

```scala
// 추상 베이스
abstract class Counter(n: Int) extends Module {
  val io = IO(new Bundle {
    val cnt  = Output(UInt(8.W))
    val tick = Output(Bool())
  })
}

// 구현체들
class WhenCounter(n: Int) extends Counter(n) {
  val cntReg = RegInit(0.U(8.W))
  cntReg := cntReg + 1.U
  when(cntReg === (n-1).U) { cntReg := 0.U }
  io.tick := cntReg === (n-1).U
  io.cnt := cntReg
}

class MuxCounter(n: Int) extends Counter(n) {
  val cntReg = RegInit(0.U(8.W))
  cntReg := Mux(cntReg === (n-1).U, 0.U, cntReg + 1.U)
  io.tick := cntReg === (n-1).U
  io.cnt := cntReg
}
```

### 제네릭 FIFO 패턴

```scala
abstract class Fifo[T <: Data](gen: T, val depth: Int) extends Module {
  val io = IO(new FifoIO(gen))
  require(depth > 0, "Number of buffer elements needs to be larger than 0")
}

class BubbleFifo[T <: Data](gen: T, depth: Int) extends Fifo(gen, depth) {
  // 구현...
}

// 사용: Module(new BubbleFifo(UInt(8.W), 4))
// 또는: Module(new BubbleFifo(new MyBundle(), 8))
```

## 제네릭 타입 파라미터 `[T <: Data]`

```scala
// 제네릭 Mux 함수
def myMux[T <: Data](sel: Bool, tVal: T, fVal: T): T = {
  Mux(sel, tVal, fVal)
}
```

### Manifest 요구 (Vec/Module 생성 시)

```scala
class ArbiterTree[T <: Data: Manifest](gen: T, n: Int) extends Module {
  // [T <: Data: Manifest] — Vec(n, gen) 생성에 Manifest 필요
  val io = IO(new Bundle {
    val in  = Input(Vec(n, gen))
    val out = Output(gen)
  })
  // ...
}
```

## 함수 기반 하드웨어 생성기

### 기본: 함수 = 하드웨어 인스턴스

```scala
// 호출할 때마다 새로운 하드웨어 생성
def adder(x: UInt, y: UInt) = x + y

val sum1 = adder(a, b)  // 가산기 1 생성
val sum2 = adder(c, d)  // 가산기 2 생성 (별개 하드웨어)
```

### 파이프라인 지연

```scala
def delay(x: UInt) = RegNext(x)

// 2단 파이프라인
val delayed2 = delay(delay(din))
```

### 튜플 반환

```scala
def compare(a: UInt, b: UInt) = {
  val equ = a === b
  val gt = a > b
  (equ, gt)
}

// 사용법 1: 튜플 접근
val cmp = compare(inA, inB)
val equResult = cmp._1
val gtResult = cmp._2

// 사용법 2: 구조 분해
val (equ, gt) = compare(inA, inB)
```

### 카운터 생성 함수

```scala
def genCounter(n: Int) = {
  val cntReg = RegInit(0.U(8.W))
  cntReg := Mux(cntReg === n.U, 0.U, cntReg + 1.U)
  cntReg
}

// 여러 독립 카운터 간단 생성
val count10 = genCounter(10)
val count99 = genCounter(99)
```

## 함수형 프로그래밍 패턴

### reduce — 선형 체인

```scala
// Vec의 모든 원소를 더하는 선형 체인
val sum = vec.reduce(_ + _)

// 동일: vec(0) + vec(1) + vec(2) + ...
// 깊이: O(n) — 타이밍 비최적
```

### reduceTree — 균형 트리

```scala
// Vec의 모든 원소를 더하는 균형 트리
val sum = vec.reduceTree(_ + _)

// 깊이: O(log n) — 타이밍 최적
```

### 최솟값 찾기

```scala
val min = vec.reduceTree((x, y) => Mux(x < y, x, y))
```

### 최솟값 + 인덱스 (Bundle + reduceTree)

```scala
class ValIdx extends Bundle {
  val v = UInt(w.W)
  val idx = UInt(8.W)
}

val vecTwo = Wire(Vec(n, new ValIdx()))
for (i <- 0 until n) {
  vecTwo(i).v := vec(i)
  vecTwo(i).idx := i.U
}

val result = vecTwo.reduceTree((x, y) => Mux(x.v < y.v, x, y))
// result.v = 최솟값, result.idx = 인덱스
```

### zipWithIndex + map + reduce

```scala
val (minVal, minIdx) = vec.zipWithIndex
  .map(x => (x._1, x._2.U))
  .reduce((x, y) => (
    Mux(x._1 < y._1, x._1, y._1),
    Mux(x._1 < y._1, x._2, y._2)
  ))
```

## Optional IO 포트

```scala
class RegisterFile(debug: Boolean = false) extends Module {
  val io = IO(new Bundle {
    val rdData = Output(UInt(32.W))
    val debugPort = if (debug) Some(Output(Vec(32, UInt(32.W)))) else None
  })
  // debug 포트 사용
  if (debug) {
    io.debugPort.get := regFile
  }
}
```

## Gotchas

| 함정 | 설명 |
|------|------|
| **함수 호출 = 새 하드웨어** | `adder(x,y)`를 두 번 호출하면 **2개** 가산기 생성. 공유 불가 |
| **`reduce` vs `reduceTree`** | `reduce` = O(n) 깊이 (선형), `reduceTree` = O(log n) (균형). 타이밍 크리티컬 경로는 항상 `reduceTree` |
| **`[T <: Data: Manifest]`** | Vec/Module 내부에서 제네릭 타입 사용 시 `: Manifest` 컨텍스트 바운드 필수 |
| **기본값 제공** | 재사용 모듈에는 `class Foo(w: Int = 32)` 처럼 합리적 기본값 |
| **`require()` 위치** | 클래스 본문 맨 앞에 배치 — 잘못된 파라미터를 합성 전에 차단 |
| **abstract class vs trait** | 생성자 파라미터 필요 → abstract class. 믹스인 → trait |
| **`WireDefault` vs `cloneType`** | 타입 복제 필요 시 `Wire(gen.cloneType)` 또는 `WireDefault(gen)` |
| **`isPow2` 유틸** | `chisel3.util.isPow2(w)` — 2의 거듭제곱 검증에 사용 |
