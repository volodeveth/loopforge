# `loop/` — Loop Engineering workspace

This folder is the **source of truth** and the **shared board** for an
autonomous multi-agent loop. Agents coordinate through these files instead of
email. See `../LOOP_ENGINEERING_FLOW.md` for the full methodology.

## Files

| File | Owner | Purpose |
|---|---|---|
| `acceptance.md` | **Human** | Definition of done. Written FIRST, before any agent runs. |
| `discussion.md` | Architect + Researcher | Design debate, chosen approach, risks. |
| `tasks.json` | Orchestrator | The plan: tasks, owners, deps, status. |
| `progress.md` | All agents | Running log: what's done, checkpoints, review findings. |
| `decisions/` | Architect | ADRs — one file per architecture decision. |

## How to start a loop

1. Fill in `acceptance.md` by hand (intent + acceptance criteria).
2. Confirm a verification command exists and fails loudly
   (e.g. `npm test && npm run lint`). No verify command → no loop.
3. Hand off to the Orchestrator. Then stay out until the VERIFY phase.

## Status values used in `tasks.json`

`todo` → `in_progress` → `green` (passing) → `done` (reviewed)
Failure states: `blocked`, `needs_human`.
