---
name: chisel-basics
description: Use when writing Chisel module definitions, IO ports, data types (UInt, SInt, Bool, Bundle, Vec), Wire/Reg declarations, or combinational logic (when/switch/Mux). Use as the primary Chisel syntax reference for code generation.
---

# Chisel Basics — 모듈, 타입, 조합논리

## Import

```scala
import chisel3._
import chisel3.util._  // DecoupledIO, switch/is, log2Ceil 등
```

## 모듈 정의 패턴

```scala
class MyModule extends Module {
  val io = IO(new Bundle {
    val in  = Input(UInt(8.W))
    val out = Output(UInt(8.W))
  })

  io.out := io.in
}
```

## 데이터 타입

### 기본 타입
```scala
UInt(8.W)    // 8비트 부호없는 정수
SInt(10.W)   // 10비트 부호있는 정수
Bool()       // 1비트 불리언
Bits(8.W)    // 8비트 (연산 불가, 거의 사용 안함)
```

### 상수 리터럴
```scala
0.U           // 폭 추론된 UInt 0
3.U(4.W)      // 4비트 UInt 3
-3.S          // 폭 추론된 SInt -3
true.B        // Bool true
false.B       // Bool false
"hff".U       // 16진수 255
"o377".U      // 8진수 255
"b1111_1111".U // 2진수 255 (언더스코어 가독성용)
'A'.U         // 문자 → UInt (65)
```

### 폭 지정
```scala
val n = 10
n.W            // 폭 객체
UInt(n.W)      // 파라미터 기반 폭
```

## Wire와 Reg

### Wire — 조합 신호
```scala
val w = Wire(UInt(8.W))     // 미초기화 Wire (반드시 할당 필요)
w := someValue

val wd = WireDefault(0.U)   // 기본값 있는 Wire (더 안전)
when (cond) {
  wd := 3.U
}
```

### Reg — 레지스터 (순차 논리)
```scala
val reg = Reg(UInt(4.W))         // 리셋 없는 레지스터
reg := inputVal

val regInit = RegInit(0.U(4.W))  // 리셋 값 있는 레지스터
regInit := inputVal

val regNext = RegNext(inputVal)  // 1사이클 지연 (가장 간결)
val regNextInit = RegNext(inputVal, 0.U)  // 리셋 값 포함

// Enable 레지스터
val enableReg = RegEnable(inputVal, enable)               // 리셋 없음
val enableRegInit = RegEnable(inputVal, 0.U(4.W), enable) // 리셋 포함
```

## Bundle — 복합 타입

```scala
// 정의
class Channel extends Bundle {
  val data = UInt(32.W)
  val valid = Bool()
}

// 사용
val ch = Wire(new Channel())
ch.data := 123.U
ch.valid := true.B

// Bundle을 IO로
val io = IO(new Bundle {
  val tx = Output(new Channel())
  val rx = Input(new Channel())
})
```

### Bundle 레지스터 초기화
```scala
// Wire로 초기값 설정 후 RegInit에 전달
val initVal = Wire(new Channel())
initVal.data := 0.U
initVal.valid := false.B
val channelReg = RegInit(initVal)
```

## Vec — 벡터/배열

```scala
// Wire Vec
val v = Wire(Vec(3, UInt(4.W)))
v(0) := 1.U
v(1) := 3.U
v(2) := 5.U
val elem = v(index)  // 동적 인덱싱

// Reg Vec (레지스터 파일)
val regFile = Reg(Vec(32, UInt(32.W)))
regFile(wrIdx) := wrData
val rdData = regFile(rdIdx)

// VecInit — 초기값으로 생성
val initVec = VecInit(1.U(3.W), 2.U, 3.U)
val fromSignals = VecInit(sigA, sigB, sigC)  // 기존 신호로 Vec

// RegInit + VecInit — 리셋 가능 Vec 레지스터
val resetRegFile = RegInit(VecInit(Seq.fill(32)(0.U(32.W))))

// Vec of Bundle
val vecBundle = Wire(Vec(8, new Channel()))

// Bundle containing Vec
class BundleWithVec extends Bundle {
  val field = UInt(8.W)
  val vector = Vec(4, UInt(8.W))
}
```

## 조합 논리

### when / elsewhen / otherwise
```scala
val w = Wire(UInt(4.W))
w := 0.U  // 기본값 (항상 먼저 할당)

when (condA) {
  w := 1.U
} .elsewhen (condB) {
  w := 2.U
} .otherwise {
  w := 3.U
}
```

### Mux
```scala
val result = Mux(sel, trueVal, falseVal)
```

### switch / is (ChiselEnum 또는 UInt 매칭)
```scala
import chisel3.util._

switch (selector) {
  is (0.U) { out := a }
  is (1.U) { out := b }
  is (2.U) { out := c }
}
```

## 비트 연산

```scala
// 비트 추출
val sign = x(31)         // 단일 비트
val lowByte = x(7, 0)    // 범위 추출

// 비트 연결
val word = highByte ## lowByte  // Cat 연산자

// 논리 연산
val and = a & b    // bitwise AND
val or  = a | b    // bitwise OR
val xor = a ^ b    // bitwise XOR
val inv = ~a       // bitwise NOT

// 산술 연산
val sum  = a + b
val diff = a - b
val neg  = -a
val prod = a * b
val quot = a / b
val rem  = a % b
```

## 타입 변환

```scala
val uint = sint.asUInt    // SInt → UInt
val sint = uint.asSInt    // UInt → SInt
val bool = uint.asBool    // UInt(1.W) → Bool
val uint = vecBool.asUInt // Vec[Bool] → UInt
val bundle = uint.asTypeOf(new MyBundle())  // UInt → Bundle (비트 재해석)
```

## 비교 연산

```scala
val eq  = a === b   // 하드웨어 동등 비교
val neq = a =/= b   // 하드웨어 부등 비교
val gt  = a > b
val gte = a >= b
val lt  = a < b
val lte = a <= b
```

## 부분 비트 할당 (Workaround)

Chisel은 `x(7,0) := y` 형태의 부분 할당을 지원하지 않음.

**해결책 1: Bundle 사용**
```scala
class Split extends Bundle {
  val high = UInt(8.W)
  val low = UInt(8.W)
}
val split = Wire(new Split())
split.low := lowByte
split.high := highByte
val combined = split.asUInt
```

**해결책 2: Vec[Bool] 사용**
```scala
val bits = Wire(Vec(4, Bool()))
bits(0) := data(0)
bits(1) := data(1)
bits(2) := data(2)
bits(3) := data(3)
val result = bits.asUInt
```

## 엣지 검출

```scala
val risingEdge  = din & !RegNext(din)    // 상승 엣지
val fallingEdge = !din & RegNext(din)    // 하강 엣지
```

## Gotchas

| 함정 | 올바른 사용 |
|------|------------|
| `UInt(8)` | `UInt(8.W)` — `.W` 필수 |
| `RegInit(0.U)` = 1비트 레지스터 | `RegInit(0.U(32.W))` — 폭 명시 |
| `==` (Scala 비교) | `===` (하드웨어 비교) |
| `!=` (Scala) | `=/=` (하드웨어) |
| Scala `if/else` → 하드웨어 | `when/otherwise` 또는 `Mux` 사용 |
| `Wire(UInt())` 미할당 | 반드시 모든 경로에서 할당하거나 `WireDefault` 사용 |
| `x(7,0) := y` 부분 할당 | Bundle 또는 Vec[Bool] workaround 필요 |
| `:=`는 마지막 연결 우선 | 여러 곳에서 같은 Wire에 할당하면 마지막만 유효 |
| `VecInit` 폭 추론 | 첫 번째 원소에 명시적 폭 지정: `VecInit(1.U(3.W), 2.U, 3.U)` |
| Bundle 레지스터 초기화 | `Wire`로 초기값 만든 후 `RegInit(wire)` |
| `Reg(Vec(...))` 리셋 없음 | 리셋 필요 시 `RegInit(VecInit(Seq.fill(n)(0.U)))` |
