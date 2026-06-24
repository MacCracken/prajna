# Changelog

Format: [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [Unreleased]

### Added
- **M2.c.2 — MAML meta-training on sine tasks** (`src/sine.cyr`): meta-train the MLP
  initialization across sampled sine tasks with the full second-order meta-grad +
  **meta-batch** (16 tasks/step, cuts gradient noise) SGD. The held-out 10-shot
  adaptation loss descends **monotonically 2.48 → 1.56** (~37%) over 1000 steps — the
  "MAML works on AGNOS" result, sovereign from the metal up. Inner lr made configurable
  (`nm_set_alpha`); `sine_eval` (held-out) + `sine_train_batch_step` (silent). Test
  asserts the improvement deterministically.
- **M2.c.1 — sine-task substrate + engine generalization** (`src/sine.cyr`): a
  sovereign **`f64_sin`** (π via a 245850922/78256779 convergent + range-reduced 8-term
  Taylor; validated at 0, π/6, π/2, π, −π/2 within 1e-5), the sine-task sampler
  (`y = A·sin(x+φ)`, A∈[0.1,5], φ∈[0,π], via `tyche`), and a refactor making the
  `mamlnl.cyr` engine **dimension/data-configurable** (`nm_config`). **Proof of
  generalization**: the three FD gates pass at the sine config **K=1, H=8** on a sampled
  task — the verified R-operator works beyond the M2.b.2 fixed dims. No new deps (sin is
  hand-rolled, not borrowed from abaco).
- **M2.b.2 — nonlinear MAML second-order meta-gradient via the R-operator**
  (`src/mamlnl.cyr`): the reference's hardest derivation. The tanh makes the support
  Hessian **θ-dependent**, so the HVP `H_s·v` is computed by the hand-derived
  **Pearlmutter R-operator** (forward-mode differentiation propagated through the
  gradient computation), then the full meta-grad `gθ' − α·(H_s·gθ')` through the inner
  step. Equations independently confirmed **three ways** (R-operator / direct-unroll /
  graph-backprop) + a NumPy finite-difference HVP cross-check, then verified in-code by a
  **three-level FD gate**: ∇Ls, the HVP (vs FD-of-gradient), and the meta-grad (vs
  FD-of-meta-loss) — all green; FOMAML observably differs; meta-descent monotone.
- **M2.b.1 — 1-hidden-layer tanh MLP on `rosnet`** (`src/mlp.cyr`): prajna's first
  NONLINEAR model + its hand-derived first-order backprop (`Z=X·W1+b1`, `Hh=tanh(Z)`,
  `Y=Hh·W2+b2`, MSE; `linear_bwd` for the matrix products, manual `tanh'`). **FD gate
  matches analytic ∇Ls on all 13 params.** The tanh makes the support Hessian
  θ-dependent — the foundation for M2.b.2's double-backward (R-operator HVP). Added the
  `ganita` stdlib module (`f64_tanh`). `cyrius test` now gates M1 + M2.a + M2.b.1.
- **M2.a — linear-regression MAML on `rosnet`** (`src/maml.cyr`): the second-order
  meta-gradient on actual rosnet tensors with **coupled** parameters (single linear
  layer, K=3→1, MSE, one inner SGD step). The meta-grad `dLq/dθ = gθ' − α·(H_s·gθ')`
  is hand-derived; for linear+MSE the Hessian-vector product `H_s·v` is computed with
  one extra `linear_fwd` + `linear_bwd` (no Hessian matrix) — the matrix generalization
  of M1's `(1 − α·Ls'')` factor.
- **FD gate** matches the analytic meta-grad on **all 4 parameters exactly**; FOMAML
  is dramatically off (on `W[0]` it has the **wrong sign**: `−0.393` vs true `+0.031`)
  — the second-order term is load-bearing. Meta-descent reduces the loss monotonically.
- Wired `[deps.rosnet]` 0.2.0 + `[deps.tyche]` 0.1.1 (prajna is now a sibling-on-rosnet,
  like attn11/tarka/tentib). `src/main.cyr` + `[build].test` exercise M1 and M2.a.

## [0.1.0] — 2026-06-24

### Added
- Initial project scaffold (`cyrius init`); name `prajna` (प्रज्ञा), identity, and the
  attn11/tarka/tentib boundary documented.
- **M1 — scalar second-order meta-gradient core** (`src/meta.cyr`): the family's first
  nested gradient, differentiating the query loss *through* one inner SGD step (scalar
  quadratic MAML). Analytic `dM/dt = Lq'(t')·(1 − α·Ls'')`, **finite-difference-gated**
  over the two-level computation (analytic `−1.919999` vs FD `−1.920000`, `|Δ| = 0`),
  with the FOMAML contrast (`|Δ| = 0.48`) proving the second-order term is real, and
  meta-descent to the optimum (`θ: 0.5 → 3.499`).
- CI/release workflows corrected to install the toolchain via the upstream `install.sh`
  (the `cyrius.cyml` pin is the single source of truth; patra is the reference).
