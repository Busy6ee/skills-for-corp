# Component Catalog — 상세 레퍼런스

## UART 설계

### Tx (송신)

```
구조: Idle → Start bit → 8 data bits (LSB first) → Stop bit → Idle
클럭: 시스템 클럭을 보드레이트로 분주
보드레이트 계산: val divisor = (frequency + baudRate/2) / baudRate - 1
```

FSM 상태: `idle`, `start`, `data`, `stop`
- `idle`: txd = 1 (마크 상태), valid 입력 시 start로 전이
- `start`: txd = 0, 1비트 시간 후 data로
- `data`: txd = shiftReg(0), 8비트 전송 후 stop으로
- `stop`: txd = 1, 1비트 시간 후 idle로

### Rx (수신)

```
Start bit 검출 → 비트 중앙에서 샘플링 → 8비트 수신 → Stop bit 확인
```

비트 중앙 샘플링: `cntReg === (divisor/2)` 에서 시작, 이후 `divisor` 간격으로 샘플

## Adder 계열

| 타입 | 깊이 | 면적 | 용도 |
|------|------|------|------|
| CarryRipple | O(n) | O(n) | 소형, 저주파 |
| CarrySelect | O(√n) | O(2n) | 중형 |
| CarrySkip | O(√n) | O(n) | 중형, 면적 효율 |
| CarryLookahead | O(log n) | O(n log n) | 고속 |
| PrefixAdder (Brent-Kung) | O(log n) | O(n log n) | 최고속 |

```scala
// 추상 베이스
abstract class AbstractAdder(w: Int) extends Module {
  require(w > 0, "adders should have a positive width")
  val io = IO(new Bundle {
    val a   = Input(UInt(w.W))
    val b   = Input(UInt(w.W))
    val cin = Input(Bool())
    val sum = Output(UInt(w.W))
    val cout = Output(Bool())
  })
}
```

## Arbiter Tree

`reduceTree` + 2-way arbiter를 결합한 공정 중재기:

```scala
class ArbiterTree[T <: Data: Manifest](gen: T, n: Int) extends Module {
  val io = IO(new Bundle {
    val in  = Vec(n, Flipped(new DecoupledIO(gen)))
    val out = new DecoupledIO(gen)
  })
  // 2-way arbiter를 reduceTree로 트리 구성
  // FSM으로 last-served 추적하여 공정성 보장
}
```

## ALU 패턴

```scala
val alu = WireDefault(0.U(16.W))
switch(io.fn) {
  is(0.U) { alu := io.a + io.b }
  is(1.U) { alu := io.a - io.b }
  is(2.U) { alu := io.a | io.b }
  is(3.U) { alu := io.a & io.b }
}
io.result := alu
```
