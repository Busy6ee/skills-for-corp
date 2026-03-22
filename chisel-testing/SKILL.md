---
name: chisel-testing
description: Use when writing ChiselTest test benches, using poke/expect/step/peek operations, or debugging Chisel module behavior through simulation. Use when test compilation or simulation errors occur in Chisel designs.
---

# Chisel Testing — ChiselTest 패턴

## 기본 구조 (Chisel 6.x + chiseltest 6.0.0)

```scala
import chisel3._
import chiseltest._
import org.scalatest.flatspec.AnyFlatSpec

class MyModuleTest extends AnyFlatSpec with ChiselScalatestTester {
  "MyModule" should "produce correct output" in {
    test(new MyModule()) { dut =>
      dut.io.in.poke(5.U)
      dut.clock.step()
      dut.io.out.expect(5.U)
    }
  }
}
```

## 핵심 API

### 입력 설정
```scala
dut.io.a.poke(3.U)          // UInt 입력
dut.io.enable.poke(true.B)  // Bool 입력
dut.io.data.poke(-5.S)      // SInt 입력
```

### 출력 검증
```scala
dut.io.out.expect(7.U)           // 값 검증 (실패 시 에러)
dut.io.valid.expect(true.B)      // Bool 검증
```

### 출력 읽기 (Scala 측 연산용)
```scala
val value = dut.io.out.peekInt()    // BigInt 반환
val flag = dut.io.valid.peekBoolean() // Boolean 반환
```

### 클럭 제어
```scala
dut.clock.step()     // 1 사이클 진행
dut.clock.step(10)   // 10 사이클 진행
```

## 조합 논리 테스트

```scala
test(new CombModule()) { dut =>
  dut.io.a.poke(3.U)
  dut.io.b.poke(4.U)
  dut.io.out.expect(7.U)  // step() 불필요 — 조합 논리는 즉시 반영
}
```

## 순차 논리 테스트

```scala
test(new Counter(10)) { dut =>
  for (i <- 0 until 10) {
    dut.io.cnt.expect(i.U)
    dut.clock.step()
  }
  dut.io.cnt.expect(0.U)  // 10에서 wrap-around
}
```

## FSM 테스트

```scala
test(new SimpleFsm()) { dut =>
  // 초기 상태 확인
  dut.io.ringBell.expect(false.B)

  // green → orange 전이
  dut.io.badEvent.poke(true.B)
  dut.clock.step()
  dut.io.ringBell.expect(false.B)

  // orange → red 전이
  dut.clock.step()
  dut.io.ringBell.expect(true.B)

  // red → green (clear)
  dut.io.badEvent.poke(false.B)
  dut.io.clear.poke(true.B)
  dut.clock.step()
  dut.io.ringBell.expect(false.B)
}
```

## 파라메트릭 테스트

```scala
class CounterTest extends AnyFlatSpec with ChiselScalatestTester {
  // 여러 설정으로 동일 테스트 반복
  for (n <- Seq(4, 7, 8, 13)) {
    s"Counter($n)" should s"count to ${n-1}" in {
      test(new WhenCounter(n)) { dut =>
        for (i <- 0 until 3 * n) {
          dut.io.cnt.expect((i % n).U)
          dut.clock.step()
        }
      }
    }
  }
}
```

## Trait 기반 테스트 재사용

```scala
trait CountTest {
  def testFn(dut: Counter): Unit = {
    val n = dut.io.cnt.getWidth  // 파라미터 접근
    for (i <- 0 until 100) {
      dut.clock.step()
    }
  }
}

class WhenCounterTest extends AnyFlatSpec with ChiselScalatestTester with CountTest {
  "WhenCounter" should "count" in {
    test(new WhenCounter(10)) { dut => testFn(dut) }
  }
}
```

## printf 디버깅

```scala
class MyDebugModule extends Module {
  val io = IO(new Bundle {
    val in = Input(UInt(8.W))
    val out = Output(UInt(8.W))
  })

  val reg = RegInit(0.U(8.W))
  reg := io.in
  io.out := reg

  // 시뮬레이션 중 매 사이클 출력
  printf("cycle: reg=%d, in=%d\n", reg, io.in)
}
```

## 파형 생성 (VCD)

```scala
test(new MyModule()).withAnnotations(Seq(WriteVcdAnnotation)) { dut =>
  // 테스트 코드...
  dut.clock.step(100)
}
// test_run_dir/<test-name>/ 에 .vcd 파일 생성
```

## Fork — 병렬 스레드 테스트

```scala
test(new MyModule()) { dut =>
  // 메인 스레드: 데이터 전송
  fork {
    for (i <- 0 until 10) {
      dut.io.in.poke(i.U)
      dut.clock.step()
    }
  }
  // 메인 스레드: 데이터 수신 확인
  dut.clock.step(2)  // 파이프라인 딜레이
  for (i <- 0 until 10) {
    dut.io.out.expect(i.U)
    dut.clock.step()
  }
}
```

## 실행 명령

```bash
sbt test                          # 전체 테스트
sbt "testOnly *MyModuleTest"      # 특정 테스트 클래스
sbt "testOnly *MyModuleTest -- -z \"should count\"" # 특정 테스트 케이스
```

## Gotchas

| 함정 | 설명 |
|------|------|
| `poke(3)` | `poke(3.U)` — Chisel 리터럴 필수 |
| 순차 로직에서 즉시 expect | `step()` 후 `expect` — 레지스터는 다음 사이클에 반영 |
| `peekInt()` 반환형 | `BigInt` 반환, `Int` 아님 |
| 조합 로직에서 불필요한 step | 조합 논리는 `step()` 없이 즉시 결과 반영 |
| 테스트 내 `println` vs `printf` | `println` = Scala 측 (호스트), `printf` = 하드웨어 측 (시뮬레이션) |
| Chisel 7.x API 변경 | `chiseltest` → `scalatest` + `ChiselSim`, API 크게 변경 |
| VCD 파일 위치 | `test_run_dir/<테스트명>/` 디렉토리에 생성 |
| `fork` 타이밍 | fork 스레드는 같은 시뮬레이션 시간에서 시작 |
