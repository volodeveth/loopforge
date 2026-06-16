# LoopForge

> Forge large features in an **autonomous multi-agent loop**.

You stay involved only at the **start** (intent) and the **end** (verification).
A team of role-separated agents plans the work, runs a strict TDD loop, and
checkpoints itself along the way.

This is the engineering-honest version of "loop engineering": **spec-first
planning + clean agent roles + automated verification.** It drops the theatrical
parts (email inboxes, agents "polling mail every 5 minutes") in favor of
mechanisms that are faster, cheaper, and auditable.

> Loop Engineering is not magic. It is CI/CD discipline applied to AI agents.
> Use it only when the task is large, the acceptance criteria are writable, and
> the repo can be automatically verified. Otherwise, just prompt.

## What's in here

| Path | What it is |
|---|---|
| `skills/loopforge/SKILL.md` | The skill: interviews you, fills the spec, plans, runs the loop. |
| `LOOP_ENGINEERING_FLOW.md` | Full methodology, phase by phase (English). |
| `LOOP_ENGINEERING_FLOW_UA.md` | Same methodology in Ukrainian. |
| `loop/` | The workspace templates (the shared "board" agents coordinate through). |
| `loop/examples/` | Filled-in `acceptance.md` + `tasks.json` on a sample feature. |

## Install (Claude Code plugin)

This repo is a Claude Code plugin. Once published to GitHub, anyone can use it:

```
/plugin marketplace add volodeveth/loopforge
/plugin install loopforge
```

Or drop the skill into your personal skills directory:

```
cp -r skills/loopforge ~/.claude/skills/
```

## Use it

In Claude Code, just say:

```
run a loop for <your big feature>
```

or invoke the skill directly. It will:

1. **Gate-check** that a loop is even worth it (and that a verify command exists).
2. **Interview you** step by step — feature, acceptance criteria, non-goals,
   constraints, verification command, scope.
3. **Write `loop/acceptance.md`** from your confirmed answers.
4. **Plan `loop/tasks.json`** (architect + researcher pass, ADRs, task graph).
5. **Run the autonomous TDD loop** — failing test → minimal code → green → gate,
   checkpointing after every phase.
6. **Hand back to you** to verify it does what you actually wanted, then ship.

## The one rule that makes or breaks it

**No verification command, no loop.** Tests / types / linters are the loop's
source of truth. Without them, agents loop blindly and drift. If your project
can't be automatically verified, set that up first — or use plain prompting.

## Author

Built by [volodeveth](https://volodeveth.vercel.app/) — portfolio & more projects.

## License

MIT
