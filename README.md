# loop-engineering

A Claude Code plugin marketplace containing **loop-engineering**: an end-to-end
"loop engineering" QA workflow for Claude Code. It chains together build,
review, debugging, QA, verification, over-engineering cleanup, security, and
production-readiness into one repeatable cycle that keeps running until the
codebase is actually clean — not just until the first pass looks okay.

This is *not* the same thing as Claude Code's built-in `/loop` command
(which repeats a prompt on a timer/interval). loop-engineering is a fixed,
opinionated 11-step quality sequence you run once per feature/change, and it
loops internally until every step passes clean in the same run.

---

## Why this exists

Most "build a feature" sessions go: write code, maybe skim it, ship it. Bugs,
security holes, and over-engineered slop slip through because nothing forces
a second look — and even when a review step *does* run, its findings don't
automatically get re-checked after the fix. loop-engineering makes the second
(and third, and fourth) look mandatory and automatic: every stage's output
feeds the next, and the whole cycle repeats until a full pass changes
nothing and the stated goal is met.

---

## Prerequisites

loop-engineering is an orchestrator — it doesn't reimplement review, debugging, or
security scanning itself, it calls out to other skills by name. Before
running `/loop-engineering:run`, make sure these are present in your session:

| Skill it invokes | Used at | Source | How to get it |
|---|---|---|---|
| `ponytail` | Step 1 (build) | `ponytail` marketplace | `/plugin marketplace add DietrichGebert/ponytail` then `/plugin install ponytail@ponytail` |
| `code-review` | Step 3 | Built into Claude Code | Nothing to do — ships with the CLI |
| `debugging-code` | Step 4 | `debug-skill` plugin, `claude-community` marketplace | `/plugin marketplace add anthropics/claude-plugins-community` then `/plugin install debug-skill@claude-community` — **optional**, step 4 falls back to manual debugging if this isn't installed. Also needs its own native `dap` CLI binary installed separately — see the Windows note below, the plugin's own install script is macOS/Linux-only. |
| `verify` | Step 6 | Built into Claude Code | Nothing to do — ships with the CLI |
| `ponytail-review` | Step 7 | `ponytail` marketplace | included in the `ponytail` plugin above |
| `ponytail-audit` | Step 7 (pass 1 only) | `ponytail` marketplace | included in the `ponytail` plugin above |
| `security-review` | Step 8 | Built into Claude Code | Nothing to do — ships with the CLI |

Check what you already have with the **Manage Plugins** panel (or
`installed_plugins.json` under your Claude Code config folder) before
installing anything twice.

After installing any plugin, **restart or reload your Claude Code session**
— skills/commands are only discovered at session start, so a plugin you
just installed won't autocomplete or be invocable until you do.

**Working directory:** run `/loop-engineering:run` with Claude Code's
working directory set to the target project's git repo root. Step 8
(`security-review`) checks the session's actual working directory, not a
subdirectory something merely `cd`s into — if you launch from a parent
directory, step 8 will report it can't find a git repo even though the
project is right there.

**Windows users — installing `dap` for step 4:** the `debug-skill` plugin's
bundled installer (`install-dap.sh`) only auto-detects macOS and Linux
(`uname`-based), and its per-language backend docs assume Homebrew/apt. On
native Windows (cmd/PowerShell), install `dap` via Go instead, which works
cross-platform:

```
go install github.com/AlmogBaku/debug-skill/cmd/dap@latest
```

This needs [Go](https://go.dev/dl/) installed, and puts `dap.exe` under
`%USERPROFILE%\go\bin` — make sure that's on your PATH. Per-language
backends: Python (`pip install debugpy`) and Go (`go install
github.com/go-delve/delve/cmd/dlv@latest`) both work fine on Windows this
way; Node/TypeScript (js-debug) should auto-discover a VS Code/Cursor
install same as elsewhere. Rust/C/C++ (`lldb-dap`) has **no documented
Windows install path** in the plugin as of this writing — if you need that,
use WSL, or skip installing `dap` entirely and let step 4 fall back to
manual debugging.

---

## Install loop-engineering itself

```
/plugin marketplace add Vishalgoud3105/loop-engineering
/plugin install loop-engineering@loop-engineering
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
"done" for this project. If you don't pass one, loop-engineering looks for an
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
plugins/loop-engineering/
  .claude-plugin/plugin.json         # plugin manifest
  skills/
    run/SKILL.md                     # /loop-engineering:run  — the 11-step loop
    create/SKILL.md                  # /loop-engineering:create — custom variants
```

Claude Code plugin commands are always namespaced `plugin-name:skill-name`
— there's no way to expose a bare `/loop-engineering`, which is why the main
workflow is `/loop-engineering:run` rather than just `/loop-engineering`.
The plugin name and the marketplace/repo name both happen to be
`loop-engineering` here, but they're independent identifiers — the plugin
name is what drives the command prefix (see Install section).

---

## Contributing (for anyone who forks this repo)

This is a personal marketplace (one owner, one plugin) rather than a
community-submission repo — there's no process here for submitting changes
back to `Vishalgoud3105/loop-engineering` itself. But if you fork it to make
your own variant:

1. Edit the relevant `SKILL.md` under `plugins/loop-engineering/skills/`.
2. Commit and push to *your* fork.
3. Anyone who ran `/plugin marketplace add` against *your* fork's URL picks
   up your changes the next time Claude Code refreshes marketplaces (or via
   `/plugin marketplace update`, if your version supports it) — no
   reinstall needed on their end.

## Maintainer note: official directory submission

To make `loop-engineering` discoverable without anyone needing to run
`/plugin marketplace add Vishalgoud3105/loop-engineering` first, it can be
submitted to Anthropic's official community plugin directory instead of
staying a self-hosted marketplace. See
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
*not* a code-debugging tool and loop-engineering never treats it as one. The
actual debugger this plugin uses is the separate `debugging-code` skill
from the `debug-skill` community plugin.

**Can I use the built-in `/goal` command with this?** Not from inside the
skill — `/goal` is a chat-box-only feature backed by a client-side hook,
and a skill has no mechanism to invoke it. loop-engineering achieves the same
"keep working until a condition holds" behavior natively via step 0 (goal
statement) and step 10 (repeat-until-met), so you don't need `/goal` at
all when using `/loop-engineering:run`.

**Step 8 says it can't find a git repository.** `security-review` checks
Claude Code's actual session working directory. Make sure you launched
Claude Code (or `cd`'d before starting the session) inside the target
project's git repo root, not a parent folder — see Prerequisites above.

---

## Tested

Before writing the sequence above, it was dry-run end to end against a
throwaway sandbox project (a small Python CLI) with deliberately injected
issues, following `run/SKILL.md`'s instructions exactly and invoking the
real underlying skills:

- **Step 1 (ponytail):** built a clean, minimal implementation on the first try.
- **Step 3 (code-review):** caught an injected command-injection bug, an
  off-by-one, and a silently-swallowed missing-file error — correctly
  scoped to correctness only, ignored the over-engineering left for step 7.
- **Step 4 (debug):** root-caused the injection by replacing the vulnerable
  call outright rather than patching around it; re-verified live afterward.
- **Step 5 (QA) / Step 6 (verify):** confirmed correct behavior across
  normal input, empty file, missing argument, and adjacent probes (directory
  passed as a file, punctuation-only input).
- **Step 7 (ponytail-review/audit):** caught a speculative Strategy/Factory
  abstraction with a single implementation, cut ~18 lines with no behavior
  change.
- **Step 9 (production check):** confirmed no secrets, debug output, or
  dead code.
- **Step 10 (repeat):** correctly re-ran after pass 1 made changes, then
  correctly stopped on a clean pass 2.

This dry run is what surfaced the `dap` binary requirement and the working
directory requirement documented above.
