# Changelog

Format: [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [Unreleased]

### Added
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
