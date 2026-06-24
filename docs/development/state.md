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
  hand-derived first-order backprop. The nonlinear foundation.
- `src/mamlnl.cyr` — **M2.b.2**: nonlinear MAML + the **Pearlmutter R-operator**
  second-order meta-gradient (θ-dependent Hessian). `nm_support_fwd_grad` / `nm_adapt`
  / `nm_query_grad_adapted` / `nm_rpass` (R-forward + R-reverse → `H_s·v`) /
  `nm_meta_grad` / three FD gates (`nm_grad_gate`, `nm_hvp_gate`, `nm_meta_grad_gate`)
  + `nm_2nd_observable`.
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

## Dependencies

Direct (declared in `cyrius.cyml`):
- stdlib — string, fmt, alloc, io, vec, str, syscalls, assert, bench, math, **ganita**
  (`f64_tanh`).
- **`rosnet` 0.2.0** (f64 tensors + `linear_fwd`/`linear_bwd`) + **`tyche` 0.1.1**
  (PRNG; mandatory link dep for rosnet's `t_randn`). `akshara` joins at **M4**.

## Consumers

_None yet._

## Next

**M2.c** — the demonstration. Sample sine-wave tasks with `tyche`; meta-train the MLP
initialization across tasks (full second-order vs FOMAML/Reptile); show K-shot fast
adaptation to a new task beats a non-meta baseline; benchmark vs the MAML sine-regression
setup. **Honest-negative eligible**: does the full 2nd-order term beat FOMAML *at this
tiny scale*? See [`roadmap.md`](roadmap.md).
