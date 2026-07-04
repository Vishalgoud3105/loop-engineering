---
name: loop-engineering-swap
description: Swap/reorder two or more steps in an existing loop-engineering workflow — the baseline loop-engineering-run, or an already-created custom loop from loop-engineering-create-loop or loop-engineering-addstep. Companion to loop-engineering-addstep, for reordering instead of adding/removing. Use when the user runs /loop-engineering-swap followed by which workflow and which steps to swap, e.g. "/loop-engineering-swap - in loop-engineering-run, swap the debug and QA steps" or "/loop-engineering-swap - in loop-engineering-backend-only, swap steps 2 and 4".
---

$ARGUMENTS names which workflow to modify and which steps to swap (by
number or by description — resolve either form against the target's actual
step list).

Do this:

1. **Identify the target.**
   - If $ARGUMENTS says "run", "baseline", or names nothing else specific,
     the target is the baseline `loop-engineering-run` sequence.
   - If $ARGUMENTS names an existing project-local custom skill (something
     already created by `loop-engineering-create-loop` or
     `loop-engineering-addstep`, found under
     `.claude/skills/loop-engineering-*/SKILL.md` in the current project),
     the target is that file.
   - Can't find a match → ask the user which workflow they mean, listing
     what's actually available in the current project plus the baseline.
2. Locate the named steps in the target's step list and swap their
   positions, renumbering every step sequentially afterward. The
   goal-setting step (step 0 in the baseline) and the final `Repeat` step
   are fixed bookends — never swap those out of position, only reorder the
   steps between them.
3. **Where to write the result — this depends on the target, same rule
   `loop-engineering-addstep` follows:**
   - Target was the baseline → NEVER edit
     `plugins/loop-engineering/skills/loop-engineering-run/SKILL.md`. Write
     a new skill instead to `.claude/skills/loop-engineering-run-<slug>/SKILL.md`
     (auto-name the slug from what was swapped if the user didn't give one,
     e.g. swapping debug and QA → `debug-qa-swapped`).
   - Target was an existing custom skill you already created → that file is
     already project-owned, not part of this plugin, so edit it in place.
     Update its `description:` frontmatter to reflect the new order.
4. Give the result a `description:` frontmatter line stating what was
   swapped, so it's easy to tell apart later.
5. End your reply with this exact callout, unmissable, its own paragraph —
   a skill has no way to trigger a reload itself:

   > ⚠️ **Restart required before this change takes effect.**
   > VSCode extension: `Ctrl+Shift+P` → **"Developer: Reload Window"**.
   > Standalone CLI: exit and restart `claude`. Skills are only loaded at
   > session start — even editing an existing skill's steps needs a reload
   > before the new order is picked up.
6. Do not run the workflow yourself in this same turn — just report exactly
   what was swapped and the command name to use next time.
