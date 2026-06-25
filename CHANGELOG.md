# Changelog

Format: [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [Unreleased]

## [0.9.0] — 2026-06-24

**Release candidate for 1.0.** All v1.0 criteria are met; 1.0.0 will be a clean version cut
with a small doc pass. No code changes to the M1–M5 reference or the 0.6.x hardening — this is
the API-freeze + documentation milestone.

### Added
- **`docs/api.md` — the public API, frozen at 1.0.** Documents the consumable `fdgate.cyr`
  surface and the per-milestone reference interface, with an explicit stability policy (the
  **gates** are the contract, not the literal digits). See ADR 0001.
- **`docs/benchmarks.md`** — footprint (224 KB binary, 2,246 lines, 3 deps, no libc/BLAS/autodiff),
  correctness (every gradient FD-gated, param counts), and convergence numbers per milestone.
- **`examples/gate_consumer.cyr` + `examples/README.md`** — a buildable downstream-consumer
  example that vendors `fdgate` to FD-gate an external gradient (`f(x)=x³`) and shows NaN rejection.
- **`docs/adr/0001-api-freeze.md`** — records the freeze decision and its scope.

### Changed
- **`README.md`** — expanded with the full M1–M5 reference summary and a documentation index
  (API, benchmarks, security, examples, roadmap, ADRs).

## [0.6.3] — 2026-06-24

Final patch of the 0.6.x hardening arc — refactor (audit R-2, R-3). **Arc complete.**

### Changed
- **R-2/R-3: the FD-gate infrastructure is centralized in `src/fdgate.cyr`.** The
  central-difference arithmetic (`fd_central`), the perturbation step (`fd_eps`, a shared 1e-5
  — meta's gate moved from 1e-4 to match, still green), and the `n/10` literal (`ften`) were
  copy-pasted across the modules; they now live in one place. 8 modules use `fd_central`/`fd_eps`,
  5 use `ften`; `mlp`'s `ften` and `ewc`'s `ften2` duplicates are gone. Behaviour is identical —
  the FD gates themselves verify the central-diff is correct (all M1–M5 gates remain green).
- **Deliberately NOT extracted** (audit R-1 + the per-gate perturb loop): the segment
  param-addressers and each gate's forward call stay in their modules. Each module is a readable
  standalone reference, and extracting the forward call would need function-value indirection —
  more risk and less clarity than the duplication it removes.

## [0.6.2] — 2026-06-24

Third patch of the 0.6.x hardening arc — security / supply-chain (audit S-1, M-1).

### Added
- **`SECURITY.md` — threat model + supply-chain posture (audit S-1).** Documents that prajna's
  attack surface is intentionally minimal (no untrusted input; `akshara` loaders stubbed to −1;
  non-crypto `tyche` PRNG; fully seeded/deterministic), so "security" here is numerical/memory
  correctness. **Dep-pin audit** (GitHub tags API, 2026-06-24): `rosnet 0.2.0`, `tyche 0.1.1`,
  `akshara 0.1.0` are all first-party and pinned at their **latest published tags**.

### Fixed
- **M-1: scratch buffers sized by `max(Ms,Mq)·max(K,H)`.** `M_ys`/`M_dy`/`M_dxd` (maml) and
  `NdxD`/`Ntmp` (mamlnl) are reused across the support and query passes and as `dX` dumps; they
  were sized for the support shape only (correct only while `Ms==Mq` and `H≥K`). Now sized by
  the max so a future `Ms≠Mq` config cannot overrun them. (Scratch only — no behavior change;
  all gates remain green.)

## [0.6.1] — 2026-06-24

Second patch of the 0.6.x hardening arc — numerical robustness (audit N-2, N-3).

### Fixed
- **N-2: `softmax_xent_fwd` floored the `ln` argument.** A target prob that underflows to 0
  (logit far below the row max → `exp` underflows) gave `ln 0 = −Inf` → NaN loss. The target
  prob is now floored to 1e-18 before `ln`, so the NLL stays finite. New test: a row with the
  target logit 1000 below the max yields a finite NLL.
- **N-3: divergence is now diagnosed, not silently swallowed.** The training-monotonicity
  checks used bare `f64_le(after, before)`, which (per N-1) returns 1 when `after` is NaN —
  a diverged run would silently *pass*. New `fd_le_finite` (in `fdgate.cyr`) requires the
  result to be finite; all 6 training checks in the demo + 6 in the test now use it. `print_f6`
  prints `nan` / `+inf` / `-inf` instead of the cryptic `--.------`. Negative test:
  `fd_le_finite(NaN, x) == 0`.

## [0.6.0] — 2026-06-24

First patch of the 0.6.x hardening arc (audit → NaN-safe gates). See
[`docs/development/audit-0.6.md`](docs/development/audit-0.6.md).

### Security
- **Fixed audit N-1: the FD gates were NaN-blind.** Every gate compared with
  `f64_le(|analytic − fd|, tol)`, and cyrius `f64_le` is `!f64_gt`, so `f64_le(NaN, tol) == 1`
  (probed) — a **NaN/Inf gradient silently PASSED** the gate instead of failing it. The
  verification harness that is prajna's whole correctness story could not catch its most likely
  failure mode. New `src/fdgate.cyr` adds a finite-guard (`fd_finite` via `f64_gt`, which is
  IEEE-correct on NaN) and routes every gate through `fd_match` / `fd_exceeds`, so a non-finite
  difference now **fails** the gate. All 9 gates + 3 observables across M1–M5 converted; a
  negative test asserts `fd_match(NaN,·)==0` and `fd_match(Inf,·)==0`. All M1–M5 gates remain
  green (finite-case behaviour unchanged).

## [0.5.0] — 2026-06-24

M5 — continual-learning durability, and the **completion of the M1–M5 roadmap**. prajna is now
a feature-complete sovereign meta-learning reference: 2nd-order MAML (R-operator), learned
optimizers (feedforward + recurrent), text few-shot on `akshara`, and forgetting-resistant
sequential adaptation. Remaining before v1.0: the freeze cycle (API + benchmarks + audit).

### Added
- **M5 — continual-learning durability (EWC + experience replay)** (`src/ewc.cyr`): the
  "don't catastrophically forget" safety glue for on-device self-adaptation. A 1→8→1 tanh MLP
  (backprop **FD-gated on all 25 params**) fits task A, then task B. **Naive** sequential
  B-training forgets A (loss 0.05 → 0.56); **experience replay** (interleaving A's data) keeps
  A essentially perfect (→ 0.00006) *and* fits B (0.178) — the robust durability method. **EWC**
  (diagonal-Fisher quadratic penalty `λF(θ−θ*)`, Kirkpatrick 2017) is implemented + FD-gateable,
  but reported **honestly as ineffective at this single-layer toy scale** (0.65, ≈ naive — the
  Fisher concentrates on shared output weights, no useful λ window). Test gates the backprop +
  the replay retention. **Completes the M1–M5 roadmap.**

## [0.4.0] — 2026-06-24

M4 — text few-shot meta-learning, the bridge into the ML family. prajna now consumes the
shared `akshara` tokenizer (with attn11/tarka/tentib), runs a tiny next-token LM, and
meta-learns to adapt it to a new text task in one inner step.

### Added
- **M4.b — MAML over text tasks: "MAML works on text"** (`src/textmaml.cyr`): few-shot
  meta-learning on the M4.a LM. Each task = a cyclic shift map `next[c]=(c+k) mod V` (k
  sampled); FOMAML meta-learns an init that adapts to a **new shift in one inner step**.
  Held-out few-shot adaptation NLL drops **2.37 → 0.49** (~79%, monotone) over meta-training
  — from uniform (ln 11) to a sharp fit. (The inner lr must be large enough — α=0.5 left one
  step unable to move an 11-way softmax; α=3 fits.) Reuses text.cyr's softmax-xent +
  akshara vocab; gradient-clipped meta-batch SGD. Test asserts the held-out improvement.
- **M4.a — next-token LM on `akshara`-tokenized text** (`src/text.cyr`): the text-model
  foundation for M4's text few-shot meta-learning, and the **tie-in to the rest of the ML
  family** (attn11/tarka/tentib share the `akshara` tokenizer). Tokenize text → next-token
  pairs; a tiny LM = token **embedding** → linear head → **softmax cross-entropy** (ported
  from attn11's ops via tentib). Hand-derived backprop incl. the embedding **scatter-gradient**
  — **FD-gated on all 72 params** ("hello world" → V=8, M=10; initial NLL 2.106 ≈ ln 8). Added
  `[deps.akshara]` 0.1.0. `cyrius test` gates it.

## [0.3.0] — 2026-06-24

M3 — the learned optimizer: meta-learn the *update rule* itself (not an initialization),
via BPTT through an unrolled optimization. Feedforward and recurrent optimizers, both
hand-derived and FD-gated; the meta-trained optimizer beats hand-tuned SGD.

### Added
- **M3.c — recurrent (RNN) learned optimizer** (`src/lrnn.cyr`): the Andrychowicz canonical —
  the optimizer carries a **hidden state across optimization steps** (a vanilla-RNN cell;
  the LSTM is the same idea with gates). The meta-gradient now needs BPTT through **two
  coupled recurrences** — the θ-trajectory `D_t` *and* the optimizer state `ds` (carried back
  via `dsf = Wsᵀ·da`). Hand-derived, **FD-gated on all 29 params** (first build); meta-trains
  to **0.085** held-out loss — beating the feedforward optimizer (0.096) and SGD (0.107), the
  recurrence buying extra expressiveness. Test gates the BPTT + the meta-training.
- **M3.b — meta-train the learned optimizer; it beats hand-tuned SGD** (`src/lopt.cyr`):
  meta-train `g_φ` (meta-batch SGD over the BPTT grad, **element-wise gradient clipping** for
  stability) across sampled quadratic optimizees (curvature `a∈[0.5,2.5]`, target `c∈[-2,2]`).
  Held-out trajectory loss drops **23.1 → 0.096**, and the meta-trained optimizer **beats the
  best fixed-lr SGD** (`0.096 < 0.107`) — its tanh nonlinearity adapts the step to the gradient
  magnitude where a single lr can't. Test asserts both the improvement and the SGD-beat.
- **M3.a — learned-optimizer BPTT meta-gradient** (`src/lopt.cyr`): the core of M3, the
  *second* realization of learn-to-learn — meta-learn the **update rule** (a 1→6→1 tanh net
  `g_φ` mapping gradient → update), not an initialization. Meta-loss = Σ optimizee losses
  over an **8-step unrolled** optimization of a quadratic; the meta-gradient `∂L/∂φ` is the
  hand-derived **BPTT** recurrence `D_t = D_{t+1} + dg·a + [t≥1]·a(θ_t−c)` (the `dg·a` term
  routes the gradient back through the optimizee Hessian). **FD-gated on all 19 optimizer
  params** (passed first build). `cyrius test` gates it.

## [0.2.0] — 2026-06-24

The MAML milestone (M2): the second-order meta-gradient proven from scalar → linear →
nonlinear, then demonstrated on sine-wave few-shot meta-learning. prajna is now a
sibling-on-rosnet that meta-learns end-to-end, sovereign from the metal up.

### Added
- **M2.c.3 — full-2nd-order vs FOMAML benchmark (the honest-negative)** (`src/sine.cyr`,
  `src/main.cyr`): meta-train both rules from an **identical** init + task schedule (only
  the meta-grad differs). **Finding: at this tiny scale they are statistically equivalent**
  (full `1.692` vs FOMAML `1.690`, |diff| ~0.1%) — the expensive Hessian-vector product
  buys little here, exactly Finn et al. 2017's FOMAML result, reproduced sovereignly. The
  reference's worth is that it can rigorously compute and compare both. `sine_benchmark` +
  `sine_train_batch_step_m(full)`; test asserts the FOMAML path also meta-learns.
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
