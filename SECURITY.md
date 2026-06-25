# Security Policy — prajna

prajna is a **sovereign meta-learning reference binary** in Cyrius: hand-derived second-order
gradients on raw `f64`, no autodiff / BLAS / libc. This document is its threat model and
supply-chain posture, established in the **0.6.x hardening arc** (audit:
[`docs/development/audit-0.6.md`](docs/development/audit-0.6.md)).

## Threat model — attack surface is intentionally minimal

- **No untrusted input.** Every task is hardcoded or synthetically sampled in-process. prajna
  reads no files, no stdin, no network. The only file-capable dependency (`akshara`) has its
  loader symbols (`secure_read_file` / `read_stdin` / `file_seek`) **stubbed to return −1** in
  prajna's build units — no syscalls, no file path is ever opened. There is no parser, no
  deserializer, and no user-controlled length/index reaching memory operations.
- **Randomness is non-cryptographic by design.** `tyche` is a statistical PRNG used only for
  task sampling and weight initialization; prajna makes **no security claim** on it and never
  uses it for keys, nonces, or tokens. (Cryptographic randomness in the AGNOS ecosystem lives
  in `sigil`, not here.)
- **Determinism.** All randomness is seeded (`rng_seed`), so every run is reproducible — there
  is no entropy-dependent behavior to exploit.

Because there is no untrusted input, "security" for prajna is primarily a **numerical/memory
correctness** property: a silent NaN, an unguarded buffer, or a verification gate that can be
fooled. The 0.6.x arc targets exactly these:

- **0.6.0** — the FD gates were NaN-blind (`f64_le(NaN,·)==1` let a NaN gradient pass). Fixed
  with a finite-guard (`src/fdgate.cyr`); every gate now fails on NaN/Inf.
- **0.6.1** — floored `ln` in softmax cross-entropy; `fd_le_finite` makes a diverged (NaN)
  training result fail rather than silently pass.
- **0.6.2** — this document + the scratch-buffer hardening below.

## Memory-safety posture

prajna uses raw `t_alloc` + `load64`/`store64` with manual indexing (no bounds-checked
containers — that is the sovereign-Cyrius idiom). Correctness rests on each scratch buffer being
sized for the largest shape it ever holds. As of **0.6.2**, the buffers reused across the
support/query passes and as `dX` dumps are **sized by `max(Ms,Mq)·max(K,H)`** (see `maml.cyr`,
`mamlnl.cyr`), so a future config with `Ms ≠ Mq` cannot overrun them. Index arithmetic operates
on demo-scale dimensions (`V ≤ 11`, `M ≤ 40`) well within `i64`.

## Supply chain

All dependencies are **first-party** (the `MacCracken` org) and **pinned at their latest
published tags** (audited 2026-06-24 via the GitHub tags API):

| dep | pinned | role |
|-----|--------|------|
| `rosnet` | 0.2.0 | f64 tensors + `linear_fwd`/`linear_bwd` |
| `tyche`  | 0.1.1 | statistical PRNG (non-crypto) |
| `akshara`| 0.1.0 | byte-vocab tokenizer (loaders stubbed) |
| `cyrius` | 6.2.39 | toolchain (manifest `${file:VERSION}` for prajna itself) |

The build pins are the single source of truth; releases re-clone each tag in a clean CI
environment. There are no third-party or registry dependencies.

## Reporting

prajna is a **1.0 stable reference** in the AGNOS ecosystem — research/reference software with no
production deployment surface. Report suspected correctness or safety issues via the repository's
issue tracker.
