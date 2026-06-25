# prajna — Current State

> Refreshed every release. CLAUDE.md is preferences/process/procedures (durable);
> this file is **state** (volatile).

## Version

**1.0.0** — **first stable release**, cut 2026-06-24. The complete sovereign meta-learning
reference: M1–M5 (2nd-order MAML / learned optimizers / text few-shot / continual learning),
hardened across the 0.6.x arc, public API frozen ([`api.md`](../api.md) + ADR 0001). Shipped
with [`benchmarks.md`](../benchmarks.md), a downstream-consumer example ([`examples/`](../../examples/)),
and `SECURITY.md`. (M1–M5 = 0.1.0–0.5.0; hardening = 0.6.0–0.6.3; 0.9.0 = the RC.)

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
  `sine_sanity` (3 FD gates at the sine config — generalization); **M2.c.2/.3**: `sine_eval`,
  `sine_opt_init`, `sine_train_batch_step_m(full)` (meta-batch SGD; full or FOMAML),
  `sine_benchmark`. `nm_set_alpha` makes the inner lr configurable.
- `src/lopt.cyr` — **M3.a/.b**: feedforward learned optimizer (1→6→1 tanh `g_φ`,
  gradient→update) unrolled 8 steps on a quadratic optimizee. `lo_forward`/`lo_meta_grad` (BPTT)
  /`lo_gate` (FD over 19 params); `lo_sample_task`/`lo_train_step_batch` (gradient-clipped
  meta-batch SGD)/`lo_eval_learned`/`lo_eval_sgd` (SGD baseline).
- `src/lrnn.cyr` — **M3.c**: recurrent (RNN) learned optimizer — hidden state across steps;
  `lr_forward`/`lr_meta_grad` (two-recurrence BPTT)/`lr_gate` (FD over 29 params); reuses
  `lopt.cyr`'s optimizee + sampler. The 2nd realization of learn-to-learn (meta-learn the
  *update rule*), feedforward + recurrent.
- `src/text.cyr` — **M4.a**: next-token LM on `akshara`-tokenized text (token embedding →
  linear head → softmax cross-entropy, ported from attn11). `tx_init`/`tx_forward`/`tx_backward`
  (incl. embedding scatter-grad)/`tx_gate` (FD). The tie-in to the ML family (shared tokenizer).
- `src/textmaml.cyr` — **M4.b**: MAML over text shift-tasks on the M4.a LM. `tm_init`,
  `tm_set_support`/`tm_set_query`, `tm_adapt` (inner step), `tm_meta_grad` (FOMAML),
  `tm_query_nll`, `tm_train_step` (gradient-clipped meta-batch), `tm_eval` (held-out NLL).
- `src/ewc.cyr` — **M5**: continual-learning durability. 1→8→1 tanh MLP (`ec_forward`/
  `ec_backward`/`ec_grad_gate`); `ec_fisher` (diagonal Fisher), `ec_snapshot`/`ec_reset`,
  `ec_step` (EWC penalty), `ec_set_ab` (experience replay). Replay = the robust method.
- `src/fdgate.cyr` — **0.6.0**: NaN-safe FD-gate comparison (`fd_finite`/`fd_match`/`fd_exceeds`).
  Every gate routes through it; a NaN/Inf difference *fails* the gate (cyrius `f64_le(NaN,·)==1`
  used to let it pass). Shared chokepoint for all 9 gates + 3 observables.
- `src/main.cyr` — demo: M1 → M2.* → M3.* → M4.* → M5, per-stage gates. Exit 0 iff all pass.
- `src/test.cyr` — `[build].test`: NaN-guard assertions + all FD gates + M2.c/M3/M4/M5 results.

## Verification (1.0.0, 2026-06-24)

- `cyrius build` → OK; `./build/prajna` → exit 0; `cyrius test` → 2 passed / 0 failed;
  all 13 `src/*.cyr` modules lint clean (0 warnings); the `examples/` consumer builds + exits 0.
- **Every hand-derived gradient is FD-gated** and NaN-safe (0.6.0): M1 (scalar) · M2.a (4 params) ·
  M2.b.1 (13) · M2.b.2 three-level R-operator gate · M3.a (19) · M3.c (29) · M4.a (72) · M5 (25).
- **Headline convergence**: M3.b learned optimizer **beats best fixed-lr SGD** (0.096 vs 0.107);
  M4.b text-MAML held-out NLL **2.37 → 0.49**; M5 replay retains task A (0.56 naive → 0.00006).
- **Full footprint + per-milestone numbers**: [`benchmarks.md`](../benchmarks.md) (the authoritative
  benchmark record — this section is a summary, not a duplicate).

## Dependencies

Direct (declared in `cyrius.cyml`):
- stdlib — string, fmt, alloc, io, vec, str, syscalls, assert, bench, math, **ganita**
  (`f64_tanh`).
- **`rosnet` 0.2.0** (f64 tensors + `linear_fwd`/`linear_bwd`) + **`tyche` 0.1.1**
  (PRNG; mandatory link dep for rosnet's `t_randn`). `akshara` joins at **M4**.

## Consumers

- [`examples/gate_consumer.cyr`](../../examples/) — vendors `fdgate` to FD-gate an external
  gradient (the reference example for downstream consumption).
- Pattern-shared with the sibling references (attn11/tarka/tentib all FD-gate hand-derived
  gradients the same way).

## Next

**1.0.0 shipped — no active engineering.** prajna is feature-complete and stable; the API is
frozen (a breaking change is a 2.0). Post-1.0 work is maintenance only: dep folds when
rosnet/tyche/akshara cut new GA tags, and any future ML-reference ideas the family surfaces
(none planned). See [`roadmap.md`](roadmap.md).
