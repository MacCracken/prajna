# prajna — Current State

> Refreshed every release. CLAUDE.md is preferences/process/procedures (durable);
> this file is **state** (volatile).

## Version

**0.2.0** — the MAML milestone (M1 + M2), cut 2026-06-24. The second-order meta-gradient
proven scalar → linear → nonlinear (R-operator) + demonstrated on sine few-shot
meta-learning. (0.1.0 = scaffold + M1.) Next: **M3** (learned optimizer).

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
  `sine_sanity` (3 FD gates at the sine config — generalization); **M2.c.2 meta-training**:
  `sine_eval` (held-out avg adapt loss), `sine_opt_init` + `sine_train_batch_step`
  (16-task meta-batch SGD step). `nm_set_alpha` makes the inner lr configurable.
- `src/main.cyr` — demo: M1 → M2.a → M2.b.1, with per-stage gates. Exit 0 iff all pass.
- `src/test.cyr` — `[build].test`: asserts M1 + M2.a + M2.b.1 gates.

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
  Finn 2017 FOMAML honest-negative). Demo run exit 0 (all gates + results); `cyrius test`
  green (FD gates M1/M2.a/M2.b.1/M2.b.2 + meta-training improvement + FOMAML path).

## Dependencies

Direct (declared in `cyrius.cyml`):
- stdlib — string, fmt, alloc, io, vec, str, syscalls, assert, bench, math, **ganita**
  (`f64_tanh`).
- **`rosnet` 0.2.0** (f64 tensors + `linear_fwd`/`linear_bwd`) + **`tyche` 0.1.1**
  (PRNG; mandatory link dep for rosnet's `t_randn`). `akshara` joins at **M4**.

## Consumers

_None yet._

## Next

**M3 — learned optimizer** (→ v0.3.0): meta-learn the *update rule* (a small net that
emits the step), the second realization of learn-to-learn on the same meta-grad machinery
("learning to learn by gradient descent by gradient descent", Andrychowicz 2016). See
[`roadmap.md`](roadmap.md).
