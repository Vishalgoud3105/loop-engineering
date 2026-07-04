---
name: loop-engineering-run
description: Run Vishal's "loop engineering" workflow - set a goal from the spec doc, build with ponytail (YAGNI), then code-review, debug, QA, verify against that goal, over-engineering check, security-review, and a production-grade check. Repeat until clean AND the goal is met. Use when the user says "run the loop", "loop check", "loop testing", or invokes /loop-engineering-run. Not the same as the built-in /loop or /goal commands.
---

Run the full loop-engineering QA cycle on the current project. This is a fixed
11-step sequence (0-10) — do not skip steps, and do not stop after the first
pass if step 9 or 10 finds problems.

Argument (optional): $ARGUMENTS may name a target directory and/or a path to
a spec/PDF/context doc. If empty, infer the target from the current working
directory and look for an obvious spec (README, PRD, docs/*.pdf) before
asking the user.

Note on scope: the built-in `/goal` command is a chat-box feature backed by a
client-side hook — a skill has no way to trigger it. Step 0 below achieves
the same effect natively, driving the whole loop, not just verify.

Note on working directory: `security-review` (step 8) checks Claude Code's
actual session working directory, not a directory a shell command `cd`s
into. Run this skill with Claude Code's working directory set to the target
project's git repo root — if it's launched from a parent directory, step 8
will report the repo can't be found even though the project is right there.

0. **Set the goal.** Before step 1, read the spec/context doc and write down
   a single concrete, checkable goal statement (what "done" looks like —
   e.g. "all endpoints in spec.md are implemented and pass their described
   behavior, with no extra unrequested endpoints"). This goal is the
   condition step 6 checks and the condition step 10's repeat depends on —
   carry it through the whole loop, not just step 6.

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
   `debugging-code` skill (breakpoints, step-through, live-variable
   inspection, call-stack tracing) is available in this session, invoke it
   via the Skill tool (`debugging-code:debugging-code`) to investigate
   flagged issues before fixing. Note: the built-in `/debug` command is
   unrelated — it only toggles CLI diagnostic logging, not a code-debugging
   tool, so never treat it as this step's debugger. `debugging-code` also
   needs its own native `dap` CLI binary installed separately (not just the
   plugin) — check whether it's on PATH first (`command -v dap` on
   macOS/Linux/Git-Bash, `where dap` on native Windows cmd/PowerShell), and
   never install it without telling the user first. If `debugging-code` or
   `dap` isn't available, trace and fix manually.
5. **QA.** Do static and live analysis covering: normal use cases, edge
   cases, boundary/invalid inputs, and realistic unseen scenarios. Where the
   project has a test runner, write/run unit and A-B style tests for the
   changed behavior.
6. **Verify.** Invoke the `verify` skill to confirm the working code
   actually satisfies the goal statement from step 0. Only run this on
   changes with a runtime surface — skip if the diff is docs/tests only.
7. **Simplify.** Invoke the `ponytail-review` skill on the staged diff to
   catch over-engineering left over from step 1: reinvented stdlib, unneeded
   dependencies, speculative abstractions, dead flexibility. Fix what it
   finds. Only on the first pass of this loop (or if the user explicitly
   asks for a full sweep), also invoke `ponytail-audit` for a whole-repo
   check — it's expensive, so don't repeat it every pass.
8. **Security.** Invoke the `security-review` skill. Fix every vulnerability
   it reports before continuing.
9. **Production check.** Review the result against a plain production bar:
   error handling at trust boundaries, no secrets/debug output left in,
   no dead/commented-out code, dependencies pinned appropriately. Fix
   anything that fails this bar.
10. **Repeat.** If steps 3-9 made any changes, OR the goal statement from
    step 0 isn't yet satisfied, go back to step 2 (re-stage) and run the
    cycle again. Stop only when a full pass makes zero changes AND the goal
    is satisfied.

Report at the end: how many passes it took, and a one-line summary of what
each pass fixed. Do not produce a long narrative report unless asked.
