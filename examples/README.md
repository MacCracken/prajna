# prajna — examples

Worked, buildable examples of consuming prajna. Built from the repo root so the project
manifest (`cyrius.cyml`) resolves the standard library + deps.

## `gate_consumer.cyr` — vendor `fdgate` to gate your own gradient

prajna's one directly-reusable module is [`src/fdgate.cyr`](../src/fdgate.cyr): the NaN-safe
finite-difference-gate infrastructure. This example uses it to verify a hand-derived derivative
(`f(x)=x³` → `f'(x)=3x²`) against central finite differences — and shows that a NaN gradient is
**rejected** rather than silently accepted (the safety dividend from the 0.6.0 hardening).

```sh
cyrius build examples/gate_consumer.cyr build/gate_consumer
./build/gate_consumer        # exit 0 iff the gate passes
```

Expected output:

```
consumer: gating f'(x)=3x^2 against finite differences via prajna fdgate
  PASS: analytic grad matches FD on x=1..5; a NaN gradient is correctly rejected
```

This is the pattern every sibling reference (attn11/tarka/tentib) uses internally to keep
hand-derived gradients honest — `fdgate` is the part of prajna meant to be reused directly.
