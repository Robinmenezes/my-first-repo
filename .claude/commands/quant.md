---
name: quant
description: Build, re-evaluate, or extend a quantitative model using the QUANT.md playbook.
allowed_tools: ["Bash", "Read", "Write", "Edit", "Grep", "Glob", "WebFetch", "WebSearch"]
---

# /quant

Drive a quantitative modeling task end-to-end using the protocol in
`QUANT.md` at the repo root.

Invocation: `/quant <problem statement>`

If no problem statement is given, ask the user for one before proceeding.

## Required Reading (always, before acting)

1. Read `QUANT.md` in full.
2. Read `data/registry.yaml` if it exists (skip silently if not).
3. List existing entries in `notebooks/` and `src/models/` to see prior work
   on the same case.

## Execution Protocol

Follow `QUANT.md` sections in order:

1. **Classify** the problem into a §2 bucket. State the bucket and why.
2. **Frame** the problem per §6 step 1: metric, baseline target, downstream decision.
3. **Pick a protocol branch** from §5. State the branch.
4. **Data check** per §3: confirm the dataset exists in the registry, or
   propose a new entry. Run the data quality protocol before fitting anything.
5. **Run the §6 Loop**, one iteration at a time:
   - Always implement the §4 baseline first.
   - Apply §8 validation gates after every iteration.
   - Use §7 auto-research triggers as written — do not improvise.
6. **Emit §9 artifacts** for any iteration that passes §8.

## Hard Stops

The §10 anti-patterns are blocking. If the user asks you to violate one
(e.g. "skip the baseline", "tune on test"), refuse and explain which §10
rule applies.

## Notes

- Treat `QUANT.md` as the source of truth; if its protocol conflicts with
  your instincts, follow `QUANT.md` and propose an edit to it instead of
  silently deviating.
- Log every research action (queries, findings, decisions) in the
  notebook's "research log" section per §7.
- Keep iterations small: one method change per loop pass.
- When in doubt about case classification or protocol choice, ask the user
  rather than guessing.
