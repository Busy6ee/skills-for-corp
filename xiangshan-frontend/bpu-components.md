# BPU Predictor Components — Detailed Reference

## TAGE (TAgged GEometric History Length Predictor)

### Concept

TAGE consists of multiple tagged tables with different history lengths and a single base predictor. It uses geometrically increasing history lengths to capture a wide variety of branch patterns.

### Structure

```
Base Predictor (bimodal)
  │
  ├── Tagged Table 1 (history len = L1)
  ├── Tagged Table 2 (history len = L2)
  ├── Tagged Table 3 (history len = L3)
  ├── ...
  └── Tagged Table N (history len = LN)

  L1 < L2 < L3 < ... < LN (geometrically increasing)
```

### Entry Structure

```scala
// Entry for each tagged table
class TageEntry extends Bundle {
  val valid  = Bool()
  val tag    = UInt(tagLen.W)      // hash of PC + history
  val ctr    = UInt(3.W)           // 3-bit saturating counter (prediction confidence)
  val useful = UInt(2.W)           // usefulness counter (replacement protection)
}
```

### Prediction Algorithm

1. All tables are looked up simultaneously using PC + history
2. The prediction from the **longest-history table with a tag hit** is used
3. If no tag hits, the base predictor is used
4. `ctr >= 4` → taken, `ctr < 4` → not-taken (3-bit counter)

### Update Algorithm

```
if (misprediction):
  - Decrement provider's ctr
  - Allocate a new entry in a longer-history table (replace entry with useful == 0)
  - If allocation fails, decrement useful in all tables (aging)

if (correct prediction):
  - Increment provider's ctr
  - If altpred differs from the outcome, increment provider's useful
```

### XiangShan TAGE Parameters

```scala
// TageTableInfos in Parameters.scala
// (nRows, histLen, tagLen)
Seq(
  (4096, 8,  8),   // Table 0: short history
  (4096, 13, 8),
  (4096, 32, 8),
  (4096, 119, 8),  // Table N: long history
)

// ITTageTableInfos
Seq(
  (256, 4,  9),
  (256, 8,  9),
  (512, 13, 9),
  (512, 16, 9),
  (512, 32, 9),
)
```

> The geometrically increasing history lengths are the core design principle of TAGE.

## SC (Statistical Corrector)

An auxiliary predictor that statistically corrects TAGE predictions.

### How It Works

```
TAGE prediction result (taken/not-taken + confidence)
  ↓
SC table lookup (multiple history lengths)
  ↓
Compute weighted sum
  ↓
If weighted sum exceeds threshold → flip TAGE prediction (override)
If weighted sum is below threshold → keep TAGE prediction
```

### Key Points
- Correction is only attempted when TAGE predicts with **low confidence**
- SC operates in the s3 stage (after the TAGE result is available)
- Override frequency is low, but contributes to accuracy improvement

## ITTage (Indirect Target TAGE)

Predicts the **target address** of indirect branches (e.g., `jalr`).

### Differences from TAGE

| Aspect | TAGE | ITTage |
|--------|------|--------|
| Prediction target | Conditional branch **direction** (taken/not-taken) | Indirect branch **target address** |
| Entry content | 3-bit counter | Target address (partial) |
| Activation | All conditional branches | `jalr`, indirect `jmp` only |

### Entry Structure

```scala
class ITTageEntry extends Bundle {
  val valid  = Bool()
  val tag    = UInt(tagLen.W)
  val target = UInt(targetLen.W)   // predicted target address
  val useful = UInt(2.W)
}
```

## RAS (Return Address Stack)

Predicts the target address of function returns (`ret`).

### Structure

```
RAS Stack (LIFO)
  ├── Committed RAS (RasSize = 16) ── committed state
  └── Speculative RAS (RasSpecSize = 32) ── speculative state

call instruction → push (return address)
ret instruction → pop (predicted target)
redirect → restore speculative RAS
```

### Speculative RAS Management

```
BPU detects call → push onto speculative RAS
BPU detects ret → pop from speculative RAS
mispred redirect → restore RAS state from FTQ snapshot
commit → update committed RAS
```

> RAS overflow/underflow can occur with deeply nested function calls. Tunable via the `RasSize`/`RasSpecSize` parameters.

## BPU Predictor Pipeline Chain

```scala
// branchPredictor definition in Parameters.scala
val ftb    = Module(new FTB)
val uftb   = Module(new FauFTB)   // Fast uFTB (1-cycle prediction)
val tage   = Module(new Tage_SC)  // TAGE + SC combined
val ras    = Module(new RAS)
val ittage = Module(new ITTage)
// Pipeline: uftb -> tage -> ftb -> ittage -> ras
```

## BPU History Management

### Global History Register (GHR)

```scala
// Global branch history — shared by all predictors
// HistoryLength = allHistLens.max + numBr * FtqSize + 9 (minimum 256)
val globalHistory = RegInit(0.U(HistoryLength.W))
```

### Folded History

```scala
// Fold long history into a short hash (for table indexing)
class FoldedHistory(len: Int, compLen: Int) {
  // XOR-fold len-bit history into compLen bits
}
```

- Each TAGE table uses a folded history matched to its own history length
- Longer history lengths can capture more complex patterns

## Predictor Pipeline Timing

| Stage | Predictor | Latency |
|-------|-----------|---------|
| s1 | FauFTB, FTB lookup | 1 cycle (fastest) |
| s2 | TAGE, ITTage, RAS | 2 cycles |
| s3 | SC (TAGE correction) | 3 cycles |

- s1 prediction is the fastest but least accurate
- Overrides at s2/s3 incur a 1–2 cycle bubble
- Most branches are predicted correctly at s1 (overrides are infrequent)
