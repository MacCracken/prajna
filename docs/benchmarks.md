# prajna — Benchmarks

prajna is a **correctness-first reference**, not a throughput engine: everything is single-threaded
`f64` with no BLAS, no SIMD, no GPU. The meaningful numbers are therefore (1) the verification that
every hand-derived gradient matches finite differences, (2) the meta-learning convergence each
milestone achieves, and (3) the sovereign footprint. Measured 2026-06-24 on the dev host (`CYRIUS_DCE=1`).

## Footprint

| metric | value |
|--------|-------|
| Binary size (DCE) | **224 KB** (static ELF) |
| Source | **2,246 lines** across 13 Cyrius modules |
| Dependencies | **3** (rosnet, tyche, akshara) — all first-party, no libc/BLAS/autodiff |
| Clean build | ~340 ms |
| `cyrius test` | ~290 ms |
| Full demo (`./build/prajna`) | ~7.4 s (dominated by the meta-training loops) |

## Correctness — every hand-derived gradient is finite-difference-gated

| milestone | what is gated | params | result |
|-----------|---------------|:------:|--------|
| M1   | scalar 2nd-order meta-gradient | 1 | matches FD; FOMAML differs by 0.48 (2nd-order term real) |
| M2.a | linear-MAML meta-gradient (rosnet) | 4 | matches FD exactly |
| M2.b.1 | tanh-MLP backprop | 13 | matches FD |
| M2.b.2 | ∇Ls · **R-operator HVP** · full meta-grad | 13 | **all 3 gates** pass |
| M2.c | R-operator generalizes at the sine config (K=1,H=8) | — | 3 gates pass on a sampled task |
| M3.a | learned-optimizer **BPTT** meta-grad | 19 | matches FD |
| M3.c | recurrent-optimizer **two-recurrence BPTT** | 29 | matches FD |
| M4.a | embedding + head + softmax-xent backprop | 72 | matches FD |
| M5   | continual-learning MLP backprop | 25 | matches FD |
| 0.6.0 | gates reject NaN/Inf (not silently pass) | — | negative test green |

## Convergence

| milestone | metric | before → after |
|-----------|--------|----------------|
| M1   | meta-descent (θ → optimum 3.5) | 0.500 → **3.493** |
| M2.c | held-out 10-shot adapt loss (20 tasks) | 2.48 → **1.598** |
| M2.c.3 | full-2nd-order vs FOMAML (honest-negative at toy scale) | 1.692 ≈ 1.690 |
| M3.b | learned optimizer **vs best fixed-lr SGD** | 0.096 **beats** 0.107 |
| M3.b | feedforward optimizer (held-out trajectory) | 23.14 → **0.096** |
| M3.c | recurrent optimizer (held-out trajectory) | 12.94 → **0.085** (beats feedforward) |
| M4.b | text-MAML held-out NLL (1-step adapt to a new shift) | 2.371 → **0.494** |
| M5   | task-A loss after learning task B (naive vs **replay**) | 0.56 (forgot) vs **0.00006** (retained) |

## Methodology

- Single-threaded `f64` (stored as `i64` bit-patterns); no BLAS, no parallelism.
- All randomness is seeded (`tyche`), so every number above is **reproducible** run-to-run.
- Finite-difference gates use central differences with step `fd_eps = 1e-5` (see `src/fdgate.cyr`).
- "Beats SGD" (M3.b) is a controlled comparison: identical optimizee tasks, the learned optimizer
  vs the best fixed learning rate found by sweep.
