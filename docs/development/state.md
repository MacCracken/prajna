# prajna — Current State

> Refreshed every release. CLAUDE.md is preferences/process/procedures (durable);
> this file is **state** (volatile).

## Version

**0.6.0** — first patch of the **0.6.x hardening arc**, cut 2026-06-24: **NaN-safe FD gates**
(audit N-1). The gates were NaN-blind (`f64_le(NaN,·)==1` → a NaN gradient silently passed);
now routed through `src/fdgate.cyr`'s finite-guard so non-finite diffs *fail*. (M1–M5 shipped
0.1.0–0.5.0.) Next: **0.6.1** numerical robustness.

## Toolchain

- **Cyrius pin**: `6.2.39` (in `cyrius.cyml [package].cyrius`; set by `cyrius init`).
- CI/release install the toolchain via upstream `install.sh` (patra reference).

## Surface

- `src/meta.cyr` — **M1**: scalar quadratic MAML meta-gradient (pure scalar f64).
- `src/maml.cyr` — **M2.a**: linear-regression MAML on **rosnet** (K=3→1 linear layer,
  MSE, one inner SGD step). `maml_init` / `maml_adapt` / `maml_meta_loss` /
  `maml_meta_grad` (full 2nd-order into `M_oW/M_ob`; FOMAML in `M_gW/M_gb`) /
  `maml_fd_grad` / `maml_gate` / `maml_2nd_observable`. The HVP `H_s·gθ'` is one
  `linear_fwd` + `linear_bwd` (constant Hessian for linear+MSE).
- `src/mlp.cyr` — **M2.b.1**: 1-hidden-layer tanh MLP on **rosnet** (K=2→H=3→1) +
  hand-derived first-order backprop. The nonlinear foundation.
- `src/mamlnl.cyr` — **M2.b.2**: nonlinear MAML + the **Pearlmutter R-operator**
  second-order meta-gradient (θ-dependent Hessian). `nm_support_fwd_grad` / `nm_adapt`
  / `nm_query_grad_adapted` / `nm_rpass` (R-forward + R-reverse → `H_s·v`) /
  `nm_meta_grad` / three FD gates + `nm_2nd_observable`. **Dimension-configurable**
  (`nm_config(k,h,ms,mq)`) so M2.c reuses the verified engine; `nm_init` = the fixed
  M2.b.2 gate config.
- `src/sine.cyr` — **M2.c.1/.2**: sovereign `f64_sin` (range-reduced Taylor) + `f64_pi`;
  sine-task sampler (`sine_sample_task`/`sine_fill`) over `tyche`; `sine_init_params`;
  `sine_sanity` (3 FD gates at the sine config — generalization); **M2.c.2/.3**: `sine_eval`,
  `sine_opt_init`, `sine_train_batch_step_m(full)` (meta-batch SGD; full or FOMAML),
  `sine_benchmark`. `nm_set_alpha` makes the inner lr configurable.
- `src/lopt.cyr` — **M3.a/.b**: feedforward learned optimizer (1→6→1 tanh `g_φ`,
  gradient→update) unrolled 8 steps on a quadratic optimizee. `lo_forward`/`lo_meta_grad` (BPTT)
  /`lo_gate` (FD over 19 params); `lo_sample_task`/`lo_train_step_batch` (gradient-clipped
  meta-batch SGD)/`lo_eval_learned`/`lo_eval_sgd` (SGD baseline).
- `src/lrnn.cyr` — **M3.c**: recurrent (RNN) learned optimizer — hidden state across steps;
  `lr_forward`/`lr_meta_grad` (two-recurrence BPTT)/`lr_gate` (FD over 29 params); reuses
  `lopt.cyr`'s optimizee + sampler. The 2nd realization of learn-to-learn (meta-learn the
  *update rule*), feedforward + recurrent.
- `src/text.cyr` — **M4.a**: next-token LM on `akshara`-tokenized text (token embedding →
  linear head → softmax cross-entropy, ported from attn11). `tx_init`/`tx_forward`/`tx_backward`
  (incl. embedding scatter-grad)/`tx_gate` (FD). The tie-in to the ML family (shared tokenizer).
- `src/textmaml.cyr` — **M4.b**: MAML over text shift-tasks on the M4.a LM. `tm_init`,
  `tm_set_support`/`tm_set_query`, `tm_adapt` (inner step), `tm_meta_grad` (FOMAML),
  `tm_query_nll`, `tm_train_step` (gradient-clipped meta-batch), `tm_eval` (held-out NLL).
- `src/ewc.cyr` — **M5**: continual-learning durability. 1→8→1 tanh MLP (`ec_forward`/
  `ec_backward`/`ec_grad_gate`); `ec_fisher` (diagonal Fisher), `ec_snapshot`/`ec_reset`,
  `ec_step` (EWC penalty), `ec_set_ab` (experience replay). Replay = the robust method.
- `src/fdgate.cyr` — **0.6.0**: NaN-safe FD-gate comparison (`fd_finite`/`fd_match`/`fd_exceeds`).
  Every gate routes through it; a NaN/Inf difference *fails* the gate (cyrius `f64_le(NaN,·)==1`
  used to let it pass). Shared chokepoint for all 9 gates + 3 observables.
- `src/main.cyr` — demo: M1 → M2.* → M3.* → M4.* → M5, per-stage gates. Exit 0 iff all pass.
- `src/test.cyr` — `[build].test`: NaN-guard assertions + all FD gates + M2.c/M3/M4/M5 results.

## Verification (2026-06-24)

- `cyrius build` → OK; `./build/prajna` → exit 0; `cyrius test` → 2 passed / 0 failed;
  all 5 src files lint clean (0 warnings).
- **M1**: `|full − FD| = 0` ; `|FOMAML − FD| = 0.48` ; meta-descent `θ: 0.5 → 3.499`.
- **M2.a**: full meta-grad matches FD on **all 4 params exactly**; FOMAML wrong-signed
  on `W[0]` (`−0.393` vs `+0.031`); meta-descent loss `0.200 → 0.053` monotone.
- **M2.b.1**: support `Ls = 0.072669`; hand-derived ∇Ls matches FD on **all 13 params**.
- **M2.b.2**: **three-level FD gate all PASS** — ∇Ls, the R-operator HVP (vs
  FD-of-gradient), and the full second-order meta-grad (vs FD-of-meta-loss); FOMAML
  observably differs (θ-dependent Hessian is real); meta-descent `0.050 → 0.030`.
- **M2.c.1**: `f64_sin` valid at known angles (<1e-5); the three FD gates **pass at the
  K=1,H=8 sine config** on a sampled task — the R-operator engine generalizes.
- **M2.c.2**: held-out 10-shot adapt loss descends **monotonically 2.476 → 1.560**
  (~37%) over 1000 meta-training steps (K=1, H=40, 16-task meta-batch) — meta-training
  measurably improves few-shot adaptation.
- **M2.c.3**: full-2nd-order vs FOMAML from identical init+schedule → **statistically
  equivalent** (1.692 vs 1.690, |diff| ~0.1%) — the HVP buys little at this scale (the
  Finn 2017 FOMAML honest-negative).
- **M3.a**: learned-optimizer BPTT meta-grad **matches FD on all 19 optimizer params**
  (untrained unroll L=32.47, T=8).
- **M3.b**: meta-training drops held-out trajectory loss **23.1 → 0.096**; the meta-trained
  optimizer **beats the best fixed-lr SGD** (0.096 vs 0.107).
- **M3.c**: recurrent-optimizer two-recurrence BPTT **matches FD on all 29 params**;
  meta-trains to **0.085** — beating feedforward (0.096) and SGD (0.107).
- **M4.a**: next-token LM on akshara-tokenized "hello world" (V=8, M=10 pairs, NLL 2.106 ≈
  ln 8); embedding+head+softmax-xent backprop **matches FD on all 72 params**.
- **M4.b**: text-MAML over cyclic-shift tasks — held-out 1-step adaptation NLL **2.37 → 0.49**
  (~79%) over meta-training.
- **M5**: MLP backprop FD-gated (25 params); naive sequential forgets A (0.05→0.56),
  **experience replay retains A** (→0.00006) + fits B (0.178). EWC honest-negative at toy scale.
  Demo run exit 0; `cyrius test` green (all FD gates + M2.c + M3.a/b/c + M4.a/b + M5).

## Dependencies

Direct (declared in `cyrius.cyml`):
- stdlib — string, fmt, alloc, io, vec, str, syscalls, assert, bench, math, **ganita**
  (`f64_tanh`).
- **`rosnet` 0.2.0** (f64 tensors + `linear_fwd`/`linear_bwd`) + **`tyche` 0.1.1**
  (PRNG; mandatory link dep for rosnet's `t_randn`). `akshara` joins at **M4**.

## Consumers

_None yet._

## Next

**0.6.x hardening arc** (grounded in [`audit-0.6.md`](audit-0.6.md)). **0.6.0 ✅ done** (NaN-safe
gates). Remaining: **0.6.1** numerical robustness (N-2 ln floor + N-3 NaN fail-fast in training
loops), **0.6.2** security/supply-chain (S-1 threat model + dep pins, M-1 scratch asserts),
**0.6.3** refactor (R-1/R-2/R-3 shared param-addressing + gate harness). Then the v1.0 freeze
cycle. See [`roadmap.md`](roadmap.md).
