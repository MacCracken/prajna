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

### M1 — Scalar second-order meta-gradient (→ v0.2.0) — ✅ core landed 2026-06-24
The minimal **exact** proof of the primitive, pure scalar f64 (no rosnet yet):
- `src/meta.cyr` — scalar quadratic MAML: support/query losses, the inner SGD step,
  the meta-loss, and the analytic **second-order** meta-gradient
  `dM/dt = Lq'(t')·(1 − α·Ls'')`.
- **FD gate** green: analytic `−1.919999` vs central finite-diff `−1.920000`.
- **Honest contrast**: FOMAML (first-order, drops the `(1−α·Ls'')` factor) differs
  from FD by exactly `0.48` — the second-order term is *observably real*.
- Meta-descent converges `θ: 0.5 → 3.499` to the optimum `3.5`.
- `cyrius test` asserts gate + observability + descent.
- **Acceptance met**; tag `0.2.0` is the user's to cut.

### M2 — MAML few-shot regression on rosnet (v0.3.0)
Scale the primitive from scalar to a real model.
- **Dep gate**: add `rosnet` (f64 tensors + `linear_fwd`/`linear_bwd`) and `tyche`
  (task sampling). The meta-gradient backprops *through* the inner adapt step of a
  small MLP — hand-derived on rosnet's first-order ops (rosnet stays first-order;
  prajna supplies the nesting).
- **Demo**: meta-learn an initialization on sampled sine-wave tasks; show K-shot fast
  adaptation (1–few steps) beats a non-meta baseline. Benchmark vs the MAML paper's
  sine-regression setup (B-series fairness rules).
- **Honest-negative eligible**: report whether the full second-order term beats
  FOMAML/Reptile *at this tiny scale* — a finding either way (attn11-MTP style).

### M3 — Learned optimizer (v0.4.0)
The second realization of learn-to-learn on the same meta-grad machinery.
- A small net meta-trained to emit the update step ("learning to learn by gradient
  descent by gradient descent", Andrychowicz 2016) — meta-learn the *update rule*
  rather than the *initialization*.

### M4 — Text few-shot / attn11 tie-in (v0.5.0)
- **Dep gate**: add `akshara` (tokenizer). Meta-learn over a tiny LM few-shot task —
  the bridge into the rest of the ML family.

### M5 — Continual-learning durability (v0.6.0)
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
