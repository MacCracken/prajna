# prajna — 0.6.x Audit / Hardening / Security / Refactor Arc

**Opened:** 2026-06-24 (after 0.5.0 / M1–M5 complete). **Goal:** harden the meta-learning
reference before the v1.0 freeze. Each theme below is its own **0.6.x patch**.

prajna's attack surface is intentionally tiny: a single-binary reference with **no untrusted
input** — every task is hardcoded or synthetic, the only file-capable dep (`akshara`) has its
loader symbols stubbed to `-1` (no syscalls), and randomness is a **non-crypto** statistical
PRNG (`tyche`) used only for task sampling + weight init. So "security" here is about
*numerical/memory correctness as a safety property*, not network/input threats.

## Findings (full read-through of all 11 modules + main + test, 2026-06-24)

| ID | Sev | Area | Finding |
|----|-----|------|---------|
| **N-1** | **HIGH** | numerical / verification | **The FD gates are NaN-blind.** Every gate is `f64_le(\|analytic−fd\|, tol)`. Demonstrated (M5 EWC divergence): `f64_le(NaN, x)` returns **1** in cyrius f64 — so a NaN/Inf gradient **silently PASSES** the gate instead of failing. The harness that *is* prajna's correctness story can't catch the one failure mode (NaN) that a future change is most likely to introduce. |
| N-2 | MED | numerical | `softmax_xent_fwd` calls `f64_ln(prob)` with no floor — a prob that underflows to 0 → `ln 0 = −Inf` → NaN loss (then N-1 hides it). attn11/tentib clamp; prajna should too. |
| N-3 | MED | numerical | No fail-fast on NaN/Inf in the training loops (`tm_train_step`, `lo/lr_train_step_batch`, `ec_step`, `sine_train_batch_step_m`). A diverging run produces `--.------` silently rather than aborting with a diagnosis. |
| M-1 | MED | memory | Scratch buffers are sized assuming **Ms == Mq** and fixed dim relations (`M_dxd=Ms*K` reused for `Mq*K` query dumps; `NdxD=Ms*H` for query; `NdWD=K*H` holds both `H` and `K*H` dumps). Correct for every current config, but a future `Ms≠Mq` or dim change overruns with no guard. Add `assert`s / size-by-max. |
| S-1 | LOW | supply-chain | Dep pins (`rosnet 0.2.0`, `tyche 0.1.1`, `akshara 0.1.0`, `cyrius 6.2.39`) — confirm each tag is current + intended; record the (first-party) provenance + the deliberate non-crypto-PRNG choice in a threat-model note. |
| R-1 | LOW | refactor | 8 near-identical `X_param_ptr`/`X_grad_val` segment-addressing pairs (mamlnl already generalized via `nm_at`). |
| R-2 | LOW | refactor | 8 near-identical FD-gate harnesses (`X_fd_grad` + `X_gate`). **Couples to N-1**: a shared gate helper means the NaN-guard is written once, not 8×. |
| R-3 | LOW | refactor | `ften`/`ften2` (both `n/10`) duplicated to avoid a name clash. |

## The 0.6.x patch plan

- **0.6.0 — Audit + NaN-safe gates (this doc + fix N-1).** Probe cyrius's f64 NaN-comparison
  behavior, add a finite-guard so every FD gate **fails** on NaN/Inf (not passes), re-run all
  gates green. The critical correctness fix; everything else builds on a trustworthy harness.
- **0.6.1 — Numerical robustness (N-2, N-3).** Floor `ln`'s argument in softmax-xent; add
  NaN/Inf fail-fast (or guard) in the training loops so divergence is diagnosed, not silent.
- **0.6.2 — Security / supply-chain (S-1, M-1).** Threat-model note (no untrusted input,
  non-crypto PRNG, stubbed loaders); dep-pin audit; `assert`/size-by-max the shared scratch.
- **0.6.3 — Refactor (R-1, R-2, R-3).** Extract the shared param-addressing + FD-gate harness
  (the N-1 guard lives in one place), dedup `ften`, shrink the surface before the v1.0 freeze.

> After 0.6.x the code is hardened and the harness is trustworthy → the **v1.0 freeze cycle**
> (API freeze + `docs/api.md`, `docs/benchmarks.md`, downstream consumer, security sign-off).
