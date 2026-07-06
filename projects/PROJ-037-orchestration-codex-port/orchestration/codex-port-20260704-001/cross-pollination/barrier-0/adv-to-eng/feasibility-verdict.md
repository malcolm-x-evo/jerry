# Barrier 0 Feasibility Verdict — Adversary to Engineering

> **Barrier:** 0 (Feasibility Gate)
> **Issued by:** adv-executor (S-003/S-002/S-007 multi-strategy execution)
> **To:** Phase 1 eng-architect (ADR author)
> **Date:** 2026-07-04
> **Strategies Applied:** S-003 Steelman → S-002 Devil's Advocate → S-007 Constitutional AI Critique (H-16 compliant ordering)
> **Findings report:** `adv/findings/barrier-0-findings.md`

---

## Document Sections

| Section | Purpose |
|---------|---------|
| [Conflict Reconciliation](#conflict-reconciliation) | Which finding is correct and why |
| [Verdict](#verdict) | GO / NO-GO / GO-WITH-CONSTRAINTS |
| [Recommended Architecture](#recommended-architecture) | Option A vs B vs defer |
| [Required De-Risking](#required-de-risking) | Live test specification |
| [CSP Corrections](#csp-corrections) | Must-fix before Phase 2 |
| [Phase 1 ADR Guidance](#phase-1-adr-guidance) | What the ADR must address |

---

## Conflict Reconciliation

### The Conflict

| Worker | Claim | Confidence | Core Evidence |
|--------|-------|-----------|---------------|
| eng-architect | No native subagent spawning for `codex exec` skills in 0.142.5 | 0.88 | `enable_fanout: under development, false`; adversary/eng-team ports explicit "no sub-agent spawning"; skill-creator conditional language |
| ps-researcher | Subagents ARE available — `agents.max_depth=1`, `max_threads=6`, natural-language invocation in skill body | 0.92 | `developers.openai.com/codex/subagents` (official docs); design-choice reading of existing ports |

### Finding: The Two Workers Address Different Scopes

The conflict is resolvable without either source being wrong. The workers are making claims about different scopes of the same question:

- **ps-researcher** is correct that Codex has a subagent architecture at the system level. `agents.max_depth=1`, `agents.max_threads=6`, and the `developers.openai.com/codex/subagents` documentation are real. This architecture exists.

- **eng-architect** is correct that subagent spawning is not available to `codex exec` skill bodies in 0.142.5 as currently configured. `enable_fanout: under development, false` directly names the capability that would enable in-skill parallel fan-out spawning — and that capability is not yet enabled. Both production-active Codex ports state categorically "no sub-agent spawning" as a runtime fact, not a preference.

**Most likely reconciliation:** Codex's subagent architecture (as documented) operates in interactive TUI mode where `multi_agent: stable, true` is active. The capability for a skill body to programmatically trigger subagent spawning during a `codex exec` non-interactive run is what `enable_fanout` is being developed to provide — and it is not yet available.

### Weight of Evidence Assessment

| Evidence Item | Favors | Weight | Rationale |
|---------------|--------|--------|-----------|
| `enable_fanout: under development, false` | eng-architect (no subagents for exec) | HIGH | Direct, specific, operationally grounded. Names the capability; its undeveloped state implies the capability is not yet available. |
| adversary SKILL.md: "no sub-agent spawning" | eng-architect | HIGH | Primary evidence: production port, written by practitioner with direct access to the same runtime, categorical factual statement not design rationale. |
| eng-team SKILL.md: identical language | eng-architect | HIGH | Corroborating primary evidence from a second independent port. |
| `developers.openai.com/codex/subagents` | ps-researcher | MEDIUM-HIGH | Official docs, high authority — but system-level description that may not apply to `codex exec` skill context specifically. |
| `agents.max_depth=1`, `max_threads=6` | ps-researcher | MEDIUM | Real operational parameters — but for what invocation context is unconfirmed by ps-researcher. |
| "design choice not technical limit" (ps-researcher inference) | ps-researcher | LOW | Unsupported inference; contradicted by ports' categorical "no" language. |
| No code example for in-body subagent invocation | eng-architect | MEDIUM | ps-researcher's own admission; undermines applicability claim. |
| skill-creator conditional: "Only do this when it is possible" | eng-architect | MEDIUM | Conditional language implies context-gating; available in interactive sessions, not universally. |

**Net assessment:** eng-architect's position carries more weight on the operationally specific question ("does subagent spawning work in a `codex exec` skill invocation right now?"). ps-researcher's position correctly identifies that the broader Codex system supports subagents, which makes the question non-trivially open and warrants a live test rather than dismissal.

**ps-researcher's CRITICAL CAVEAT — acknowledged:** "no code example found for in-body subagent invocation." A technical capability without a reproducible invocation example cannot be treated as confirmed for architectural planning purposes.

---

## Verdict

**GO-WITH-CONSTRAINTS**

The port is feasible. The sequential-role architecture (Option A) is the safe, confirmed baseline. The subagent architecture question (Option B) is OPEN — not confirmed and not definitively ruled out — and requires a 30-minute live test to resolve before Phase 1 commits to it.

This verdict activates the following constraints:

| Constraint | Action Required |
|-----------|----------------|
| Architecture: use sequential-role as Phase 1 ADR baseline | Phase 1 eng-architect documents Option A as the default; Option B as a conditional path pending live test |
| Live subagent test required | Must execute before Phase 1 ADR is finalized if Option B is under consideration |
| CSP standard corrections required | CC-001 (Critical) must be fixed before Phase 2 begins; CSP is a prerequisite to template generation |
| Feature-flag upgrade path documented | Phase 1 ADR must document `enable_fanout` going `true` as the trigger for Option B re-evaluation |

---

## Recommended Architecture

**Decision: Proceed to Phase 1 with Option A (sequential-role) as baseline; defer Option A vs. B final selection to Phase 1 ADR after a minimal live subagent test.**

| Option | Pattern | Fidelity | Complexity | Readiness |
|--------|---------|----------|------------|-----------|
| A — Sequential-Role | Adopt orch-planner, orch-tracker, orch-synthesizer as sequential roles in one context; ORCHESTRATION.yaml as sync primitive | Lower (no true parallelism) | Low | CONFIRMED by existing ports |
| B — Subagent-Parallel | Use Codex subagent spawning for worker agents; `max_depth=1` provides single-level nesting; natural-language invocation in body | Higher (preserves parallel pipeline capability) | Medium | UNCONFIRMED — requires live test |

**Rationale for deferral rather than immediate Option A commitment:**

ps-researcher's evidence is substantial enough that dismissing Option B without a live test would be premature. The official docs and architectural parameters are real. If the live test confirms subagent spawning works from `codex exec` skill bodies, Option B would deliver significantly higher fidelity to the Jerry `/orchestration` original — preserving parallel pipeline execution, independent context isolation per worker role, and genuine fan-out coordination. The 30-minute investment in the live test is justified by the potential architecture upside.

**If live test confirms Option B works:**
- Phase 1 ADR documents Option B as the primary architecture
- orch-planner, orch-tracker, orch-synthesizer become TOML agent definitions in `~/.codex/agents/` (or `.codex/agents/`)
- SKILL.md body instructs Codex to spawn these agents for their respective workflow phases
- Max_depth=1 constraint naturally enforces P-003 single-level nesting
- Note P-003 alignment explicitly in Phase 1 ADR

**If live test confirms Option B does NOT work:**
- Phase 1 ADR documents Option A only, with the `enable_fanout` upgrade path as deferred decision (DEC-001)
- Sequential-role architecture: orch-planner, orch-tracker, orch-synthesizer definitions become `references/` role files
- ORCHESTRATION.yaml is the single coordination surface (already P-002 primary mechanism)
- Document this as a reversible decision per ADR conventions

---

## Required De-Risking

### Live Subagent Test — Required Before Phase 1 ADR Is Finalized

**Purpose:** Definitively resolve whether a `codex exec` non-interactive skill invocation can spawn subagents via natural language instructions in the SKILL.md body.

**Test specification:**

**Step 1 — Create a minimal TOML agent definition:**
```
~/.codex/agents/test-worker.toml
  name = "test-worker"
  description = "A minimal test worker that lists files and reports back"
  developer_instructions = "List the files in the current directory and return the list."
```

**Step 2 — Create a minimal skill with in-body subagent instruction:**
```
~/.codex/skills/subagent-probe/SKILL.md
---
name: subagent-probe
description: Tests whether subagent spawning works from codex exec. Use when testing subagent capability.
---
# Subagent Probe Skill

Your task is to spawn a subagent using the test-worker agent to list files, then return the subagent's result.
Use the test-worker agent to perform the task and report what it returns.
```

**Step 3 — Execute via `codex exec`:**
```bash
codex exec "Use the subagent-probe skill to test subagent spawning"
```

**Observe:**
- Does a subagent spawn? (Look for evidence in the output or session logs)
- Does the result return from the subagent correctly?
- Is there any error about subagent spawning not being available in exec mode?
- What does `codex --json exec` output show for agent events?

**Decision gate:**
- **Subagent spawned successfully** → Option B is viable; proceed to Phase 1 ADR with Option B as primary
- **No subagent spawned / error returned** → Option A confirmed; proceed with sequential-role baseline
- **Ambiguous** → Attempt `spawn_agents_on_csv` experimental tool as alternative mechanism; if still ambiguous, default to Option A with Phase 1 ADR note

**Estimated time:** 30 minutes. This is the single highest-ROI action before Phase 1 begins.

---

## CSP Corrections

**CC-001 is Critical — these corrections are BLOCKED FROM PROCEEDING TO PHASE 2 without resolution.**

The CSP standard (`standards/codex-skill-port-standard.md`) contains two factual errors in the Frontmatter Standards section that will cause validator failures in Phase 2 template generation.

### Correction 1: "Codex ignores them" → Validator error

**Current CSP text (Frontmatter Standards, [VERIFY] row):**
> "Dropped in the Codex SKILL.md — Codex **ignores** them."

**Corrected text:**
> "Dropped in the Codex SKILL.md. Fields outside the allowed set (`name`, `description`, `license`, `allowed-tools`, `metadata`) cause a `quick_validate.py` validator error (FAIL — not silently ignored). Jerry's `version` and `activation-keywords` fields will fail validation and MUST be excluded from the Codex SKILL.md."

**Evidence:** `quick_validate.py` lines 40-49 (installed Codex 0.142.5): `if unexpected_keys: return False, f"Unexpected key(s) in SKILL.md frontmatter: {unexpected}. Allowed properties are: {allowed}"` — explicit failure, not ignore.

### Correction 2: `allowed-tools` status clarification

**Current CSP text:**
> "`allowed-tools` — Dropped" (implied in the [VERIFY] row — same claim as above)

**Corrected text:**
> "`allowed-tools` — **Permitted** by the Codex 0.142.5 validator (it is in the allowed set). However: (a) it has different Codex-native semantics (space-separated tool names in agentskills.io format, e.g., `Bash(git:*) Read`) incompatible with Jerry's value; (b) enforcement is experimental and unreliable per GitHub issue; (c) existing port convention (adversary, eng-team) omits it. Recommendation: omit from Codex SKILL.md unless specific tool restriction is required; NEVER copy Jerry's `allowed-tools` value."

**Responsible party for corrections:** Phase 1 eng-architect applies corrections to CSP as part of Phase 1 deliverables. Adversary validates corrections before Phase 2 start.

---

## Phase 1 ADR Guidance

The Phase 1 ADR must address the following items. This list is the mandatory scope derived from Barrier 0 analysis.

| # | Required ADR Item | Source Finding |
|---|---|---|
| 1 | Subagent test result: confirm Option A or Option B as primary architecture | DA-001, DA-002, DA-003 |
| 2 | Each orchestration primitive mapping (P-1 through P-11) from eng-arch-001 — adopt or refine | SM-002 |
| 3 | `enable_fanout` upgrade path as a deferred, reversible decision (DEC-001) | DA-003 |
| 4 | ORCHESTRATION.yaml as the primary coordination and state persistence mechanism | SM-002 |
| 5 | orch-planner, orch-tracker, orch-synthesizer as either role files (Option A) or TOML agent definitions (Option B) | SM-001, SM-002 |
| 6 | P-003 alignment: note that Codex `max_depth=1` mechanically enforces the single-level nesting constraint if Option B is adopted | CC-002 |
| 7 | Memory-Keeper MCP gap: file-based ORCHESTRATION.yaml replaces MCP; justify per P-002 primacy | SM-002 |
| 8 | Version independence: `codex-x.y.z` versioning in SKILL.md body header; divergence note from Jerry source version | CSP |
| 9 | Future upgrade path: when `multi_agent_v2` stabilizes, re-evaluate port architecture | SM-002 |

---

## Barrier 0 Gate Status

| Gate Check | Status | Notes |
|------------|--------|-------|
| Is the port feasible? | PASS | Sequential-role maps all primitives (functional parity) |
| Is the primary architecture confirmed? | CONDITIONAL | Option A confirmed; Option B requires live test |
| CSP corrections complete? | BLOCKED | CC-001 Critical — must fix before Phase 2 |
| P-003 compliance confirmed? | PASS | Both options satisfy P-003 / H-01 |
| Conflict reconciled? | PASS | Scope difference identified; live test resolves remaining ambiguity |
| Phase 1 ADR scope defined? | PASS | 9-item mandatory scope above |

**Overall Barrier 0 Gate: GO-WITH-CONSTRAINTS**
- Constraint 1: Live subagent test before Phase 1 ADR is finalized
- Constraint 2: CSP corrections before Phase 2 begins

---

*Verdict issued by: adv-executor (Barrier 0 adversary role)*
*Strategies: S-003 (Steelman) → S-002 (Devil's Advocate) → S-007 (Constitutional AI Critique)*
*H-16 ordering confirmed: Steelman executed before Devil's Advocate*
*P-003 compliant: no subagents spawned in this execution*
*Findings detail: `adv/findings/barrier-0-findings.md`*
*Next artifact: Phase 1 ADR by eng-architect*
*Created: 2026-07-04*
