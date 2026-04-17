# Kernel 02 — vector_add + 1.0f

Smallest possible delta from kernel 01: a single extra addition in the expression.

## Source

```cuda
__global__ void vector_add_plus1(const float* a, const float* b, float* c, int n) {
    int i = blockIdx.x * blockDim.x + threadIdx.x;
    if (i < n) {
        c[i] = a[i] + b[i] + 1.0f;
    }
}
```

One character difference in the source: `+ 1.0f` appended to the expression.

## Compile

```
nvcc -arch=sm_120 -o 02_vector_add_plus1 02_vector_add_plus1.cu
cuobjdump --dump-sass 02_vector_add_plus1
```

## SASS diff from kernel 01

Instructions 0x00 through 0x100 are **byte-for-byte identical** to kernel 01. The entire prologue, bounds check, pointer loads, address arithmetic, and memory loads do not change.

The only difference is in the compute section:

| Address | Kernel 01 | Kernel 02 |
|---------|-----------|-----------|
| 0x110 | `FADD R9, R2, R5` | `FADD R0, R2, R5` |
| 0x120 | `STG.E ..., R9` | `FADD R9, R0, 1` |
| 0x130 | `EXIT` | `STG.E ..., R9` |
| 0x140 | (padding) | `EXIT` |

One extra instruction. One register renamed. Everything else identical.

## Analysis

### No fusion

Two FADDs stay two FADDs. ptxas does not combine them into an FFMA because FFMA computes `a * b + c`, not `a + b + c`. The two operations do not fit the FMA algebraic template.

This establishes the rule: **FMA fusion is syntactically strict**. The source expression must literally contain a multiply followed by an add to trigger fusion. Adding three operands does not.

### Register recycling

In kernel 01, the single FADD wrote to R9 and STG consumed R9. Done.

In kernel 02, there are two FADDs. The first one produces an intermediate value that must live somewhere while the second FADD reads it. ptxas chose to write the first FADD into R0.

Why R0? Because R0 held threadIdx.x during the prologue (via `S2R R0, SR_TID.X`), and after the IMAD at 0x50 computed `i` from it, R0 is never read again. The value is dead. ptxas's liveness analysis detects this and recycles R0 as scratch space for the intermediate `a[i] + b[i]` value.

Net effect: one additional arithmetic operation, zero additional register pressure.

This is a general pattern: ptxas tracks register liveness precisely and reuses dead registers whenever possible, regardless of their original semantic purpose.

### The "stable infrastructure" observation

Every non-compute instruction is identical between the two kernels. The prologue, bounds check, and memory operations form a stable infrastructure that does not change when the algorithm changes. Only the compute section grows.

This confirms the method: **controlled variation isolates exactly one pattern at a time**. A one-character source change produces a one-instruction SASS change, and the register recycling logic is the only surprise. If we had changed more at once, we would not know which part of the source caused which part of the SASS change.

## Patterns identified (new)

Two new observations on top of kernel 01:

1. **FMA fusion is strict.** Only `a * b + c` patterns fuse. `a + b + c` produces two separate FADDs.

2. **Dead register recycling.** ptxas reuses registers whose values are no longer live, even across semantic boundaries (threadIdx's R0 becomes an FP scratch register). No pressure increase for additional computation as long as there is dead room in the register file.

## Instruction count breakdown

| Role | Count | Δ from K01 |
|---|---|---|
| Setup + bounds | 7 | 0 |
| Pointer loads | 4 | 0 |
| Address + memory | 6 | 0 |
| Compute | 2 | +1 |
| Store + exit | 2 | 0 |
| **Total** | **21** | **+1** |

Exactly one extra instruction for one extra source-level operation. No hidden overhead, no optimization surprises. This is the cleanest possible controlled variation.

## What this kernel does not show

Same limitations as kernel 01. The minimal delta did not introduce any new infrastructure. No loops, no shared memory, no new control flow, no tensor cores.

## Takeaway for SASS reading

When diffing two SASS dumps:
- Most instructions will be identical. Focus on what changes.
- Register renaming may occur even in the identical-looking sections as a consequence of changes elsewhere. Track destinations, not just opcodes.
- The absence of fusion is as informative as its presence. If you see two arithmetic instructions where you expected one, it usually means the source-level pattern did not match FMA.