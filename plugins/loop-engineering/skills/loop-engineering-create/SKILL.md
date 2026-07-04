---
name: loop-engineering-create
description: Create a new custom loop-engineering workflow variant, saved as a project-level skill the user can invoke by name. Use when the user runs /loop-engineering-create followed by a description of a custom loop, e.g. "/loop-engineering-create - skip security-review, add a load test step".
---

$ARGUMENTS describes a custom variant of the loop-engineering workflow (see
the `loop-engineering-run` skill for the baseline 11-step sequence, steps
0-10: set goal -> build with ponytail -> stage -> code-review -> debug -> QA
-> verify -> simplify (ponytail-review/audit) -> security-review ->
production check -> repeat).

Do this:

1. If $ARGUMENTS doesn't include a short name for the workflow, ask the user
   for one (a few words, e.g. "backend-only" or "no-security").
2. Slugify that name (lowercase, hyphens) and write a new skill file to
   `.claude/skills/loop-engineering-<slug>/SKILL.md` in the CURRENT project
   (create the directories if needed). Same naming pattern as the plugin's
   own commands (`loop-engineering-run`, `loop-engineering-create`) and
   `ponytail`'s (`ponytail-review`, `ponytail-audit`) — the plugin name
   repeated as a prefix, no `-custom-` infix, no colon. Base the content on
   the baseline 11-step sequence, applying whatever the user's description
   changes (add, remove, or reorder steps; swap which skills get invoked;
   change the verify target, etc.). Keep the same imperative, step-by-step
   style as the baseline skill, and keep an explicit numbered step list.
3. Give the new skill a `description:` frontmatter line stating what makes
   it different from the baseline, so it's easy to tell apart later.
4. Tell the user plainly: this new skill won't be invocable until they
   restart or reload the Claude Code session (skills are only discovered at
   session start). Once reloaded, they can run it as
   `/loop-engineering-<slug>` — a project skill, not part of this plugin,
   so it stays local to this project and won't follow them elsewhere.
5. Do not run the new workflow yourself in this same turn — just create the
   file and report the exact command name for next time.
