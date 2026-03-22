# Chisel Type Reference

## Operator Tables

### Bitwise Operations (UInt, SInt, Bool)
| Operation | Syntax | Description |
|-----------|--------|-------------|
| AND | `a & b` | bitwise AND |
| OR | `a \| b` | bitwise OR |
| XOR | `a ^ b` | bitwise XOR |
| NOT | `~a` | bitwise negation |
| Shift Left | `a << n` | Shift left by n bits |
| Shift Right | `a >> n` | Shift right by n bits |
| Bit Extract | `a(n)` | Extract the n-th bit |
| Range Extract | `a(hi, lo)` | Extract bits in hi:lo range |
| Concatenate | `a ## b` | Bit concatenation (a is upper) |

### Arithmetic Operations (UInt, SInt)
| Operation | Syntax | Result Width |
|-----------|--------|-------------|
| Add | `a + b` | max(w(a), w(b)) |
| Sub | `a - b` | max(w(a), w(b)) |
| Mul | `a * b` | w(a) + w(b) |
| Div | `a / b` | w(a) |
| Mod | `a % b` | min(w(a), w(b)) |
| Negate | `-a` | w(a) |

### Comparison Operations
| Operation | Syntax | Returns |
|-----------|--------|---------|
| Equal | `a === b` | Bool |
| Not Equal | `a =/= b` | Bool |
| Greater | `a > b` | Bool |
| Greater Equal | `a >= b` | Bool |
| Less | `a < b` | Bool |
| Less Equal | `a <= b` | Bool |

### Bool-Specific
| Operation | Syntax |
|-----------|--------|
| AND | `a && b` |
| OR | `a \|\| b` |
| NOT | `!a` |

## Type Conversion Cheat Sheet

| From | To | Method |
|------|----|--------|
| `SInt` | `UInt` | `.asUInt` |
| `UInt` | `SInt` | `.asSInt` |
| `UInt(1.W)` | `Bool` | `.asBool` |
| `Bool` | `UInt` | `.asUInt` |
| `Vec[Bool]` | `UInt` | `.asUInt` |
| `UInt` | `Vec[Bool]` | `VecInit(u.asBools)` |
| `UInt` | `Bundle` | `.asTypeOf(new MyBundle())` |
| `Bundle` | `UInt` | `.asUInt` |

## Width Inference Rules

- Operation result widths are automatically inferred (see arithmetic operations table above)
- `Wire(UInt())` — when width is unspecified, it is inferred from the connected signal
- `RegInit(0.U)` — **inferred as 1-bit** (caution!)
- Constants: `0.U` -> 1-bit, `255.U` -> 8-bit (minimum required width)
- Explicit width specification is always safe: `0.U(32.W)`

## Mux Variants

```scala
Mux(cond, thenVal, elseVal)           // 2-way mux
MuxCase(default, Seq(c1 -> v1, c2 -> v2))  // priority mux
MuxLookup(sel, default)(Seq(0.U -> a, 1.U -> b))  // lookup mux
```
