---
name: loop-engineering-addstep
description: Add, remove, or skip steps from the baseline loop-engineering-run 11-step sequence, saved as a new project-level variant skill — never modifies the shipped baseline itself. Use when the user runs /loop-engineering-addstep followed by a description of what to add and/or remove, e.g. "/loop-engineering-addstep - skip security-review, add a load test step after QA". To instead build an unrelated loop from scratch, use /loop-engineering-create-loop instead.
---

$ARGUMENTS describes step-level changes to apply to the baseline
`loop-engineering-run` sequence (steps 0-10: set goal -> build with ponytail
-> stage -> code-review -> debug -> QA -> verify -> simplify
(ponytail-review/audit) -> security-review -> production check -> repeat).
It can add a step, remove/skip a step, or do both in the same call.

This command NEVER edits
`plugins/loop-engineering/skills/loop-engineering-run/SKILL.md` — the
plugin's shipped baseline stays exactly as installed. Every change here
produces a separate new skill file instead, so the original 11-step form is
always available and untouched.

Do this:

1. If $ARGUMENTS doesn't include a short name for the resulting variant, ask
   the user for one (a few words, e.g. "no-security" or "with-load-test").
2. Start from the baseline's exact step list and content, then apply only
   what was asked: add a described step at the right point in the sequence,
   and/or remove/skip a named step. Renumber the remaining steps
   sequentially. Keep step 0 (set the goal) and the final `Repeat` step
   intact regardless of what else changes — that bookend shape is the part
   of "my plugin form" that must stay intact even as the steps between them
   change.
3. Write the result to `.claude/skills/loop-engineering-<slug>/SKILL.md` in
   the CURRENT project (create directories if needed) — same naming
   pattern as the plugin's own commands and `ponytail`'s (`ponytail-review`,
   `ponytail-audit`): the plugin name repeated as a prefix, no colon. If a
   skill already exists at that path, ask before overwriting it.
4. Give the new skill a `description:` frontmatter line stating exactly
   what differs from the baseline (which step was added/removed), so it's
   easy to tell apart later.
5. Tell the user plainly: this new skill won't be invocable until they
   restart or reload the Claude Code session (skills are only discovered at
   session start). Once reloaded, they can run it as
   `/loop-engineering-<slug>` — a project skill, not part of this plugin,
   so it stays local to this project and won't follow them elsewhere.
6. Do not run the new workflow yourself in this same turn — just create the
   file and report the exact command name and exactly what changed from the
   baseline.
