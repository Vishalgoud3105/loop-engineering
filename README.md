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
security scanning itself, it calls out to other skills by name.

**Auto-install (Claude Code v2.1.110+):** `plugin.json` declares `ponytail`
and `debugging-code` as dependencies, and the marketplace allowlists both for
cross-marketplace installs. Installing `loop-engineering` installs them
too, automatically, no separate steps — **as long as their marketplaces
are already known to your Claude Code setup.** If `ponytail`'s marketplace
or `claude-community` hasn't been added yet, that dependency stays
unresolved until you add it — Claude Code can't auto-add an unknown
marketplace, only auto-install a plugin from one it already knows about.
Re-running `/plugin install loop-engineering@loop-engineering` (or your
version's `/reload-plugins` equivalent) re-resolves anything still missing
once the marketplace is added.

If you're on an older Claude Code version, or a dependency didn't resolve,
here's the manual path for each:

| Skill it invokes | Used at | Source | How to get it |
|---|---|---|---|
| `ponytail` | Step 1 (build) | `ponytail` marketplace | `/plugin marketplace add DietrichGebert/ponytail` then `/plugin install ponytail@ponytail` |
| `code-review` | Step 3 | Built into Claude Code | Nothing to do — ships with the CLI |
| `debugging-code` | Step 4 | `debugging-code` plugin (same underlying package is also listed as `debug-skill` — identical repo/commit, either name installs it), `claude-community` marketplace | `/plugin marketplace add anthropics/claude-plugins-community` then `/plugin install debugging-code@claude-community` — **optional**, step 4 falls back to manual debugging if this isn't installed. Also needs its own native `dap` CLI binary installed separately — see the Windows note below, the plugin's own install script is macOS/Linux-only, and there's no auto-install mechanism for that binary regardless of Claude Code version. |
| `verify` | Step 6 | Built into Claude Code | Nothing to do — ships with the CLI |
| `ponytail-review` | Step 7 | `ponytail` marketplace | included in the `ponytail` plugin above |
| `ponytail-audit` | Step 7 (pass 1 only) | `ponytail` marketplace | included in the `ponytail` plugin above |
| `security-review` | Step 8 | Built into Claude Code | Nothing to do — ships with the CLI |

Check what you already have before installing anything twice: VSCode
extension → **Manage Plugins** panel; standalone CLI → `/plugin` command;
either way, `installed_plugins.json` under your Claude Code config folder
has the raw list.

**OS compatibility:** loop-engineering's own skill files (`run`, `scratch`,
`addstep`, `swap`) are plain instructions with no embedded shell scripts —
nothing in the plugin itself is OS-specific. Its dependencies: `code-review`, `verify`,
and `security-review` ship with Claude Code on every OS; `ponytail` ships
proper Windows (PowerShell) and Unix (bash) variants for all its hooks and
only needs Node.js (already required by Claude Code itself). The one
partial exception is `debugging-code`'s `dap` binary — optional, step 4
falls back to manual debugging without it — see the Windows note below.

After installing any plugin, **restart or reload your Claude Code session**
— skills/commands are only discovered at session start, so a plugin you
just installed won't autocomplete or be invocable until you do. VSCode
extension: `Ctrl+Shift+P` → **"Developer: Reload Window"**. Standalone CLI:
exit and restart `claude`. There's no way for a plugin to trigger this
itself — it's a manual step every time, for any plugin, not specific to
loop-engineering.

**Working directory:** run `/loop-engineering-run` with Claude Code's
working directory set to the target project's git repo root. Step 8
(`security-review`) checks the session's actual working directory, not a
subdirectory something merely `cd`s into — if you launch from a parent
directory, step 8 will report it can't find a git repo even though the
project is right there. VSCode extension: open the project's repo root as
your workspace folder. Standalone CLI: `cd` into the repo root *before*
running `claude`, not after it starts.

**Windows users — installing `dap` for step 4:** the `debug-skill` plugin's
bundled installer (`install-dap.sh`) only auto-detects macOS and Linux
(`uname`-based) and its docs never mention Windows, but a Windows binary
does exist — the plugin's own release CI builds `dap-windows-amd64.exe` on
every release, it's just not linked from any install doc. Two options, no
Go required for the first one:

- **Download the prebuilt binary (no Go needed):** grab
  `dap-windows-amd64.exe` from the [latest release](https://github.com/AlmogBaku/debug-skill/releases/latest),
  rename it to `dap.exe`, and put it in a folder that's on your PATH.
- **Or, if you have [Go](https://go.dev/dl/) installed:**
  ```
  go install github.com/AlmogBaku/debug-skill/cmd/dap@latest
  ```
  This puts `dap.exe` under `%USERPROFILE%\go\bin` — make sure that's on
  your PATH.

Per-language backends: Python (`pip install debugpy`) and Go (`go install
github.com/go-delve/delve/cmd/dlv@latest`) both work fine on Windows;
Node/TypeScript (js-debug) should auto-discover a VS Code/Cursor install
same as elsewhere. Rust/C/C++ (`lldb-dap`) has **no documented Windows
install path** in the plugin as of this writing — if you need that, use
WSL, or skip installing `dap` entirely and let step 4 fall back to manual
debugging.

---

## Install loop-engineering itself

```
/plugin marketplace add Vishalgoud3105/loop-engineering
/plugin install loop-engineering@loop-engineering
```

Requires Claude Code with plugin support. Same rule as above: restart or
reload your session afterward so `/loop-engineering-run`,
`/loop-engineering-scratch`, `/loop-engineering-addstep`, and
`/loop-engineering-swap` are discovered.

---

## Usage

### Run the loop

```
/loop-engineering-run
/loop-engineering-run path/to/spec.md
/loop-engineering-run path/to/spec.pdf
```

The argument is optional — a path to whatever spec/PRD/context doc defines
"done" for this project. If you don't pass one, loop-engineering looks for an
obvious candidate (README, PRD, `docs/*.pdf`) in the current project before
asking you for one.

You don't have to type the slash command — Claude also picks this skill up
from plain language, since that's part of what the skill tells it to watch
for. Saying any of these in chat works the same as typing the command:

- "run the loop"
- "loop check"
- "loop testing"
- "loop engineering"
- "check loop analysis"

**Disambiguation from the built-in `loop` skill:** Claude Code ships its own
unrelated `loop` skill for recurring/scheduled tasks (e.g. "check the deploy
every 5 minutes"). loop-engineering's skill description explicitly tells
Claude the difference — if your phrasing mentions an interval or "every N
minutes/hours," that routes to the built-in scheduler, not this plugin. This
skill always means one build-and-QA pass (which internally repeats itself
at step 10 until clean), never a recurring background task.

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

### Create a custom loop

Two separate commands, depending on what you want:

**`/loop-engineering-scratch`** — a loop built from scratch, unrelated
to code QA. Describe your own steps and it designs a workflow around them
instead of forcing the baseline's steps onto it:
```
/loop-engineering-scratch - a content-review loop: draft, fact-check,
tone-check, publish, repeat until no edits are needed
```
Name your workflow explicitly if you want a specific command
(`/loop-engineering-scratch - name: content-review, ...`) — if you
don't name it, it auto-names itself from the description instead of
asking. Can be run any number of times to build different loop workflows,
each independent, each under its own name. The only thing every generated
loop keeps from the baseline is the *shape*, not the steps: state a
concrete goal up front, then repeat until a full pass makes no changes and
that goal holds — which steps exist, what they check, what skills they
call is entirely up to the description.

**`/loop-engineering-addstep`** — modify the *existing* 11-step baseline:
add a step, remove/skip a step, or both in one call. This never edits the
plugin's own shipped `loop-engineering-run` — it always writes a separate
variant, so the original 11-step form stays exactly as installed:
```
/loop-engineering-addstep - skip security-review, add a load test step after QA
/loop-engineering-addstep - backend only, no ponytail-audit sweep
```
Same auto-naming as `scratch` if you don't name it explicitly — but
written to `.claude/skills/loop-engineering-run-<slug>/SKILL.md` (note the
extra `-run-`), so its command is `/loop-engineering-run-<slug>`. That
extra segment is deliberate: it marks the result as a *variant of the run
baseline*, distinct from `scratch`'s unrelated from-scratch loops
(`loop-engineering-<slug>`, no `-run-`).

**`/loop-engineering-swap`** — reorder steps instead of adding/removing
them, a companion to `addstep` for exactly that gap. Works on either the
baseline or a custom loop you already created:
```
/loop-engineering-swap - in loop-engineering-run, swap the debug and QA steps
/loop-engineering-swap - in loop-engineering-backend-only, swap steps 2 and 4
```
Important nuance in where it writes the result, since the two targets have
different ownership: swapping in the **baseline** never edits
`loop-engineering-run` itself — it writes a new variant to
`loop-engineering-run-<slug>`, same rule as `addstep`. Swapping in a
**custom loop you already created** edits that file directly, since it's
already a project file you own, not part of this plugin.

A note on syntax: the command name itself is always fixed
(`/loop-engineering-swap`) — a slash command can't have dynamic content
like step numbers baked into the command string. Which workflow and which
steps to swap are arguments you write after the command, same as every
other command in this plugin.

All three generator commands (`scratch`, `addstep`, `swap`) follow the
same underlying convention as the plugin's own commands and `ponytail`'s
(`ponytail-review`, `ponytail-audit`): the plugin name repeated as a
prefix, no colon, no `-custom-` infix.

**Important limitation (all three commands):** a generated/edited skill is
a plain project skill, not part of this plugin — plugins can't add
commands to themselves at runtime, and Claude Code only discovers skills
at session startup. So:

- `scratch` output is invoked as `/loop-engineering-<slug>`;
  `addstep`/`swap`-on-baseline output is invoked as
  `/loop-engineering-run-<slug>`; `swap`-on-an-existing-custom-loop keeps
  that loop's existing command name (it edited the file in place, it
  didn't create a new one).
- It won't show up (or won't reflect the change) until you restart or
  reload your Claude Code session — VSCode extension: `Ctrl+Shift+P` →
  **"Developer: Reload Window"**; standalone CLI: exit and restart
  `claude`. None of these commands can trigger this for you — a skill has
  no access to VSCode's or the CLI's controls, so all three end their
  output with an explicit callout telling you to do it.
- Anything these commands write lives in *your project*, not in this
  plugin — it won't follow you to other repos, and updating the plugin
  won't touch it.

You can create as many of these as you want, one per project, each named
whatever you asked for (or whatever it auto-named itself).

---

## Repo layout

```
.claude-plugin/marketplace.json              # marketplace listing (this repo)
plugins/loop-engineering/
  .claude-plugin/plugin.json                 # plugin manifest
  skills/
    loop-engineering-run/SKILL.md            # /loop-engineering-run  — the 11-step loop
    loop-engineering-scratch/SKILL.md    # /loop-engineering-scratch — new loop from scratch
    loop-engineering-addstep/SKILL.md        # /loop-engineering-addstep — add/remove a baseline step
    loop-engineering-swap/SKILL.md           # /loop-engineering-swap — reorder steps
```

Claude Code plugin commands are always namespaced `plugin-name:skill-name`
under the hood — there's no way to make a plugin skill respond to a bare
`/skill-name` directly. The workaround (same one `ponytail` uses for
`ponytail-review`/`ponytail-audit`): name the skill's own directory with the
plugin name repeated as a prefix (`loop-engineering-run`, not just `run`),
so it reads and types as a clean hyphenated command instead of a colon
pair. The plugin name and the marketplace/repo name both happen to be
`loop-engineering` here too, but those are independent identifiers — see
Install section.

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

**"No matching commands" / "Unknown command" when I type the skill name.**
Some installed skills aren't meant to be typed directly (chat box in the
VSCode extension, prompt in the standalone CLI) — they're invoked by
Claude via the Skill tool when relevant (that's how `debugging-code` works
here). Also, Claude Code only refreshes its command index at session
start, so a fresh install always needs a restart before autocomplete picks
it up — VSCode: `Ctrl+Shift+P` → **"Developer: Reload Window"**; CLI: exit
and restart `claude`.

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
all when using `/loop-engineering-run`.

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
