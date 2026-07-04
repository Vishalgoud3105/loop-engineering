# loop-check

A Claude Code plugin marketplace containing **loop-check**: an end-to-end
"loop engineering" QA workflow for Claude Code. It chains together build,
review, debugging, QA, verification, over-engineering cleanup, security, and
production-readiness into one repeatable cycle that keeps running until the
codebase is actually clean — not just until the first pass looks okay.

This is *not* the same thing as Claude Code's built-in `/loop` command
(which repeats a prompt on a timer/interval). loop-check is a fixed,
opinionated 11-step quality sequence you run once per feature/change, and it
loops internally until every step passes clean in the same run.

---

## Why this exists

Most "build a feature" sessions go: write code, maybe skim it, ship it. Bugs,
security holes, and over-engineered slop slip through because nothing forces
a second look — and even when a review step *does* run, its findings don't
automatically get re-checked after the fix. loop-check makes the second
(and third, and fourth) look mandatory and automatic: every stage's output
feeds the next, and the whole cycle repeats until a full pass changes
nothing and the stated goal is met.

---

## Prerequisites

loop-check is an orchestrator — it doesn't reimplement review, debugging, or
security scanning itself, it calls out to other skills by name. Before
running `/loop-engineering:run`, make sure these are present in your session:

| Skill it invokes | Used at | Source | How to get it |
|---|---|---|---|
| `code-review` | Step 3 | Built into Claude Code | Nothing to do — ships with the CLI |
| `verify` | Step 6 | Built into Claude Code | Nothing to do — ships with the CLI |
| `security-review` | Step 8 | Built into Claude Code | Nothing to do — ships with the CLI |
| `ponytail` | Step 1 (build) | `ponytail` marketplace | `/plugin marketplace add DietrichGebert/ponytail` then `/plugin install ponytail@ponytail` |
| `ponytail-review` | Step 7 | `ponytail` marketplace | included in the `ponytail` plugin above |
| `ponytail-audit` | Step 7 (pass 1 only) | `ponytail` marketplace | included in the `ponytail` plugin above |
| `debugging-code` | Step 4 | `debug-skill` plugin, `claude-community` marketplace | `/plugin marketplace add anthropics/claude-plugins-community` then `/plugin install debug-skill@claude-community` — **optional**, step 4 falls back to manual debugging if this isn't installed |

Check what you already have with the **Manage Plugins** panel (or
`installed_plugins.json` under your Claude Code config folder) before
installing anything twice.

After installing any plugin, **restart or reload your Claude Code session**
— skills/commands are only discovered at session start, so a plugin you
just installed won't autocomplete or be invocable until you do.

---

## Install loop-check itself

```
/plugin marketplace add Vishalgoud3105/loop-check
/plugin install loop-engineering@loop-check
```

Requires Claude Code with plugin support. Same rule as above: restart or
reload your session afterward so `/loop-engineering:run` and
`/loop-engineering:create` are discovered.

---

## Usage

### Run the loop

```
/loop-engineering:run
/loop-engineering:run path/to/spec.md
/loop-engineering:run path/to/spec.pdf
```

The argument is optional — a path to whatever spec/PRD/context doc defines
"done" for this project. If you don't pass one, loop-check looks for an
obvious candidate (README, PRD, `docs/*.pdf`) in the current project before
asking you for one.

### What actually happens, step by step

**Step 0 — Set the goal.** Reads the spec doc and writes down one concrete,
checkable goal statement — what "done" looks like. This isn't the built-in
`/goal` command (that's a chat-box-only feature backed by a client-side
hook; a skill has no way to trigger it). Instead, the goal statement is
carried through the rest of the run as plain context, and it's what steps 6
and 10 check against.

**Step 1 — Build.** Invokes `ponytail` to build or extend the codebase,
following YAGNI: reuse what's already there, reach for stdlib/native
platform features before writing new code, no speculative abstractions.

**Step 2 — Stage.** `git add`s everything so the review steps below see the
full diff. Initializes a git repo first if there isn't one yet.

**Step 3 — Review.** Runs `code-review` on the staged diff.

**Step 4 — Debug.** Fixes every issue code-review flagged, root cause first
(checks other callers of the same function, not just the one call site the
issue was found in). If the `debugging-code` skill is installed, it's used
to investigate live behavior — set breakpoints, inspect variables, step
through execution — before making the fix. Falls back to manual tracing if
that skill isn't installed.

**Step 5 — QA.** Static and live analysis: normal use cases, edge cases,
boundary/invalid inputs, and realistic scenarios nobody explicitly asked
about. Writes/runs unit and A-B style tests for whatever changed, where the
project has a test runner.

**Step 6 — Verify.** Runs `verify` to confirm the working code actually
satisfies the goal statement from step 0 — not just "does it run," but "does
it do the thing the spec asked for." Skipped for diffs with no runtime
surface (docs/tests-only changes).

**Step 7 — Simplify.** Runs `ponytail-review` on the diff to catch
over-engineering left over from step 1: reinvented standard library code,
unneeded dependencies, speculative abstractions, dead flexibility. On the
first pass only (or if you explicitly ask for a full sweep), also runs
`ponytail-audit` across the whole repo — that one's expensive, so it doesn't
repeat every pass.

**Step 8 — Security.** Runs `security-review` and fixes every vulnerability
it reports before moving on.

**Step 9 — Production check.** A plain bar: error handling at trust
boundaries, no secrets or debug output left in, no dead/commented-out code,
dependencies pinned appropriately. Anything that fails gets fixed.

**Step 10 — Repeat.** If steps 3-9 made *any* change, or the step-0 goal
isn't satisfied yet, go back to step 2 and run the whole cycle again. Only
stops when a full pass makes zero changes and the goal is met.

At the end you get a short report: how many passes it took and a one-line
summary of what each pass fixed — not a long narrative unless you ask for
one.

---

### Create a custom variant

```
/loop-engineering:create - <description of what should be different>
```

Examples:
```
/loop-engineering:create - skip security-review, add a load test step
/loop-engineering:create - backend only, no ponytail-audit sweep
```

This asks you for a short name if you didn't give one, then writes a new
project-local skill to:

```
.claude/skills/loop-engineering-custom-<slug>/SKILL.md
```

based on the baseline 11-step sequence with your requested changes applied.

**Important limitation:** this new skill is a plain project skill, not a
plugin skill, so it can't share the `loop-engineering:` namespace — plugins can't
add commands to themselves at runtime, and Claude Code only discovers
skills at session startup. So:

- It's invoked as `/loop-engineering-custom-<slug>` (flat name, no colon).
- It won't show up until you restart or reload your Claude Code session
  after it's created.
- It lives in *your project*, not in this plugin — it won't follow you to
  other repos, and updating the plugin won't touch it.

You can create as many of these as you want, one per project, each named
whatever you asked for.

---

## Repo layout

```
.claude-plugin/marketplace.json      # marketplace listing (this repo)
plugins/loop-check/
  .claude-plugin/plugin.json         # plugin manifest
  skills/
    run/SKILL.md                     # /loop-engineering:run  — the 11-step loop
    create/SKILL.md                  # /loop-engineering:create — custom variants
```

Claude Code plugin commands are always namespaced `plugin-name:skill-name`
— there's no way to expose a bare `/loop-engineering`, which is why the main
workflow is `/loop-engineering:run` rather than just `/loop-engineering`.
Note the plugin is named `loop-engineering` while the marketplace/repo is
named `loop-check` — those are two independent identifiers (see Install
section); the plugin name is what drives the command prefix.

---

## Publishing changes / contributing

This is a personal marketplace (one owner, one plugin) rather than a
community-submission repo, but if you fork it:

1. Edit the relevant `SKILL.md` under `plugins/loop-check/skills/`.
2. Commit and push.
3. Anyone who already ran `/plugin marketplace add` against your fork picks
   up the update the next time Claude Code refreshes marketplaces (or via
   `/plugin marketplace update`, if your version supports it).

To submit `loop-check` itself to Anthropic's official community plugin
directory instead of distributing it as your own marketplace, see
[Plugins — Submit your plugin](https://code.claude.com/docs/en/plugins.md#submit-your-plugin-to-the-community-marketplace)
(requires running `claude plugin validate` first).

---

## Troubleshooting

**"No matching commands" when I type the skill name.** Some installed
skills aren't meant to be typed directly in the chat box — they're invoked
by Claude via the Skill tool when relevant (that's how `debugging-code`
works here). Also, Claude Code generally only refreshes its command index
on session start/reload, so a fresh install often needs a restart before
autocomplete picks it up.

**`/debug` doesn't do what I expect.** The built-in `/debug` command only
toggles CLI diagnostic logging (equivalent to `claude --debug`) — it is
*not* a code-debugging tool and loop-check never treats it as one. The
actual debugger this plugin uses is the separate `debugging-code` skill
from the `debug-skill` community plugin.

**Can I use the built-in `/goal` command with this?** Not from inside the
skill — `/goal` is a chat-box-only feature backed by a client-side hook,
and a skill has no mechanism to invoke it. loop-check achieves the same
"keep working until a condition holds" behavior natively via step 0 (goal
statement) and step 10 (repeat-until-met), so you don't need `/goal` at
all when using `/loop-engineering:run`.
