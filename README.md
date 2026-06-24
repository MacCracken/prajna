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

## Build

```sh
cyrius deps                               # resolve sibling deps (none beyond stdlib at M1)
cyrius build src/main.cyr build/prajna    # compile the demo
cyrius test                               # run the FD-gate assertions
```

## Where it sits

- The roadmap (M1 → v1.0): [`docs/development/roadmap.md`](docs/development/roadmap.md)
- Why it exists / why it is a *reference* and not RSI orchestration: agnosticos
  `docs/development/planning/self-improvement-lane.md`

## License

GPL-3.0-only
