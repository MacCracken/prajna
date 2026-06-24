# Changelog

Format: [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [Unreleased]

### Added
- **M1 — scalar second-order meta-gradient core** (`src/meta.cyr`): the family's
  first nested gradient, differentiating the query loss *through* one inner SGD step
  (scalar quadratic MAML). Analytic `dM/dt = Lq'(t')·(1 − α·Ls'')`.
- **Finite-difference gate** over the two-level computation: analytic `−1.919999` vs
  central finite-diff `−1.920000` (`|Δ| = 0`).
- **Second-order observability**: FOMAML (first-order) differs from finite-diff by
  `0.48`, proving the `(1 − α·Ls'')` term is real — the honest contrast.
- Demo (`src/main.cyr`) with meta-descent to the optimum (`θ: 0.5 → 3.499`, opt 3.5);
  `[build].test` (`src/test.cyr`) asserting gate + observability + descent.

## [0.1.0]

### Added
- Initial project scaffold (`cyrius init`); name `prajna` (प्रज्ञा), identity, and the
  attn11/tarka/tentib boundary documented.
