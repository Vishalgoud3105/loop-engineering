---
name: loop-engineering-edit
description: Add, remove, or skip steps in an existing loop-engineering workflow — the baseline loop-engineering-run (coding-focused), or any custom loop you already created with loop-engineering-newloop, coding or not (a content-review loop, a research loop, anything). Never modifies the baseline itself. Use when the user runs /loop-engineering-edit followed by which workflow and what to add/remove, e.g. "/loop-engineering-edit - skip security-review, add a load test step after QA" or "/loop-engineering-edit - in loop-engineering-content-review, add a legal-check step after fact-check". If the user doesn't name the resulting variant, auto-name it from the description. To instead build an unrelated loop from scratch, use /loop-engineering-newloop instead.
---

Add, remove, or skip steps in an existing loop-engineering workflow —
works on the coding baseline or on any custom loop you already built,
whatever domain it's for.

$ARGUMENTS names which workflow to modify and describes the step-level
changes: add a step, remove/skip a step, or both in the same call.

Do this:

1. **Identify the target.**
   - If $ARGUMENTS says "run", "baseline", or names nothing else specific,
     the target is the baseline `loop-engineering-run` sequence (steps
     0-10: set goal -> build with ponytail -> stage -> code-review -> debug
     -> QA -> verify -> simplify (ponytail-review/audit) -> security-review
     -> production check -> repeat).
   - If $ARGUMENTS names an existing project-local custom skill (something
     already created by `loop-engineering-newloop` or a previous
     `loop-engineering-edit`/`loop-engineering-swap` call, found under
     `.claude/skills/loop-engineering-*/SKILL.md` in the current project —
     coding or not, e.g. a content-review or research loop), the target is
     that file.
   - Can't find a match → ask the user which workflow they mean, listing
     what's actually available in the current project plus the baseline.
2. **Naming.** If $ARGUMENTS explicitly names the resulting variant,
   slugify that (lowercase, hyphens). If it doesn't, auto-generate a short
   slug from what changed — don't ask, just pick something that reads
   clearly (e.g. skipping security-review and adding a load test →
   `no-security-load-test`) and tell the user what you named it.
3. Start from the target's exact step list and content, then apply only
   what was asked: add a described step at the right point in the
   sequence, and/or remove/skip a named step. Renumber the remaining steps
   sequentially. Four elements are protected regardless of what else
   changes: the goal-setting step, the final `Repeat` step, any
   `loop-state.md` memory read/write, and any human-gate step (a step that
   pauses for approval before publishing/sending/deploying/spending/
   deleting). That anatomy — goal, memory, gate, repeat — is what makes it
   a loop-engineering workflow at all. If the user explicitly asks to
   remove a human gate, warn once about what will then auto-execute
   unattended, and only proceed if they confirm.
4. **Where to write the result — this depends on the target, same rule
   `loop-engineering-swap` follows:**
   - Target was the baseline → NEVER edit
     `plugins/loop-engineering/skills/loop-engineering-run/SKILL.md`. Write
     a new skill instead to `.claude/skills/loop-engineering-run-<slug>/SKILL.md`
     — the extra `-run-` segment marks it as a *variant of the coding
     baseline*, distinct from `loop-engineering-newloop`'s
     `loop-engineering-<slug>` (unrelated from-scratch loops, any domain).
     Same underlying naming pattern as `ponytail`'s (`ponytail-review`,
     `ponytail-audit`): the plugin name repeated as a prefix, no colon.
   - Target was an existing custom skill you already created → that file
     is already project-owned, not part of this plugin, so edit it in
     place. Update its `description:` frontmatter to reflect the change.
   If a skill already exists at the destination path, ask before
   overwriting it.
5. Give the result a `description:` frontmatter line stating exactly what
   was added/removed, so it's easy to tell apart later.
6. End your reply with this exact callout, unmissable, its own paragraph —
   a skill has no way to trigger a reload itself, so this is the only way
   the user finds out it's needed:

   > ⚠️ **Restart required before this change takes effect.**
   > VSCode extension: `Ctrl+Shift+P` → **"Developer: Reload Window"**.
   > Standalone CLI: exit and restart `claude`. Skills are only loaded at
   > session start — even editing an existing skill's steps needs a reload
   > before the new step order or content is picked up.
7. Do not run the new/edited workflow yourself in this same turn — just
   report exactly what changed and the command name to use next time.
