# ADR-001: Orchestration Skill Codex Port — Architecture Decision

> **Document ID:** PROJ-037-ADR-001
> **Agent:** eng-architect (Phase 1 — Port Architecture)
> **Workflow:** codex-port-20260704-001
> **Date:** 2026-07-04
> **Status:** Accepted
> **Criticality:** C3 — Significant (multi-file, multi-session artifact, reversible)
> **Quality target:** >= 0.92

---

## Document Sections

| Section | Purpose |
|---------|---------|
| [L0: Executive Summary](#l0-executive-summary) | Decision and business impact in plain language |
| [Context](#context) | Why this decision was needed, what was discovered |
| [Decision](#decision) | Option A — sequential-role; what it means in full |
| [Consequences](#consequences) | What the decision enables, forecloses, and defers |
| [Primitive Mapping Table](#appendix-a-primitive-mapping-table) | Every orchestration primitive and its Codex equivalent |
| [File Manifest](#appendix-b-file-manifest) | Exact files Phase 2 must produce |
| [Subagent-Ready Upgrade Path](#appendix-c-subagent-ready-upgrade-path) | How to promote to Option B when the blockers clear |
| [CSP Corrections Applied](#appendix-d-csp-corrections-applied-ac-12) | Frontmatter standard corrections from Phase 0 |
| [Phase 3 Verification Hooks](#appendix-e-phase-3-verification-hooks) | Live test criteria for proof of port function |
| [L2: Strategic Implications](#l2-strategic-implications) | Long-term trade-offs, divergence posture, upgrade signals |

---

## L0: Executive Summary

The Jerry `/orchestration` skill is being ported to OpenAI Codex CLI (version 0.142.5) so it can
be used in non-interactive `codex exec` runs alongside the already-ported `adversary` and `eng-team`
skills.

A live test (Phase 0) confirmed that Codex's subagent spawning mechanism is wired in the CLI but
currently blocked by a service-tier entitlement error on this account, and the parallel fan-out
flag (`enable_fanout`) is not yet released. As a result, true parallel pipeline execution is
unavailable in the Codex port.

The decision is to build a **sequential-role port** — identical to the pattern used by the
`adversary` and `eng-team` Codex ports that are already in production use. The three orchestration
agents (`orch-planner`, `orch-tracker`, `orch-synthesizer`) become role files loaded on demand
rather than spawned subagents. ORCHESTRATION.yaml file-based state replaces Memory-Keeper MCP
(which is not registered in the Codex environment). The YAML `execution_mode: PARALLEL` field
becomes planning metadata only.

**Business impact:** The ported skill delivers structured workflow planning, persistent state
tracking, quality gate enforcement, checkpoint/resume, and cross-pollination handoffs. Users gain
orchestration capability in Codex with no new infrastructure dependencies. **Honest disclosure
(P-022):** Automated worker-agent coordination is NOT available in this Codex port; users invoke
work agents via separate `codex exec` calls and report completion to the orch-tracker role. The
skill's value is structured planning, persistent state tracking, quality-gate enforcement, and
synthesis coordination — not automatic multi-agent execution. (Wall-clock parallelism is likewise
unavailable, but the source skill does not actually deliver it either — it executes sequentially
within a single LLM context.)

---

## Context

### Background

PROJ-037 is porting the Jerry `/orchestration` Claude skill to Codex CLI 0.142.5 for use in
non-interactive `codex exec` workflows. Two previous Jerry ports (`adversary`, `eng-team`) use the
established sequential-role pattern: multi-agent logic is expressed as named roles adopted one at a
time within a single Codex execution context.

The `orchestration` skill is the most complex port attempted because it:
- Coordinates 3 reusable agents across multi-phase workflows
- Maintains persistent YAML state that spans sessions
- Supports parallel pipelines, sync barriers, and fan-out patterns
- Uses Memory-Keeper MCP for cross-session supplemental state
- Invokes the Agent tool (Task) for worker spawning

These features required a Phase 0 spike to determine whether any native Codex parallel execution
mechanism could underpin the port, or whether the sequential-role fallback was required.

### Phase 0 Findings (Grounding Evidence)

**Finding 1 — Live subagent test (decisive).**
A minimal TOML agent (`test-worker.toml`) was created and a `codex exec` prompt instructed Codex
to delegate to it. Codex exposed and invoked `SpawnAgent` three times. Each invocation failed:
`ERROR codex_core::tools::router: error=spawn_agent could not resolve the child model for service
tier validation`. `proof.txt` was NOT created. Spawning is architecturally present but operationally
blocked on this account.
Source: `eng/phase-0/eng-arch-001/live-subagent-test.md`

**Finding 2 — Feature flags.**
`codex features list` shows: `multi_agent: stable, true` (UI/interactive scope); `enable_fanout:
under development, false` (the in-skill parallel spawn gate — not available); `multi_agent_v2:
under development, false` (next-generation mechanism, not yet released).
Source: `eng/phase-0/eng-arch-001/codex-subagent-mechanism.md`

**Finding 3 — Production port precedent.**
Both `~/.codex/skills/adversary/SKILL.md` and `~/.codex/skills/eng-team/SKILL.md` explicitly
state: "Codex has a single execution context and no sub-agent spawning, so the N 'agents' are
**roles** you adopt one at a time by loading the matching file in `references/`." These ports were
authored after the `multi_agent: stable, true` feature existed and represent the confirmed pattern.
Source: `eng/phase-0/eng-arch-001/codex-subagent-mechanism.md` §2

**Finding 4 — Frontmatter validation.**
The Codex `quick_validate.py` enforces a strict allowlist:
`{"name", "description", "license", "allowed-tools", "metadata"}`. Jerry's `version` and
`activation-keywords` fields are NOT in the allowlist and cause a validator ERROR (not silent ignore).
The `allowed-tools` field IS permitted but has different Codex-native semantics; existing port
convention omits it.
Source: `eng/phase-0/ps-rsch-001/codex-official-standards.md` §1, Resolved-VERIFY Table

**Finding 5 — Memory-Keeper not available.**
`codex mcp list` shows no Memory-Keeper MCP server registered. Codex's MCP integration requires
explicit `codex mcp add` registration. ORCHESTRATION.yaml file-based persistence is already the
P-002 primary mechanism; Memory-Keeper was supplemental. No functional loss from its absence.
Source: `eng/phase-0/eng-arch-001/codex-subagent-mechanism.md` §3

**Finding 6 — Jerry source executes sequentially in practice.**
The Jerry `/orchestration` skill's `execution_mode: PARALLEL` in ORCHESTRATION.yaml is planning
metadata, not an actual concurrent execution mechanism. Within a single Claude context, pipelines
execute sequentially even when marked PARALLEL. The Codex port is functionally equivalent.
Source: `eng/phase-0/eng-arch-001/codex-subagent-mechanism.md` L0, L2

**Adversary verdict (Barrier 0):** GO-WITH-CONSTRAINTS. Option A confirmed as baseline.
Option B (subagent-parallel) BLOCKED by live test result. CSP corrections CC-001 required before
Phase 2.
Source: `cross-pollination/barrier-0/adv-to-eng/feasibility-verdict.md`

---

## Decision

**Selected: Option A — Sequential-Role Port**

The port maps each Jerry orchestration agent to a named role file in `references/`. Within a single
`codex exec` context, roles are adopted sequentially by reading the matching file and following its
instructions. No subagent spawning occurs. ORCHESTRATION.yaml on disk is the coordination surface.

This matches the pattern established by the `adversary` and `eng-team` Codex ports verbatim.

### Decision Rationale

| Factor | Assessment |
|--------|------------|
| Live subagent test | FAILED — `SpawnAgent` blocked by service-tier/child-model error ×3 |
| `enable_fanout` | `under development, false` — parallel fan-out not available |
| Production precedent | Both adversary and eng-team use sequential-role; confirmed in production |
| Jerry source parallelism | Planning metadata only; source executes sequentially in Claude context |
| Memory-Keeper availability | Not registered in Codex; file-based state is sufficient and P-002 primary |
| Frontmatter constraint | Strict allowlist confirmed; Jerry-only fields must be dropped |
| Coordination limitation | All primitives map to Codex file/role primitives EXCEPT automated worker-agent coordination (manual `codex exec` + report to orch-tracker); see Addendum A.1 |

### What "Sequential-Role" Means for Orchestration

The Jerry orchestration skill uses three agents in distinct temporal roles. The Codex port
preserves this sequencing explicitly:

| Step | Jerry source | Codex port |
|------|-------------|------------|
| 1 | `orch-planner` spawned to produce plan | Adopt orch-planner role: read `references/orch-planner.md`, produce `ORCHESTRATION_PLAN.md` and `ORCHESTRATION.yaml` |
| 2-N | Worker agents execute per plan | Workflow-defined work agents execute in context (not orchestration's responsibility) |
| After each completion | `orch-tracker` spawned to update state | Re-adopt orch-tracker role: read `references/orch-tracker.md`, read ORCHESTRATION.yaml, write updated state |
| After each phase/barrier | Quality gate: creator → critic cycle | Quality gate role cycle within context (creator output → critic assessment in same context) |
| Final | `orch-synthesizer` spawned for synthesis | Adopt orch-synthesizer role: read `references/orch-synthesizer.md`, read all artifacts, write synthesis |

**Critical distinction from adversary/eng-team:** The `orch-tracker` role is **re-entrant** — it
is adopted multiple times during a workflow (once after each agent or phase completion). The
`orch-planner` and `orch-synthesizer` roles are adopted once each. The SKILL.md body must make
this re-entrant pattern explicit.

### SKILL.md Frontmatter Decision

The Codex SKILL.md frontmatter contains ONLY `name` and `description`, exactly matching the
adversary and eng-team port pattern:

```yaml
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
```

Jerry-only fields (`version: "2.2.0"`, `activation-keywords`, `allowed-tools` with Jerry
tool-list value) are dropped. The port version is tracked in the SKILL.md body header as
`codex-1.0.0`.

---

## Consequences

### Enabled by this decision

1. **Immediate deployability.** The sequential-role port works today on Codex 0.142.5 with no
   account-tier changes, no `enable_fanout` release, no MCP server configuration. Zero new
   infrastructure dependencies.

2. **Structured-coordination capability, with an explicit limitation (P-022 honest disclosure).**
   Structured workflow planning, persistent YAML state, checkpoint/resume, quality gates
   (creator-critic-revision cycle), cross-pollination handoffs (file-based), and synthesis are
   available. **Automated worker-agent coordination is NOT available in this Codex port:** users
   invoke work agents via separate `codex exec` calls and report completion to the orch-tracker role.
   The skill's value is structured planning, persistent state tracking, quality-gate enforcement, and
   synthesis coordination — not automatic multi-agent execution (see Addendum A.1).

3. **Consistency with established port pattern.** The `adversary` and `eng-team` ports set the
   convention; this port follows it exactly. Users familiar with those ports will recognize the
   mental model immediately.

4. **No write-concurrency issues.** Sequential execution makes ORCHESTRATION.yaml the safe single
   coordination surface with no file-contention risk.

5. **P-003 compliant by design.** No subagent spawning means no recursive delegation, no context
   isolation per agent. The single-context constraint is enforced architecturally, not just by
   convention.

### Foreclosed by this decision

1. **True wall-clock parallelism is not available.** When a workflow declares
   `execution_mode: PARALLEL`, the Codex port executes the agents sequentially within one context.
   Workflows that rely on actual concurrent execution time savings cannot obtain them.

2. **Independent context isolation per agent is not available.** In the Jerry source, each spawned
   worker agent has a fresh context window (no accumulated reasoning from prior agents). In the
   Codex port, all role adoptions share the same context window. Long workflows may experience
   context pressure.

3. **Memory-Keeper cross-session search is not available.** The MCP-based ability to search across
   multiple workflow sessions is not available. Resumption is limited to reading ORCHESTRATION.yaml
   from the current project directory.

### Deferred by this decision

**DEC-001 (reversible): Option B subagent-parallel upgrade path.**
When both conditions clear — (a) the service-tier/child-model entitlement error is resolved on
this account and (b) `enable_fanout` ships and goes `true` — the sequential-role architecture can
be upgraded to genuine parallel subagent execution. The upgrade path is documented in
[Appendix C](#appendix-c-subagent-ready-upgrade-path). This deferral is explicitly reversible and
non-breaking; the upgrade path produces a new SKILL.md body and TOML agent definitions without
invalidating existing ORCHESTRATION.yaml files.

**DEC-002 (reversible): `multi_agent_v2` evaluation.**
When `multi_agent_v2` stabilizes (currently `under development, false`), re-evaluate whether it
provides in-skill subagent spawning semantics that differ from the current `SpawnAgent` mechanism.
Trigger: `codex features list` shows `multi_agent_v2: stable, true`.

---

## Appendix A: Primitive Mapping Table

Each orchestration primitive in the Jerry source is mapped to its Codex equivalent under the
sequential-role architecture. Confidence is grounded in Phase 0 evidence.

| # | Primitive | Jerry Implementation | Codex Mapping (Option A) | Confidence | Residual Gap |
|---|-----------|---------------------|--------------------------|------------|--------------|
| P-1 | Parallel pipeline execution | Two+ agent pipelines in `execution_queue.groups` with `execution_mode: PARALLEL` running concurrently | Sequential phase execution in one context. `execution_mode: PARALLEL` is informational metadata only; the SKILL.md body must state this explicitly. Pipelines A and B execute as sequential role phases. | High (gap confirmed by live test) | Wall-clock parallelism unavailable. Functional outputs identical. |
| P-2 | Sync barrier | Named checkpoint where all pipelines converge; barrier `depends_on` conditions must all be `COMPLETE` before crossing | Sequential role-transition with quality gate. Barrier becomes an explicit step: creator output → orch-tracker records score → critic role assesses → gate check → COMPLETE or REVISE. Barrier entry/exit criteria documented in references/orch-tracker.md | High | Enforcement is LLM-behavioral rather than mechanistic. Mitigation: precise barrier completion criteria in role file. |
| P-3 | Cross-pollination handoff | `barriers[].artifacts.{direction}` paths; one pipeline writes a handoff artifact, the other reads it before Phase N+1 | File-based handoff. Role A writes artifact to `orchestration/{workflow_id}/cross-pollination/{barrier_id}/{source}-to-{target}/handoff.md`. Role B reads it before proceeding. Codex Read/Write tools support this directly. | High | None. File write then file read is functionally equivalent to the cross-pollination artifact. |
| P-4 | ORCHESTRATION.yaml state | YAML SSOT; read-before-write protocol; atomic updates via Edit tool | Preserved as-is. Codex Write/Edit/Read tools support YAML directly. The role files instruct orch-tracker to read before any write (already mandated by the source agent). | High | Memory-Keeper supplemental layer absent (see P-8). No functional loss. |
| P-5 | Checkpoint / resume | `checkpoints[]` entries in ORCHESTRATION.yaml; `resumption` section with `files_to_read`; `codex resume --last` for interactive sessions | YAML state + `codex exec` re-run pointing at existing ORCHESTRATION.yaml. orch-tracker writes checkpoint entries; `resumption.files_to_read` lists artifacts to load on resume. Resume instruction: run `codex exec` with the prompt "Resume workflow from ORCHESTRATION.yaml in {project_dir}" — the skill reads the resumption section and continues from `resumption.next_step`. | Medium | `codex exec resume` (interactive session resume) works at session level. For non-interactive resume, YAML must carry sufficient recovery context in `resumption.next_step`. Design ORCHESTRATION.yaml `resumption` section to be fully self-describing. |
| P-6 | Quality gate (phase/barrier) | S-014 LLM-as-Judge >= 0.92; minimum 3 creator-critic-revision iterations; H-13/H-14 | Sequential role cycle in single context: creator role produces output → orch-tracker role records state → critic role (LLM self-assessment applying S-014 rubric) → if score < 0.92 and iterations < 3: revise; if >= 0.92: PASS; if 3 iterations exhausted: ESCALATE. Pattern identical to adversary port's executor-scorer sequence. | High (conditional on V-7a calibration) | **Major risk (FM-007):** same-context critic anchoring can inflate gate scores (no fresh context isolation). Mitigation: V-7a runs the gate once inline and once as a fresh `codex exec` (artifact+rubric only); if divergence > 0.08 (RT-M-012), add explicit context-clearing to the orch-tracker role. |
| P-7 | Agent tool (worker spawn) | `Task(description="...", subagent_type="general-purpose", prompt="...")` in orch-planner, orch-tracker, orch-synthesizer `<invocation>` sections | NOT available. Agent/Task tool requires T5 orchestration tier and SpawnAgent which is blocked. The `<invocation>` sections in Jerry agent files become role-prompt instructions in references/ files instead. | High (gap confirmed by live test) | Context isolation per agent unavailable. Role files absorb the invocation template as inline instructions. |
| P-8 | Memory-Keeper MCP | `mcp__memory-keeper__context_save/get/search`; key `jerry/{project}/orchestration/{id}` | Not available (no Memory-Keeper MCP server in Codex environment). ORCHESTRATION.yaml file-based persistence is the sole cross-session mechanism. This is already P-002 primary; Memory-Keeper was supplemental. No functional loss for the primary use cases. | High | Cross-session search across multiple workflow IDs is unavailable. Single-workflow resumption via ORCHESTRATION.yaml is fully supported. |
| P-9 | Fan-out / fan-in pattern | Multiple agents in one `execution_queue.groups` entry with `execution_mode: PARALLEL` | Sequential fan-out approximation. Each fan-out agent's role is adopted sequentially; results aggregate in ORCHESTRATION.yaml before the fan-in synthesizer role. Functional output identical; timing serialized. | High | Wall-clock time serialization only. All outputs are file-based; synthesis role reads all of them regardless of creation order. |
| P-10 | Workflow ID + dynamic paths | `workflow.id`, `pipelines.{x}.short_alias`, dynamic path scheme `orchestration/{id}/{alias}/phase-N/{agent_id}/` | Preserved as-is. Path scheme is purely file-system naming. ORCHESTRATION.yaml carries all ID and alias values. orch-tracker role resolves paths from YAML before writing (existing protocol carried forward). | High | None. |
| P-11 | Criticality-based adversarial strategy selection | C1-C4 levels; required strategy sets per quality-enforcement SSOT; embedded in ORCHESTRATION_PLAN.md by orch-planner | Preserved as-is. Strategy selection is LLM-behavioral, embedded in orch-planner role prompt. The Codex `adversary` skill (`~/.codex/skills/adversary/`) is already installed and available for external adversarial review requests. Quality gate cycling (P-6) implements the in-workflow strategy execution inline. | High | None. |

---

## Appendix B: File Manifest

Phase 2 (Build) must produce exactly these files. The canonical location is
`~/.codex/skills/orchestration/`. The git-tracked mirror is
`codex-ports/orchestration/` in the Jerry repository (CSP-03).

| # | File path (relative to skill root) | Source / Generation method | Phase 0 evidence |
|---|------------------------------------|---------------------------|------------------|
| 1 | `SKILL.md` | Authored from `templates/codex-ported-SKILL.template.md` applying the port decisions in this ADR. Frontmatter: `name` + `description` only. Body: divergence header (`codex-1.0.0`, forked from `/orchestration` v2.2.0, 2026-07-04) + Codex-specific instructions for the 3-role sequential cycle. | adversary/eng-team SKILL.md pattern; ps-rsch-001 frontmatter findings |
| 2 | `agents/openai.yaml` | Authored from `templates/codex-ported-openai.template.yaml`. Fields: `interface.display_name`, `interface.short_description` (25-64 chars), `interface.default_prompt` (descriptive sentence; no `$<skill-name>` literal — existing port convention omits it). No `dependencies` block (no MCP). `policy.allow_implicit_invocation: true`. | adversary/eng-team openai.yaml pattern; ps-rsch-001 §3 |
| 3 | `references/orch-planner.md` | Distilled from `skills/orchestration/agents/orch-planner.md`. Strip Jerry-specific: `<invocation>` Task() template, `<memory_keeper_integration>` section, `mcpServers` frontmatter. Retain: identity, expertise, cognitive mode, workflow ID generation, alias resolution, quality gate planning, output format (ORCHESTRATION_PLAN.md + ORCHESTRATION.yaml), forbidden actions, guardrails. Rewrite `<capabilities>` to remove Agent tool; tools available to Codex are Read/Write/Edit/Glob/Grep/Bash. | eng-arch-001 §2; adversary/references/ pattern |
| 4 | `references/orch-tracker.md` | Distilled from `skills/orchestration/agents/orch-tracker.md`. Strip: `<invocation>` Task() template, `<memory_keeper_integration>`. Retain: identity, state update protocol, dynamic path resolution, quality score tracking, gate enforcement (PASS/REVISE/ESCALATED), checkpoint creation, output format. Add explicit note: "This role is re-entrant — adopt it after every agent or phase completion." | eng-arch-001 §2; adversary/references/ pattern |
| 5 | `references/orch-synthesizer.md` | Distilled from `skills/orchestration/agents/orch-synthesizer.md`. Strip: `<invocation>` Task() template, `<memory_keeper_integration>`. Retain: identity, synthesis protocol (5 steps), adversarial synthesis protocol (S-014, S-013, S-003), quality trend analysis, output format (final synthesis document). | eng-arch-001 §2; adversary/references/ pattern |
| 6 | `docs/PATTERNS.md` | Carried from `skills/orchestration/docs/PATTERNS.md` with one targeted edit: add a note to Pattern 1 (Cross-Pollinated Pipeline) and Pattern 3 (Fan-Out/Fan-In) stating "In the Codex port, PARALLEL execution groups are executed sequentially (see SKILL.md divergence note). Functional outputs are identical." No other changes needed. | eng-arch-001 §4 (P-1, P-9 mappings) |
| 7 | `docs/STATE_SCHEMA.md` | Carried from `skills/orchestration/docs/STATE_SCHEMA.md` with one targeted edit: remove the `mcpServers: memory-keeper: true` front-matter equivalents if any appear in schema narrative; add a note in the `resumption` section: "In the Codex port, Memory-Keeper MCP is unavailable; ORCHESTRATION.yaml `resumption` section is the sole cross-session mechanism. Ensure `resumption.files_to_read` is complete and `resumption.next_step` is self-describing." | eng-arch-001 §3 (P-8 mapping) |
| 8 | `templates/ORCHESTRATION.template.yaml` | Carried verbatim from `skills/orchestration/templates/ORCHESTRATION.template.yaml`. No changes required — schema is file-native and Codex-agnostic. | eng-arch-001 §4 (P-4 mapping) |
| 9 | `templates/ORCHESTRATION_PLAN.template.md` | Carried verbatim from `skills/orchestration/templates/ORCHESTRATION_PLAN.template.md`. No changes required — markdown template is tool-agnostic. | eng-arch-001 §4 (P-4 mapping) |
| 10 | `templates/ORCHESTRATION_WORKTRACKER.template.md` | Carried verbatim from `skills/orchestration/templates/ORCHESTRATION_WORKTRACKER.template.md`. No changes required — markdown template is tool-agnostic. | eng-arch-001 §4 (P-4 mapping) |

**Total: 10 files.** All 10 are required before the port is considered complete for Phase 3
verification.

**Phase 2 template fill-in reference for `agents/openai.yaml`:**

```yaml
# agents/openai.yaml — Codex port version: codex-1.0.0
interface:
  display_name: "Orchestration — Multi-Agent Workflow Coordinator"
  short_description: "Multi-agent workflow orchestration with state tracking and checkpointing."
  default_prompt: "Plan and execute a structured multi-agent workflow with phase tracking, sync barriers, quality gates, and checkpoint resumption using the orchestration skill."

policy:
  allow_implicit_invocation: true
```

Note on `default_prompt`: The CSP standard cites the openai_yaml.md spec requirement to mention
`$<skill-name>`. The production `adversary` and `eng-team` ports omit the literal `$adversary` /
`$eng-team` reference and use descriptive prose instead. Phase 2 SHOULD follow the production
port convention (descriptive prose). If the Codex skill-creator validation requires the `$`-prefix
literal in a future version, add `$orchestration` to the sentence at that time.

---

## Appendix C: Subagent-Ready Upgrade Path

**DEC-001: Deferred. Reversible. Non-blocking.**

The sequential-role port (Option A) is designed so that the natural-language delegation phrasing
in the SKILL.md body is already compatible with genuine subagent invocation under Option B. The
upgrade path does not require changing ORCHESTRATION.yaml schema, templates, or documents.

### Current behavior (Option A)

The SKILL.md body instructs Codex with natural-language role adoption:

> "To create the workflow plan: read `references/orch-planner.md` and adopt the orch-planner role.
> Follow the role's instructions to produce ORCHESTRATION_PLAN.md and ORCHESTRATION.yaml."

Codex executes this as an in-context role adoption: the file is read, and Codex follows the role
instructions within the existing context window.

### Upgrade preconditions (both must be true)

| Precondition | Current state | Signal that it has cleared |
|-------------|---------------|---------------------------|
| (a) Service-tier / child-model entitlement resolved | BLOCKED — `SpawnAgent` fails with child-model error ×3 | `codex exec` with a TOML agent delegate produces a file without error |
| (b) `enable_fanout` released and active | `under development, false` | `codex features list` shows `enable_fanout: stable, true` |

### Upgrade actions (when both preconditions clear)

1. **Create TOML agent definitions** from the role files:
   - `~/.codex/agents/orch-planner.toml` — content: `name`, `description`, `developer_instructions` drawn from `references/orch-planner.md` role body
   - `~/.codex/agents/orch-tracker.toml` — same pattern
   - `~/.codex/agents/orch-synthesizer.toml` — same pattern

2. **Update SKILL.md body** delegation language to:
   > "To create the workflow plan: delegate to the orch-planner agent to produce ORCHESTRATION_PLAN.md and ORCHESTRATION.yaml."
   (Codex's natural-language orchestration will trigger SpawnAgent against the TOML definition.)

3. **Add fan-out invocation** for Pattern 3 (Fan-Out/Fan-In): when `enable_fanout` is active,
   the SKILL.md body can instruct fan-out worker agents to execute in parallel, with
   `agents.max_threads` (default 6) as the concurrency ceiling.

4. **Bump port version** to `codex-2.0.0` in the SKILL.md body header to mark the Option B
   transition. Update `codex-ports/orchestration/SKILL.md` in the Jerry repository.

5. **Verify** via the Phase 3 test suite (extended): a sample workflow must show two agents
   executing in parallel (different output files created concurrently) and the fan-in synthesizer
   consuming both.

### What does NOT change in the upgrade

- ORCHESTRATION.yaml schema and all template files are unchanged
- Docs, patterns, and state schema remain valid
- Existing ORCHESTRATION.yaml files from Option A workflows are fully forward-compatible
- Role file content in `references/` is preserved; the TOML `developer_instructions` field can
  reference or copy the role file content

---

## Appendix D: CSP Corrections Applied (AC-1.2)

The Codex Skill Port Standard (`standards/codex-skill-port-standard.md`) required corrections
from Phase 0 findings before Phase 2 can proceed. Both CC-001 items from the Barrier 0 adversary
verdict are addressed here and must be confirmed as applied to the standard.

### CC-001-A: Validator error, not silent ignore

**Phase 0 finding (ps-rsch-001, Resolved-VERIFY Table):**
`quick_validate.py` lines 40-49: `if unexpected_keys: return False, f"Unexpected key(s) in
SKILL.md frontmatter: {unexpected}. Allowed properties are: {allowed}"` — this is a failure
return, not silent ignore.

**CSP correction required (Barrier 0 feasibility-verdict.md CC-001):**
The CSP Frontmatter Standards table row for Jerry-only fields must read:

> "MUST be dropped — they are NOT in the allowlist and cause a **validator ERROR** (not silent
> ignore). Jerry's `version` and `activation-keywords` fields will fail `quick_validate.py` and
> MUST be excluded from the Codex SKILL.md."

**Status:** The CSP (`standards/codex-skill-port-standard.md`) was updated to reflect this
correction (Frontmatter Standards table, row "Codex validator allowlist" and "Jerry-only fields")
as part of Phase 0 deliverables. Confirm the file reflects this before Phase 2 template
instantiation.

### CC-001-B: `allowed-tools` is permitted but carry-over is forbidden

**Phase 0 finding (ps-rsch-001):**
`allowed-tools` IS in the validator allowlist. However: (a) it uses Codex-native semantics
(space-separated tool string: `Bash(git:*) Read`), not Jerry's format; (b) enforcement is
experimental; (c) existing port convention omits it.

**CSP correction required:**
The CSP must state:

> "`allowed-tools` — **Permitted** by the Codex 0.142.5 validator. However: Codex semantics
> differ (space-separated string: `Bash(git:*) Read`); enforcement is experimental; existing port
> convention (adversary, eng-team) omits it. **MUST NOT** copy Jerry's `allowed-tools` value.
> Recommend omitting unless specific tool restriction is required."

**Status:** The CSP was updated as part of Phase 0 deliverables (see CSP Frontmatter Standards
table, `allowed-tools` row marked "RESOLVED"). Confirm before Phase 2.

### Template edit status: no residual issues

The `codex-ported-SKILL.template.md` template correctly contains only `name: {skill-name}` and
`description: >-` in frontmatter (no `version`, `activation-keywords`). The template is
consistent with the corrected CSP. No further template edits are required for CSP compliance.

The `codex-ported-openai.template.yaml` template correctly omits `dependencies` by default
(commented out) and sets `allow_implicit_invocation: true`. No changes required.

---

## Appendix E: Phase 3 Verification Hooks

A successful Phase 3 live verification run must demonstrate all of the following criteria against
the built port (`~/.codex/skills/orchestration/` installed, files from the Phase 2 manifest).

| # | Criterion | Test action | PASS condition | Evidence to capture |
|---|-----------|-------------|----------------|---------------------|
| V-1 | Frontmatter valid | Run `quick_validate.py` against `SKILL.md` | Exits with PASS; no unexpected keys error | CLI output showing "VALID" or equivalent |
| V-2 | Implicit skill trigger | Run `codex exec "I need to orchestrate a multi-phase workflow with sync barriers"` (no explicit `$orchestration`) | Codex selects the orchestration skill and adopts orch-planner role without explicit invocation | Session transcript showing skill selection |
| V-3 | orch-planner role adoption | Run `codex exec "Use orchestration to plan a two-phase sequential workflow for PROJ-037-test"` | `ORCHESTRATION_PLAN.md` and `ORCHESTRATION.yaml` files are created on disk in the working directory | Verify files exist: `ls ORCHESTRATION_PLAN.md ORCHESTRATION.yaml` |
| V-4 | ORCHESTRATION.yaml schema validity | Inspect the file created in V-3 | File is valid YAML; contains required fields: `workflow.id`, `workflow.status: PLANNED`, `pipelines`, `checkpoints`, `resumption` | `yq . ORCHESTRATION.yaml` returns without parse error |
| V-5 | orch-tracker re-entrant adoption | Run `codex exec "Update ORCHESTRATION.yaml: agent phase-1-agent-001 is now COMPLETE, artifact at orchestration/test-001/ps/phase-1/agent-001/output.md"` pointing at the YAML from V-3 | ORCHESTRATION.yaml is updated: agent status = COMPLETE, artifact path recorded, metrics recalculated | Diff ORCHESTRATION.yaml before and after |
| V-6 | Checkpoint creation | Run V-5 with the prompt including "Phase 1 is now complete, create a checkpoint" | `checkpoints.entries` in ORCHESTRATION.yaml has a new entry with `trigger: PHASE_COMPLETE` and `recovery_point` populated | Inspect `checkpoints.entries` in YAML |
| V-7 | Quality gate enforcement | Run `codex exec` instructing the skill to run a quality gate on a sample phase output | The skill produces a quality assessment with a numeric score (0.0-1.0) and a PASS/REVISE/ESCALATED verdict | Verify score is numeric and verdict is one of the three valid values |
| V-8 | Cross-pollination handoff | Run a mini two-phase workflow where Phase 1 writes a handoff artifact and the skill instructs Phase 2 to read it before proceeding | File `orchestration/test-001/cross-pollination/barrier-1/{dir}/handoff.md` exists on disk | `ls orchestration/test-001/cross-pollination/barrier-1/` |
| V-9 | orch-synthesizer role adoption | After V-5/V-6: run `codex exec "All phases are complete for workflow test-001, run final synthesis"` | `synthesis/{workflow_id}-final-synthesis.md` created on disk; ORCHESTRATION.yaml `workflow.status` = COMPLETE | Verify synthesis file exists and YAML status field |
| V-10 | Resume from checkpoint | Mark ORCHESTRATION.yaml `workflow.status: PAUSED`; run `codex exec "Resume workflow test-001 from ORCHESTRATION.yaml"` | Codex reads `resumption.files_to_read` and `resumption.next_step`; resumes from correct point without re-running completed phases | Session transcript shows checkpoint reading and correct resume point |

**Sample workflow for V-3 through V-9 (minimum viable end-to-end):**

```
Workflow ID: codex-verify-20260704-001
Pipeline A: problem-solving (alias: ps), 2 phases
No Pipeline B (sequential pattern, no sync barriers) — simplest possible proof
Phase 1 output: orchestration/codex-verify-20260704-001/ps/phase-1/agent-001/research.md (mock)
Phase 2 output: orchestration/codex-verify-20260704-001/ps/phase-2/agent-002/analysis.md (mock)
Final synthesis: orchestration/codex-verify-20260704-001/synthesis/final-synthesis.md
```

This sample workflow exercises all 3 roles (planner, tracker ×2, synthesizer), checkpointing,
and state progression without requiring the full complexity of a cross-pollinated two-pipeline
workflow. A cross-pollinated verification (P-3 handoff) can be run as a supplemental V-8 test.

---

## L2: Strategic Implications

### Long-term architecture posture

The sequential-role pattern is the permanent architecture for Codex 0.142.5 and should not be
treated as a temporary workaround. Both `adversary` and `eng-team` production ports have operated
on this pattern without deficiency reports. The functional value of the orchestration skill —
structured workflow coordination, state tracking, quality gating, and cross-pollination — is
delivered without true parallelism.

### Divergence point and version tracking

The Codex port version line `codex-1.0.0` starts fresh, independent of the Jerry source
`v2.2.0`. The SKILL.md body header records:
- Forked from: Jerry `/orchestration` v2.2.0
- Divergence date: 2026-07-04
- Divergences: (1) `execution_mode: PARALLEL` informational only; (2) Memory-Keeper MCP absent;
  (3) Agent tool (subagent spawning) absent; (4) all three agents are roles, not subagents

Future Jerry source upgrades to `/orchestration` (e.g., new patterns, schema additions, agent
capability enhancements) may not map automatically to the Codex port if they depend on the Agent
tool, Memory-Keeper, or a new tool not available in Codex. Review each source version bump against
the divergence list before applying to the Codex port.

### Security posture

The sequential-role pattern is architecturally simpler from a security perspective than subagent
spawning: there is one context, one file-system actor, and one trust domain. No cross-context
token transfer occurs. ORCHESTRATION.yaml write operations are the only persistent mutation
surface; all writes are explicit and governed by the orch-tracker role protocol (read before write,
atomic update, no partial state).

### NIST CSF 2.0 alignment

| CSF Function | This decision |
|-------------|---------------|
| Identify | Orchestration plan explicitly identifies all agents, phases, and trust boundaries per workflow |
| Protect | File-based persistence (P-002) ensures state is not lost; quality gate enforcement (>= 0.92) prevents low-quality artifacts from propagating |
| Detect | ORCHESTRATION.yaml `blockers.active` and `quality.phase_scores` enable workflow anomaly detection |
| Respond | Checkpoint/resume mechanism supports graceful recovery from session interruption |
| Recover | `resumption.files_to_read` and `resumption.next_step` provide self-describing recovery instructions |

### Upgrade signals to watch

| Signal | Monitoring action | Implication |
|--------|-------------------|-------------|
| `codex features list` shows `enable_fanout: stable, true` | Quarterly check or when Codex CLI version bumps | One of two preconditions for DEC-001 upgrade cleared |
| `SpawnAgent` succeeds in a fresh live test (no child-model error) | Test when account tier changes or Codex CLI major version releases | Second precondition for DEC-001 upgrade cleared |
| `codex features list` shows `multi_agent_v2: stable, true` | Quarterly check | Evaluate DEC-002: whether v2 provides in-skill spawning semantics |
| Jerry source `/orchestration` version bump | On each Jerry release that touches `skills/orchestration/` | Review against divergence list; patch Codex port if applicable |

---

*ADR ID: PROJ-037-ADR-001*
*Agent: eng-architect (Phase 1 — worker, no subagents spawned, P-003 compliant)*
*Workflow: codex-port-20260704-001*
*Evidence grounding: Phase 0 artifacts (codex-subagent-mechanism.md, codex-official-standards.md, live-subagent-test.md, feasibility-verdict.md)*
*Handoff target: Phase 2 (Build) — eng-lead or direct build execution*
*Created: 2026-07-04*

---

## Addendum A — Barrier 1 Required Revisions (iteration 2)

> Incorporated after the Barrier 1 adversarial design review (see
> `cross-pollination/barrier-1/adv-to-eng/design-review.md`). These are BINDING requirements on the
> Phase 2 build and Phase 3 verification. They resolve the review's Major findings.
>
> **Source citations:** each `FM-xxx` ID and its RPN value below is drawn from the FMEA table in
> `adv/findings/barrier-1-findings.md` §FMEA (adv-executor, Barrier 1 iteration 1): FM-005 RPN 384
> (cross-pollination read), FM-004 RPN 336 (tracker saturation), FM-007 RPN 294 (critic anchoring),
> FM-002 RPN 240 (workflow loop), FM-008 RPN 240 (resume detection), FM-006 RPN 54 (default_prompt),
> FM-001 RPN 18 (frontmatter validator).

### A.1 Honest disclosure (P-022) — supersedes the "functional parity" claim
The L0 Executive Summary and Consequences MUST state, verbatim intent: "Automated worker-agent
coordination is NOT available in this Codex port. Users invoke work agents via separate `codex exec`
calls and report completion to the orch-tracker role. The skill's value is structured planning,
persistent state tracking, quality-gate enforcement, and synthesis coordination." Remove any
unqualified "functional parity" wording.

### A.2 Binding Phase 2 build requirements
- **B-R1 (FM-005, RPN 384):** `references/orch-planner.md` AND `templates/ORCHESTRATION_PLAN.template.md`
  MUST include explicit cross-pollination READ instructions — before a receiving pipeline's next
  phase runs, it MUST read the other pipeline's `cross-pollination/{barrier}/{src}-to-{dst}/handoff.md`.
- **B-R2 (FM-002, RPN 240):** `SKILL.md` body MUST contain an explicit NUMBERED workflow loop:
  (1) adopt orch-planner; (2) per phase: (2a) work agent executes, (2b) adopt orch-tracker → update
  ORCHESTRATION.yaml, (2c) check gate score, (2d) PASS→proceed / <3 iters→revise / else→ESCALATE;
  (3) adopt orch-synthesizer.
- **B-R3 (FM-008, RPN 240):** `SKILL.md` body MUST specify resume detection — on any invocation,
  check for an existing ORCHESTRATION.yaml with `workflow.status: PAUSED` and branch to resume rather
  than overwrite.
- **B-R4 (FM-004, RPN 336):** `references/orch-tracker.md` MUST instruct offset/limit reads on
  ORCHESTRATION.yaml when it exceeds ~2,000 tokens (load only workflow.status, current phase, quality
  metrics), because orch-tracker is re-entrant (adopted once per phase/agent completion).
- **B-R5 (FM-006):** resolve `agents/openai.yaml` `default_prompt` — either include `$orchestration`
  or document the production-port precedent as a CSP-04 exception.

### A.3 Binding Phase 3 verification hooks (supersede the minimal sample workflow)
- **V-7a (FM-007):** run one quality gate inline AND once as a fresh `codex exec` (artifact+rubric
  only); if score divergence > 0.08 (RT-M-012), add explicit context-clearing to orch-tracker.
- **V-8b (FM-005):** after a handoff artifact is written, assert the receiving pipeline's next phase
  performs a Read of that handoff before generating output (session transcript must show the Read).
- **V-12 (FM-004):** adopt orch-tracker 6+ times in one session; verify ORCHESTRATION.yaml state
  accuracy on the 6th adoption.
- **Sample workflow upgrade (DA-002):** the Phase 3 live run MUST use a 2-pipeline, 2-phase-each
  workflow WITH real bidirectional cross-pollination — not the minimal 1-pipeline/2-phase workflow.

### A.4 Acceptance
Phase 2 is not COMPLETE unless B-R1..B-R5 are implemented; Phase 3 is not COMPLETE unless V-7a, V-8b,
V-12 pass on the upgraded sample workflow (AC-3.2).
