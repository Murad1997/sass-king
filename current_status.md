# Findings and open hypotheses

Running log of observations and hypotheses, organized by kernel chapter.

Notation:
* **[OBS]** verified observation from a SASS dump
* **[HYP]** open hypothesis, to be tested
* **[RES]** resolved hypothesis (rejected or confirmed)

---

## Kernel 01 vector_add (baseline)

### Observations

* [OBS] Every kernel has a 6 section prologue: stack pointer init, thread and block ID setup, argument loads, index computation, bounds check, global descriptor load.
* [OBS] `R1` is the stack pointer by ABI convention, loaded from `c[0x0][0x37c]`, initialized even when no spilling occurs.
* [OBS] Kernel arguments and launch parameters live in constant memory bank 0 at fixed offsets (`c[0x0][...]`).
* [OBS] `LDC` loads into a per thread register, `LDCU` into a uniform register. ptxas classifies automatically based on data flow.
* [OBS] Per thread registers R0 to R255 versus uniform registers UR0 to UR63 (or UR0 to UR255 on SM100+) are two physically separate register files.
* [OBS] Values shared across the warp (blockIdx, kernel args, n) go into UR. Values unique per thread (threadIdx, computed indices, loaded data) go into R.
* [OBS] Global memory accesses on SM120 use descriptor based addressing: `desc[UR][R.64]`. The descriptor is loaded once from `c[0x0][0x358]` and shared across all global accesses.
* [OBS] `IMAD.WIDE` produces a 64 bit result (register pair) from a 32 bit multiply. Used for pointer arithmetic.
* [OBS] Bounds check uses predication (`@P0 EXIT`), not a branch. Zero divergence cost.
* [OBS] Each independent pointer load gets its own scoreboard, allowing dependent address computations to proceed independently.
* [OBS] Co-consumed loads share a single scoreboard. Two LDGs feeding one FADD both go on SB4, one wait covers both.
* [OBS] ptxas interleaves IMAD.WIDE (address computation) with LDG (memory load) so that multiple global loads are in flight simultaneously.
* [OBS] The store is fire and forget, no scoreboard set, since no downstream instruction reads it back.
* [OBS] NOP padding at the end of the kernel aligns the kernel size to a 512 byte boundary. Never executed.
* [OBS] On a minimal elementwise kernel, the useful compute ratio is 1 FADD out of 20 instructions = 5%. Signature of a memory bound kernel.
* [OBS] Only pointers to arrays live in constant memory. The data of the input arrays themselves lives in global memory and is accessed via LDG.
* [OBS] Patterns not yet observed in kernel 01 (to be introduced later): loops (backward BRA), divergent control flow (BSSY/BSYNC), shared memory (LDS/STS/BAR.SYNC), vectorized memory (LDG.E.128), register spilling (STL/LDL), tensor core, atomics, warp level primitives.

### Reference SASS reading template (established here)

Used as a first pass filter in every subsequent kernel audit:

1. Skip the prologue by pattern matching the ID setup sequence.
2. Find the bounds check or early exit. Look for the first `@P EXIT`.
3. Identify the loaded arguments. Pointers come from constant memory in a block near the start of the body.
4. Locate the compute section. Arithmetic instructions that correspond to the algorithm.
5. Check the stores. STG at the end writes the results.
6. Everything else is plumbing: address arithmetic, scoreboard bookkeeping, control flow around loops.

### Open hypotheses

* [HYP] Cross pipeline ALU to CBU transfer costs ~13 cycles versus ~5 for intra ALU. Observed on ISETP to `@P EXIT`. To microbenchmark.
* [HYP] Predicate to arithmetic consumer latency versus predicate to branch latency. Not measured.
* [HYP] Useful compute ratio thresholds (20% and 50%) for memory bound versus latency bound versus compute bound classification. Not calibrated against the actual roofline.

---

## Kernel 02 vector_add_plus1 (a + b + 1.0f)

### Observations

* [OBS] `a + b + c` does **not** fuse. Two separate FADD instructions emitted. FMA fusion is syntactically strict: the source must literally contain a multiply followed by an add.
* [OBS] ptxas tracks register liveness and recycles dead registers across semantic boundaries. R0 (holding threadIdx.x) becomes FP scratch for the first FADD once threadIdx.x is dead.
* [OBS] Adding one source level operation produces exactly one additional SASS instruction, no hidden overhead.
* [OBS] Non compute infrastructure (prologue, bounds check, memory ops) is byte for byte identical between kernels 01 and 02. Only the compute section grows.

### Diagnostic takeaways

* [OBS] When diffing two SASS dumps, most instructions will be identical. Focus on what changes.
* [OBS] Register renaming may occur even in identical looking sections as a consequence of changes elsewhere. Track destinations, not just opcodes.
* [OBS] The absence of fusion is as informative as its presence. Two arithmetic instructions where you expected one means the source level pattern did not match FMA.

---

## Kernel 03 vector_fma (a*b + c)

### Observations

* [OBS] FMA fusion is active: `a*b + c` compiles to a single FFMA.
* [OBS] Fusion is syntactic, the multiply and the add must be direct operands of each other.
* [OBS] Six scoreboards per warp (SB0 to SB5). Hard budget.
* [OBS] Scoreboards and stall count are independent mechanisms. Stall count is for instruction cadence, scoreboards are for data ready signaling.
* [OBS] Fixed latency operations (FFMA, IMAD, ISETP) use only stall count, no scoreboard. ptxas knows their latency at compile time.
* [OBS] Variable latency operations (LDG, LDS, S2R, MUFU, BAR) use scoreboards.
* [OBS] A warp stalls if at least one of two conditions holds: internal stall counter nonzero, or scoreboard wait mask on the next instruction not satisfied.
* [OBS] Scoreboard grouping for co-consumed producers. Three LDGs feeding one FFMA all go on SB4, one wait covers all three.
* [OBS] Yield flag is set on exactly those instructions that wait on a scoreboard. Tells the scheduler it can switch warps during the wait.
* [OBS] Uniform registers live in a distinct SRAM with a separate pipeline, not a mode of the main register file.
* [OBS] Uniform register file is smaller (64 registers on SM80 to SM89, expanded to 256 on SM100+) and the uniform pipeline has a narrower set of supported operations.
* [OBS] Adding one memory operand costs about 3 extra SASS instructions: LDC pointer, IMAD.WIDE address, LDG load.
* [OBS] In kernel 03, adding the `d` pointer grew plumbing by 3 instructions but compute stayed at 1 instruction thanks to FMA fusion. The fusion absorbed what would have been FMUL + FADD.
* [OBS] Uniform datapath benefits: no duplication of the value in the main register file, separate pipeline avoids contention with per thread arithmetic, lower energy per operation (one broadcast instead of 32 reads).
* [OBS] `ISETP` feeding `@P EXIT` has stall=13, much larger than typical 5 for ALU instructions.

### Open hypotheses

* [HYP] The 13 cycle stall on ISETP to `@P EXIT` is caused by cross pipeline transfer from ALU to CBU. To microbenchmark by comparing ISETP to arithmetic consumer versus ISETP to branch.
* [HYP] Throughput of LDG grouped on 1 SB versus split across N SB. Not measured.
* [HYP] Whether the 6 scoreboards per warp budget is effectively 6 or smaller due to hardware constraints. Not measured.

---

## Kernel 04 vector_loop (runtime trip count)

### Observations

* [OBS] Runtime trip count loops produce a cascade of unrolled paths. For this kernel: skip if K<1, scalar tail if K<4, unroll by 4 if 4≤K<16, unroll by 16 if K≥16.
* [OBS] Generalized Duff's device pattern. A 5 line loop produces ~80 SASS instructions across 4 execution paths.
* [OBS] ptxas inverts the loop counter: negates UR5 to count down toward 0 instead of up toward K. Saves a register and allows comparison against a small immediate.
* [OBS] First guard uses signed comparison (handles K≤0 correctly), subsequent guards use unsigned comparison once sign is known.
* [OBS] Remainder precomputed at dispatch: `UR4 = K AND 3` (= K mod 4) is extracted once and later consumed by the scalar tail loop.
* [OBS] Loop counter update (UIADD3) and termination test (UISETP) are interleaved between FFMAs in the loop body, not placed at the tail. Keeps arithmetic pipelines saturated.
* [OBS] HFMA2 used as a constant loader. `HFMA2 R2, -RZ, RZ, 1.875, 0.00931549` loads the FP32 value `1.001f` by exploiting the bit pattern equivalence between two packed FP16 values and one FP32.
* [OBS] HFMA2 is chosen when the kernel is compute heavy, MOV is chosen in lighter blocks. Evidence of pipeline balancing heuristic by ptxas.
* [OBS] Late pointer load for store destination when the store value has a long dependency chain. Minimizes register live range.
* [OBS] `UIADD3` at 0x360 has stall=12, larger than expected for a uniform datapath arithmetic instruction.
* [OBS] Source to SASS expansion ratio for this kernel is approximately 15x (5 source lines, ~77 useful instructions).
* [OBS] Methodological validation: a "smallest possible runtime loop" produced a multi path code expansion, an undocumented constant loading trick, and several optimization decisions that depend on kernel wide heuristics not visible at source level. Controlled variation is the only reliable way to map source patterns to SASS patterns because any a priori model of "what the SASS should look like" will be wrong in at least one significant way.
* [OBS] Henry Zhu (December 2025) documented that ptxas's decision between HFMA2, IMAD, and MOV for constant loading can be affected by surprising inputs including the name of the CUDA function, with performance impacts of ±20% in real kernels. Signal that ptxas scheduling heuristics for constant loading are not fully stable across compiler versions.

### Open hypotheses

* [HYP] Relative throughput of HFMA2 vs MOV vs IMAD for constant loading in compute heavy blocks. To microbenchmark by forcing each via PTX inline.
* [HYP] Per SM partition i-cache size on SM120 is ~8 KB. Unverified.
* [HYP] FMA, FMAHEAVY, FMALITE are distinct physical pipelines or submodes of the same pipeline. To microbenchmark by saturating one and observing the other.
* [HYP] `UPLOP3.LUT` with all UPT inputs and specific LUT values is an idiomatic "initialize UP to constant" pattern. Semantics not fully reverse engineered.
* [HYP] The anomalous UIADD3 stall=12 is caused by cross pipeline transfer. Not verified.

---

## Kernel 05 vector_loop_fixed (compile time loop count)

### Observations

* [OBS] Compile time loop bounds eliminate the entire unrolling cascade. ptxas emits exactly N copies of the body in straight line, no dispatch, no backward branch.
* [OBS] Refactoring runtime trip counts to compile time collapses SASS size drastically. Kernel 04 had ~80 instructions for runtime K, kernel 05 has 17 instructions for K=8.
* [OBS] FFMA on SM120 has one immediate slot, reserved for the addend (`src3`). The multiplier sources (`src1`, `src2`) must be registers.
* [OBS] Any non trivial FP32 constant used as a multiplier must be materialized in a register first. HFMA2 is the systematic idiom ptxas uses for this.
* [OBS] HFMA2 is used regardless of constant value (small, large, irrational, exact power of 2), number of uses (one or many), or kernel structure (loop or straight line).
* [OBS] Register allocation is context dependent. The same value can land in different registers across related kernels based on global liveness analysis.
* [OBS] Last operation in a chain may target a different register than the chain to break dependency with downstream instruction. Observed: last FFMA writes R3 to free R0 and avoid a tight dependency with STG.
* [OBS] Useful compute ratio reaches 47% in kernel 05 (8 FFMA out of 17 instructions), up from 5% in kernel 01. Direct effect of raising arithmetic intensity.

### Practical consequences for constants

Three optimization strategies derived from the FFMA immediate slot rule:

1. **Reuse constants.** Restructure expressions so the same constant is used multiple times. ptxas materializes each unique constant only once.
2. **Use constants as addends when possible.** Constants in `src3` position are encoded as immediates with no materialization cost. Structuring formulas so constants are added rather than multiplied saves instructions.
3. **Pass constants as kernel arguments.** Constants in kernel args live in constant memory and load via LDC, which uses a different pipeline. Can be advantageous if the FMA pipeline is the bottleneck and the LDC can be hidden behind other prologue work.

### Resolved hypotheses

* [RES] The cascade in kernel 04 was caused by runtime trip count. Fixing the count removes the cascade. **Confirmed.**

### Open hypotheses

* [HYP] Why HFMA2 specifically versus MOV or IMAD for constant materialization. Three candidate reasons (pipeline affinity with consuming FFMAs, compact encoding, hardcoded heuristic), none directly measured.
* [HYP] Whether FFMA with immediate in `src1` or `src2` exists at all in any SM120 SASS dump. Would invalidate the addend only rule.
* [HYP] Whether the HFMA2 idiom persists on SM80 or SM89. Older architectures may use different defaults.

---

## Kernel 06 vector_smem (shared memory scalar)

### Observations

* [OBS] Shared memory base is constructed via `UMOV UR4, 0x400` then `ULEA UR4, UR5, UR4, 0x18` where UR5 is `SR_CgaCtaId`.
* [OBS] Shared addressing does not use a descriptor. Direct `[R]` or `[R+UR]`, unlike global memory which uses `desc[UR][R.64]`.
* [OBS] `BAR.SYNC.DEFER_BLOCKING 0x0` is the SASS form of `__syncthreads()` on SM120.
* [OBS] BAR.SYNC runs on the ADU pipeline, not CBU as initially intuited.
* [OBS] `__syncthreads()` always uses barrier 0. CUDA exposes 16 hardware barriers per block.
* [OBS] Modulo by a runtime variable generates a CALL to `__cuda_sm20_rem_u16`, a ~21 instruction function placed after EXIT.
* [OBS] The modulo function uses MUFU.RCP on the XU pipeline. First XU usage in our kernels.
* [OBS] I2F and F2I conversions in the modulo function also run on XU.
* [OBS] CALL.REL.NOINC requires the caller to write the return address into a register (MOV Rn, offset) before the call. RET.REL.NODEC reads it back.
* [OBS] u16 masking (`LOP3.LUT R, R, 0xffff, RZ, 0xc0`) before the modulo CALL is mandated by the u16 ABI of the helper function, not defensive.
* [OBS] The LOP3 mask in shared addressing encodes the modulo bound as `(size_in_floats - 1) * 4`. Verified for shared sizes 128, 256, 512.

### Variant 06b (`% 256`)

* [OBS] Modulo by a power of 2 constant fuses into LEA + LOP3 mask. Two instructions total. No CALL, no slowpath.

### Variant 06c (`% 255`)

* [OBS] Modulo by a non power of 2 constant inlines a reciprocal multiplication sequence. ~10 instructions inline, no CALL.
* [OBS] The inline sequence uses IMAD with magic number 0x809, SHF.R.U32.HI, and PRMT for byte manipulation.

### Variant 06d (`& 255`)

* [RES] `& 255` and `% 256` produce byte identical SASS. ptxas canonicalizes them to the same internal representation. **Confirmed.**

### Variant 06e (128 floats shared, 128 threads)

* [RES] `UMOV UR4, 0x400` does **not** encode shared memory size. Value unchanged at 0x400 despite half size. **Rejected.**
* [OBS] The LOP3 mask scales with shared size: 128 floats produces mask 0x1fc = (128-1) × 4.
* [OBS] Bounds check fusion: `ISETP.GT.U32.AND P0, ..., R6, 0x7f, PT` combined with `ISETP.GE.OR P0, ..., R7, UR5, P0`. ptxas fuses two conditions into one predicate via the `.OR` modifier.

### Variant 06f (512 threads, 512 floats shared)

* [RES] `UMOV UR4, 0x400` does **not** encode block size. Value unchanged at 0x400 despite doubled block. **Rejected.**
* [OBS] The LOP3 mask scales: 512 floats produces mask 0x7fc = (512-1) × 4.

### Variant 06g (two shared buffers)

* [OBS] For N shared buffers, ptxas generates one UMOV and derives the other N-1 bases via UIADD3. Not N separate UMOVs.
* [OBS] Shared buffers are laid out consecutively in memory. Offset between two 256 float buffers = 0x400 = 1024 bytes (size of the first in bytes). No automatic padding.
* [OBS] `[R+UR]` addressing mode lets a single per thread address register be reused with different uniform offsets to read multiple buffers at the same position.
* [OBS] Consecutive STS can be emitted without intermediate BAR.SYNC. One final `__syncthreads()` covers all stores.

### Variant 06h (integer division by runtime)

* [RES] Division and modulo use **separate functions**. Same reciprocal multiplication core, different final step. Division stops at the quotient. Modulo computes quotient then reconstructs remainder via IMAD. **Rejected that they share the function.**
* [OBS] Practical: `q = a / b; r = a % b;` generates two CALLs. `q = a / b; r = a - q * b;` generates one CALL plus one IMAD.

### Variant 06i (two modulos, two runtime divisors)

* [RES] Multiple modulo calls in the same kernel share a **single instance** of the helper function. Two CALLs target the same address. **Confirmed.**
* [OBS] Binary size stays constant regardless of the number of calls.
* [OBS] Modulo helper ABI: R8 u16 dividend input, R9 u16 divisor input and remainder output, Rn return address written by caller via MOV.
* [OBS] Runtime cost is not amortized. Each call executes the full function (~30 cycles minimum with MUFU.RCP).

### Open hypotheses

* [HYP] `UMOV UR4, 0x400` is an architectural constant unrelated to kernel parameters. Exact role unknown. To investigate in kernel 07.
* [HYP] `SR_CgaCtaId` falls back to a default value when the kernel does not declare `__cluster_dims__`. Unverified.
* [HYP] `.DEFER_BLOCKING` on BAR.SYNC is an SM90+ modifier. To verify by comparing with SM80 or SM89 dumps.
* [HYP] PRMT with `0x9910` and `0x7710` in the `% 255` function handles byte level manipulation for u16 truncation. Not analyzed.
* [HYP] `HFMA2 R3, -RZ, RZ, 15, 0` and the subnormal FSETP in the modulo helper handle edge cases of the reciprocal multiplication. Role not fully worked out.
* [HYP] The return address register choice (R7 in kernel 06, R10 in kernel 06i) depends on caller register pressure at the call site. To test with a caller under low pressure.
* [HYP] Why the modulo function has a symbolic label (`__cuda_sm20_rem_u16`) in the dump but the division function does not.

---

## Cross chapter summary

### Pipelines observed so far

| Pipeline | Instructions observed |
|---|---|
| FMA | FFMA, FADD, FMUL, IMAD, HFMA2 |
| ALU | ISETP, FSETP, MOV, LEA, LOP3, FSEL |
| LSU | LDG, STG, LDS, STS |
| ADU | LDC, S2R, BAR.SYNC |
| DCC | LDCU |
| UNIFORM | S2UR, UMOV, ULEA, UIADD3, UISETP, UPLOP3, ULOP3 |
| XU | MUFU.RCP, I2F, F2I (first observed in kernel 06 modulo helper) |
| CBU | EXIT, BRA, CALL, RET |

### Cross cutting observations

* [OBS] Every SASS instruction is 128 bits (16 bytes), fixed size since Volta.
* [OBS] Stall count field is 4 bits (values 0 to 15). Stall=15 repeated on consecutive instructions signals an ILP shortage or pipeline saturation.
* [OBS] Yield flag is set on every instruction with a scoreboard wait.
* [OBS] Predication used by ptxas for short conditional blocks, BRA for longer blocks. Predication evaluates every instruction in the block even when masked off, branches let the warp skip entirely when uniform.
* [OBS] NOP padding at the end of every kernel aligns total size to a memory boundary (typically 128 or 512 bytes). Never executed.

### Global diagnostic workflow

When opening any SASS dump for performance work:

1. Skip the prologue by pattern matching the 6 section skeleton.
2. Find the hot region (BRA backward for a loop body, or the main compute block).
3. Count the useful compute ratio (FFMA + FADD + FMUL + MMA divided by total body instructions).
4. Grep for artifact signals: STL/LDL (spill), CALL (slowpath), abnormal size (unroll cascade).
5. Trace scoreboards: which SB is used, who produces, who waits.
6. Check stall counts. Stall=15 repeated signals a problem.
7. Verify fusion. Count FFMAs vs separate FMUL+FADD chains.
8. Correlate with NCU. SASS identifies the "who", NCU quantifies the "how much".
