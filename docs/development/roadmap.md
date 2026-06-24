# prajna — Roadmap

> Milestone plan through v1.0. State lives in [`state.md`](state.md); this file is
> the sequencing — what ships, in what order, against what dependency gates.
>
> **Thesis:** prove the family's **first second-order / nested gradient** — the
> meta-gradient, differentiating a loss *through* an inner SGD step (MAML /
> learned-optimizer) — is expressible assembly-up in everything-is-i64 Cyrius,
> hand-derived and finite-difference-gated over a two-level computation.

## v1.0 criteria (mirrors tarka / tentib)

- [ ] Public meta-learning API frozen — documented + tested
- [ ] Every hand-derived backward (incl. the **second-order** meta-grad) finite-difference grad-checked
- [ ] Benchmarks captured in `docs/benchmarks.md` (vs the MAML sine-regression reference)
- [ ] At least one downstream consumer green
- [ ] CHANGELOG complete from v0.1.0 onward
- [ ] Security audit pass (`docs/audit/YYYY-MM-DD-audit.md`)

## Milestones

### M0 — Scaffold (v0.1.0) — ✅ shipped 2026-06-24
- `cyrius init` scaffold; identity + the attn11/tarka/tentib boundary documented.
- Name = `prajna` (प्रज्ञा), completing the cognition cluster pramana → tarka → prajna.

### M1 — Scalar second-order meta-gradient — ✅ shipped in **0.1.0** (2026-06-24)
The minimal **exact** proof of the primitive, pure scalar f64 (no rosnet):
- `src/meta.cyr` — scalar quadratic MAML; analytic **second-order** meta-gradient
  `dM/dt = Lq'(t')·(1 − α·Ls'')`. **FD gate** green (analytic `−1.919999` vs FD
  `−1.920000`); FOMAML differs by `0.48` (the second-order term is *observably real*);
  meta-descent `θ: 0.5 → 3.499` to the optimum `3.5`. Released as part of `0.1.0`.

### M2 — MAML few-shot regression on rosnet (→ v0.2.0)
Scale the primitive from scalar to a real model. Broken into verified sub-bites:
- **M2.a — linear-MAML on rosnet — ✅ landed 2026-06-24** (`src/maml.cyr`). Wired
  `rosnet` 0.2.0 + `tyche` 0.1.1 (prajna is now a sibling-on-rosnet). Single linear
  layer (K=3→1, MSE, one inner SGD step); the second-order meta-grad
  `dLq/dθ = gθ' − α·(H_s·gθ')` hand-derived, the HVP `H_s·v` computed with one extra
  `linear_fwd`+`linear_bwd` (constant Hessian for linear+MSE). **FD gate matches on
  all 4 params exactly**; FOMAML wrong-signed on `W[0]` (`−0.393` vs `+0.031`);
  meta-descent monotone. `cyrius test` gates M1 + M2.a.
- **M2.b — MLP + nonlinearity** (the θ-dependent Hessian). Sub-split:
  - **M2.b.1 — tanh MLP forward + first-order backprop — ✅ landed 2026-06-24**
    (`src/mlp.cyr`): K=2→H=3→1 tanh MLP on rosnet; hand-derived ∇Ls **FD-gated on all
    13 params**. Added the `ganita` stdlib module (`f64_tanh`). The nonlinear foundation.
  - **M2.b.2 — the R-operator HVP / double-backward — ✅ landed 2026-06-24**
    (`src/mamlnl.cyr`): the θ-dependent Hessian's HVP `H_s·v` via the hand-derived
    **Pearlmutter R-operator**, then the full second-order meta-grad `gθ' − α·(H_s·gθ')`
    through the MLP's inner step. Equations confirmed three ways + NumPy FD cross-check;
    **three-level FD gate all green** (∇Ls / HVP-vs-FD-of-gradient / meta-grad-vs-FD-of-
    meta-loss); FOMAML observably differs; meta-descent monotone. **M2.b complete.**
- **M2.c — sine-task few-shot meta-learning** (the demonstration). Sub-split:
  - **M2.c.1 — sine substrate + engine generalization — ✅ landed 2026-06-24**
    (`src/sine.cyr`): sovereign `f64_sin` (range-reduced Taylor, validated), the sine-task
    sampler (`y=A·sin(x+φ)` via `tyche`), and the `nm_config` refactor making the engine
    dimension/data-configurable. The three FD gates pass at the sine config (K=1, H=8) on a
    sampled task — the verified R-operator generalizes.
  - **M2.c.2 — the meta-training loop — ✅ landed 2026-06-24** (`src/sine.cyr`):
    meta-train the MLP initialization across sampled sine tasks (full 2nd-order meta-grad
    + 16-task meta-batch SGD). Held-out 10-shot adaptation loss descends **monotonically
    2.48 → 1.56** over 1000 steps — meta-training measurably improves few-shot adaptation
    ("MAML works on AGNOS"). Inner lr configurable; test asserts the improvement.
  - **M2.c.3 — the benchmark — ✅ landed 2026-06-24** (`src/sine.cyr`): full-2nd-order vs
    FOMAML from an identical init+schedule. **Honest-negative finding: at this tiny scale
    they are statistically equivalent** (full `1.692` vs FOMAML `1.690`, |diff| ~0.1%) —
    the HVP buys little here, exactly Finn et al. 2017's FOMAML result. (Reptile is
    degenerate at one inner step — needs a multi-step inner loop, deferred.) **M2 complete.**

> **▶ M1 + M2 SHIPPED as `0.2.0` (2026-06-24).** The second-order meta-gradient is proven
> scalar → linear → nonlinear (R-operator) and demonstrated on sine few-shot meta-learning.

### M3 — Learned optimizer (→ v0.3.0)
The second realization of learn-to-learn: meta-learn the *update rule* (a small net that
emits the optimization step) rather than the *initialization* ("learning to learn by
gradient descent by gradient descent", Andrychowicz 2016). Sub-split:
- **M3.a — the BPTT meta-gradient core — ✅ landed 2026-06-24** (`src/lopt.cyr`): a 1→6→1
  tanh optimizer `g_φ` (gradient → update), unrolled 8 steps on a quadratic optimizee;
  meta-loss = Σ optimizee losses; meta-grad `∂L/∂φ` via the hand-derived **BPTT** recurrence
  (`dg·a` routes through the optimizee Hessian). **FD-gated on all 19 params** (first build).
- **M3.b — meta-train the optimizer; it beats SGD — ✅ landed 2026-06-24** (`src/lopt.cyr`):
  meta-train `φ` (meta-batch SGD over the BPTT grad, **gradient-clipped** for stability) across
  sampled quadratics. Held-out trajectory loss **23.1 → 0.096**; the meta-trained optimizer
  **beats the best fixed-lr SGD** (0.096 vs 0.107) — genuine "learning to learn." Test asserts
  the improvement + the SGD-beat.
- **M3.c — recurrent (RNN) optimizer — ✅ landed 2026-06-24** (`src/lrnn.cyr`): the Andrychowicz
  canonical — a hidden state carried across optimization steps (vanilla-RNN cell; LSTM = same
  idea + gates). Meta-grad via BPTT through **two coupled recurrences** (θ-trajectory `D` +
  state `ds`). **FD-gated on all 29 params** (first build); meta-trains to **0.085**, beating
  both the feedforward optimizer (0.096) and SGD (0.107). **M3 complete.**

> **▶ M3 SHIPPED as `0.3.0` (2026-06-24).** Meta-learning the update rule — feedforward and
> recurrent learned optimizers, hand-derived BPTT, FD-gated, beating hand-tuned SGD.

### M4 — Text few-shot / attn11 tie-in (→ v0.4.0)
Meta-learn over a tiny LM few-shot task; the bridge into the ML family (shared `akshara`
tokenizer). Sub-split:
- **M4.a — the text-model foundation — ✅ landed 2026-06-24** (`src/text.cyr`): wired
  `akshara` (the family tokenizer); a tiny next-token LM = token embedding → linear head →
  softmax cross-entropy (ported from attn11). Hand-derived backprop incl. the embedding
  scatter-gradient — **FD-gated on all 72 params** ("hello world" → V=8, M=10 pairs).
- **M4.b — MAML over text tasks — ✅ landed 2026-06-24** (`src/textmaml.cyr`): each task =
  a cyclic shift map `next[c]=(c+k) mod V`; FOMAML meta-learns an init that adapts to a new
  shift in **one inner step**. Held-out few-shot adaptation NLL **2.37 → 0.49** (~79%, monotone)
  — "MAML works on text," sovereignly. Gradient-clipped meta-batch SGD; test asserts the
  improvement. **M4 complete.**

> **▶ M4 SHIPPED as `0.4.0` (2026-06-24).** Text few-shot meta-learning on the shared
> `akshara` tokenizer — prajna is wired into the ML family's data layer (attn11/tarka/tentib).

### M5 — Continual-learning durability (→ v0.5.0)
- **EWC** (Fisher penalty) + experience replay so sequential meta-adaptations don't
  catastrophically forget — the safety glue the self-improvement-lane flags as
  mandatory for any on-device self-updater.

## Out of scope (for v1.0)

- **The test-time-memory / Titans incarnation** (TTT/Titans learned-memory at
  inference) — that is the `generative-paradigms.md` **Beyond / Type-2** rung, gated
  behind the Type-2 recurrent reference; prajna only supplies its differentiable
  core. Not built here until Type-2 exists.
- **GPU** (mabda) — CPU f64 only; the meta-grad is the point, not throughput.
- **The recursive-self-improvement orchestration** (generate→verify→retrain) — a
  *tarka recipe*, mapped in agnosticos `planning/self-improvement-lane.md`, not prajna.
- **RL / reward / search** — tarka's, always.
