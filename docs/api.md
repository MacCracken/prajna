# prajna — Public API (frozen at 1.0)

This is the stable surface of prajna. From **1.0.0** onward these signatures are **frozen**:
a breaking change to anything below requires a **major** version bump (2.0). Internal scratch
buffers, allocation helpers, and the per-module `*_param_ptr`/`*_grad_val` addressers are
**not** part of the frozen surface and may change in minor releases.

prajna is a **reference binary**: the primary artifact is `./build/prajna` (the demo, exit 0 ⇔
all gates pass) and `cyrius test`. The one **directly consumable library module** is `fdgate.cyr`;
the rest of the API is the per-milestone *reference interface* — stable so the algorithms can be
studied and adapted, but exercised through the demo/test rather than linked.

## Conventions

- Everything is `f64` stored as an `i64` bit-pattern; all functions are `i64`-typed. Tensors are
  raw `t_alloc` buffers (row-major). There is no autodiff, BLAS, or libc.
- Each milestone module follows the same shape: `X_init()` (allocate + seed) → `X_forward()` /
  `X_backward()` (or `X_meta_grad()`) → `X_gate()` (returns 1 iff the hand-derived gradient
  matches finite differences) → optional `X_train_*` / `X_eval_*`.
- Gates return `1` (pass) / `0` (fail) and are **NaN-safe** as of 0.6.0 (see `fdgate`).

## `fdgate.cyr` — finite-difference-gate infrastructure (consumable)

The reusable core. Any sovereign-Cyrius project that hand-derives gradients can vendor this module
to gate them safely. See [`examples/`](../examples/) for a standalone consumer.

| function | contract |
|----------|----------|
| `fd_big() -> i64` | a finite sentinel (1e18) above any real gradient/diff |
| `fd_finite(v) -> i64` | `1` iff `v` is finite (rejects NaN, ±Inf) — uses NaN-safe `f64_gt` |
| `fd_match(analytic, fd, tol) -> i64` | `1` iff `analytic` matches `fd` within `tol` **and the diff is finite** (the gate pass condition; a NaN gradient FAILS) |
| `fd_exceeds(analytic, fd, tol) -> i64` | `1` iff they differ by **more** than `tol`, finitely (the "2nd-order observable" check) |
| `fd_le_finite(a, b) -> i64` | `1` iff `a` is finite and `a <= b` (training-monotonicity check that a diverged NaN can't fool) |
| `fd_eps() -> i64` | the central-difference step (1e-5) |
| `fd_central(hi, lo, eps) -> i64` | `(hi - lo) / (2·eps)` |
| `ften(n) -> i64` | `n/10` as `f64` (param/data init literal) |

## Meta-gradient core — M1 (`meta.cyr`)

`meta_alpha()`, `inner_step(t)`, `meta_loss(t)`, `meta_grad_full(t)` (2nd-order),
`meta_grad_fomaml(t)` (1st-order), `meta_grad_fd(t)` (FD truth), `gate_pass(t)`,
`second_order_observable(t)`.

## MAML — M2 (`maml.cyr`, `mlp.cyr`, `mamlnl.cyr`, `sine.cyr`)

- **Linear (M2.a)** `maml.cyr`: `maml_init()`, `maml_adapt()`, `maml_meta_loss()`,
  `maml_meta_grad()`, `maml_gate()`, `maml_2nd_observable()`, `maml_an_full(idx)`,
  `maml_an_fomaml(idx)`, `maml_fd_grad(idx)`; dims `maml_k/maml_ms/maml_mq/maml_alpha`.
- **Nonlinear backprop (M2.b.1)** `mlp.cyr`: `mlp_init()`, `mlp_forward()`, `mlp_backward()`,
  `mlp_grad_gate()`; dims `mlp_k/mlp_h/mlp_ms/mlp_np`.
- **R-operator engine (M2.b.2)** `mamlnl.cyr`: `nm_init()` (gate config) / `nm_config(k,h,ms,mq)`
  (arbitrary dims) / `nm_set_alpha(a)`; `nm_support_fwd_grad()`, `nm_adapt()`,
  `nm_query_grad_adapted()`, `nm_rpass(vW1,vb1,vW2,vb2)` (the HVP), `nm_meta_grad()`,
  `nm_meta_loss()`; gates `nm_grad_gate()`, `nm_hvp_gate()`, `nm_meta_grad_gate()`,
  `nm_2nd_observable()`; accessors `nm_meta_val/nm_fomaml_val/nm_hvp_val/nm_param_ptr`.
- **Sine few-shot (M2.c)** `sine.cyr`: `f64_sin(x)`, `f64_pi()`, `sine_validate()`,
  `sine_sample_task()`, `sine_init_params()`, `sine_sanity()`, `sine_eval(n,seed)`,
  `sine_opt_init()`, `sine_train_batch_step(B,mlr)` / `sine_train_batch_step_m(B,mlr,full)`,
  `sine_benchmark(full)`.

## Learned optimizers — M3 (`lopt.cyr`, `lrnn.cyr`)

- **Feedforward (M3.a/b)** `lopt.cyr`: `lo_init()`, `lo_set_task(a,c,th0)`, `lo_forward()`,
  `lo_meta_grad()` (BPTT), `lo_gate()`, `lo_opt_init()`, `lo_sample_task()`,
  `lo_train_step_batch(B,mlr)`, `lo_eval_learned(n,seed)`, `lo_sgd_unroll(lr)`,
  `lo_eval_sgd(n,seed,lr)`.
- **Recurrent (M3.c)** `lrnn.cyr`: `lr_init()`, `lr_forward()`, `lr_meta_grad()`
  (two-recurrence BPTT), `lr_gate()`, `lr_train_step_batch(B,mlr)`, `lr_eval_learned(n,seed)`.

## Text few-shot — M4 (`text.cyr`, `textmaml.cyr`)

- **Next-token LM (M4.a)** `text.cyr`: `softmax_xent_fwd(logits,targets,probs,M,V)`,
  `softmax_xent_bwd(probs,targets,dlogits,M,V)`, `tx_init(text,len,D)`, `tx_forward()`,
  `tx_backward()`, `tx_gate()`.
- **Text-MAML (M4.b)** `textmaml.cyr`: `tm_init()`, `tm_adapt(alpha)`, `tm_meta_grad(alpha)`
  (FOMAML), `tm_query_nll(alpha)`, `tm_train_step(B,mlr,alpha)`, `tm_eval(ntasks,seed,alpha)`.

## Continual learning — M5 (`ewc.cyr`)

`ec_init()`, `ec_set_a()` / `ec_set_b()` / `ec_set_ab()` (replay), `ec_forward()`,
`ec_backward()`, `ec_grad_gate()`, `ec_fisher()` (diagonal Fisher), `ec_snapshot()` / `ec_reset()`,
`ec_step(lr, ewc, lam)` (EWC penalty when `ewc==1`), `ec_train(steps,lr,ewc,lam)`,
`ec_loss_a()` / `ec_loss_b()`.

## Stability policy

- **Frozen at 1.0**: every function listed above (name, arity, and meaning).
- **Not frozen**: module-global buffer names (`M_*`, `N*`, `LO_*`, `TX_*`, `EC_*`, …), the
  `*_param_ptr`/`*_grad_val`/`*_fd_*` addressers, and the demo's exact printed text.
- Numerical results may shift in the last decimal across toolchain versions; the **gates** (FD
  match, monotonic improvement, "beats SGD") are the stable contract, not the literal digits.
