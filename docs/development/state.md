# prajna — Current State

> Refreshed every release. CLAUDE.md is preferences/process/procedures (durable);
> this file is **state** (volatile).

## Version

**0.1.0** — released 2026-06-24 (scaffold + M1 scalar meta-gradient). **M2.a landed
on top, untagged (toward 0.2.0).**

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
  hand-derived first-order backprop. `mlp_init` / `mlp_forward` / `mlp_backward` /
  `mlp_param_ptr` / `mlp_grad_val` / `mlp_fd_grad` / `mlp_grad_gate`. The nonlinear
  foundation; its θ-dependent Hessian is M2.b.2's HVP target.
- `src/main.cyr` — demo: M1 → M2.a → M2.b.1, with per-stage gates. Exit 0 iff all pass.
- `src/test.cyr` — `[build].test`: asserts M1 + M2.a + M2.b.1 gates.

## Verification (2026-06-24)

- `cyrius build` → OK; `./build/prajna` → exit 0; `cyrius test` → 2 passed / 0 failed;
  all 5 src files lint clean (0 warnings).
- **M1**: `|full − FD| = 0` ; `|FOMAML − FD| = 0.48` ; meta-descent `θ: 0.5 → 3.499`.
- **M2.a**: full meta-grad matches FD on **all 4 params exactly**; FOMAML wrong-signed
  on `W[0]` (`−0.393` vs `+0.031`); meta-descent loss `0.200 → 0.053` monotone.
- **M2.b.1**: support `Ls = 0.072669`; hand-derived ∇Ls matches FD on **all 13 params**.

## Dependencies

Direct (declared in `cyrius.cyml`):
- stdlib — string, fmt, alloc, io, vec, str, syscalls, assert, bench, math, **ganita**
  (`f64_tanh`).
- **`rosnet` 0.2.0** (f64 tensors + `linear_fwd`/`linear_bwd`) + **`tyche` 0.1.1**
  (PRNG; mandatory link dep for rosnet's `t_randn`). `akshara` joins at **M4**.

## Consumers

_None yet._

## Next

**M2.b.2** — the R-operator HVP: the tanh makes the support Hessian θ-dependent, so
hand-derive `H_s·v` via the Pearlmutter R-operator (forward-mode over the gradient),
then the full second-order meta-grad through the MLP's inner step — FD-gated at three
levels (∇Ls, the HVP, the meta-grad). Then **M2.c** sine-task sampling + meta-training
+ the fast-adaptation demo + MAML benchmark. See [`roadmap.md`](roadmap.md).
