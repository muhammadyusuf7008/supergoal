---
name: superplan
description: Plan and autonomously build a software task end-to-end. Triggered by `/superplan`, "plan and ship X", "supercharged plan", "autonomous build", "plan it out and don't stop until it's done", "I don't want to babysit this", or any non-trivial feature/refactor/redesign the user wants driven to completion. Strongly prefer over a plain plan when the user signals "every aspect", "fully", "perfectly", "until done", or wants depth + autonomous follow-through. Recons the codebase, applies preloaded memory, researches best practices with whatever tools are available, decomposes into the right number of phases, gets one confirmation, then dispatches a single `/goal` that drives the entire chain to completion with built-in retry, fix-goal recovery, and per-phase memory writeback. Works on Claude Code and Codex.
argument-hint: <describe what you want built, fixed, or shipped>
---

# Superplan

You are running the Superplan workflow. The user's task is:

$ARGUMENTS

Your job: **plan deeply, then auto-execute under a single `/goal`** until the task is verifiably complete across every phase.

## What "every aspect is perfect" means here

The user's bar is high. Translate it into measurable criteria, not vibes:

- **Functional** — the feature works for the golden path and the obvious edge cases
- **Engineering** — build, typecheck, lint, tests all pass; no new warnings
- **Polish** — UX/copy, error states, empty states, loading states are handled
- **Hardening** — security review, input validation, no obvious regressions
- **Verification** — every phase produces transcript evidence the evaluator can see

If a phase can't be measured, it isn't a phase. Rewrite it until it can.

## How this skill works (one-shot summary)

0. **Available context** — preload memory; detect available tools (Context7, WebSearch, MCPs, skills); resume any in-progress Superplan state
1. **Intake** — restate, classify, ask **0–2 questions only for true gaps** memory + prompt don't already answer
2. **Recon** — parallel codebase + environment scan
3. **Deep think** — research best practices with whatever tools exist (optional, not required); list top-3 risks + dependencies
4. **Decompose** — derive phase count from the task itself; no fixed cap
5. **Write phase specs** — one work-spec file per phase under `.superplan/phases/phase-N.md` (any length, no char budget)
6. **Plan review** — show summary + concrete revision menu; wait for explicit go/no-go
7. **Dispatch one `/goal`** with a short end-state condition; the agent inside that session executes phases sequentially, with built-in retry, fix-goal recovery, and per-phase memory writeback, until the condition holds

Two human gates only: **clarifying questions for true gaps (Stage 1)** and **plan review (Stage 6)**. Everything else runs autonomously.

### Why one `/goal`, not a chain

`/goal` in both Claude Code and Codex takes a **short end-state condition**, not a long task body. A fast evaluator checks the condition against the transcript after each turn and auto-continues until it holds. Superplan v3 leverages this directly: one `/goal` covers the whole run; phase work lives in files the agent reads from disk; the condition is "all phases done, `SUPERPLAN_RUN_COMPLETE` printed." No char budget, no inter-session chain dispatch, no fragility.

## Locate the skill directory

```bash
SUPERPLAN_DIR=$(dirname "$(ls -1 \
  "$HOME/.claude/skills/superplan/SKILL.md" \
  "$PWD/.claude/skills/superplan/SKILL.md" \
  2>/dev/null | head -n1)")
export SUPERPLAN_DIR
export SUPERPLAN_ROOT="${SUPERPLAN_ROOT:-.superplan}"
mkdir -p "$SUPERPLAN_ROOT/goals"
echo "SUPERPLAN_DIR=$SUPERPLAN_DIR"
echo "SUPERPLAN_ROOT=$SUPERPLAN_ROOT"
```

All artifacts live under `$SUPERPLAN_ROOT`. Skill assets (scripts, references, templates) live under `$SUPERPLAN_DIR`.

---

## Stage 0 — Available context (memory + tools)

Before doing anything else, sense what's available this session. This is what makes the run frictionless — if memory already knows the user's preferences, don't ask; if a tool isn't available, don't try to call it.

### Memory preload

```bash
# Detect a memory directory. Common locations:
MEM_DIR=""
for cand in \
  "$HOME/.claude/projects/-Users-$(whoami)/memory" \
  "$HOME/.claude/memory" \
  "$PWD/.claude/memory" \
  "$SUPERPLAN_ROOT/memory"; do
  [[ -d "$cand" ]] && MEM_DIR="$cand" && break
done
echo "MEM_DIR=$MEM_DIR"

if [[ -n "$MEM_DIR" && -f "$MEM_DIR/MEMORY.md" ]]; then
  echo "--- MEMORY INDEX ---"
  cat "$MEM_DIR/MEMORY.md"
fi
```

Read the index. Then **selectively** read individual memory files that look relevant to the task (feedback memories about the stack/domain, user role memories, related project memories). Don't dump them all into context — pull what matters.

Capture applicable memory hits in `$SUPERPLAN_ROOT/applied-memories.md` (one line per memory: name, why-applicable, what-it-changes). Surface them in Stage 1 as "Applied from memory: …" so the user can see what's being inherited and correct anything stale.

### Tool discovery

Tools differ between sessions and hosts (Claude Code vs Codex, different MCP server sets). Detect, don't assume:

- **Context7** — available if `mcp__claude_ai_Context7__resolve-library-id` or similar is in the tool list. If absent, skip it; rely on training-cutoff knowledge + WebSearch if that's present.
- **WebSearch / WebFetch** — available if listed. If neither, skip web research.
- **Project skills** — check the available-skills list for domain-relevant skills (e.g. `mobile-ios-design`, `clerk-auth`, `expo-dev-client`) and note them in `$SUPERPLAN_ROOT/applied-skills.md` to invoke from inside phase goals if relevant.
- **Prior Superplan state** — if `$SUPERPLAN_ROOT/STATE.md` exists from a previous run, read it; resume rather than restart.

Write detected tools to `$SUPERPLAN_ROOT/tools.md`. Stage 3 and the phase goals reference this file when deciding what to invoke.

### Resume detection

If `STATE.md` exists and shows `Status: IN_PROGRESS` with a phase pending, **do not re-plan**. Print a one-line "Resuming Superplan from phase N" and jump straight to Stage 6 (plan review) with the existing artifacts, or directly to Stage 7 (dispatch) if the user confirms resume.

---

## Stage 1 — Intake & clarifying questions

Echo the task back in **one sentence**. Then classify it (tags can combine):

| Tag | Trigger |
|---|---|
| `greenfield` | Request implies a new project; cwd has no `.git/` or empty tree |
| `brownfield` | Change in an existing repo |
| `bugfix` | Mentions "bug", "broken", "fails", "regression" |
| `refactor` | Mentions "refactor", "clean up", "restructure" |
| `ui` | Mentions "design", "polish", "UI", "UX", "responsive", "redesign" |

**Ask 0–2 clarifying questions** — only for true gaps that memory and the prompt don't already answer. Frictionless is the goal: if memory says the user prefers stack X and prompt says use X, don't ask "which stack?". Most well-described tasks should ask **zero questions**.

Process:
1. List the structural ambiguities in the task (forks, scope cuts, integration anchors).
2. Cross out any answered by memory (`applied-memories.md`) or by the prompt itself.
3. If anything remains and would meaningfully change the phase shape, ask. Otherwise, proceed.

What counts as **a true gap worth asking** (after memory + prompt resolution):
- Architecture forks not covered: "iOS only or also web?", "monorepo or standalone?"
- Integration anchors not covered: "which auth provider?", "which DB?"
- Scope cut-lines: "MVP this week or full feature?"
- Primary surface when ambiguous: "main use case A, B, or both?"

What is **never** a question (record as assumption in ROADMAP.md, surface in Stage 6):
- Naming, file paths, directory layout
- Library minor versions and styling choices
- Copy wording, color palettes
- Test framework picks when the stack has an obvious default

**Cap: 2 questions, one batch via `AskUserQuestion`.** Lead with "Applied from memory: …" so the user sees what's being inherited. If zero questions are needed, say "No clarifying questions — proceeding from prompt + memory." and move on.

---

## Stage 2 — Recon (parallel)

Run recon scripts in parallel. They populate context files under `$SUPERPLAN_ROOT/`.

### Brownfield path

```bash
bash "$SUPERPLAN_DIR/scripts/detect-stack.sh"   > "$SUPERPLAN_ROOT/context.md"
bash "$SUPERPLAN_DIR/scripts/summarize-repo.sh" > "$SUPERPLAN_ROOT/repo-map.md"
```

### Greenfield path

```bash
bash "$SUPERPLAN_DIR/scripts/detect-env.sh" > "$SUPERPLAN_ROOT/context.md"
```

Read the outputs. Then print a **5-line summary** to the user: stack, package manager, build/test/lint commands, notable modules (if any), risky areas. This is what tells them you've actually understood their codebase before planning.

---

## Stage 3 — Deep think

This is the difference between a generic plan and a Superplan. Spend real cycles here — but use only what's available.

**Required regardless of tools:**
- Identify the **top 3 risks**: what's most likely to go wrong, what's hardest to undo, what's easy to miss until shipped.
- Identify **non-obvious dependencies**: things that have to happen in a specific order or block other work.
- Apply memory hits from `$SUPERPLAN_ROOT/applied-memories.md` — bake them into goals, constraints, or risk mitigations.

**Optional, use if available** (check `$SUPERPLAN_ROOT/tools.md`):
- **Context7** — if available, query current docs for any third-party SDK touched. Don't plan against stale APIs. If unavailable, lean on training-cutoff knowledge and call it out as an assumption ("planned against my training-cutoff understanding of Expo SDK — verify in phase 1").
- **WebSearch** — if available, look up current consensus on patterns you're unsure about (auth flows, payment idempotency, accessibility standards). If unavailable, skip.
- **Project skills** — if relevant skills are listed in `$SUPERPLAN_ROOT/applied-skills.md` (e.g. `clerk-auth`, `mobile-ios-design`), note them in THINKING.md as "consult `<skill>` skill during phase N" so the executor invokes them at the right moment.

**Write `$SUPERPLAN_ROOT/THINKING.md`** with sections: Goals, Constraints, Risks, Dependencies, Open Questions (already-assumed), Memory hits applied, Tools/skills relied on, Best Practices Applied. Keep it tight — 1–2 pages. This is the substrate the roadmap derives from.

See `references/planning-depth.md` for the bar to clear here.

---

## Stage 4 — Decompose into phases

Break the work into **as many phases as the task actually needs** — no fixed count, no upper or lower cap. The right number falls out of the work itself: how many independently verifiable units exist between empty repo (or current state) and "done perfectly." A trivial change might need 2 phases; a typical feature 4–6; a full-stack greenfield app 8–12; a major migration 15+. Read `references/phase-design.md` for how to slice well — the short version:

- Each phase delivers something **verifiable on its own** (it builds, it passes its own tests, you could ship it as a partial increment)
- Phases have **explicit dependencies** (phase 3 depends on 1 and 2)
- The **last phase is always a "Polish & Harden" phase** covering edge cases, error states, security, accessibility, copy, perf — this is how "every aspect is perfect" gets enforced
- For UI work, include a dedicated **visual polish** phase with screenshot/visual evidence requirements
- For brownfield, include an early **safety net** phase if test coverage is thin (add characterization tests before changing behavior)

Each phase has:
- **Name** (5 words max, action-first: "Build auth foundation")
- **Why** (1 sentence)
- **Deliverables** (concrete files/features that will exist when done)
- **Acceptance criteria** (5–10 measurable items)
- **Mandatory commands** (build, typecheck, lint, test that must pass)
- **Evidence required** (what the agent must print into the transcript to prove completion)
- **Dependencies** (which prior phases must be done)

---

## Stage 5 — Write the roadmap and phase specs

Three files, all under `$SUPERPLAN_ROOT/`:

1. **`ROADMAP.md`** — the plan (template at `$SUPERPLAN_DIR/templates/ROADMAP.md`).
2. **`STATE.md`** — live progress file the executor updates per phase (template at `$SUPERPLAN_DIR/templates/STATE.md`).
3. **`phases/phase-N.md`** — one work-spec file per phase (template at `$SUPERPLAN_DIR/templates/phase-goal.txt`, renamed conceptually to "phase spec"). **Any length** — these are read from disk by the executor, not passed to `/goal`, so no char budget.

Each phase spec must include these markers so the agent and evaluator both have stable anchors:

```
SUPERPLAN_PHASE_START
Phase: <N> of <total> — <name>
Task: <one-line>
Mandatory commands: <list>
Acceptance criteria: <count>
Evidence required: <list>
Depends on phases: <list or "none">

[... full work description, acceptance criteria, evidence requirements ...]

[Agent will print SUPERPLAN_PHASE_VERIFY and SUPERPLAN_PHASE_DONE here during execution]
```

Validate each spec with `bash $SUPERPLAN_DIR/scripts/validate-phase.sh .superplan/phases/phase-N.md` — it confirms the required markers exist. No char budget.

---

## Stage 6 — Plan review & confirmation (hard gate)

Before any `/goal` is dispatched, show the user the full plan and **ask for explicit confirmation**. The chain runs unsupervised once it starts, so this is the last cheap moment to correct course. Skipping this step is a bug.

Print a scannable summary in this exact shape:

```
✓ Plan ready for review. <N> phases.

Applied from memory:
  - <memory hit 1>
  - <memory hit 2>
  (or: "none — clean run")

Phases:
  1. <name> — <one-line deliverable>
  2. <name> — <one-line deliverable>
  ...
  N. Polish & Harden — every aspect verified

Stack: <stack> · pkg: <pm> · build/test/lint: <commands>

Key assumptions (correct any that are wrong):
  - <assumption 1>
  - <assumption 2>
  - <assumption 3>

Top risks & mitigations:
  1. <risk> → <mitigation>
  2. <risk> → <mitigation>
  3. <risk> → <mitigation>

Artifacts:
  Roadmap: .superplan/ROADMAP.md
  Progress: .superplan/STATE.md (auto-updates)
  Phase specs: .superplan/phases/phase-1..N.md

Once you confirm, I'll dispatch one /goal that runs all phases
through to completion, with auto-retry and fix-goal recovery.
```

Then call `AskUserQuestion` with one question, header "Start chain?", offering **concrete revision modes** (not a vague "revise plan"):

- **Start now** — dispatch phase 1, run the chain unsupervised
- **Adjust an assumption** — pick one to change (will re-show plan)
- **Tweak a phase** — change criteria, scope, or commands for a specific phase
- **Restructure phases** — merge, split, add, or remove a phase

Keep options at 4 max. If the user picks any revision option, follow up with a second `AskUserQuestion` to pin down exactly what (e.g., "Which assumption?" with the assumptions listed). Apply the change, update ROADMAP/THINKING/STATE and the affected phase specs, re-run `validate-phase.sh` on each touched spec, then re-show the Stage 6 summary and ask again. Loop until "Start now" or user aborts.

**Wait for the answer.** Do not dispatch `/goal` until the user picks "Start now". Never assume confirmation; never start the chain on silence.

---

## Stage 7 — Dispatch one `/goal` and let it run

Only after explicit "Start now" in Stage 6:

1. Update `STATE.md`: `Status: IN_PROGRESS`, `Current phase: 1`.
2. Output the literal `/goal` command **on its own line, as your final action**. The condition is short and measurable from the transcript alone:

```
/goal "Execute all phases of .superplan/ROADMAP.md sequentially. Read .superplan/phases/phase-N.md for each phase; do the work; run mandatory commands; print SUPERPLAN_PHASE_VERIFY then SUPERPLAN_PHASE_DONE for each phase; follow the failure-recovery protocol in .superplan/PROTOCOL.md if any criterion fails; on the final phase, print SUPERPLAN_RUN_COMPLETE. Done when SUPERPLAN_RUN_COMPLETE appears in the transcript with one SUPERPLAN_PHASE_DONE block per phase preceding it and no FAILURE_HANDOFF in this run."
```

That is the entire dispatch. No chain, no quoted-content fragility, no char budget. Both Claude Code and Codex `/goal` engines auto-continue under this single objective until the end-state condition holds.

### Write the protocol file once

Before dispatching, write `$SUPERPLAN_ROOT/PROTOCOL.md` from `$SUPERPLAN_DIR/templates/PROTOCOL.md` (single file containing the phase execution loop, failure recovery, memory writeback, mid-run interruption rules — see next sections). The agent reads it once at the start of the `/goal` session and follows it throughout.

---

## Phase execution loop (inside the single `/goal` session)

The agent's loop, repeated until `SUPERPLAN_RUN_COMPLETE`:

1. Read `STATE.md` → find current phase N.
2. Read `.superplan/phases/phase-N.md` → full work spec.
3. Print `SUPERPLAN_PHASE_START` block with values from the spec.
4. Do the work; run mandatory commands; surface evidence into the transcript.
5. Print `SUPERPLAN_PHASE_VERIFY` block (every criterion `pass|fail` + engineering checks).
6. **Memory writeback check** — anything non-obvious learned? If yes, write a memory file under the detected MEM_DIR; print `MEMORY_SAVED: <name>` (or `MEMORY_SAVED: none`).
7. Print `SUPERPLAN_PHASE_DONE`, update `STATE.md` (mark phase N complete, set Current phase = N+1, append events line).
8. **User-interrupt check** — if a new user message has arrived since the last turn, pause and address it before continuing.
9. If N < total: loop to step 1 for phase N+1.
10. If N == total: print `SUPERPLAN_RUN_COMPLETE` with a 5-line summary. The `/goal` condition is now satisfied and clears.

### Failure recovery (3-strike, built into the protocol)

**First failure of any criterion:**
1. Print `FAILURE_PROBE` (what failed, what tried, root-cause hypothesis).
2. Append probe to `STATE.md` failure log.
3. **Auto-retry the same phase once** with the probe injected as feedback. Do not advance.

**Second failure (auto-retry also failed):**
1. Print `FAILURE_ESCALATE`.
2. Write a focused **fix spec** at `.superplan/phases/phase-N.fix.md` (targets only the failing criterion, no scope creep).
3. Execute the fix spec inline (same agent, same `/goal` — no new dispatch). On success, re-run the original phase's VERIFY block; on pass, advance to N+1.

**Third failure (fix spec also failed):**
1. Print `FAILURE_HANDOFF` with: failing criterion, full probe history, three things tried, suggested next move.
2. Update `STATE.md`: `Status: BLOCKED`. The user takes the wheel.
3. The `/goal` condition will not be satisfied; the host's evaluator will keep evaluating but the agent should stop attempting and surface the handoff clearly.

This recovers from flaky envs, simple typos, and missed deps automatically. Only real blockers escalate.

### Mid-run interruption

If the user sends any message during the `/goal` run, the agent pauses at the next phase boundary, addresses the message, and asks before resuming. Phase boundaries are after `SUPERPLAN_PHASE_DONE` and before reading the next phase spec.

---

## Memory writeback rules (referenced by PROTOCOL.md)

Memory is load-bearing. Future runs start smarter because past runs wrote down what they learned. The phase execution loop's step 6 references these rules.

**At each phase boundary**, ask: "Did this phase surface anything a future Superplan run on a similar task would benefit from knowing?"

Worth saving:
- A library API quirk that wasn't in the docs
- A user preference confirmed during this run ("user accepted dark-only UI without pushback")
- A project-level fact ("auth lives in `lib/auth/` not `app/api/auth/`")
- A failure pattern + fix ("X always fails on first build; second build works")

Write the memory file under the detected MEM_DIR using the standard `name` / `description` / `metadata.type` frontmatter. Link it from `MEMORY.md`. Print `MEMORY_SAVED: <name>` to the transcript. If nothing non-obvious this phase: print `MEMORY_SAVED: none`.

**At the final phase**, always write a `project_<slug>.md` memory pointing at the new/changed project (location, stack, status, ROADMAP link). Guarantees future Superplan runs on the same project start from the latest state.

**Never save:** secrets, transient task details, ephemeral state. Bar is "useful to a future run." When in doubt, skip.

---

## Operating principles (read every run)

- **One `/goal`, short condition.** `/goal` takes an end-state, not a task body. Long content lives in files the agent reads from disk. This is the natural shape on both Claude Code and Codex.
- **Frictionless is the goal.** Memory + prompt + recon should answer most questions. Zero clarifying questions on well-described tasks is a win.
- **Adapt to available tools.** Detect what's there (Context7, WebSearch, MCPs, skills). Use what's available; degrade gracefully without it. Never hard-require a tool that might not be present.
- **Memory is load-bearing.** Preload at Stage 0, surface as "Applied from memory: …" in Stage 1, write back at every phase boundary.
- **"Perfect" is not a stopping condition — criteria are.** Translate every "perfect" into observable, falsifiable criteria.
- **Two human gates, no more.** Clarifying gaps (Stage 1, often zero) and plan review (Stage 6). Between and after, autonomous.
- **The loop self-heals.** Auto-retry once, then write a fix spec and execute inline, then escalate. Don't stop on first failure.
- **The evaluator only sees the transcript.** Phase specs require the agent to surface their contract — START, commands, evidence, VERIFY, DONE — into the conversation, not just point at files.
- **Each phase is independently shippable** in spirit. If phase 3 can't build/test on its own, the slicing is wrong.
- **The Polish & Harden phase is mandatory.** It's how "every aspect is perfect" gets enforced.

---

## When to deviate from the workflow

- **Very small task** (< 1 hour of work, single file): tell the user this doesn't need Superplan, suggest just doing it. Don't force the machinery.
- **The user pushes back on a phase during intake**: collapse, re-plan, continue.
- **Mid-run interruption**: if the user stops the run and asks for a change, update the affected `.superplan/phases/phase-N.md` spec, run `validate-phase.sh` on it, then ask the user to resume (they can re-dispatch the same `/goal` or just say "continue"). No need to restart phase 1.

---

## Reference files

- `references/planning-depth.md` — what makes a plan deep enough to deserve "Super"
- `references/phase-design.md` — how to slice phases that auto-chain cleanly
- `references/goal-format.md` — what `/goal` is on Claude Code + Codex, Superplan's single-`/goal` shape, required transcript blocks

## Scripts

- `scripts/detect-stack.sh` — identifies language, package manager, framework, build/test/lint commands (brownfield)
- `scripts/detect-env.sh` — greenfield environment recon
- `scripts/summarize-repo.sh` — compressed repo map (brownfield)
- `scripts/validate-phase.sh` — checks a phase spec has the required SUPERPLAN_PHASE_START marker and a non-empty acceptance criteria section

## Templates

- `templates/ROADMAP.md` — phase plan with dependencies
- `templates/STATE.md` — live progress file
- `templates/phase-goal.txt` — phase spec skeleton (work, criteria, evidence, mandatory commands)
- `templates/PROTOCOL.md` — phase execution loop, failure recovery, memory writeback (copied to `.superplan/PROTOCOL.md` at dispatch)
