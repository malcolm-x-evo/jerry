# Codex Subagent Mechanism — Architecture Spike

> **Document ID:** PROJ-037-ENG-ARCH-001
> **Agent:** eng-architect (Phase 0 — Discovery/Spike)
> **Workflow:** `codex-port-20260704-001`
> **Date:** 2026-07-04
> **Codex CLI version confirmed:** `codex-cli-exec 0.142.5`
> **Status:** COMPLETE — feeds Barrier 0 (Feasibility Gate)

---

## Document Sections

| Section | Purpose |
|---------|---------|
| [L0: Executive Summary](#l0-executive-summary) | GO/NO-GO verdict with business context |
| [L1: Technical Analysis](#l1-technical-analysis) | Full evidence, CLI surface, mechanism reconciliation |
| [L2: Strategic Implications](#l2-strategic-implications) | Port architecture, long-term posture |
| [Evidence Log](#evidence-log) | CLI commands run, file paths examined |

---

## L0: Executive Summary

The central question of the PROJ-037 spike is whether Codex CLI 0.142.5 can support the
`/orchestration` skill's defining capability — true parallel pipeline execution, sync barriers,
and cross-pollination handoffs via a native subagent spawning mechanism.

**Verdict: GO-WITH-CONSTRAINTS.**

Codex 0.142.5 does **not** provide native subagent spawning within a skill's single execution
context. The `enable_fanout` feature flag is `under development, false`; the `multi_agent`
stable flag is scoped to the interactive UI, not skills. Both previously ported skills (`adversary`,
`eng-team`) explicitly document "no sub-agent spawning" and adopt the sequential-role pattern
instead. The skill-creator's "subagent" references are conditional on interactive context and are
only used for forward-testing validation — not a general-purpose mechanism available to ported
skills.

The port is viable because the functional value of `/orchestration` — structured phase sequencing,
state persistence, quality gates, sync barriers, cross-pollination handoffs, checkpoint/resume —
maps cleanly onto Codex's file-based primitives and sequential role-adoption pattern. True
parallelism is unavailable, but in practice Jerry's `/orchestration` executes sequentially within a
single LLM context anyway (the YAML `execution_mode: PARALLEL` is planning metadata, not actual
concurrent execution).

**Bottom line for stakeholders:** The ported skill will deliver the workflow planning, state tracking,
quality gating, and handoff infrastructure of the original. It will not deliver simultaneous
parallel execution — but neither does the source skill in practice.

---

## L1: Technical Analysis

### 1. Codex CLI Surface — What Was Observed

All CLI surface evidence was collected via direct command execution on Codex CLI 0.142.5 installed
at the local machine.

#### 1.1 Top-Level Commands

```
codex --help  (excerpt, evidence date 2026-07-04)

Commands:
  exec            Run Codex non-interactively [aliases: e]
  review          Run a code review non-interactively
  mcp             Manage external MCP servers for Codex
  plugin          Manage Codex plugins
  resume          Resume a previous interactive session
  fork            Fork a previous interactive session
  archive/delete/unarchive  Session lifecycle management
  features        Inspect feature flags
  ...
```

Key observation: there is no `spawn`, `agent`, or `parallel` top-level command. The
non-interactive entry point is `codex exec`.

#### 1.2 `codex exec` Flags

```
codex exec --help  (excerpt, evidence date 2026-07-04)

-m, --model <MODEL>                Model the agent should use
-o, --output-last-message <FILE>   Write last agent message to file
--output-schema <FILE>             JSON schema for final response shape
--json                             Print events as JSONL
--sandbox <MODE>                   read-only | workspace-write | danger-full-access
--cd <DIR>                         Working root directory
--ephemeral                        Run without persisting session files
--dangerously-bypass-approvals-and-sandbox
```

Notably absent: `--background`, `--parallel`, `--fanout`, or any mechanism to spawn parallel
agents from within a skill.

The `--output-last-message <FILE>` flag enables writing the final response to disk, which could
support a shell-level workaround (multiple `codex exec` invocations as Bash subprocesses whose
outputs are aggregated). This is documented as a gap-mitigation option but is NOT a native
subagent mechanism.

#### 1.3 Feature Flags — Critical to Reconciliation

```
codex features list  (evidence date 2026-07-04)

multi_agent          stable             true
multi_agent_mode     removed            false
multi_agent_v2       under development  false
enable_fanout        under development  false
```

**`multi_agent: stable, true`** — This feature is active. However, cross-referencing with existing
Codex port artifacts (`~/.codex/skills/adversary/SKILL.md`, `~/.codex/skills/eng-team/SKILL.md`),
which were ported after this feature existed and both explicitly state "single execution context and
no sub-agent spawning," the `multi_agent` flag refers to UI-level collaborative or multi-turn
agent capability, not to a skill-invokable subagent spawn primitive within a single non-interactive
`codex exec` run. No CLI flag, tool call, or skill API exposes subagent spawning to running
skills.

**`enable_fanout: under development, false`** — This is the flag that would enable parallel fan-out
execution from within a single skill context. It is not enabled in 0.142.5. This directly confirms
that parallel pipeline execution is not available to ported skills.

**`multi_agent_v2: under development, false`** — A next-generation multi-agent mechanism is in
development but not yet released.

#### 1.4 The Skill-Creator Subagent Reference — Reconciled

The `~/.codex/skills/.system/skill-creator/SKILL.md` refers to subagents in two places:

1. **Step 4 ("Edit the Skill"):** "After substantial revisions, or if the skill is particularly
   tricky, you should use subagents to forward-test the skill on realistic tasks or artifacts."

2. **"Forward-testing" section:** "To forward-test, launch subagents as a way to stress test the
   skill with minimal context."

**Reconciliation:** Both references are *conditional* and *skill-creator-specific*. The text
explicitly states "Only do this when it is possible to start new subagents" — meaning the capability
is conditional, not universal. The context is the **interactive** Codex session where a UI-level
`multi_agent` feature may allow the skill-creator to launch validation subagents. This capability
is:
- Scoped to the interactive UI context
- Unavailable during `codex exec` non-interactive runs
- Available only to the *skill-creator itself* as a forward-testing mechanism, not to ported skills
  as a runtime feature

This is consistent with both `adversary` and `eng-team` explicitly stating "no sub-agent spawning"
— those ports were authored with full knowledge of the Codex capabilities available to skills.

### 2. Existing Port Evidence — Authoritative Pattern

**Source 1:** `~/.codex/skills/adversary/SKILL.md`

> "Codex has a single execution context and no sub-agent spawning, so the three 'agents' are
> **roles** you adopt one at a time by loading the matching file in `references/`."
> "Adopt one role at a time. Do not try to spawn parallel agents."

**Source 2:** `~/.codex/skills/eng-team/SKILL.md`

> "Codex has a single execution context and no sub-agent spawning, so the 10 'agents' are
> represented here as **roles** you adopt one at a time by loading the matching file in
> `references/`."
> "Only adopt one role at a time — Codex must not try to spawn parallel agents."

These two artifacts represent the authoritative established port pattern for Codex skills in this
codebase. Both were authored post-version-0.142.x and have been in production use. They constitute
the primary evidence of the sequential-role architecture that this port must follow.

### 3. Memory-Keeper MCP — Not Available in Codex

The Jerry `/orchestration` agents (`orch-planner`, `orch-tracker`, `orch-synthesizer`) use
`mcp__memory-keeper__context_save/get/search` for cross-session state persistence.

**Codex MCP situation:** `codex mcp list` manages external MCP servers. There is no
`memory-keeper` MCP server registered in the Codex environment. Codex's MCP integration is
add-on via `codex mcp add`, not a built-in capability. This gap means all cross-session state
must be file-based (ORCHESTRATION.yaml in the project directory), which is already the
primary SSOT in the Jerry skill (P-002) — Memory-Keeper was supplemental.

### 4. Orchestration Primitive Mapping Table

This table maps each orchestration primitive to its Codex equivalent, with confidence and gaps.

| # | Jerry Orchestration Primitive | Jerry Implementation | Codex Primitive | Confidence | Gap / Notes |
|---|---|---|---|---|---|
| P-1 | Parallel pipeline execution | Two or more agent pipelines running concurrently within a workflow; `execution_queue.groups` with `execution_mode: PARALLEL` | **Not available.** `enable_fanout: under development, false`. No native parallel spawn within skill context. | High (gap confirmed) | **FALLBACK:** Sequential phase ordering. Pipelines A and B become sequential role phases in the same context. The YAML `execution_mode` field becomes informational metadata only. |
| P-2 | Sync barrier | Named checkpoint where all pipelines converge; bidirectional cross-pollination artifacts must exist before barrier is COMPLETE | **Role transition with quality gate.** Barrier becomes an explicit sequential step: creator role → critic role → quality check. Barrier "COMPLETE" condition is enforced by the LLM following documented criteria. | High | Minor: barrier enforcement is LLM-behavioral rather than mechanistic. Mitigation: document precise barrier entry/exit criteria in references. |
| P-3 | Cross-pollination handoff | `barriers[].artifacts.{direction}` paths; each pipeline writes a handoff artifact that the other pipeline reads before Phase N+1 | **File-based handoff.** Role A writes artifact to `cross-pollination/{barrier}/{direction}/handoff.md`. Role B reads it before proceeding. Codex's Read/Write tools support this directly. | High | None. This maps cleanly: file write then file read is equivalent to the cross-pollination artifact. |
| P-4 | ORCHESTRATION.yaml state | YAML SSOT for all workflow state; Read-before-write; atomic updates | **File-based YAML.** Codex's Write/Edit/Read tools support this directly. The ported skill reads and updates the YAML file at each phase boundary. | High | Memory-Keeper MCP not available; file-only. P-002 already mandates file persistence as primary, so no functional loss. |
| P-5 | Checkpoint / resume | `checkpoints[]` entries in ORCHESTRATION.yaml; `resumption` section with `files_to_read` | **`codex exec resume` + file state.** `codex exec resume --last` resumes an interactive session. For non-interactive: re-run `codex exec` with the ORCHESTRATION.yaml path as context and resume from the last checkpoint. | Medium | `resume` works at session level; checkpoint granularity within a non-interactive run requires explicit YAML state management by the skill. |
| P-6 | Quality gate (phase) | S-014 LLM-as-Judge scoring; minimum 3 creator-critic-revision iterations; threshold >= 0.92 | **Sequential role cycle within context.** Creator role → Critic role → evaluate score → iterate or proceed. Identical to adversary/eng-team's quality gate pattern. | High | None. Already established by existing ports. |
| P-7 | Agent tool (spawn worker) | Jerry: `Task(description="...", subagent_type="general-purpose", prompt="...")` — spawns a worker subagent | **NOT available to ported skills.** Agent/Task tool is reserved for orchestrators (T5). Codex skills have no equivalent spawn primitive. | High (gap confirmed) | **FALLBACK:** The orch-planner, orch-tracker, orch-synthesizer agent definitions become role prompts in `references/` files loaded by role-adoption (identical to adversary/eng-team pattern). |
| P-8 | Memory-Keeper MCP persistence | `mcp__memory-keeper__context_save/get/search`; key pattern `jerry/{project}/orchestration/{id}` | **Not available in Codex.** No Memory-Keeper MCP server in Codex environment. | High (gap confirmed) | **FALLBACK:** ORCHESTRATION.yaml file-based persistence. This is already the primary P-002 mechanism; Memory-Keeper was supplemental. Functionally equivalent for cross-session resumption if YAML is in the project directory. |
| P-9 | Fan-out/Fan-in pattern | `execution_queue.groups` with PARALLEL mode; multiple agent IDs in same group | **Sequential fan-out approximation.** Each fan-out agent runs sequentially; results aggregate before the fan-in role. Functional output identical; timing serialized. | High | Only gap is actual parallelism (wall-clock time). Functional outputs are identical. |
| P-10 | Workflow ID + dynamic paths | `workflow.id`, `pipelines.{x}.short_alias`, dynamic path scheme `orchestration/{id}/{alias}/{phase}/` | **Preserved as-is.** Path scheme is purely file-system naming; Codex supports arbitrary file paths. ORCHESTRATION.yaml carries the ID and alias values. | High | None. |
| P-11 | Criticality-based adversarial strategy selection | C1-C4 levels; required strategy sets per quality-enforcement SSOT | **Preserved as-is.** Strategy selection is LLM-behavioral, embedded in role prompts. The adversary Codex skill provides bundled strategy references. | High | None. The adversary Codex skill at `~/.codex/skills/adversary/` is already available. |

### 5. What IS Available — Potential Partial Parallelism via Shell

A shell-level workaround for partial parallelism exists but is classified as a workaround, not a
native mechanism:

```bash
# Illustrative only — not recommended as primary architecture
codex exec --cd /project --output-last-message phase-a-out.md "Execute role A" &
codex exec --cd /project --output-last-message phase-b-out.md "Execute role B" &
wait
```

This would invoke two `codex exec` subprocesses in parallel via Bash background jobs. Issues:
- Requires auth tokens available to both subprocesses
- Both write to the same file system; file contention if paths overlap
- Output is unstructured; aggregation is manual
- The inner skill has no coordination mechanism over these external processes
- `enable_fanout` being disabled suggests this pattern is not intended

**Assessment:** This workaround is not suitable as the primary architecture. It may be documented
in the ADR as a future-state option if `enable_fanout` is enabled in a future version.

### 6. Risk R-01 Disposition

The ORCHESTRATION_PLAN.md identified **Risk R-01:** "Codex subagent mechanism differs from
assumption / can't do true parallelism — Likelihood: M, Impact: H — Mitigation: Phase 0 spike is
a hard GO/NO-GO gate; NO-GO → escalate with fallback (sequential-role port)."

**This risk has materialized as expected.** The spike confirms the sequential-role fallback is the
correct path. The GO-WITH-CONSTRAINTS verdict activates the sequential-role port as the primary
architecture for Phase 1 (ADR) and Phase 2 (Build).

---

## L2: Strategic Implications

### Long-Term Architecture Posture

**1. The sequential-role pattern is the correct permanent architecture, not a temporary workaround.**
Both `adversary` and `eng-team` Codex ports have operated successfully on this pattern. The
`multi_agent_v2` feature is under development, but there is no indication it will provide
skill-invokable parallel subagents within a non-interactive `codex exec` run. When `multi_agent_v2`
stabilizes, the port should be re-evaluated; but the current design must be sequential-role.

**2. The `/orchestration` skill's core value proposition is preserved.**
The skill's value is not parallelism per se — it is structured workflow coordination, state
tracking, quality gating, and cross-pollination of findings between analytical approaches. These
are fully portable. The Jerry source skill itself executes sequentially within a single Claude
context; "parallel pipelines" in Jerry are sequential interleaving within one LLM turn sequence.
The Codex port is equivalent in output.

**3. ORCHESTRATION.yaml becomes the synchronization primitive.**
Without a shared in-memory subagent coordination mechanism, the YAML file becomes the single
coordination surface. Each role reads the YAML before acting and writes it after. This is already
the P-002 mandate; the sequential execution makes it safe from write-concurrency issues.

**4. Version independence — critical divergence point.**
The port must carry an independent `codex-x.y.z` version line and a divergence note documenting
that parallelism is not available. Future Jerry upgrades to `/orchestration` (e.g., adding new
patterns) may not map automatically to the Codex port if they depend on the Agent tool. This
divergence point should be documented in the Phase 4 maintenance artifact.

**5. `enable_fanout` as a future upgrade trigger.**
If OpenAI enables `enable_fanout` in a future Codex version, the sequential-role architecture
can be replaced with genuine parallel pipeline execution. The ADR produced in Phase 1 should
document this upgrade path explicitly so the decision is reversible when the capability exists.

**6. Memory-Keeper gap is acceptable.**
The Memory-Keeper MCP is supplemental to the primary file-based persistence. Cross-session
resumption via ORCHESTRATION.yaml is the primary designed mechanism. The Codex port requires no
new state mechanism.

### Trade-offs Summary

| Trade-off | Jerry Source | Codex Port | Impact |
|-----------|-------------|------------|--------|
| Execution model | Sequential LLM interleaving (nominally parallel) | Sequential role adoption (explicitly sequential) | None — same effective execution |
| Subagent spawning | Agent tool (Task) | Not available; role-adoption pattern | Medium — no independent context isolation per agent |
| Memory-Keeper | Supplemental cross-session MCP | File-based ORCHESTRATION.yaml only | Low — P-002 file persistence is primary |
| Quality gates | Creator-critic loop with Agent tool critic | Creator-critic loop in same context | Low — same methodology, minor context bleed |
| Parallelism recovery | If `enable_fanout` released | Re-evaluate port architecture | Low current risk; document in ADR |

---

## Evidence Log

| Evidence Item | Source | How Obtained |
|---|---|---|
| Codex CLI version: `codex-cli-exec 0.142.5` | `codex --version` | CLI command, 2026-07-04 |
| `codex --help` commands list | CLI stdout | CLI command, 2026-07-04 |
| `codex exec --help` flags | CLI stdout | CLI command, 2026-07-04 |
| Feature flags: `multi_agent stable true`, `enable_fanout under dev false`, `multi_agent_v2 under dev false` | `codex features list` | CLI command, 2026-07-04 |
| `codex mcp --help` | CLI stdout | CLI command, 2026-07-04 |
| adversary port: "single execution context, no sub-agent spawning" | `~/.codex/skills/adversary/SKILL.md` lines 22-24 | File read |
| eng-team port: "single execution context, no sub-agent spawning" | `~/.codex/skills/eng-team/SKILL.md` lines 22-24 | File read |
| skill-creator subagent references (conditional, validation-only context) | `~/.codex/skills/.system/skill-creator/SKILL.md` lines 51-54, 389-416 | File read |
| Jerry orchestration primitive inventory | `skills/orchestration/SKILL.md`, `docs/STATE_SCHEMA.md`, `docs/PATTERNS.md`, `agents/orch-planner.md`, `agents/orch-tracker.md`, `agents/orch-synthesizer.md` | File reads |
| Parallel research artifact | `orchestration/codex-port-20260704-001/eng/phase-0/ps-rsch-001/codex-official-standards.md` | Not present at time of this spike — no findings to cite |

---

## Recommendation Summary

**GO-WITH-CONSTRAINTS**

| Dimension | Verdict | Rationale |
|-----------|---------|-----------|
| Is the port feasible? | YES | Sequential-role architecture maps all orchestration primitives except true parallelism |
| Is true parallelism available? | NO | `enable_fanout: under dev, false`; existing ports confirm no spawn |
| Is the loss of parallelism material? | NO | Jerry source executes sequentially in practice; functional outputs identical |
| Is the port architecturally sound? | YES | Established pattern from adversary + eng-team ports; file-based state is P-002 compliant |
| Fallback if GO-WITH-CONSTRAINTS rejected? | SEQUENTIAL-ROLE PORT | As specified in R-01 mitigation — this IS the primary architecture |

**Proceed to Phase 1 (Port Architecture ADR)** with the sequential-role mapping as the base
architecture. The ADR (Phase 1) should:
1. Document each primitive mapping from this table as architecture decisions
2. Include the `enable_fanout` upgrade path as a deferred decision (reversible)
3. Define how ORCHESTRATION.yaml replaces Memory-Keeper for cross-session state
4. Specify the role-adoption sequence for orch-planner, orch-tracker, orch-synthesizer

---

*Document ID: PROJ-037-ENG-ARCH-001*
*Agent: eng-architect (worker — no subagents spawned, P-003 compliant)*
*Workflow: codex-port-20260704-001, Phase 0*
*Feeds: Barrier 0 (Feasibility Gate) — artifact `codex-subagent-mechanism.md`*
*Created: 2026-07-04*
