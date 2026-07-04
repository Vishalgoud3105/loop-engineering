---
name: run
description: Run Vishal's "loop engineering" workflow - build with ponytail (YAGNI), then code-review, debug, QA, verify against the stated goal, security-review, and a production-grade check. Repeat until clean. Use when the user says "run the loop", "loop check", "loop testing", or invokes /loop-check:run. Not the same as the built-in /loop command.
---

Run the full loop-engineering QA cycle on the current project. This is a fixed
9-step sequence — do not skip steps, and do not stop after the first pass if
step 8 or 9 finds problems.

Argument (optional): $ARGUMENTS may name a target directory and/or a path to
a spec/PDF/context doc to verify against in step 6. If empty, infer the
target from the current working directory and look for an obvious spec
(README, PRD, docs/*.pdf) before asking the user.

Steps, in order:

1. **Build.** Invoke the `ponytail` skill to build or extend the codebase.
   Follow YAGNI: reuse existing code, stdlib, and native platform features
   before writing new code. Do not generate speculative abstractions.
2. **Stage.** `git add` all generated/changed files so the review steps below
   see the full diff. If this isn't a git repo, `git init` first (ask the
   user before doing so if the directory has unrelated existing content).
3. **Review.** Invoke the `code-review` skill on the staged diff.
4. **Debug.** For every issue code-review flagged, fix the root cause (not
   just the symptom — check other callers of the same function). If the
   `debug-skill` plugin (community marketplace, root cause / breakpoint /
   live-variable debugging) is installed in this session, invoke it to
   investigate flagged issues before fixing. Note: the built-in `/debug`
   command is unrelated — it only toggles CLI diagnostic logging, not a
   code-debugging tool, so never treat it as this step's debugger. If
   `debug-skill` isn't installed, trace and fix manually.
5. **QA.** Do static and live analysis covering: normal use cases, edge
   cases, boundary/invalid inputs, and realistic unseen scenarios. Where the
   project has a test runner, write/run unit and A-B style tests for the
   changed behavior.
6. **Verify.** Invoke the `verify` skill to confirm the working code
   actually satisfies the project's stated goal or spec doc (from
   $ARGUMENTS, or the one found in step 0). Only run this on changes with a
   runtime surface — skip if the diff is docs/tests only.
7. **Security.** Invoke the `security-review` skill. Fix every vulnerability
   it reports before continuing.
8. **Production check.** Review the result against a plain production bar:
   error handling at trust boundaries, no secrets/debug output left in,
   no dead/commented-out code, dependencies pinned appropriately. Fix
   anything that fails this bar.
9. **Repeat.** If steps 3-8 made any changes, go back to step 2 (re-stage)
   and run the cycle again. Stop only when a full pass makes zero changes.

Report at the end: how many passes it took, and a one-line summary of what
each pass fixed. Do not produce a long narrative report unless asked.
