# prajna

> प्रज्ञा — *the cognition that refines itself.* The sovereign **meta-learning /
> learn-to-learn** reference, written in [Cyrius](https://github.com/MacCracken/cyrius).

prajna is a sibling reference to [attn11](https://github.com/MacCracken/attn11)
(transformer), [tarka](https://github.com/MacCracken/tarka) (RL / reasoning), and
[tentib](https://github.com/MacCracken/tentib) (ternary), on the same f64
substrate. Where those three prove **first-order** learning (attention gradient,
policy gradient, the STE surrogate), prajna proves the family's **first
second-order / nested gradient**:

> **bi-level optimization — a model that learns *how* to learn — is expressible
> assembly-up in everything-is-i64 Cyrius**, hand-derived (no autodiff, no BLAS, no
> libc) and **finite-difference-gated over a two-level computation**.

The lead algorithm is **MAML** (meta-learn an initialization so a new task is solved
in a few gradient steps); the sibling is **learned optimizers** (meta-learn the
update rule). The hard, genuinely-new primitive is the **meta-gradient**:
differentiating the post-adaptation loss *through* the inner SGD step.

## M1 — the scalar meta-gradient (verified)

The minimal exact proof of the primitive (scalar quadratic MAML): the analytic
second-order meta-gradient matches central finite differences, the first-order
approximation (FOMAML) measurably does **not** (proving the second-order term is
real), and meta-descent converges to the optimum.

```
$ cyrius build src/main.cyr build/prajna && ./build/prajna
  meta-grad full  = -1.919999   (2nd-order, analytic)
  meta-grad FOMAML= -2.399999   (1st-order approx)
  meta-grad FD    = -1.920000   (finite-diff truth)
  GATE  : PASS  (2nd-order meta-grad matches finite-diff)
  2nd-ORD: OBSERVABLE  (FOMAML differs from FD -> the term is real)
  ... meta-descent: theta 0.5 -> 3.499  (optimum 3.5)
```

M1 above is the minimal worked example; the demo runs the full reference end-to-end.

## The full reference (M1–M5, complete)

Every hand-derived gradient is finite-difference-gated. `./build/prajna` exits 0 iff all gates pass.

- **M1** — scalar second-order meta-gradient (the MAML primitive).
- **M2 — MAML**: linear (M2.a) → nonlinear via the **Pearlmutter R-operator** double-backward
  (M2.b) → sine few-shot meta-learning with a sovereign `f64_sin` (M2.c).
- **M3 — learned optimizers**: meta-learn the *update rule* via **BPTT** through an unroll —
  feedforward (M3.a/b, beats best fixed-lr SGD) and recurrent (M3.c).
- **M4 — text few-shot**: a tiny next-token LM on the shared **akshara** tokenizer (M4.a), then
  MAML over text tasks — "MAML works on text" (M4.b).
- **M5 — continual learning**: experience replay (+ EWC) so sequential adaptation doesn't forget.

Hardened across the **0.6.x arc** (NaN-safe gates, numerical robustness, security audit, refactor).

## Build

```sh
cyrius deps                               # resolve sibling deps (rosnet, tyche, akshara)
cyrius build src/main.cyr build/prajna    # compile the demo
cyrius test                               # run the FD-gate assertions
./build/prajna                            # the full M1–M5 reference (exit 0 = all gates pass)
```

## Documentation

- **API** (frozen at 1.0): [`docs/api.md`](docs/api.md)
- **Benchmarks** (footprint, correctness, convergence): [`docs/benchmarks.md`](docs/benchmarks.md)
- **Security** (threat model + supply chain): [`SECURITY.md`](SECURITY.md)
- **Examples** (consume `fdgate`): [`examples/`](examples/)
- **Roadmap & state**: [`docs/development/roadmap.md`](docs/development/roadmap.md) ·
  [`docs/development/state.md`](docs/development/state.md)
- **ADRs**: [`docs/adr/`](docs/adr/)

## Where it sits

- Sibling reference to attn11 / tarka / tentib on the same f64 substrate (rosnet/tyche/akshara).
- Why it exists / why it is a *reference* and not RSI orchestration: agnosticos
  `docs/development/planning/self-improvement-lane.md`

## License

GPL-3.0-only
