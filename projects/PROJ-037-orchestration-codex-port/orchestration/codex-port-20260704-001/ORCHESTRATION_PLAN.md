# ORCHESTRATION_PLAN.md

> **Document ID:** PROJ-037-ORCH-PLAN
> **Project:** PROJ-037-orchestration-codex-port
> **Workflow ID:** `codex-port-20260704-001`
> **Status:** ACTIVE (awaiting user approval to execute)
> **Version:** 2.0
> **Created:** 2026-07-04
> **Last Updated:** 2026-07-04

---

## 1. Executive Summary

Port the Jerry `/orchestration` skill so it runs as a native **Codex CLI** skill under
`~/.codex/skills/orchestration/`, matching the convention already established by the ported
`adversary/` and `eng-team/` Codex skills. The port must preserve orchestration's defining
capability — **multi-agent coordination** (subagent spawning, parallel pipelines, sync barriers,
cross-pollination, state tracking) — mapped onto Codex's own subagent mechanism, and must be
**verified by a live end-to-end run inside Codex**, not merely structurally validated.

**GitHub Issue:** [geekatron/jerry#315](https://github.com/geekatron/jerry/issues/315) (assigned to user).

**Division of labor (per user):**
- **Planning:** Fable — this session, foreground, orchestrator role.
- **Execution:** Sonnet-tier `/eng-team` agents spawned as **background** workers (Agent tool `run_in_background: true`).
- **Validation:** Sonnet-tier `/adversary` gates **and** the Phase 3 QA `codex` live-run, also **background** workers.
- **Runtime model of the ported skill:** model-agnostic (Codex picks its own model at run time).

> The background-Sonnet execution/validation policy is encoded in `ORCHESTRATION.yaml` under the
> `execution:` block and per-agent `model: sonnet` / `background: true` fields.

**Current State:** Plan authored; not yet executed. Awaiting approval.

**Orchestration Pattern:** Sequential build pipeline with cross-pollinated adversarial quality gates
(Pattern 2 "Sequential with Checkpoints" + Adversarial Quality Mode barrier gates). Two logical
pipelines cross-pollinate at every barrier: **ENG** (build) and **ADV** (quality).

### 1.1 Workflow Identification

| Field | Value | Source |
|-------|-------|--------|
| Workflow ID | `codex-port-20260704-001` | auto |
| ID Format | `{purpose}-{YYYYMMDD}-{NNN}` | semantic-date-seq |
| Base Path | `orchestration/codex-port-20260704-001/` | Dynamic |
| Criticality | **C3** (Significant: 10+ files, external tool, multi-session, live execution) | quality-enforcement SSOT |
| Quality threshold | >= 0.92 weighted composite (H-13) | quality-enforcement SSOT |

**Artifact Output Locations:**
- Pipeline ENG (build): `orchestration/codex-port-20260704-001/eng/`
- Pipeline ADV (quality): `orchestration/codex-port-20260704-001/adv/`
- Cross-pollination: `orchestration/codex-port-20260704-001/cross-pollination/`
- **Ported skill (primary deliverable):** `~/.codex/skills/orchestration/`
- **Repo tracked copy:** `codex-ports/orchestration/` (git-tracked mirror; see Risk R-05)

---

## 2. Workflow Architecture

### 2.1 Pipeline Diagram

```
    PIPELINE ENG (build - eng-team)                 PIPELINE ADV (quality - adversary)
    ===============================                 ==================================

┌──────────────────────────────────┐
│ PHASE 0: DISCOVERY / SPIKE        │
│ ──────────────────────────────── │            (ADV joins from Barrier 0)
│ • eng-architect + research        │
│ Confirm Codex subagent + parallel │
│ mechanism; how skills express it. │
│ STATUS: PENDING                   │
└─────────────────┬────────────────┘
                  ▼
    ╔═══════════════════════════════════════════════════════════╗
    ║   BARRIER 0: FEASIBILITY GATE  (adv C3 review)            ║
    ║   eng→adv: codex-subagent-mechanism.md                    ║
    ║   adv→eng: feasibility-verdict.md (GO / NO-GO / REVISE)   ║
    ╚═══════════════════════════════════════════════════════════╝
                  ▼
┌──────────────────────────────────┐
│ PHASE 1: PORT ARCHITECTURE (ADR)  │
│ • eng-architect                   │
│ Map pipelines/barriers/cross-poll │
│ /state → Codex subagents + files  │
│ STATUS: PENDING                   │
└─────────────────┬────────────────┘
                  ▼
    ╔═══════════════════════════════════════════════════════════╗
    ║   BARRIER 1: DESIGN GATE  (adv C3 review of ADR)         ║
    ╚═══════════════════════════════════════════════════════════╝
                  ▼
┌──────────────────────────────────┐
│ PHASE 2: BUILD THE PORT           │
│ • eng-lead + eng-devsecops        │
│ Write SKILL.md, references/*,     │
│ agents/openai.yaml, docs,         │
│ templates → ~/.codex + repo copy  │
│ STATUS: PENDING                   │
└─────────────────┬────────────────┘
                  ▼
    ╔═══════════════════════════════════════════════════════════╗
    ║   BARRIER 2: BUILD GATE  (adv C3 review of artifacts)    ║
    ╚═══════════════════════════════════════════════════════════╝
                  ▼
┌──────────────────────────────────┐
│ PHASE 3: LIVE-RUN VERIFICATION    │
│ • eng-qa (design sample workflow) │
│ Actually run `codex` on ported    │
│ skill end-to-end; capture evidence│
│ STATUS: PENDING                   │
└─────────────────┬────────────────┘
                  ▼
    ╔═══════════════════════════════════════════════════════════╗
    ║   BARRIER 3: VERIFICATION GATE                           ║
    ║   eng-reviewer final gate + adv verifies run evidence    ║
    ╚═══════════════════════════════════════════════════════════╝
                  ▼
┌──────────────────────────────────┐
│ PHASE 4: SYNTHESIS & HANDOFF      │
│ • orch-synthesizer                │
│ Divergence note, maintenance doc, │
│ final synthesis                   │
│ STATUS: PENDING                   │
└──────────────────────────────────┘
```

### 2.2 Orchestration Pattern Classification

| Pattern | Applied | Description |
|---------|---------|-------------|
| Sequential | Yes | Phases 0→4 execute in order |
| Concurrent | Partial | Within Phase 2/3, eng sub-tasks fan out; ADV runs concurrently at each gate |
| Barrier Sync | Yes | 4 barriers; adversarial cross-pollination each |
| Hierarchical | Yes | Main context (Fable orchestrator) delegates to workers; P-003 single level |

---

## 3. Phase Definitions

### 3.1 Pipeline ENG Phases (build)

| Phase | Name | Purpose | Agents | Status |
|-------|------|---------|--------|--------|
| 0 | Discovery/Spike | Nail Codex's exact subagent + parallel mechanism and skill expression | eng-architect, ps-researcher | PENDING |
| 1 | Port Architecture | ADR mapping orchestration semantics → Codex primitives | eng-architect | PENDING |
| 2 | Build | Write ported skill files to `~/.codex` + repo mirror | eng-lead, eng-devsecops | PENDING |
| 3 | Live-Run Verification (QA) | Run `codex` to validate the changes work as intended; fix → re-run loop, **max 8 iterations before human review** | eng-qa | PENDING |
| 4 | Synthesis | Divergence note + maintenance handoff | orch-synthesizer | PENDING |

### 3.2 Pipeline ADV Phases (quality)

| Phase | Name | Purpose | Agents | Status |
|-------|------|---------|--------|--------|
| B0 | Feasibility review | GO/NO-GO on the mechanism the port depends on | adv-executor, adv-scorer | PENDING |
| B1 | Design review | C3 adversarial review of the port ADR | adv-executor, adv-scorer | PENDING |
| B2 | Build review | C3 adversarial review of ported artifacts | adv-executor, adv-scorer | PENDING |
| B3 | Evidence review | Verify live-run evidence is genuine and complete | adv-executor, adv-scorer | PENDING |

---

## 4. Sync Barrier Protocol

### 4.1 Barrier Transition Rules

```
1. PRE-BARRIER CHECK
   □ Phase agents completed; artifacts exist and valid
2. CROSS-POLLINATION EXECUTION
   □ ENG hands deliverable to ADV; ADV scores with S-014 + S-002 + S-007 (C3 set)
   □ If score < 0.92 → revision (S-003 Steelman + feedback), max 3 iterations
3. POST-BARRIER VERIFICATION
   □ Score >= 0.92 → PASS; verdict handed back to ENG; barrier COMPLETE
   □ 3 failed iterations → ESCALATE to user (AE-006, C3+)
```

### 4.2 Barrier Definitions

| Barrier | After Phase | Artifacts | Status |
|---------|-------------|-----------|--------|
| barrier-0 | Phase 0 | eng→adv: `codex-subagent-mechanism.md`; adv→eng: `feasibility-verdict.md` | PENDING |
| barrier-1 | Phase 1 | eng→adv: `ADR-001-codex-port.md`; adv→eng: `design-review.md` | PENDING |
| barrier-2 | Phase 2 | eng→adv: build manifest; adv→eng: `build-review.md` | PENDING |
| barrier-3 | Phase 3 | eng→adv: `live-run-evidence.md`; adv→eng: `verification-verdict.md` | PENDING |

---

## 5. Agent Registry

| Agent ID | Pipeline | Role | Input Artifacts | Output Artifacts | Model | Status |
|----------|----------|------|-----------------|------------------|-------|--------|
| eng-arch-001 | ENG | Discovery + architecture | Jerry `/orchestration` source, Codex docs | mechanism spike, ADR | sonnet | PENDING |
| ps-rsch-001 | ENG | Codex mechanism research | Codex CLI/docs, skill-creator | research notes | sonnet | PENDING |
| eng-lead-001 | ENG | Build coordination + write files | ADR | ported skill files | sonnet | PENDING |
| eng-dso-001 | ENG | Sync to `~/.codex`, safety of file ops | build plan | install + repo mirror | sonnet | PENDING |
| eng-qa-001 | ENG | Live-run test design + execution | ported skill | live-run evidence | sonnet | PENDING |
| eng-rev-001 | ENG | Final review gate | all artifacts | final gate report | sonnet | PENDING |
| adv-exec-001 | ADV | Execute adversarial strategies | phase deliverables | finding reports | sonnet | PENDING |
| adv-score-001 | ADV | S-014 scoring | phase deliverables | rubric scores | sonnet | PENDING |
| orch-synth-001 | ENG | Final synthesis | all | workflow synthesis | fable | PENDING |

---

## 6. State Management

### 6.1 State Files

| File | Purpose |
|------|---------|
| `ORCHESTRATION.yaml` | Machine-readable state (SSOT) |
| `ORCHESTRATION_WORKTRACKER.md` | Tactical execution documentation |
| `ORCHESTRATION_PLAN.md` | This file — strategic context |

### 6.2 Checkpoint Strategy

| Trigger | When | Purpose |
|---------|------|---------|
| PHASE_COMPLETE | After each phase | Phase-level rollback |
| BARRIER_COMPLETE | After each adversarial gate | Recovery + audit of quality decision |
| MANUAL | User-triggered | Debug/inspection |

---

## 7. Execution Constraints

### 7.1 Hard Constraints (Jerry Constitution)

| Constraint | ID | Enforcement |
|------------|----|-------------|
| Single agent nesting | P-003 | Fable orchestrator → worker only; workers do not spawn workers |
| File persistence | P-002 | All state to filesystem |
| No deception | P-022 | Live-run evidence must be real command output, never fabricated |
| User authority | P-020 | User approves this plan and each escalation |
| Active project | H-04 | `JERRY_PROJECT=PROJ-037-orchestration-codex-port` |
| GitHub parity | H-32 | Work items mirrored to GitHub Issues (jerry repo) |

### 7.2 Soft Constraints

| Constraint | Value | Rationale |
|------------|-------|-----------|
| Max barrier retries | 3 | Adversarial gate circuit breaker (H-14/AE-006) |
| Max QA codex-run iterations | 8 | Phase 3 live-run validation loop; human review required after 8 failed runs (user directive) |
| Checkpoint frequency | BARRIER | Recovery granularity |
| Writes outside repo | Only `~/.codex/skills/orchestration/` | Least surprise; mirror kept in repo |

---

## 8. Success Criteria

### 8.1 Phase Exit Criteria

| Phase | Criterion | Validation |
|-------|-----------|------------|
| 0 | Codex subagent/parallel mechanism documented with evidence | ADV GO verdict; cites real Codex behavior/docs |
| 1 | ADR maps every orchestration primitive (pipeline, barrier, cross-poll, state) to a Codex primitive | ADV design review >= 0.92 |
| 2 | Ported skill files exist, load without error, follow codex convention (SKILL.md + references/ + agents/openai.yaml) | `codex` loads skill; ADV build review >= 0.92 |
| 3 | Ported skill runs end-to-end in a real `codex` invocation and changes work as intended | Captured terminal evidence; QA loop converges within 8 codex runs (else human review); ADV confirms genuine |
| 4 | Divergence note + maintenance doc produced | Files exist; synthesis complete |

### 8.2 Workflow Completion Criteria

| Criterion | Validation |
|-----------|------------|
| All phases COMPLETE | ORCHESTRATION.yaml all phase status = COMPLETE |
| All barriers PASS | All 4 barrier scores >= 0.92 |
| **Live run verified** | Real Codex run transcript captured in `live-run-evidence.md` |
| Final synthesis created | `synthesis/codex-port-final.md` exists |

---

## 9. Risk Mitigations

| ID | Risk | Likelihood | Impact | Mitigation |
|----|------|------------|--------|------------|
| R-01 | Codex subagent mechanism differs from assumption / can't do true parallelism | M | H | Phase 0 spike is a hard GO/NO-GO gate before any build; NO-GO → escalate with fallback (sequential-role port) |
| R-02 | Live run needs a model/auth Codex can't reach non-interactively | M | M | Phase 3 verifies auth (`codex doctor`, existing `auth.json`) and uses `codex exec` with a minimal workflow |
| R-03 | Cross-pollination semantics don't map cleanly to Codex subagents | M | M | ADR (Phase 1) must show the mapping explicitly; ADV design gate blocks if hand-wavy |
| R-04 | Writing to `~/.codex` corrupts existing skills | L | H | eng-devsecops backs up existing skill dir; repo mirror is source of truth; no destructive ops |
| R-05 | Port drifts from Jerry source over time | H | L | Independent `codex-x.y.z` version + divergence note (matches adversary/eng-team convention); repo mirror tracked in git |
| R-06 | Skill name collision / codex expects different frontmatter | L | M | Phase 0 documents codex SKILL.md schema; Phase 2 conforms |

---

## 10. Resumption Context

### 10.1 Current Execution State

```
WORKFLOW STATUS AS OF 2026-07-04
================================
Phase 0 (Discovery):     PENDING (awaiting approval)
Phase 1 (Architecture):  PENDING
Phase 2 (Build):         PENDING
Phase 3 (Live-Run):      PENDING
Phase 4 (Synthesis):     PENDING
Barriers 0-3:            PENDING
```

### 10.2 Next Actions

1. **User approves this plan** (or requests changes).
2. Set `JERRY_PROJECT=PROJ-037-orchestration-codex-port`; create worktracker entities + GitHub issues (H-32).
3. Execute Phase 0 spike (eng-architect + research on Sonnet) → Barrier 0 feasibility gate (adversary).
4. Proceed phase-by-phase, updating `ORCHESTRATION.yaml` at each barrier.

---

*Document ID: PROJ-037-ORCH-PLAN*
*Workflow ID: codex-port-20260704-001*
*Version: 2.0*
*Cross-Session Portable: All repo paths are repository-relative*
