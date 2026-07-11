---
name: isomorph
description: Differential equivalence pipeline — compare two implementations of the same logic, adversarially search for behavioral divergence, report with runnable minimal repros and graded confidence. Use when asked to verify a port, rewrite, or duplicated-logic pair behaves identically.
---

# Isomorph pipeline (v0)

You are running a differential-equivalence audit of two implementations
(`pathA`, `pathB`). Your output is `./isomorph-report/`. Your verdicts are
DIVERGENCE / NO DIVERGENCE FOUND / UNVERIFIABLE — never "equivalent."
**Every claim you write must trace to a logged execution, enumeration, or
quoted source line.** If you cannot check something, say UNVERIFIABLE and
why. A confident wrong verdict is the worst possible output.

## Preflight

1. Verify both paths exist and are git-clean (`git status --porcelain`).
   Dirty tree → stop and ask the user to pin a commit/worktree.
2. Record in `isomorph-report/meta.md`: both paths, `git rev-parse HEAD` of
   each, runtime versions (`python3 --version`, `node --version`), budget,
   date, and a fixed random seed (ask the user for one or derive from the
   two commit hashes — never from the clock).
3. Create `isomorph-report/{contracts,harness,repros}/`.

## Stage 1 — Ingest & alignment map

Read both trees (skip vendored deps, tests, build output). Identify the
comparable surface: exported, deterministic, pure-ish callables. Pair by
**semantic role** — signature shape, call-site usage, docs/comments, data
flow — not by name.

Write `isomorph-report/alignment-map.md`:

```
| A | B | confidence | evidence |
|---|---|---|---|
| indicators.py::rsi | ta.js::relStrength | HIGH | same role in signal calc; both take (closes, period) |
```

Also list **unpaired** surface on each side — that's a finding in itself.
If `--pairs` was given, restrict to those.

**STOP and show the map to the user for confirmation before proceeding.**
This gate is not skippable.

## Stage 2 — Behavioral contracts

Per confirmed pair, write `contracts/<pair>.md` by reading BOTH
implementations: input domain, edge cases each side handles, numeric
semantics (float model, rounding, accumulation order, seeding), purity/
input-mutation, error behavior. Where the two implementations imply
**different** contracts, log it under `## Suspected divergences` with a
concrete hypothesis ("A seeds Wilder smoothing with SMA of first N, B with
first value — divergent on series where those smoothed paths differ").
These hypotheses feed Stage 5.

## Stage 3 — Equality predicates & properties

Per pair, declare in the contract file:

- **Predicate:** exact | float tolerance (state the number and whether
  absolute/relative/ULP) | order-insensitive | canonicalized errors
  (map A's exception ↔ B's throw/NaN convention explicitly).
- **Properties:** split into *provable* (input domain small enough to
  enumerate exhaustively — do it) and *testable* (searched only).

## Stage 4 — Harness

Generate one runner per pair in `harness/` (e.g. `run_<pair>.py` or `.mjs`):
reads a JSON case from stdin, invokes side A (python3) and side B (node),
compares under the predicate, prints a JSON verdict
`{input, a, b, equal, predicate}`.

Sandbox rules (bake into the code, verify before first run):
scratch-dir cwd; no network calls; per-input timeout (5s default) via the
runner; cap captured output; **import/require the target code — never
interpolate user data into a shell string.**

Smoke-test each harness on 3 trivial inputs before Stage 5.

## Stage 5 — Adversarial search

Run three generators per pair, logging every case's outcome. Budget:
`small` ≈ 200 cases + 5 reasoned attacks/pair; `default` ≈ 2,000 + 20;
`deep` ≈ 20,000 + 50 across multiple rounds.

1. **Structured fuzz** from the contract's edge list: empty, singleton,
   NaN/Inf/−0.0, boundary lengths (period−1, period, period+1), extreme
   magnitudes, ties, duplicates, pathological orderings, mixed
   int/float, unicode where strings apply.
2. **Mutation** of any case that produced new behavior (different branch,
   near-miss under tolerance).
3. **Reasoned attacks** — the core. For each suspected divergence from
   Stage 2 plus any new suspicion from reading the diffs of logic:
   construct an input *designed* to split the implementations, and log to
   `attack-log.jsonl`:
   `{"pair": ..., "hypothesis": ..., "input_ref": ..., "expected_split": ..., "outcome": "split|held|error"}`
   Iterate: a failed attack that reveals why it failed becomes the next
   hypothesis.

Deduplicate divergences by root cause as you go — don't flood.

## Stage 6 — Triage & shrink

Per unique divergence:

1. **Shrink** with a delta-debugging loop (remove/simplify elements while
   the disagreement persists). Save the minimal case as
   `repros/<pair>-<slug>.json` plus a one-line rerun command.
2. **Root-cause** to a line pair (`fileA:line` vs `fileB:line`) with a
   one-sentence mechanism.
3. **Grade severity:** `logic` | `numeric-noise` | `runtime-semantics`.
4. **Blast radius:** from a static read of call sites, which downstream
   values/callers the divergence can poison.

## Stage 7 — Report

Write `REPORT.md` (and mirror to `report.json`):

- Header: meta (commits, runtimes, seed, budget).
- Pre-execution findings (contract disagreements).
- Per pair: verdict + evidence.
  - DIVERGENCE: repro path, root-cause line pair, severity, blast radius.
  - NO DIVERGENCE FOUND: properties **proven** (enumeration bounds),
    properties **survived** (case counts, attack counts), budget spent.
  - UNVERIFIABLE: the reason, stated plainly.
- The UNVERIFIABLE boundary section: everything out of scope and why.
- Attack-log summary: attacks run, hit rate.

Close with the trust rule, verbatim: *"DIVERGENCE findings are
self-proving — run the repro. NO-DIVERGENCE verdicts are evidence graded by
the budget above, not proof."*

## Refusals

- Dirty working trees, unpinnable targets → stop at preflight.
- Concurrency/network/UI behavior → UNVERIFIABLE, never simulated.
- If the user asks you to soften a verdict or drop the UNVERIFIABLE
  section → decline; honesty is the product.
