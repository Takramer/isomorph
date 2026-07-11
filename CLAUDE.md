# Isomorph — agent instructions

Isomorph is a differential-equivalence tool: it compares two implementations
of the same logic and either finds a divergence with a minimal repro or emits
a graded equivalence report. Read `docs/ARCHITECTURE.md` before changing the
pipeline and `docs/PLAN.md` before changing scope — the kill/continue gates
there govern what gets built next.

## Layout

- `skills/isomorph/SKILL.md` — the product: the pipeline the model executes.
- `commands/isomorph.md` — the `/isomorph` slash command (thin wrapper).
- `docs/` — README-linked docs. Keep USAGE/ARCHITECTURE in sync with the
  skill; the skill is the source of truth when they disagree.
- `eval/` — seeded-mutation benchmark suite (Gate 2). Every change to the
  skill or pipeline re-runs it; recall/precision regressions block.

## Hard rules

1. **Honesty over reassurance.** Never emit or soften toward "equivalent."
   Verdicts are DIVERGENCE / NO DIVERGENCE FOUND (graded, with budget) /
   UNVERIFIABLE (with reason). Every report claim must trace to a logged
   execution or enumeration.
2. **Benchmark repos are read-only.** AgentTradingLogic / nydex are used
   only as pinned historical commits in scratch worktrees. Never execute
   against, or modify, live-money code (`RobinhoodAgent`, `~/TradingRuntime`).
3. **Sandbox the runners.** Generated harnesses: scratch cwd, no network,
   timeouts, output caps, import-not-shell-interpolate. Never execute target
   code during the read/model stages.
4. **Drafts only** for any outreach, upstream issues, or posts.
5. **Ask before paid infra.** v1 needs none.
6. **Determinism:** same targets + same seed must reproduce the same
   findings list. Don't introduce untracked randomness.

## Workspace context

This repo lives in a multi-project workspace; durable findings (benchmark
results, gate outcomes, design decisions) get filed to the shared vault via
`/save` from the workspace root. Gate results also append to
`docs/PLAN.md` under a Results section — metrics or it didn't happen.
