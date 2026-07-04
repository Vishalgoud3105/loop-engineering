# loop-check

A Claude Code plugin marketplace containing the `loop-check` plugin: an
end-to-end "loop engineering" QA workflow — set a goal from your spec doc,
build with `ponytail` (YAGNI), then code-review, debug, QA, verify against
the goal, check for over-engineering, security-review, and a production-grade
check. Repeats until a full pass makes zero changes and the goal is met.

## Install

```
/plugin marketplace add Vishalgoud3105/loop-check
/plugin install loop-check@loop-check
```

## Use

```
/loop-check:run [path-to-spec-doc]
```

Runs the full 11-step loop (0-10) against the current project.

```
/loop-check:create - <description of a custom variant>
```

Writes a new project-level skill (`.claude/skills/loop-check-custom-<name>/SKILL.md`)
with a modified version of the loop. Available as `/loop-check-custom-<name>`
after restarting/reloading the session (skills load at session start).

## Steps `loop-check:run` invokes

Depends on these skills being available in your session: `ponytail`,
`code-review`, `debugging-code` (from the `debug-skill` community plugin),
`verify`, `ponytail-review`, `ponytail-audit`, `security-review`.
