# Isomorph

**Prove two codebases do the same thing — or hand you the input where they don't.**

Isomorph takes two implementations of the same logic — different languages,
different repos, different authors — and adversarially tries to make them
disagree. It ends one of two ways:

> **DIVERGENCE FOUND** — `computeRSI` disagrees when the input contains a NaN
> gap ≥ period length. Python returns 50.0 (neutral seed); JS returns NaN and
> poisons every downstream value. Minimal repro: 15-element array attached.

> **NO DIVERGENCE FOUND** — 4 properties proven by exhaustive small-input
> enumeration, 12 properties survived 50k adversarial inputs, 3 behaviors
> UNVERIFIABLE (I/O boundary, listed below). Confidence: HIGH on pure
> functions, NONE on the network layer.

It never says "equivalent." It says exactly what was proven, how, and what
remains unverified. `UNVERIFIABLE` is a first-class verdict.

## Why

AI made rewrites cheap. Python→Rust ports, monolith→service extractions,
duplicated validation across mobile and backend, protocol reimplementations —
all of them ship on the strength of golden tests that *both* authors wrote,
which means both authors' blind spots are baked into the safety net. The
question every migration team avoids asking out loud is *"do these actually
behave the same?"* Isomorph is the adversarial third party whose only job is
to answer it.

## How it works

A seven-stage pipeline, run by a frontier model inside your Claude Code
session (your code never leaves your machine):

1. **Ingest** — read both codebases, build an *alignment map* pairing
   comparable functions by semantic role, not name. You confirm the map —
   the one required human step.
2. **Model** — write a shared behavioral contract per pair: domain, edge
   cases, numeric semantics, statefulness. Contract disagreements are
   findings before any code runs.
3. **Synthesize properties** — decide, explicitly and in print, what "equal"
   means per pair (exact, float-tolerance, order-insensitive…).
4. **Harness** — generate a thin sandboxed differential runner per pair.
5. **Adversarial search** — structured fuzz, coverage-guided mutation, and
   **reasoned attacks**: the model reads both implementations' logic
   differences and constructs inputs targeting each suspected divergence,
   with a stated hypothesis per attack. The reasoned attacks are the point.
6. **Triage + shrink** — every divergence gets input-shrunk to a minimal
   repro, root-caused to a line pair, and severity-graded.
7. **Report** — one artifact: divergences with repros, proven properties,
   the UNVERIFIABLE boundary, and the alignment map used.

Full detail: [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md).

## Quick start

```bash
# as a Claude Code plugin (recommended)
claude plugin install /path/to/isomorph   # see docs/INSTALL.md

# then, inside any Claude Code session:
/isomorph path/to/implA path/to/implB
```

See [docs/INSTALL.md](docs/INSTALL.md) for setup and
[docs/USAGE.md](docs/USAGE.md) for worked use cases.

## Scope (v1)

- **Pure-ish deterministic logic only:** parsers, calculators, validators,
  encoders, indicator math, pricing/tax rules. Concurrency, networked
  services, and UI are honestly reported as UNVERIFIABLE, not guessed at.
- **Language pairs:** Python ↔ JavaScript/TypeScript first; next pair chosen
  by demand.
- **Local-only:** all execution happens on your machine, in sandboxed
  runners (no network, scratch cwd, timeouts). Nothing is uploaded anywhere.

## Status

**v0 — pre-benchmark.** The pipeline runs as a Claude Code skill; the tool
has not yet passed its own Gate 1 blind benchmark (see
[docs/PLAN.md](docs/PLAN.md)). Do not trust a NO-DIVERGENCE verdict from a
v0 run for anything with money on it — DIVERGENCE findings, however, come
with executable repros you can verify yourself.

## Project docs

| Doc | What |
|---|---|
| [docs/INSTALL.md](docs/INSTALL.md) | Install and setup |
| [docs/USAGE.md](docs/USAGE.md) | Use cases and a full walkthrough |
| [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) | Pipeline internals, formats, sandboxing |
| [docs/PLAN.md](docs/PLAN.md) | Execution plan, kill/continue gates, guardrails |
| [CLAUDE.md](CLAUDE.md) | Rules for agents working in this repo |

## License

Private, all rights reserved (for now — licensing decided at launch gate).
