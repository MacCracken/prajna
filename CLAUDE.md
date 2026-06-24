# prajna — Claude Code Instructions

> **Core rule**: this file is **preferences, process, and procedures** —
> durable rules that change rarely. Volatile state (current version,
> module line counts, supported backends, test counts, dep-gap status,
> consumers) lives in [`docs/development/state.md`](docs/development/state.md).
> Do not inline state here.

## Project Identity

**prajna** (प्रज्ञा — *discerning insight / the cognition that refines itself*, the
Nyāya/Yoga faculty of meta-cognition; in the Yoga Sūtras *ṛtaṃbharā prajñā* is the
insight that arises and refines through practice) — the sovereign **meta-learning /
learn-to-learn** reference in Cyrius. Sibling to
[attn11](https://github.com/MacCracken/attn11) (transformer),
[tarka](https://github.com/MacCracken/tarka) (RL/reasoning), and
[tentib](https://github.com/MacCracken/tentib) (ternary) on the shared
rosnet/tyche f64 substrate. It completes the Sanskrit cognition cluster
**pramana** (valid cognition) → **tarka** (inference) → **prajna** (meta-cognition).

- **Type**: Binary
- **License**: GPL-3.0-only
- **Language**: Cyrius (toolchain pinned in `cyrius.cyml [package].cyrius`)
- **Version**: `VERSION` at the project root is the source of truth — do not inline the number here
- **Standards**: [First-Party Standards](https://github.com/MacCracken/agnosticos/blob/main/docs/development/applications/first-party-standards.md) · [First-Party Documentation](https://github.com/MacCracken/agnosticos/blob/main/docs/development/applications/first-party-documentation.md)

## Goal

prajna owns the **meta-gradient** — the family's **first second-order / nested
gradient**: differentiating a loss **through** an inner SGD step (the MAML /
learned-optimizer "learning to learn" primitive). attn11, tarka, and tentib are all
**first-order** (attention grad / policy grad / STE surrogate); prajna proves that
**bi-level optimization — a model that learns *how* to learn — is expressible
assembly-up in everything-is-i64 Cyrius**, hand-derived and **finite-difference-gated
over a two-level computation**. Lead algorithm: **MAML** (meta-learn an
initialization for fast few-shot adaptation); sibling: **learned optimizers**
(meta-learn the update rule itself).

### Boundary — what prajna does and does NOT own

- **Owns**: the second-order meta-gradient primitive + its FD-gate-over-two-levels;
  MAML (learned init) and learned-optimizer (learned update rule); the inner-adapt /
  outer-meta loop structure.
- **Consumes** (sibling on the shared substrate, **not** chained through the other
  binaries): `rosnet` (f64 tensors + first-order `linear_fwd`/`linear_bwd`), `tyche`
  (task sampling / PRNG), `akshara` (tokenizer — only once a text few-shot task
  arrives at M4).
- **Does NOT own**: RL / reward / search / MCTS (**tarka**); the transformer
  architecture (**attn11**); ternary / quantization (**tentib**); the **outer
  recursive-self-improvement flywheel** (generate→verify→retrain orchestration — a
  *recipe over tarka+attn11*, mapped in agnosticos
  `docs/development/planning/self-improvement-lane.md`). prajna is the
  *differentiable meta-learning core* that exploration surfaced as the lane's **one
  genuine new primitive** — not the orchestration.
- **Relationship to the test-time-learning rung**: TTT / Titans (test-time learned
  memory — the `generative-paradigms.md` "Beyond" rung) is a *consumer/cousin* of
  prajna's nested grad; it is gated behind the **Type-2 recurrent reference** and
  would consume prajna once both exist.

## DO NOT

- **Do not commit or push** — the user handles all git operations.
- **Never use `gh`** — `curl` the GitHub API if needed.
- Do not add unnecessary dependencies (M1 is scalar f64 — stdlib `math` only;
  rosnet/tyche join at M2, akshara at M4).
- Do not re-own anything in the Boundary "does NOT own" list — if a change wants
  RL/search/a reward model, it belongs in **tarka**.

## Current State

> Volatile state lives in [`docs/development/state.md`](docs/development/state.md) —
> current version, surface area, in-flight work, consumers, dep gaps.
> Refreshed every release.

This file (`CLAUDE.md`) is durable rules.

## Scaffolding

Project was scaffolded with `cyrius init` (greenfield) or `cyrius port` (Rust → Cyrius migration). **Do not manually create project structure** — use the tools. If a tool is missing something, fix the tool.

## Quick Start

```sh
cyrius deps                          # resolve sibling deps
cyrius build src/main.cyr build/prajna
cyrius test                          # run [build].test + tests/*.tcyr
```

## Key Principles

- **Correctness over cleverness** — if it's wrong, the bugs own you
- Test after every change, not after the feature is "done"
- ONE change at a time — never bundle unrelated changes
- Research before implementation — check vidya / existing patterns
- Build with `cyrius build`, not raw `cat file | cc5` — the manifest auto-resolves deps and prepends includes
- Source files only need project includes — stdlib / external deps auto-resolve from `cyrius.cyml`
- Every buffer declaration is a contract: `var buf[N]` = N **bytes**, not N entries
- `&&` / `||` short-circuit; mixed expressions require explicit parens

## Rules (Hard Constraints)

- **Do not commit or push** — the user handles all git operations
- **Never use `gh` CLI** — use `curl` to the GitHub API if needed
- Do not skip tests before claiming changes work
- Do not use `sys_system()` with unsanitized input — command injection
- Do not trust external data (file / network / args) without validation
- Do not modify `lib/` files (vendored stdlib / dep symlinks)
- Do not hardcode toolchain versions in CI YAML — `cyrius = "X.Y.Z"` in `cyrius.cyml` is the source of truth

## Documentation

- [`docs/adr/`](docs/adr/) — Architecture Decision Records (*why X over Y?*)
- [`docs/architecture/`](docs/architecture/) — Non-obvious constraints (*what's true about the code?*)
- [`docs/guides/`](docs/guides/) — Task-oriented how-tos
- [`docs/examples/`](docs/examples/) — Runnable examples
- [`docs/development/state.md`](docs/development/state.md) — Live state snapshot
- [`docs/development/roadmap.md`](docs/development/roadmap.md) — Milestones through v1.0

## Process

1. **Work phase** — features, roadmap items, bug fixes
2. **Build check** — `cyrius build`
3. **Test + benchmark additions** for new code
4. **Internal review** — performance, memory, correctness, edge cases
5. **Documentation** — update CHANGELOG, `docs/development/state.md`, any ADR the change earned
6. **Version sync** — `VERSION`, `cyrius.cyml`, CHANGELOG header

