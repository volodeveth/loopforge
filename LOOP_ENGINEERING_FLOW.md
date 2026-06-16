# Loop Engineering — Optimal Flow

> A practical, battle-tested instruction for running an autonomous multi-agent
> "loop" on a project with Claude Code. This is the engineering-honest version
> of the methodology: spec-first planning + clean agent roles + automated
> verification. It deliberately drops the theatrical parts (email inboxes,
> agents "polling mail every 5 minutes") in favor of mechanisms that are
> faster, cheaper, and auditable.

---

## 0. Core idea (read this first)

**You are involved only at the START (intent) and the END (verification).
The middle is an autonomous loop run by a team of agents.**

Loop Engineering is NOT magic. It is CI/CD discipline applied to AI agents.
It is *better than manual prompting* only when ALL three are true:

1. The task is **large** and has **clear acceptance criteria**.
2. There is **automated verification** (tests, types, linters) — the loop's
   source of truth. Without this, agents loop blindly and drift.
3. You accept **higher token cost** in exchange for autonomy.

If any of those is false → use plain prompting instead. Do not run a loop for
small, exploratory, or undefined tasks.

```
INTENT  ──►  PLAN  ──►  [ AUTONOMOUS LOOP ]  ──►  VERIFY  ──►  SHIP
(human)     (agents)    (agents, no human)        (human)      (human)
   ▲                                                 │
   └──────────── only re-enter here on gate fail ────┘
```

---

## 1. When to use vs. when NOT to use

| Use the loop | Do NOT use the loop |
|---|---|
| Big feature / "epic" with sub-tasks | One-file change, typo, quick fix |
| Acceptance criteria are writable up front | You don't yet know what you want (exploration) |
| Tests/types/lint exist or can be added | No way to automatically verify correctness |
| Code must be tight, verified, multi-part | Throwaway script / prototype spike |
| You'll be away for a while | You need it cheap and fast |

**Rule of thumb:** if you can't write the acceptance criteria and a test for
"done," you are not ready to loop. Stop and clarify first.

---

## 2. The agent roles (clean context per role)

Each role runs as a **separate subagent with its own clean context**. Do not
make one agent do everything — separate contexts produce cleaner decisions.

| Role | Model tier | Responsibility |
|---|---|---|
| **Orchestrator / CTO** | Opus | Owns the loop. Plans, delegates, integrates results, decides gate pass/fail. The only role that talks to the human. |
| **Architect** | Opus | Designs architecture & interfaces before code. Writes the design doc + ADRs. |
| **Researcher** | Sonnet | Maps the codebase, reads docs, gathers facts. Read-only. |
| **Implementer(s)** | Sonnet | Writes code strictly to the plan. Touches only its assigned files. Can run in parallel for independent tasks. |
| **Test Author** | Sonnet | Writes the failing test FIRST (encodes acceptance criteria). Never writes production code. |
| **Reviewer** | Opus | Deep code review: bugs, antipatterns, types, smells. Read-only, reports findings. |
| **Security Auditor** | Opus | Vulnerabilities, secrets, injection, CORS, authz. Read-only. |
| **Debugger** | Sonnet | Reads logs/stack traces, fixes with minimal change. |
| **Tester** | Haiku | Runs suites, reports coverage. Routine. |

> The orchestrator does NOT write feature code itself. It delegates and
> integrates. This keeps its context clean for decisions.

---

## 3. Where agents communicate (NOT email)

Email is the worst channel: slow, polling-based, brittle, token-wasteful.
Use these instead, in priority order:

1. **Shared files in the repo** ⭐ — the "board." Agents leave durable traces
   for each other in version-controlled markdown/JSON:
   - `loop/discussion.md` — design debate & decisions
   - `loop/tasks.json` — task list, owners, status, dependencies
   - `loop/decisions/adr-*.md` — architecture decision records
   - `loop/progress.md` — running log of what's done / what's next
   This is the primary mechanism. Git history = full audit trail.

2. **Subagent dispatch (Task tool)** ⭐ — real orchestration. The orchestrator
   *calls* a subagent, passes context in the prompt, receives the result back.
   That return value IS the message. No mailbox, no polling.

3. **GitHub Issues / PR comments** — for human-visible audit trail and review
   threads. Good when you want to re-enter the loop easily.

4. **Notification webhook (Slack / Telegram / Discord)** — push events TO YOU
   ("Phase 2 done", "gate failed"). This replaces "check email every 5 min."
   It is push, not poll.

5. **Message queue / MCP shared-state server** — only if agents become
   separate long-lived processes. Overkill for a single Claude Code session.
   Don't build this until file-based coordination actually breaks down.

> Default to **files + Task subagents + a notification webhook**. That covers
> ~90% of the value with zero standing infrastructure.

---

## 4. The flow, phase by phase

### Phase 0 — Setup (once per project)

Create the loop workspace and the source-of-truth scaffolding:

```
loop/
  discussion.md        # design debate
  tasks.json           # the plan: tasks, owners, deps, status
  decisions/           # ADRs
  progress.md          # running status log
  acceptance.md        # the definition of done (human-authored)
```

Ensure the project has a **verification command** that exits non-zero on
failure. Examples:
- JS/TS: `npm test && npm run lint && npm run typecheck`
- Python: `pytest && ruff check && mypy .`

If no such command exists, **create it before looping.** No verify command =
no loop.

### Phase 1 — INTENT (human, ~5–15 min)

You describe the outcome, not the steps. Write to `loop/acceptance.md`:
- What the feature does (1 paragraph).
- **Acceptance criteria** as a checklist (these become tests).
- Explicit non-goals / out of scope.
- Constraints (perf, security, compatibility, deadlines).

Dictating by voice (e.g. WhisperFlow) is fine — but the OUTPUT must be written
criteria, not a vibe.

### Phase 2 — PLAN (agents, you don't weigh in)

1. **Architect + Researcher** read the codebase and `acceptance.md`, then
   write the design into `loop/discussion.md`: chosen approach, interfaces,
   data model, sequence of operations, risks.
2. They record key choices as ADRs in `loop/decisions/`.
3. **Orchestrator** breaks the design into `tasks.json`:
   ```json
   [
     { "id": "T1", "title": "...", "owner": "implementer-A",
       "depends_on": [], "files": ["src/x.ts"], "status": "todo",
       "acceptance": "criterion this task satisfies" }
   ]
   ```
   Mark which tasks are **independent** (parallelizable) vs. **sequential**.

> ⚠️ Anti-pattern guard: a "discussion" between agents can manufacture false
> consensus — two LLMs confidently agreeing on a wrong design. Cap the debate
> (e.g. 2 rounds max), and require every decision to cite a concrete fact from
> the code or acceptance criteria, not just opinion.

### Phase 3 — AUTONOMOUS LOOP (agents, no human)

For each task (parallelize independent ones), run **strict TDD**:

```
for each task in tasks.json (respecting depends_on):
    1. Test Author writes a FAILING test for the task's acceptance criterion.
    2. Implementer writes the MINIMAL code to make it pass.
    3. Implementer refactors while staying green.
    4. Run the per-task gate: unit + lint + types.
    5. Update tasks.json status + append to progress.md.
    6. On gate fail → Debugger fixes minimally, re-run. Max N retries,
       then escalate to human via webhook.
```

**Hard rules inside the loop (non-negotiable):**
- Never weaken or edit a test to force a pass.
- Each implementer touches only its assigned files (avoid merge chaos).
- No `any` without a comment explaining why. No `console.log` in prod code.
- No hardcoded secrets — env vars only. Parameterize all SQL.
- No empty catch blocks / swallowed errors.
- Every task ends GREEN or it is not "done."

**Checkpointing (critical):** do NOT run for hours with zero checkpoints. The
longer the unattended stretch, the more expensive a late-discovered error.
After each **phase** (group of tasks), run the FULL verification command and
write a checkpoint to `progress.md`. A loop with checkpoints beats a long
blind run every time.

### Phase 4 — REVIEW (agents, parallel)

Once all tasks are green, run these **in parallel** (independent, read-only):
- **Reviewer** — bugs, antipatterns, types, smells.
- **Security Auditor** — secrets, injection, XSS, authz, CORS.
- **Tester** — full suite + coverage report.

Findings go to `loop/progress.md`. Orchestrator triages: blocking findings
re-enter Phase 3; non-blocking are logged.

### Phase 5 — VERIFY (human — your real job)

Agents prove it works *to spec*. You verify it works *the way you actually
wanted*. Checklist:
- Run the app for real (not just tests). Exercise the feature manually.
- Confirm acceptance criteria in `acceptance.md` are genuinely met.
- Spot-check the diff for things tests can't catch (UX, naming, intent).
- Confirm the full verification command passes on a clean run.

Only after YOUR verification is the loop "done."

### Phase 6 — SHIP (human-gated)

- Conventional Commits: `feat: ...`, `fix: ...`, `chore: ...`.
- Branch → PR (never commit straight to `main`).
- Merge to `main` → auto-deploy (e.g. Vercel) only after your approval.
- Confirm risky actions (force-push, drop table, deletes) explicitly.

---

## 5. Token & cost discipline

The loop uses **far more tokens** than prompting. Control it:

- Reserve loops for big, consequential features. Small things → plain prompts.
- Cap design debate rounds (2 is usually enough).
- Use the cheapest model that fits each role (Haiku for routine, Sonnet for
  code, Opus only for planning/review/security).
- Read only the parts of files you need, not whole files.
- Checkpoint often so a failure costs one phase, not the whole run.

---

## 6. Anti-patterns (the "theater" to avoid)

| Anti-pattern | Why it's bad | Do instead |
|---|---|---|
| Email inboxes between agents, polling every 5 min | Slow, brittle, token-wasteful, no benefit over a shared file | Shared files + Task dispatch + webhook to YOU |
| Long multi-round agent "debates" | Manufactures false consensus, burns tokens | Cap rounds; require fact citations; pick and move on |
| Hours of unattended work, no checkpoints | Late-discovered errors are the most expensive | Full verify + checkpoint after each phase |
| Looping without tests | Nothing for the loop to verify against → drift | No verify command, no loop |
| One mega-agent doing everything | Context bloat, muddy decisions | Clean context per role |
| Letting agents edit tests to pass | Hides real failures | Tests are immutable during implementation |

---

## 7. Minimal checklist (print this)

```
[ ] Acceptance criteria written (acceptance.md)
[ ] Verification command exists and fails loudly
[ ] tasks.json: owners, deps, parallel/sequential marked
[ ] Design debate capped + ADRs recorded
[ ] TDD per task: failing test → minimal code → green → gate
[ ] Checkpoint + full verify after each phase
[ ] Parallel review: Reviewer ‖ Security ‖ Tester
[ ] HUMAN verify: run the app, check intent, not just tests
[ ] Conventional commit → branch → PR → approve → ship
```

---

## 8. One-paragraph summary

Define the outcome and the acceptance criteria. Let a team of role-separated
agents plan it (architect + researcher), break it into tasks, then run an
autonomous TDD loop where each task goes failing-test → minimal-code → green →
gate. Agents coordinate through version-controlled files and direct subagent
dispatch — never email. Checkpoint after every phase. Parallel review for bugs
and security. Then YOU verify it does what you actually wanted, and ship via a
PR you approve. The loop is only worth it when the task is big and there's an
automated source of truth; otherwise just prompt.
