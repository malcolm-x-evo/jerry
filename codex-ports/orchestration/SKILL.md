---
name: orchestration
description: >-
  Multi-agent workflow orchestration with state tracking, phase sequencing, sync barriers,
  cross-pollination handoffs, checkpointing, and cross-session resumption. Use when
  coordinating 3+ agents in a structured workflow, running sequential or parallel pipelines
  that need synchronization barriers, tracking workflow execution state across sessions,
  requiring checkpoint/resume for long-running processes, or planning workflows with
  quality gates and adversarial review cycles — orchestration, pipeline, workflow, multi-agent,
  phases, sync barrier, checkpoint, cross-pollination, state tracking, fan-out, fan-in,
  agent coordination. Do NOT use for single-agent tasks, simple sequential flows with no
  cross-session state, research/analysis (use $problem-solving), requirements (use $nasa-se),
  adversarial quality review (use $adversary), or transcript parsing (use $transcript).
---

# Orchestration — Multi-Agent Workflow Coordinator (Codex port)

> **Codex port version:** `codex-1.0.0` — versioned independently of the Jerry source skill.
> **Forked from:** Jerry `/orchestration` v2.2.0 (divergence point: 2026-07-04). This Codex port is
> a separate running version: it is **not** kept in lockstep with the Jerry original and may
> intentionally diverge. Bump this `codex-x.y.z` version on its own cadence when this port or its
> bundled references change; do not assume parity with `skills/orchestration/`.

> Ported from the Jerry Framework `/orchestration` Claude skill (`skills/orchestration/SKILL.md`).
> Codex has a single execution context and no sub-agent spawning, so the three orchestration
> agents (`orch-planner`, `orch-tracker`, `orch-synthesizer`) are **roles** you adopt one at a
> time by reading the matching file in `references/`. ORCHESTRATION.yaml file-based state
> replaces Memory-Keeper MCP (which is not registered in the Codex environment).

## Divergences from the Jerry source

| Capability | Jerry source | This Codex port |
|------------|-------------|-----------------|
| Agent execution | Spawned subagents (Agent/Task tool) | Roles adopted sequentially in one context |
| Parallel pipelines | `execution_mode: PARALLEL` concurrent | Sequential execution; PARALLEL is informational only |
| Memory-Keeper MCP | Supplemental cross-session search | Not available; ORCHESTRATION.yaml is the sole cross-session mechanism |
| Worker-agent coordination | Automatic via subagent spawning | Manual: user invokes work agents via separate `codex exec` calls |

## Honest capability disclosure (P-022)

**Automated worker-agent coordination is NOT available in this Codex port.** Users invoke
work agents via separate `codex exec` calls and report completion to the orch-tracker role.
The value this skill provides is: structured workflow planning, persistent YAML state tracking,
quality-gate enforcement (>= 0.92), checkpoint/resume, and synthesis coordination. It does
NOT automatically dispatch or execute work agents.

When a workflow declares `execution_mode: PARALLEL`, the Codex port executes the corresponding
agents sequentially in the same context. Functional outputs are identical; wall-clock
parallelism is not available.

## What this skill does

Provides a structured framework for coordinating multi-agent workflows. Three roles are
adopted in sequence to produce three core artifacts:

| Artifact | Format | Purpose |
|----------|--------|---------|
| `ORCHESTRATION_PLAN.md` | Markdown | Strategic context, workflow diagram, quality gates |
| `ORCHESTRATION_WORKTRACKER.md` | Markdown | Tactical execution log |
| `ORCHESTRATION.yaml` | YAML | Machine-readable state (SSOT, cross-session portable) |

## When to use / When NOT to use

**Use when:**
- Coordinating 3+ agents in a structured workflow
- Workflow spans multiple sessions and needs state persistence
- Need sync barriers or cross-pollination handoffs between pipelines
- Need checkpoint/resume for long-running processes
- Need quality gates (creator-critic-revision, >= 0.92 threshold)
- Need a final synthesis across all phase artifacts

**Do NOT use when:**
- Task requires a single agent only — orchestration overhead is unnecessary
- Simple sequential flow with no cross-session state — use direct agent invocation
- Research or analysis only — use `$problem-solving`
- Requirements or V&V — use `$nasa-se`
- Adversarial quality review — use `$adversary`
- Transcript parsing — use `$transcript`

## Roles

| Role | When adopted | Reference file |
|------|-------------|----------------|
| `orch-planner` | Once, at workflow start | `references/orch-planner.md` |
| `orch-tracker` | Re-entrantly — after EVERY agent or phase completion | `references/orch-tracker.md` |
| `orch-synthesizer` | Once, after all phases complete | `references/orch-synthesizer.md` |

**Important:** `orch-tracker` is re-entrant — it is adopted multiple times per workflow,
once after each agent or phase completes. `orch-planner` and `orch-synthesizer` are adopted
once each. The re-entrant nature of orch-tracker is the key architectural pattern of this port.

## Resume detection (B-R3)

**On every invocation, before taking any other action:**

1. Check whether an ORCHESTRATION.yaml exists in the current working directory or the
   project directory.
2. If it exists, read `workflow.status` from that file.
3. If `workflow.status: PAUSED` — branch to resume mode:
   - Read `resumption.files_to_read` and load all listed files.
   - Read `resumption.recovery_state.next_step` to find the resume point.
   - Continue the workflow from that step. Do NOT overwrite ORCHESTRATION.yaml.
4. If `workflow.status: ACTIVE` or `PLANNED` — continue the in-progress workflow.
5. If no ORCHESTRATION.yaml exists — begin a new workflow (adopt orch-planner role).

## Numbered workflow loop (B-R2)

Execute this loop for every orchestrated workflow:

**Step 1 — Plan** (adopt orch-planner role once):
- Read `references/orch-planner.md` and adopt the orch-planner role.
- Follow the role's instructions to produce `ORCHESTRATION_PLAN.md` and `ORCHESTRATION.yaml`.
- Assess criticality (C1-C4) and embed quality gate definitions in the plan.

**Step 2 — Execute phases** (repeat for each phase in the plan):

- **(2a) Work agent executes:**
  The user invokes the appropriate work agent via a separate `codex exec` call (e.g.,
  `codex exec "Use $problem-solving to ..."`) and reports back with the artifact path.
  This is a manual hand-off — the orchestration skill does not dispatch the work agent.

- **(2b) Adopt orch-tracker role** (re-entrant — once per agent/phase completion):
  Read `references/orch-tracker.md` and adopt the orch-tracker role.
  Update ORCHESTRATION.yaml: agent status to COMPLETE, artifact path registered,
  metrics recalculated. If the YAML file exceeds ~2,000 tokens, use offset/limit reads
  (load only `workflow.status`, current phase, and `quality.phase_scores` sections first).

- **(2c) Check quality gate score:**
  The orch-tracker role evaluates the phase output against the S-014 rubric (6 dimensions,
  weighted composite). Gate check:
  - Score >= 0.92: record PASS, proceed to (2d-pass).
  - Score < 0.92 AND iterations < 3: record REVISE, creator revises with critic feedback,
    repeat from (2c).
  - Score < 0.92 AND iterations >= 3: record ESCALATED, block phase transition, present
    current best result to user with full critic findings, request explicit guidance (H-31).

- **(2d) Gate outcome:**
  - PASS: Create checkpoint entry in ORCHESTRATION.yaml, proceed to next phase or barrier.
  - If this is a sync barrier with cross-pollination: the receiving pipeline's next phase
    MUST read the handoff artifact before generating any output. See cross-pollination
    protocol in `references/orch-planner.md`.

**Step 3 — Synthesize** (adopt orch-synthesizer role once, after all phases complete):
- Read `references/orch-synthesizer.md` and adopt the orch-synthesizer role.
- Read all phase and barrier artifacts listed in ORCHESTRATION.yaml.
- Produce final synthesis document.
- Mark `workflow.status: COMPLETE` in ORCHESTRATION.yaml.

## Cross-pollination handoff protocol (B-R1)

When a workflow has a sync barrier:

1. The source pipeline writes its handoff artifact to:
   `orchestration/{workflow_id}/cross-pollination/{barrier_id}/{src}-to-{dst}/handoff.md`

2. **Before the receiving pipeline's next phase generates any output**, it MUST read that
   handoff file. This read is mandatory — proceeding without it violates the barrier
   protocol and produces invalid cross-pollination.

3. Both directions must be written and read before the barrier is marked COMPLETE.

## References

- `references/orch-planner.md` — planner role: workflow design, quality gate planning, cross-pollination READ instructions
- `references/orch-tracker.md` — tracker role: state update protocol, re-entrant adoption, offset/limit reads
- `references/orch-synthesizer.md` — synthesizer role: synthesis protocol, adversarial synthesis, quality trends
- `docs/PATTERNS.md` — workflow pattern catalog with Codex divergence notes
- `docs/STATE_SCHEMA.md` — ORCHESTRATION.yaml schema specification
- `templates/ORCHESTRATION.template.yaml` — state template
- `templates/ORCHESTRATION_PLAN.template.md` — plan template (includes cross-pollination READ instructions)
- `templates/ORCHESTRATION_WORKTRACKER.template.md` — tracker template
