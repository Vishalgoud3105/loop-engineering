---
name: loop-engineering-create
description: Create a new custom loop workflow, saved as a project-level skill the user can invoke by name — either a variant of the baseline loop-engineering-run sequence, or a completely different loop built from scratch for an unrelated purpose. Use when the user runs /loop-engineering-create followed by a description, e.g. "/loop-engineering-create - skip security-review, add a load test step" or "/loop-engineering-create - a content-review loop: draft, fact-check, tone-check, publish, repeat until no edits".
---

$ARGUMENTS describes a custom loop workflow. Two shapes this can take —
figure out which one from what the user actually describes, don't force it
into the wrong one:

- **A variant of the baseline.** The description tweaks, trims, or extends
  the `loop-engineering-run` sequence (see that skill for the baseline
  11-step sequence, steps 0-10: set goal -> build with ponytail -> stage ->
  code-review -> debug -> QA -> verify -> simplify (ponytail-review/audit)
  -> security-review -> production check -> repeat). Use this shape when the
  description references the baseline's steps or skills directly (skip a
  step, add one, swap what a step invokes, etc).
- **A workflow from scratch.** The description defines its own steps
  entirely unrelated to code QA (e.g. a writing/review loop, a research
  loop, a data-pipeline loop). In this case, don't force the baseline's
  steps onto it — design whatever numbered step sequence actually matches
  what the user described, invoking whatever skills/tools fit (not
  necessarily `ponytail`, `code-review`, etc. at all). The only structural
  requirement carried over from the baseline: state 0 is a concrete,
  checkable goal, and the last step is `Repeat` — go back and re-run until a
  full pass makes no changes and the goal is satisfied. That "set a goal,
  loop until it holds" shape is what makes it a loop-engineering workflow;
  the actual steps in between are the user's to define.

Do this regardless of which shape it is:

1. If $ARGUMENTS doesn't include a short name for the workflow, ask the user
   for one (a few words, e.g. "backend-only" or "content-review").
2. Slugify that name (lowercase, hyphens) and write a new skill file to
   `.claude/skills/loop-engineering-<slug>/SKILL.md` in the CURRENT project
   (create the directories if needed). Same naming pattern as the plugin's
   own commands (`loop-engineering-run`, `loop-engineering-create`) and
   `ponytail`'s (`ponytail-review`, `ponytail-audit`) — the plugin name
   repeated as a prefix, no `-custom-` infix, no colon. Keep the same
   imperative, step-by-step style as the baseline skill, and keep an
   explicit numbered step list, whatever the steps actually are.
3. Give the new skill a `description:` frontmatter line stating what the
   workflow does and, if relevant, what makes it different from the
   baseline — so it's easy to tell apart later, and so Claude's own skill
   matching picks the right one when the user mentions it in plain language.
4. Tell the user plainly: this new skill won't be invocable until they
   restart or reload the Claude Code session (skills are only discovered at
   session start). Once reloaded, they can run it as
   `/loop-engineering-<slug>` — a project skill, not part of this plugin,
   so it stays local to this project and won't follow them elsewhere.
5. Do not run the new workflow yourself in this same turn — just create the
   file and report the exact command name for next time.
