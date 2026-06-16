---
name: loopforge
description: Use when the user wants to start a large feature or "epic" as an autonomous multi-agent loop, asks to "run a loop", "spec this out", "set up loop engineering", or hands off a big task to work unattended. Interviews the user, fills the acceptance spec, plans tasks, and runs an autonomous TDD loop with human checkpoints only at the start and end.
---

# Loop Engineering

## Overview

**You are involved only at the START (intent) and END (verification). The middle
is an autonomous TDD loop run by role-separated agents.**

Loop Engineering is CI/CD discipline applied to AI agents. It is better than
manual prompting ONLY when all three hold:

1. The task is large with clear acceptance criteria.
2. There is an automated verification command (tests/types/lint) — the loop's
   source of truth.
3. The user accepts higher token cost for autonomy.

If any is false → tell the user to use plain prompting instead, and stop.

This skill drives the full flow: **interview → fill `loop/acceptance.md` →
plan `loop/tasks.json` → run the autonomous loop → hand back for human verify.**

## When to Use

- User says: "run a loop", "loop this", "spec this epic", "set up loop engineering".
- A large, multi-part feature that the user wants built mostly unattended.
- Acceptance criteria are writable up front and the repo can be verified.

**Do NOT use when:** the task is a one-file change, a quick fix, exploratory work
with no clear "done", or there is no way to automatically verify correctness.
Say so and recommend plain prompting.

## Step 1 — Gate check (before interviewing)

Confirm out loud, in one line each:
- Is this big enough to justify a loop? If not → stop, recommend prompting.
- Does a verification command exist (or can one be created now)? If none can
  exist → stop. **No verify command, no loop.**

## Step 2 — Structured interview (one topic at a time)

Ask the user the questions below. **Ask in small batches (use AskUserQuestion
where choices are discrete), wait for answers, do not assume.** Do not move on
until a topic is answered. If the user says "you decide", pick a sensible
default and state it.

**When the user pushes back on questions ("don't ask me 20 things", "just
start"), do NOT silently invent the spec.** Instead, batch the decisions into a
few AskUserQuestion calls with a recommended default marked on every option, so
they can answer "default" in seconds — and only surface the forks where a wrong
guess is expensive (lock-out, expiry/idempotency, over-limit behavior, reuse vs
greenfield). This honors their time without skipping confirmation.

1. **Feature** — In one paragraph, what should this do and why?
2. **Acceptance criteria** — How do we know it's done? Push for *verifiable*
   statements with concrete numbers (status codes, limits, thresholds, coverage),
   not "it should work". Each criterion will become a test.
3. **Non-goals** — What is explicitly out of scope? (Prevents scope creep.)
4. **Constraints** — Performance, security, compatibility, deadline/budget.
5. **Verification command** — The exact command that proves "done" and exits
   non-zero on failure (e.g. `npm test && npm run lint && npm run typecheck`).
   If it doesn't exist yet, agree on creating it as task T0.
6. **Scope/size** — Rough number of PRs / time. Used to size the plan.

After collecting answers, **play them back as a summary and get explicit
confirmation** before writing anything.

## Step 3 — Write `loop/acceptance.md`

Fill the template at `loop/acceptance.md` from the confirmed answers. Every
acceptance criterion must be a checkbox and verifiable. Include the verification
command. (If the `loop/` workspace doesn't exist, create it — see Quick Reference.)

## Step 4 — Plan into `loop/tasks.json`

Dispatch an **Architect + Researcher** pass (clean-context subagents) to read the
codebase and `acceptance.md`, write the design into `loop/discussion.md` (cap
debate at 2 rounds, every decision cites a fact), record key choices as ADRs in
`loop/decisions/`, then break the design into `loop/tasks.json`:

- Each task: `id`, `title`, `owner`, `depends_on`, `parallel`, `files`,
  `acceptance` (which criterion it satisfies), `status`.
- Mark independent tasks `parallel: true`; sequence the rest via `depends_on`.

Get the user's OK on the plan once. Then they stay out until Step 6.

## Step 5 — Run the autonomous loop (no human)

For each task, respecting `depends_on`, run strict TDD:

1. **Test Author** writes a FAILING test for the task's acceptance criterion.
2. **Implementer** writes the MINIMAL code to pass. Touches only its `files`.
3. Refactor while green.
4. Run the per-task gate (unit + lint + types).
5. Update `tasks.json` status + append to `loop/progress.md`.
6. On gate fail → **Debugger** fixes minimally, retry. After N retries, escalate
   to the human via a one-line notification and pause.

**Hard rules (non-negotiable):** never weaken or edit a test to force a pass; no
hardcoded secrets (env only); parameterize SQL; no swallowed errors; each task
ends GREEN or it is not done.

**Checkpoint after every phase:** run the FULL verification command, write the
result to `loop/progress.md`. Never run for hours with zero checkpoints.

Then run review in parallel (read-only): **Reviewer ‖ Security Auditor ‖ Tester**.
Blocking findings re-enter the loop; non-blocking are logged.

## Step 6 — Hand back for human verification

Stop and tell the user it's ready to verify. Give them the checklist: run the app
for real, confirm acceptance criteria are genuinely met, spot-check the diff for
intent/UX, confirm the full verification command passes clean. Only after THEIR
verification is it done → then ship via a PR they approve.

## Quick Reference

| Step | Owner | Output |
|---|---|---|
| 1 Gate check | You | go / no-go |
| 2 Interview | You ↔ user | confirmed answers |
| 3 Acceptance | You | `loop/acceptance.md` |
| 4 Plan | Architect+Researcher | `loop/discussion.md`, `tasks.json`, ADRs |
| 5 Loop | role agents | green code + `progress.md` checkpoints |
| 6 Verify | Human | sign-off → ship |

**If `loop/` is missing**, create it with: `acceptance.md`, `discussion.md`,
`tasks.json`, `progress.md`, `decisions/`. See the templates and filled examples
in this repo's `loop/` folder.

## Common Mistakes

| Mistake | Fix |
|---|---|
| Starting the loop without a verify command | Stop. Create the command first (task T0). |
| Accepting vague criteria ("should work") | Push for numbers; each criterion becomes a test. |
| Filling acceptance.md before confirmation | Play back the summary, get explicit OK first. |
| Agents emailing / polling each other | Coordinate via repo files + subagent dispatch. |
| Hours unattended with no checkpoints | Full verify + checkpoint after every phase. |
| Editing a test to make it pass | Forbidden. Fix the code, never the test. |
| Looping a small/exploratory task | Recommend plain prompting and stop. |

## Communication channels (never email)

Agents coordinate through version-controlled files (`loop/*`) and direct subagent
dispatch — the returned result IS the message. Notify the *human* via a push
webhook (Slack/Telegram), not by polling a mailbox.
