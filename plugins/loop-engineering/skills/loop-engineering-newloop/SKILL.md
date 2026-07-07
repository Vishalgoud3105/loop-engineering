---
name: loop-engineering-newloop
description: Build a brand-new loop workflow from scratch (not a variant of the baseline) and save it as its own reusable slash command. Use when the user runs /loop-engineering-newloop followed by a description of a new loop, e.g. "/loop-engineering-newloop - a content-review loop with draft, fact-check, tone-check, and publish steps, repeating until no edits". If the user doesn't name the workflow, auto-name it from the description. To instead modify the existing 11-step baseline (add/remove/skip a step), use /loop-engineering-edit instead.
---

Build a brand-new loop workflow from scratch, for anything — a
content-review loop, a research loop, a data-pipeline loop, not just code
QA. Save any number of these as independent, reusable slash commands.

$ARGUMENTS describes a brand-new loop workflow — NOT a variant of
`loop-engineering-run`'s 11-step baseline (that's what
`loop-engineering-edit` is for). Design whatever numbered step sequence
actually matches the description, invoking whatever skills/tools fit — not
necessarily `ponytail`, `code-review`, `security-review`, etc. at all.

The steps in between are the user's to define, but every generated loop
MUST carry the four structural elements that make it a loop-engineering
workflow (not just a checklist):

- **Goal (step 0):** a concrete, checkable statement of what "done" looks
  like — the evaluation criterion every cycle is judged against.
- **Memory:** the loop reads/writes a `loop-state.md` file each cycle
  (goal, pass number, what was already handled, what's blocked) so a
  repeated cycle never redoes work it already did, and an interrupted
  loop resumes instead of restarting.
- **Human gate:** any step with consequences beyond the local
  project — publishing, sending to anyone, deploying, spending money,
  deleting things — must stop and ask the human for explicit approval,
  never auto-execute. The loop drafts and prepares; the human presses
  the final button. Write this gate into the generated skill at the
  exact steps it applies to.
- **Repeat (last step):** go back and re-run until a full pass makes no
  changes and the goal is satisfied — logging, not hiding, anything
  that couldn't run (honesty log inside `loop-state.md`).

Do this:

1. **Naming.** If $ARGUMENTS explicitly names the workflow, slugify that
   (lowercase, hyphens). If it doesn't, auto-generate a short slug from the
   description itself — don't ask, just pick something that reads clearly
   (e.g. a loop about drafting/fact-checking/publishing content →
   `content-review`) and tell the user what you named it so they know the
   command to invoke.
2. Write a new skill file to `.claude/skills/loop-engineering-<slug>/SKILL.md`
   in the CURRENT project (create directories if needed) — same naming
   pattern as the plugin's own commands (`loop-engineering-run`,
   `loop-engineering-newloop`) and `ponytail`'s (`ponytail-review`,
   `ponytail-audit`): the plugin name repeated as a prefix, no colon.
   If a skill already exists at that path, ask before overwriting it rather
   than silently clobbering an earlier custom loop.
3. Keep the same imperative, step-by-step style as `loop-engineering-run`,
   with an explicit numbered step list — whatever the steps actually are
   for this workflow.
4. Give the new skill a `description:` frontmatter line stating what the
   workflow does, so Claude's own skill matching picks the right one when
   the user mentions it in plain language later.
5. End your reply with this exact callout, unmissable, its own paragraph —
   a skill has no way to trigger a reload itself, so this is the only way
   the user finds out it's needed:

   > ⚠️ **Restart required before `/loop-engineering-<slug>` works.**
   > VSCode extension: `Ctrl+Shift+P` → **"Developer: Reload Window"**.
   > Standalone CLI: exit and restart `claude`. Skills are only discovered
   > at session start — typing the command before reloading will show
   > "Unknown command" even though the file now exists.
6. Do not run the new workflow yourself in this same turn — just create the
   file and report the exact command name for next time.
7. This can be invoked repeatedly, once per new loop workflow the user
   wants — each call produces its own independent skill under its own name.
