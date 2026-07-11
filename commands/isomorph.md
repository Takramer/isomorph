---
description: Compare two implementations of the same logic — find a divergence with a minimal repro, or emit a graded equivalence report.
argument-hint: <pathA> <pathB> [--pairs a,b] [--budget small|default|deep]
---

Run the Isomorph pipeline on the two implementation roots given in
`$ARGUMENTS`. Load and follow `skills/isomorph/SKILL.md` from this plugin —
it defines the seven stages, the artifact formats, the sandbox rules, and
the verdict semantics. Do not improvise a lighter-weight comparison; if the
arguments are unusable (missing paths, dirty working trees), stop and say
exactly what's needed.
