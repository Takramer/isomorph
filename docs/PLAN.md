# F1 — Isomorph: Prove Two Codebases Do the Same Thing (Fable-class project)

**Status: PROPOSED — not approved for execution.**

**What:** a tool (Claude Code plugin first, standalone product second) that takes
**two implementations of the same logic** — different languages, different repos,
different authors — and either **finds a behavioral divergence with a minimal
reproducing input**, or emits a **confidence-graded equivalence report** stating
exactly what was proven, how, and what remains unverified. Point it at a Python
engine and its JS port, a Go service and its Rust rewrite, a spec and its
implementation. It answers the question every migration team is afraid to ask
out loud: *"do these actually behave the same?"*

**Why this is Fable-only:** existing differential-testing tools (fuzzers,
property-based testing, golden vectors) all require a human to first build the
harness — align the two APIs, normalize the I/O, decide what "equal" means for
floats/dates/ordering, and pick which behaviors matter. That alignment step IS
the hard problem, and it is pure frontier-model reasoning: reading two codebases
simultaneously, constructing a shared behavioral model of both, synthesizing the
properties that model implies, and then *adversarially* hunting the inputs where
the implementations disagree. An Opus-class model produces golden tests for the
happy path. Fable-class reasoning is what finds "these two RSI implementations
disagree only when the price series contains a gap longer than the smoothing
window, because one seeds Wilder's average differently" — a divergence no
human-written fixture would think to include. The product is the reasoning; the
code around it is thin.

**Why it's credible from this workspace (without being based on it):** the
workspace already lived this failure. The AgentTradingLogic Python engine and
the nydex JS reimplementation drifted, and the 2026-07-10 audit found the drift
by hand (backtest PF 0.97 → 1.03 after fixes). That history is not the product —
it's the **benchmark with known ground truth**: Isomorph, run blind on the
pre-audit commits, must rediscover that drift on its own. Every rewrite team on
earth (Python→Rust, monolith→microservice, mobile/backend duplicated validation,
protocol reimplementations) has the same problem and currently solves it with
vibes and a few golden tests.

---

## 1. Product thesis

Code rewrites and ports are exploding — largely *because* AI makes rewrites
cheap. What AI made cheap to produce, it made expensive to trust: nobody can
say whether the port is faithful. The market's current answer is "run both in
prod and compare" (expensive, late) or "write more tests" (tests inherit the
author's blind spots — both implementations pass the tests both authors thought
of). Isomorph's answer: an adversarial third party whose only job is to make
the two implementations disagree. Output is binary and visceral:

> **DIVERGENCE FOUND** — `computeRSI` disagrees when input contains a NaN gap
> ≥ period length. Python returns 50.0 (neutral seed), JS returns NaN and
> poisons every downstream value. Minimal repro: 15-element array attached.
> Estimated blast radius: every signal on a halted ticker.

or

> **NO DIVERGENCE FOUND** — 4 properties proven by exhaustive small-input
> enumeration, 12 properties survived 50k adversarial inputs, 3 behaviors
> UNVERIFIABLE (I/O boundary, documented below). Confidence: HIGH on pure
> functions, NONE on the network layer — here is the exact boundary.

The honesty rule from the workspace's other products applies verbatim:
**graded confidence, never "it's equivalent."** UNVERIFIABLE is a first-class
verdict, stated loudly.

## 2. How it works (the pipeline)

1. **Ingest** — read both codebases; identify the *comparable surface*
   (exported pure-ish functions, matched by semantic role, not by name —
   Fable-work: `rhengine.indicators.rsi()` ↔ `src/engine/ta.js:relStrength()`).
   Produce an **alignment map** with per-pair confidence; human confirms or
   edits the map (this is the one required human step).
2. **Model** — for each aligned pair, write a shared behavioral contract in
   plain language + types: domain, edge cases each side handles, numeric
   semantics (float model, rounding, tolerance), statefulness. Disagreements
   between the *contracts* are already findings before any code runs.
3. **Synthesize properties** — from the contract, generate properties and
   equivalence predicates (exact, ULP-tolerance, order-insensitive, etc.).
   This is where "what does equal mean" gets decided explicitly and printed.
4. **Harness** — generate a thin differential harness per pair: run Python via
   subprocess, JS via node, feed identical inputs, compare under the declared
   predicate. Harness code is boring by design; the model writes it.
5. **Adversarial search** — the Fable core. Three input generators run in
   parallel: (a) structured fuzz from the contract's edge-case list, (b)
   coverage-guided mutation, (c) **reasoned attacks** — the model reads both
   implementations' *diffs in logic* and constructs inputs targeting each
   suspected divergence ("JS uses `sort()` default = lexicographic; build an
   input where numeric vs lexicographic order differ"). Reasoned attacks are
   the moat; (a) and (b) are commodity.
6. **Triage + shrink** — every divergence gets input-shrunk to a minimal repro,
   root-caused to a line pair, and severity-graded (cosmetic float noise vs.
   logic drift) with a blast-radius estimate.
7. **Report** — one artifact: divergences (repro + root cause + severity),
   proven properties, the UNVERIFIABLE boundary, and the alignment map used.
   The report is the product.

## 3. Success metrics (kill/continue gates)

- **Gate 1 — the blind benchmark (weeks 1–3):** run against AgentTradingLogic
  `rhengine` vs. nydex JS at their **pre-audit commits**, with the audit
  findings sealed. Isomorph must independently rediscover ≥ 80% of the drift
  the 2026-07-10 audit found by hand, plus shrunken repros, in < 2 hours of
  wall-clock. Miss badly → the reasoned-attack engine isn't good enough; stop
  and study why before building anything else. (Read-only against those repos;
  pinned commits; live-money code never touched.)
- **Gate 2 — seeded mutations (weeks 3–4):** mutation-test the tool itself.
  Take 5 OSS dual-implementation pairs (e.g. a JS and Python semver parser,
  markdown renderers, a TA library ported across languages), seed 20 known
  behavioral mutations, measure recall/precision. Target ≥ 85% recall,
  ≥ 90% of reports actionable (no noise floods). False-positive floods are the
  death mode — a tool that cries wolf gets uninstalled.
- **Gate 3 — strangers' repos (month 2):** run on 3 real public rewrite pairs
  found in the wild (GitHub is full of "rewritten in Rust" projects). Find ≥ 1
  real, previously unknown divergence worth an upstream issue (drafted, not
  filed, per workspace rules). A true positive on a stranger's code is the
  launch asset.
- **Gate 4 — distribution (month 2–3):** ship as a Claude Code plugin/skill
  (`/isomorph <pathA> <pathB>`). 30-day: 100 installs or 3 inbound "can this
  check our migration" conversations. The wedge audience is teams mid-rewrite —
  they have budget and a deadline.
- **North star:** confirmed real divergences found per run-hour, and the
  ratio of reasoned-attack finds to fuzz finds (if fuzzing finds everything,
  there's no moat and no reason this needs Fable).

## 4. Scope fences (v1)

- **Pure-ish functions only.** Deterministic logic: parsers, calculators,
  validators, encoders, indicator math, pricing/tax rules. NOT: concurrency,
  networked services, UI, anything whose behavior is a side effect. The
  UNVERIFIABLE verdict covers the rest honestly.
- **Two language pairs at launch:** Python↔JavaScript/TypeScript (the benchmark
  pair) and one more chosen at Gate 3 by demand (likely Python↔Rust or Go↔Rust).
- **Local execution only.** Customer code never leaves the machine — the tool
  runs inside their Claude Code session. This kills the enterprise "upload your
  codebase" objection at birth and matches the workspace's client-side-only
  privacy pattern.

## 5. Phases

### Phase 1 — Harness spine + alignment (1–2 sessions)
Ingest → alignment map → contract → generated differential harness for ONE
hand-picked function pair, end to end. No search yet; prove the plumbing.
Acceptance: harness runs both sides on 100 identical inputs and diffs under a
declared predicate, from a single command.

### Phase 2 — Reasoned attacks (2–3 sessions; the product IS this)
The adversarial generator: model reads both implementations, enumerates
semantic differences (numeric model, ordering, seeding, boundary handling,
mutation-of-input, error paths), emits targeted attack inputs with a stated
hypothesis each ("this input should split them because…"). Every attack logs
hypothesis → outcome, so the tool's reasoning is auditable. Plus commodity
fuzz as the baseline to beat.
Acceptance: on a toy pair with 5 seeded divergences, reasoned attacks find ≥ 4
and each finding's hypothesis matches the actual root cause.

### Phase 3 — The blind benchmark (Gate 1) (1–2 sessions)
Pin pre-audit commits of rhengine + nydex engine in a scratch worktree
(read-only), run full pipeline, compare against the sealed audit findings.
Score recall honestly in a findings file. This is the go/no-go for the whole
project.

### Phase 4 — Shrinking, triage, report (1–2 sessions)
Input shrinking (delta-debugging loop), severity grading, blast-radius
estimate, the single report artifact (markdown + JSON). Grounding rule: every
claim in the report traces to a logged harness execution — no narrated
findings without a repro attached.

### Phase 5 — Package + Gate 2/3 (2 sessions)
Plugin/skill packaging, the seeded-mutation eval harness (which becomes the
permanent regression suite for the tool itself), the three strangers'-repos
runs, launch drafts (X thread: "I made two 'identical' codebases fight; here's
what broke" + the OSS issue drafts). **Drafts only.**

## 6. Verification (of the tool itself)
- The seeded-mutation suite is CI: every change to Isomorph re-runs it;
  recall/precision regressions block.
- Determinism check: same repo pair + same seed → same findings list.
- Honesty audit: sample 5 reports; every "proven" claim must have the
  enumeration/execution log that proves it; every UNVERIFIABLE must say why.

## 7. Guardrails
- Read-only against all benchmark repos; live-money code (RobinhoodAgent,
  `~/TradingRuntime`) is never an execution target, only pinned historical
  commits of AgentTradingLogic/nydex in scratch worktrees.
- Generated harnesses execute customer code — sandbox the runners (no network,
  scratch-dir cwd, timeouts); never execute repo code during the *read/model*
  phases.
- No upstream issue filing, posting, or outreach without explicit approval.
- Ask before any paid infra; v1 needs none (local + existing Claude spend).

## 8. Open questions for Thomas
1. Approve Gate 1's use of pinned pre-audit AgentTradingLogic/nydex commits as
   the blind benchmark? (Read-only, scratch worktree, sealed answer key.)
2. Plugin-first (distribution rides Claude Code's ecosystem, zero infra) vs.
   CLI-first (broader audience, more packaging work)? Recommend plugin-first.
3. Name check: "Isomorph" — gut reaction?
4. Time-box: Gates 1–2 are ~4 weeks part-time. Given runway, this runs behind
   P1/P2, not instead of them — confirm that ordering.
