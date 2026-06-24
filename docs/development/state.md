# prajna — Current State

> Refreshed every release. CLAUDE.md is preferences/process/procedures (durable);
> this file is **state** (volatile).

## Version

**0.1.0** — scaffolded 2026-06-24 via `cyrius init`. M1 core landed (toward 0.2.0,
untagged). No releases tagged yet.

## Toolchain

- **Cyrius pin**: `6.2.39` (in `cyrius.cyml [package].cyrius`; set by `cyrius init`).

## Surface

- `src/meta.cyr` — **M1 meta-gradient core** (scalar quadratic MAML): `ls_grad`,
  `lq`, `lq_grad`, `inner_step`, `meta_loss`, `meta_grad_full` (2nd-order),
  `meta_grad_fomaml` (1st-order), `meta_grad_fd` (finite-diff), `gate_pass`,
  `second_order_observable`. Pure scalar f64, no autodiff.
- `src/main.cyr` — demo: prints full/FOMAML/FD meta-grads, the FD gate, and
  meta-descent to the optimum. Exit 0 iff gate + observability both pass.
- `src/test.cyr` — `[build].test`: asserts FD gate, second-order observability, and
  one-step meta-loss descent.

## Verification (2026-06-24)

- `cyrius build src/main.cyr build/prajna` → OK; `./build/prajna` → exit 0.
  - `|full − FD| = 0.000000` (gate PASS); `|FOMAML − FD| = 0.48` (2nd-order observable).
  - meta-descent `θ: 0.5 → 3.499` (optimum 3.5), meta_loss `2.88 → 0.000000`.
- `cyrius test` → 2 passed / 0 failed (+ smoke).

## Dependencies

Direct (declared in `cyrius.cyml`):
- stdlib — string, fmt, alloc, io, vec, str, syscalls, assert, bench, **math**.
- `rosnet` / `tyche` join at **M2** (MAML on an MLP); `akshara` at **M4** (text few-shot).

## Consumers

_None yet._

## Next

**M2** — MAML few-shot regression on `rosnet` (the meta-gradient backprops through an
MLP's inner adapt step). See [`roadmap.md`](roadmap.md).
