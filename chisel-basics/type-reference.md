# Chisel Type Reference

## 연산자 테이블

### 비트 연산 (UInt, SInt, Bool)
| 연산 | 구문 | 설명 |
|------|------|------|
| AND | `a & b` | bitwise AND |
| OR | `a \| b` | bitwise OR |
| XOR | `a ^ b` | bitwise XOR |
| NOT | `~a` | bitwise negation |
| Shift Left | `a << n` | n비트 왼쪽 시프트 |
| Shift Right | `a >> n` | n비트 오른쪽 시프트 |
| Bit Extract | `a(n)` | n번째 비트 |
| Range Extract | `a(hi, lo)` | hi:lo 범위 추출 |
| Concatenate | `a ## b` | 비트 연결 (a가 상위) |

### 산술 연산 (UInt, SInt)
| 연산 | 구문 | 결과 폭 |
|------|------|---------|
| Add | `a + b` | max(w(a), w(b)) |
| Sub | `a - b` | max(w(a), w(b)) |
| Mul | `a * b` | w(a) + w(b) |
| Div | `a / b` | w(a) |
| Mod | `a % b` | min(w(a), w(b)) |
| Negate | `-a` | w(a) |

### 비교 연산
| 연산 | 구문 | 반환 |
|------|------|------|
| Equal | `a === b` | Bool |
| Not Equal | `a =/= b` | Bool |
| Greater | `a > b` | Bool |
| Greater Equal | `a >= b` | Bool |
| Less | `a < b` | Bool |
| Less Equal | `a <= b` | Bool |

### Bool 전용
| 연산 | 구문 |
|------|------|
| AND | `a && b` |
| OR | `a \|\| b` |
| NOT | `!a` |

## 타입 변환 치트시트

| From | To | 방법 |
|------|----|------|
| `SInt` | `UInt` | `.asUInt` |
| `UInt` | `SInt` | `.asSInt` |
| `UInt(1.W)` | `Bool` | `.asBool` |
| `Bool` | `UInt` | `.asUInt` |
| `Vec[Bool]` | `UInt` | `.asUInt` |
| `UInt` | `Vec[Bool]` | `VecInit(u.asBools)` |
| `UInt` | `Bundle` | `.asTypeOf(new MyBundle())` |
| `Bundle` | `UInt` | `.asUInt` |

## 폭 추론 규칙

- 연산 결과 폭은 자동 추론됨 (위 산술 연산 테이블 참조)
- `Wire(UInt())` — 폭 미지정 시 연결된 신호에서 추론
- `RegInit(0.U)` — **1비트로 추론됨** (주의!)
- 상수: `0.U` → 1비트, `255.U` → 8비트 (최소 필요 폭)
- 명시적 폭 지정이 항상 안전: `0.U(32.W)`

## Mux 변형

```scala
Mux(cond, thenVal, elseVal)           // 2-way mux
MuxCase(default, Seq(c1 -> v1, c2 -> v2))  // priority mux
MuxLookup(sel, default)(Seq(0.U -> a, 1.U -> b))  // lookup mux
```
