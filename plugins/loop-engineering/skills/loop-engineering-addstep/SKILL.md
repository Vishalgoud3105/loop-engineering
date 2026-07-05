---
name: loop-engineering-addstep
description: Add, remove, or skip steps from the baseline loop-engineering-run 11-step sequence, saved as a new project-level variant skill named /loop-engineering-run-{name} — never modifies the shipped baseline itself. Use when the user runs /loop-engineering-addstep followed by a description of what to add and/or remove, e.g. "/loop-engineering-addstep - skip security-review, add a load test step after QA". If the user doesn't name the variant, auto-name it from the description. To instead build an unrelated loop from scratch, use /loop-engineering-scratch instead.
---

Add, remove, or skip steps from the baseline 11-step
`loop-engineering-run` sequence without ever touching the plugin's shipped
file — every change here produces a separate new variant skill instead.

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

1. **Naming.** If $ARGUMENTS explicitly names the variant, slugify that
   (lowercase, hyphens). If it doesn't, auto-generate a short slug from
   what changed — don't ask, just pick something that reads clearly (e.g.
   skipping security-review and adding a load test → `no-security-load-test`,
   or just `no-security` if that's the more prominent change) and tell the
   user what you named it.
2. Start from the baseline's exact step list and content, then apply only
   what was asked: add a described step at the right point in the sequence,
   and/or remove/skip a named step. Renumber the remaining steps
   sequentially. Keep step 0 (set the goal) and the final `Repeat` step
   intact regardless of what else changes — that bookend shape is the part
   of "my plugin form" that must stay intact even as the steps between them
   change.
3. Write the result to `.claude/skills/loop-engineering-run-<slug>/SKILL.md`
   in the CURRENT project (create directories if needed) — note the extra
   `-run-` segment versus `loop-engineering-scratch`'s
   `loop-engineering-<slug>`: it marks this as a *variant of the run
   baseline*, not an unrelated from-scratch loop. Same underlying pattern
   as `ponytail`'s (`ponytail-review`, `ponytail-audit`): the plugin name
   repeated as a prefix, no colon. If a skill already exists at that path,
   ask before overwriting it.
4. Give the new skill a `description:` frontmatter line stating exactly
   what differs from the baseline (which step was added/removed), so it's
   easy to tell apart later.
5. End your reply with this exact callout, unmissable, its own paragraph —
   a skill has no way to trigger a reload itself, so this is the only way
   the user finds out it's needed:

   > ⚠️ **Restart required before `/loop-engineering-run-<slug>` works.**
   > VSCode extension: `Ctrl+Shift+P` → **"Developer: Reload Window"**.
   > Standalone CLI: exit and restart `claude`. Skills are only discovered
   > at session start — typing the command before reloading will show
   > "Unknown command" even though the file now exists.
6. Do not run the new workflow yourself in this same turn — just create the
   file and report the exact command name and exactly what changed from the
   baseline.
