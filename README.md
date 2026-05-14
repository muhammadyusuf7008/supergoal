# Superplan

Plan deeply, then autonomously build until it's done.

`/superplan <what you want>` recons your codebase, pulls in your saved preferences from memory, decomposes the work into the right number of phases for the task, gets one confirmation from you, then dispatches a single `/goal` that drives the entire build through to completion with built-in retry and fix-spec recovery.

Works on **Claude Code** and **Codex** (Codex CLI).

## Why one `/goal` (not a chain)

`/goal` on both hosts takes a short **end-state condition** that an evaluator checks against the transcript after each turn — not a long task body. Superplan v3 leverages this directly: one `/goal` covers the whole run; phase work lives in files the agent reads from disk. No char budget, no inter-session chain dispatch, no fragility.

## Install

### Claude Code

Via the plugin marketplace (one-line install):

```text
/plugin install superplan@robzilla1738
```

Or clone manually:

```bash
mkdir -p ~/.claude/skills
git clone https://github.com/robzilla1738/superplan /tmp/superplan
cp -R /tmp/superplan/skills/superplan ~/.claude/skills/
```

### Codex

Add the plugin's skills directory to your Codex skills path (see `~/.codex/` setup) or copy `skills/superplan` into wherever your Codex install reads skills from.

## Use

```
/superplan build me an Expo app that converts photos to ASCII art
```

What happens:

1. **Stage 0 — Available context.** Auto-detects your memory directory, preloads relevant feedback/user/project memories, senses which tools/MCPs are available this session.
2. **Stage 1 — Intake.** Asks 0–2 clarifying questions only for true gaps memory + prompt don't already answer. Most well-described tasks ask zero.
3. **Stage 2 — Recon.** Parallel codebase/environment scan.
4. **Stage 3 — Deep think.** Identifies top-3 risks + dependencies. Uses Context7/WebSearch if available (optional, not required).
5. **Stage 4 — Decompose.** Phase count derived from the task — no fixed cap.
6. **Stage 5 — Write specs.** `ROADMAP.md` + `STATE.md` + one `phase-N.md` work spec per phase.
7. **Stage 6 — Plan review.** Shows phases, assumptions, risks, and applied memories. Concrete revision menu: **Start now / Adjust assumption / Tweak a phase / Restructure phases.**
8. **Stage 7 — Dispatch one `/goal`.** Runs phases sequentially with 3-strike auto-retry → fix-spec → handoff. Writes a memory at each phase boundary so future runs start smarter.

## Self-healing failure recovery

Built into every run:

- **First failure** of any acceptance criterion → `FAILURE_PROBE` printed, probe injected as feedback, **auto-retry once**.
- **Second failure** → `FAILURE_ESCALATE`, write a focused fix spec at `phase-N.fix.md`, execute inline.
- **Third failure** → `FAILURE_HANDOFF`, mark state `BLOCKED`, stop. User takes the wheel.

Flaky envs, typos, and missed deps self-resolve. Only real blockers escalate.

## Memory writeback

Each phase ends with a "non-obvious learnings" check. If anything a future run on a similar task would benefit from was learned (an API quirk, a confirmed user preference, a project-level fact, a failure-and-fix pattern), it's saved to your memory directory using the standard `name`/`description`/`metadata.type` frontmatter. The final phase always writes a `project_<slug>.md` memory pointing at the new/changed project.

## Artifacts a run produces

All under `.superplan/` in the project directory:

```
.superplan/
├── ROADMAP.md            full plan
├── STATE.md              live progress, updated per phase
├── THINKING.md           risks, dependencies, applied memories, best practices
├── PROTOCOL.md           execution loop + failure recovery (copied at dispatch)
├── context.md            recon output
├── repo-map.md           brownfield only
├── applied-memories.md   memory hits that informed the plan
├── tools.md              detected MCPs / skills / hosts
└── phases/
    ├── phase-1.md
    ├── phase-2.md
    ├── ...
    └── phase-N.md
```

## Skill internals

```
skills/superplan/
├── SKILL.md
├── references/
│   ├── planning-depth.md      what makes a plan deep enough to deserve "Super"
│   ├── phase-design.md        how to slice phases (adaptive count, no cap)
│   └── goal-format.md         /goal mechanics on CC + Codex, required transcript blocks
├── scripts/
│   ├── detect-env.sh          greenfield env recon
│   ├── detect-stack.sh        brownfield stack recon
│   ├── summarize-repo.sh      repo map
│   └── validate-phase.sh      checks phase spec structure
└── templates/
    ├── ROADMAP.md
    ├── STATE.md
    ├── phase-goal.txt         phase spec skeleton
    └── PROTOCOL.md            execution loop + failure recovery
```

## License

MIT. See [LICENSE](LICENSE).
