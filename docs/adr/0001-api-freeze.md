# 0001 — Freeze the public API at 1.0

**Status**: Accepted
**Date**: 2026-06-24

## Context

prajna reached feature completeness at 0.5.0 (the M1–M5 roadmap) and finished a four-patch
hardening arc at 0.6.3 (audit, NaN-safe gates, numerical robustness, security, refactor). The
remaining gate to 1.0 is a stability commitment: consumers and the sibling references
(attn11/tarka/tentib) need to know what they can rely on. But prajna is a **reference binary**,
not a conventional library — most of its ~150 functions are internal scratch plumbing, and its
numerical outputs shift in the last digit across cyrius toolchain versions. A naive "freeze every
symbol and every number" would be both meaningless and impossible to honor.

## Decision

Freeze a **defined public surface**, documented in [`docs/api.md`](../api.md), as of **1.0.0**.
In scope: the `fdgate.cyr` infrastructure (the one directly-consumable module) and the
per-milestone *reference interface* (`*_init` / `*_forward` / `*_backward` / `*_meta_grad` /
`*_gate` / `*_train_*` / `*_eval_*`). Out of scope: module-global buffer names, the
`*_param_ptr`/`*_grad_val` addressers, and the demo's exact printed text. The **contract is the
gates** (FD-match, monotonic improvement, "beats SGD"), not the literal digits. A breaking change
to anything in scope requires a major (2.0) bump.

## Consequences

- **Positive** — consumers can vendor `fdgate` and study/adapt the algorithm interfaces with a
  stability guarantee; 1.0.0 becomes a clean cut (version + a small doc pass) rather than a scramble.
- **Negative** — we now own backward compatibility for those signatures; adding a parameter to a
  frozen function is a major bump, so future extensions favor new functions over changed ones.
- **Neutral** — the digit-level results are explicitly *not* frozen, so toolchain upgrades that
  perturb the last decimal are non-events as long as the gates stay green.

## Alternatives considered

- **Freeze nothing (stay 0.x forever).** Rejected: the siblings are reaching 1.0 and the ecosystem
  needs a stable meta-learning reference to point at; perpetual 0.x signals unfinished, which is false.
- **Freeze every symbol + exact outputs.** Rejected: impossible to honor (digits drift with the
  toolchain) and meaningless (internal scratch isn't a contract anyone wants).
- **Ship `fdgate` as a separate library repo.** Rejected for now: it is small and tightly coupled
  to prajna's idioms; a split can happen later if external demand appears, without breaking this API.
