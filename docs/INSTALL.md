# Install & Setup

Isomorph v1 ships as a **Claude Code plugin**. The pipeline is executed by
the model in your session; the plugin provides the skill, the command, and
the harness conventions. There is no server, no account, no telemetry.

## Requirements

- **Claude Code** ≥ 2.x, with a frontier-class model selected. The
  adversarial-search stage depends on deep cross-codebase reasoning —
  smaller models will produce shallow attacks and untrustworthy
  NO-DIVERGENCE verdicts. If in doubt, run `/model` and pick the most
  capable model available to you.
- **Runtimes for the code under test.** For the v1 language pair:
  - Python ≥ 3.10 on `PATH` (`python3 --version`)
  - Node.js ≥ 20 on `PATH` (`node --version`)
- **Git** (used for pinning target commits and scratch worktrees).
- macOS or Linux. Windows untested.

## Install

### Option A — plugin install (recommended)

```bash
git clone git@github.com:Takramer/isomorph.git
claude plugin install ./isomorph
```

Restart your Claude Code session. `/isomorph` should now appear in the
command list.

### Option B — vendor the skill into one project

If you don't want a global plugin, copy the skill into the target project:

```bash
cp -r isomorph/skills/isomorph  your-project/.claude/skills/isomorph
```

Then invoke it with `/isomorph` from a session started in that project.

## Setup per comparison

1. **Pin your targets.** Isomorph compares *commits*, not working trees.
   Check out (or worktree) the exact revision of each implementation:

   ```bash
   git -C implA worktree add /tmp/isomorph/A <commitA>
   git -C implB worktree add /tmp/isomorph/B <commitB>
   ```

   Dirty working trees are refused by default — a finding against code that
   changes underneath you is worthless.

2. **Run:**

   ```bash
   /isomorph /tmp/isomorph/A /tmp/isomorph/B
   ```

3. **Confirm the alignment map** when prompted (stage 1 output). This is the
   one required human step: Isomorph pairs functions by semantic role and
   shows you the map with per-pair confidence; correct anything it got wrong
   before the search stages run.

4. **Read the report** at `isomorph-report/REPORT.md` (plus `report.json`
   for machines). Every divergence includes a runnable minimal repro under
   `isomorph-report/repros/` — verify them yourself; don't take the prose
   on faith.

## Sandboxing (read this)

Isomorph **executes the code under test** in stages 4–6. Runners are
generated with these constraints and you should verify them in the emitted
harness code before first run on untrusted code:

- working directory is a scratch dir, never the repo
- no network access expected; harnesses make no network calls by design
- per-input timeout (default 5s) and output-size caps
- target code is imported/required, never shelled into with user-controlled
  strings

If you are comparing code you do not trust at all, run the whole session in
a container or VM. Isomorph's sandbox is a seatbelt, not a prison.

## Uninstall

```bash
claude plugin uninstall isomorph
```

Reports live in `isomorph-report/` in your cwd; delete them whenever.

## Troubleshooting

- **`/isomorph` not found** — restart the Claude Code session after install;
  check `claude plugin list`.
- **"dirty working tree" refusal** — commit, stash, or use a pinned
  worktree as shown above.
- **Python/Node mismatch findings** — the report tags divergences it
  attributes to *runtime version* semantics separately from logic drift;
  make sure the runtimes match what each side ships with in production.
- **Runs are slow** — stage 5 is compute-heavy by design. Narrow the
  alignment map to the pairs you care about; the map is editable.
