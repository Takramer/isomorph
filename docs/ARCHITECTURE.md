# Architecture

Isomorph is a pipeline executed by a frontier model inside a Claude Code
session. The deliberate design choice: **the reasoning is the product; the
code around it is thin and inspectable.** Every stage emits a plain-text
artifact into `isomorph-report/` so the whole run is auditable after the
fact.

```
 ingest → model → properties → harness → adversarial search → triage/shrink → report
 (map)    (contracts)  (predicates)  (runners)   (attack log)      (repros)      (REPORT.md)
```

## Stage 1 — Ingest & alignment

Read both trees. Identify the *comparable surface*: exported, pure-ish,
deterministic callables. Pair them by **semantic role** — signature shape,
call-site usage, docstrings/comments, data flow — never by name alone.

Output: `alignment-map.md` — one line per pair with confidence
(HIGH/MEDIUM/LOW) and the evidence for the pairing. Unpairable functions are
listed too (they're a finding: surface that exists on one side only).

**Human gate:** the user confirms/edits the map before anything executes.
This is the only required human step and it is not skippable — a wrong
alignment makes every downstream result garbage.

## Stage 2 — Behavioral contracts

Per pair, write one contract (`contracts/<pair>.md`) covering:

- input domain (types, shapes, ranges, nullability)
- edge cases each side visibly handles (and doesn't)
- numeric semantics: float model, rounding, accumulation order, seeding
- statefulness / purity caveats, input mutation
- error behavior: what raises/returns/NaNs where

Contracts are written by reading *both* implementations. Where the two
implementations imply **different contracts**, that's a pre-execution
finding, logged with a hypothesis for stage 5 to weaponize.

## Stage 3 — Equality predicates & properties

"Equal" is never assumed. Per pair, declare in print:

- the predicate: exact | ULP/absolute float tolerance (with the number) |
  order-insensitive | canonicalized-error equivalence
- properties implied by the contract (invariants, algebraic identities,
  round-trips) split into two classes:
  - **provable** — domain small enough to enumerate exhaustively
  - **testable** — searched, never "proven"

## Stage 4 — Harness generation

One thin differential runner per pair: read a JSON case from stdin, call
side A (via `python3` subprocess) and side B (via `node`), compare under
the declared predicate, emit a JSON verdict. Boring by design; the emitted
code sits in `harness/` for inspection and re-running.

Sandbox constraints baked into every runner: scratch-dir cwd, no network
calls, per-input timeout, output caps, code imported not shell-interpolated.

## Stage 5 — Adversarial search

Three generators, in cost order:

1. **Structured fuzz** — cases from the contract's edge-case list: empties,
   singletons, NaN/Inf, boundary lengths, extreme magnitudes, ties,
   pathological ordering. Commodity.
2. **Coverage-guided mutation** — mutate survivors that reached new
   behavior. Commodity.
3. **Reasoned attacks** — the moat. The model diffs the two
   implementations' *logic* and constructs inputs targeting each suspected
   divergence. Every attack is logged to `attack-log.jsonl` as
   `{hypothesis, input, expected_split, outcome}` — the tool's reasoning is
   auditable, and the hypothesis→outcome hit-rate is the tool's own quality
   metric.

Budget (`--budget`) governs iterations of 1–2 and rounds of 3.

## Stage 6 — Triage & shrink

Every raw divergence is:

- **shrunk** — delta-debugging loop to a minimal repro, saved runnable
  under `repros/`
- **root-caused** — attributed to a specific line pair where the semantics
  split
- **graded** — `logic` (real drift) | `numeric-noise` (accumulation-order
  float dust within an honest tolerance discussion) | `runtime-semantics`
  (language/runtime behavior, e.g. sort stability, int overflow)
- **blast-radius estimated** — which callers/downstream values the
  divergence can poison, from static read of call sites

Duplicates (same root cause) are folded, not spammed.

## Stage 7 — Report

`REPORT.md` + `report.json`. Structure per pair: verdict, evidence,
budget spent. Global sections: pre-execution contract findings, the
UNVERIFIABLE boundary (with reasons), the alignment map used, and the
grounding rule — **every claim in the report traces to a logged execution
or enumeration; no narrated findings without an attached repro or log.**

## Verdict semantics

| Verdict | Meaning | Trust model |
|---|---|---|
| DIVERGENCE | disagreement found under the declared predicate | self-proving — run the repro |
| NO DIVERGENCE FOUND | properties proven and/or survived the stated budget | evidence, graded by budget; never "equivalent" |
| UNVERIFIABLE | couldn't be isolated/executed/aligned — reason stated | honest refusal |

## Why a frontier model is load-bearing

Stages 1, 2, and 5.3 are the tool. Alignment across languages, contract
extraction from two codebases at once, and attack construction from a
semantic diff ("JS seeds Wilder smoothing differently — build a series where
the seeds' smoothed paths never reconverge inside the window") are
open-ended reasoning tasks. A weaker model degrades exactly where it hurts:
it produces happy-path fuzz, misses the alignment subtleties, and returns
confident NO-DIVERGENCE verdicts that mean nothing. Hence the install doc's
model requirement, and hence the attack-log hit-rate metric: if reasoned
attacks stop out-finding fuzz, the product has no moat and the plan's kill
gates say so out loud.

## Self-verification (the tool testing itself)

- **Seeded-mutation suite** (`eval/`, Gate 2): known dual implementations
  with planted behavioral mutations; recall/precision measured on every
  change to Isomorph itself. This is the tool's CI.
- **Determinism:** same inputs + same seed → same findings list.
- **Honesty audit:** sampled reports checked claim-by-claim against logs.
