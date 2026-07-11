# Usage & Use Cases

## The command

```
/isomorph <pathA> <pathB> [--pairs name1,name2] [--budget small|default|deep]
```

- `pathA`, `pathB` — roots of the two implementations (pinned checkouts;
  see INSTALL.md).
- `--pairs` — restrict to specific aligned pairs after you've seen the map
  once.
- `--budget` — how hard stage 5 searches. `small` for a smoke pass, `deep`
  for pre-cutover sign-off.

Output lands in `./isomorph-report/`:

```
isomorph-report/
  REPORT.md          # the human artifact — read this
  report.json        # same content, machine-readable
  alignment-map.md   # the pairing that was used (you confirmed it)
  contracts/         # one behavioral contract per pair
  harness/           # generated differential runners (inspectable, rerunnable)
  repros/            # one runnable minimal repro per divergence
  attack-log.jsonl   # every reasoned attack: hypothesis → outcome
```

## Reading a verdict

Three verdict classes appear in `REPORT.md`, per pair:

- **DIVERGENCE** — inputs found where the implementations disagree under
  the declared equality predicate. Comes with: minimal repro (runnable),
  root-cause line pair, severity (`logic` / `numeric-noise` /
  `runtime-semantics`), and a blast-radius estimate.
- **NO DIVERGENCE FOUND** — graded, never absolute: which properties were
  *proven* (exhaustive small-domain enumeration), which merely *survived*
  N adversarial inputs, and the search budget spent.
- **UNVERIFIABLE** — and why: I/O boundary, nondeterminism, state we
  couldn't isolate, no plausible alignment. Silence is never a verdict.

**Trust rule:** DIVERGENCE findings are self-proving — run the repro.
NO-DIVERGENCE verdicts are evidence, not proof; weigh them by the printed
budget and property list.

---

## Use cases

### 1. The rewrite cutover (the core case)

You ported a Python service's pricing module to Rust/TypeScript and the
cutover date is real. Golden tests pass on both sides — because both test
suites encode the same author's assumptions.

```
/isomorph ./legacy-python ./new-ts --budget deep
```

What Isomorph adds over your tests: it attacks the *differences in how the
two languages and implementations behave* — float model, ordering, seeding,
error paths, boundary handling — rather than replaying the happy paths you
already thought of. Run it at every release candidate; diff the reports.

### 2. Duplicated logic that must not drift

The same validation/calculation lives twice on purpose: backend Python and
frontend TS (a trading engine and its browser backtester; a tax calculator
on server and client; input validation on mobile and API). Each side evolves
independently and drift is silent until it costs money.

Run Isomorph in CI on a schedule against the two heads. The report diff *is*
the drift alarm, and the alignment map only needs confirming when the
surface changes.

### 3. Port-of-a-library due diligence

You're about to depend on `somelib-rs`, "a Rust port of somelib.py". Before
trusting it:

```
/isomorph ./somelib.py-src ./somelib-rs-src --pairs the_five_functions_you_use
```

Twenty minutes, and you either get a repro showing where the port disagrees
or a graded statement of what was actually checked.

### 4. Spec conformance between independent implementations

Two teams implemented the same internal spec (a message codec, a fee
schedule, a scoring formula). Neither is "the reference." Isomorph doesn't
care which is right — it finds where they disagree, and the contract stage
often surfaces that the *spec itself* is ambiguous (both sides are
defensible readings). Those contract-level findings come out in stage 2
before anything even executes.

### 5. Refactor safety net within one language

`pathA` and `pathB` can be two commits of the *same* repo — pre- and
post-refactor. This is differential regression testing without writing a
single test: "prove my refactor changed nothing observable, or show me the
input where it did."

---

## Worked example (condensed)

Comparing a Python TA engine against its JS port:

```
/isomorph /tmp/iso/py-engine /tmp/iso/js-engine
```

**Stage 1** proposes an alignment map:

```
rhengine/indicators.py::rsi        <-> src/engine/ta.js::relStrength   [HIGH]
rhengine/indicators.py::atr        <-> src/engine/ta.js::atr           [HIGH]
rhengine/sizing.py::position_size  <-> src/engine/risk.js::sizeFor     [MEDIUM — arg order differs]
```

You confirm it (fixing the MEDIUM pair's arg mapping).

**Stage 2** writes contracts and immediately flags one pre-execution
finding: the Python `rsi` documents Wilder smoothing seeded with an SMA of
the first `period` values; the JS reads as seeding with the first value
alone. Suspected divergence, hypothesis logged.

**Stage 5** confirms it with a reasoned attack — a 15-bar series constructed
so the two seeds produce different smoothed paths — then fuzz finds a second,
unhypothesized one: JS `Array.prototype.sort()` default comparator
reordering ties lexicographically in the ranking step.

**Stage 6** shrinks both to minimal repros; **stage 7** reports:

```
PAIR rsi <-> relStrength           DIVERGENCE  severity=logic
  repro: repros/rsi-seed-divergence.json (15 bars)
  root cause: indicators.py:41 vs ta.js:88 (seed formula)
PAIR position_size <-> sizeFor     NO DIVERGENCE FOUND
  proven: 2 properties (exhaustive ≤4-element domains)
  survived: 50,000 adversarial inputs, 61 reasoned attacks
PAIR loadBars (I/O)                UNVERIFIABLE — network boundary
```

You run `repros/rsi-seed-divergence.json` through the emitted harness
yourself, watch the two numbers disagree, and now the fix conversation is
about a line pair, not a feeling.
