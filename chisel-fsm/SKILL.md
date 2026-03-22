---
name: chisel-fsm
description: Use when designing finite state machines in Chisel, including Mealy/Moore machines, multi-state controllers, or FSMs communicating with datapaths (timers, counters). Use when the design requires state machine, FSM, or sequential control logic.
---

# Chisel FSM — 유한 상태 머신 설계

## 기본 FSM 패턴

```scala
import chisel3._
import chisel3.util._

class SimpleFsm extends Module {
  val io = IO(new Bundle {
    val badEvent = Input(Bool())
    val clear = Input(Bool())
    val ringBell = Output(Bool())
  })

  // 1. 상태 정의 (ChiselEnum)
  object State extends ChiselEnum {
    val green, orange, red = Value
  }
  import State._

  // 2. 상태 레지스터 (초기 상태 지정)
  val stateReg = RegInit(green)

  // 3. 다음 상태 로직
  switch (stateReg) {
    is (green) {
      when(io.badEvent) {
        stateReg := orange
      }
    }
    is (orange) {
      when(io.badEvent) {
        stateReg := red
      } .elsewhen(io.clear) {
        stateReg := green
      }
    }
    is (red) {
      when(io.clear) {
        stateReg := green
      }
    }
  }

  // 4. 출력 로직 (Moore: 상태에만 의존)
  io.ringBell := stateReg === red
}
```

## Moore vs Mealy

### Moore Machine — 출력이 상태에만 의존
```scala
// 출력 로직이 switch 블록 밖에 있음
switch (stateReg) {
  is (zero) {
    when(io.din) { stateReg := one }
  }
  is (one) {
    when(!io.din) { stateReg := zero }
  }
}
// Moore 출력: 상태만으로 결정
io.risingEdge := stateReg === one
```

### Mealy Machine — 출력이 상태+입력에 의존
```scala
// 기본 출력값 설정 (중요!)
io.risingEdge := false.B

switch (stateReg) {
  is (zero) {
    when(io.din) {
      stateReg := one
      io.risingEdge := true.B  // Mealy: 전이 시 즉시 출력
    }
  }
  is (one) {
    when(!io.din) {
      stateReg := zero
    }
  }
}
```

**차이점:**
| | Moore | Mealy |
|--|-------|-------|
| 출력 | 상태에만 의존 | 상태 + 입력 |
| 타이밍 | 1사이클 지연 | 즉시 반응 |
| 글리치 | 없음 | 입력 변화 시 발생 가능 |
| 상태 수 | 더 많음 | 더 적음 |

## FSM + Datapath 통합

타이머나 카운터와 FSM을 결합하는 패턴:

```scala
class Flasher extends Module {
  val io = IO(new Bundle {
    val start = Input(Bool())
    val light = Output(Bool())
  })

  object State extends ChiselEnum {
    val off, flash1, space1, flash2, space2, flash3 = Value
  }
  import State._

  val stateReg = RegInit(off)

  // 타이머 (datapath)
  val timerReg = RegInit(0.U(16.W))
  val timerDone = timerReg === 0.U

  // FSM → Datapath: 제어 신호
  val timerLoad = WireDefault(false.B)
  val timerSelect = WireDefault(0.U(1.W))

  // 타이머 로직
  val timerVal = Mux(timerSelect === 0.U, 999.U, 499.U)
  when(timerLoad) {
    timerReg := timerVal
  } .elsewhen(timerReg =/= 0.U) {
    timerReg := timerReg - 1.U
  }

  // 출력 기본값
  io.light := false.B

  // FSM 로직
  switch(stateReg) {
    is(off) {
      when(io.start) {
        stateReg := flash1
        timerLoad := true.B
      }
    }
    is(flash1) {
      io.light := true.B
      when(timerDone) {
        stateReg := space1
        timerLoad := true.B
        timerSelect := 1.U
      }
    }
    // ... 나머지 상태들
  }
}
```

## 카운터 기반 상태 축소

상태가 반복 패턴일 때 카운터로 상태 수를 줄이는 패턴:

```scala
object State extends ChiselEnum {
  val off, flash, space = Value
}

val stateReg = RegInit(off)
val cntReg = RegInit(0.U(2.W))

switch(stateReg) {
  is(flash) {
    io.light := true.B
    when(timerDone) {
      stateReg := space
      timerLoad := true.B
    }
  }
  is(space) {
    when(timerDone) {
      when(cntReg === 2.U) {
        stateReg := off
        cntReg := 0.U
      } .otherwise {
        stateReg := flash
        cntReg := cntReg + 1.U
        timerLoad := true.B
      }
    }
  }
}
```

## 통신 FSM — FSM 간 연결

```scala
// Timer FSM (재사용 가능한 타이머)
class TimerFsm(maxCount: Int) extends Module {
  val io = IO(new Bundle {
    val start = Input(Bool())
    val done = Output(Bool())
  })

  val cntReg = RegInit(0.U(log2Ceil(maxCount + 1).W))
  io.done := cntReg === 0.U

  when(io.start) {
    cntReg := maxCount.U
  } .elsewhen(cntReg =/= 0.U) {
    cntReg := cntReg - 1.U
  }
}

// Main FSM — TimerFsm을 사용
class MainController extends Module {
  val timer = Module(new TimerFsm(1000))

  // FSM → Timer: 제어 신호
  timer.io.start := false.B

  // Timer → FSM: 상태 신호
  when(timer.io.done) {
    stateReg := nextState
  }
}
```

## Gotchas

| 함정 | 설명 |
|------|------|
| **ChiselEnum 위치** | Module 클래스 내부 또는 companion object 내에 정의. 패키지 레벨 정의 시 문제 발생 가능 |
| **`import State._` 누락** | enum 정의 후 반드시 `import State._` 해야 상태명 직접 사용 가능 |
| **기본 출력값 누락** | Mealy 출력은 `switch` 전에 기본값 설정 필수. 미설정 시 "not fully initialized" 에러 또는 래치 생성 |
| **Mealy 글리치** | Mealy 출력은 조합 경로 — 입력 글리치가 출력으로 전파. 비동기 컨텍스트에서 주의 |
| **switch에 otherwise 없음** | `switch/is`는 `otherwise` 지원 안함 — 매치되지 않는 상태는 현재 값 유지 (레지스터) |
| **when vs switch** | `switch`는 `ChiselEnum`/`UInt` 매칭용. 상태 내 조건 분기는 `when` 사용 |
| **상태 인코딩** | ChiselEnum은 기본 이진 인코딩. one-hot 필요 시 별도 처리 |
| **타이머 off-by-one** | `timerReg === 0.U`로 done 검출 시, 로드 값이 N이면 N+1 사이클 후 done |
| **WireDefault 제어 신호** | FSM 제어 출력은 `WireDefault(false.B)`로 선언 — switch 내에서 필요할 때만 `true.B` 할당 |
