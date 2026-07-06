# ORCHESTRATION.yaml State Schema

> **Version:** 1.0.0 (Codex port)
> **Skill:** orchestration
> **Format:** YAML 1.2
> **Forked from:** `skills/orchestration/docs/STATE_SCHEMA.md` (2026-07-04)
> **Codex divergence:** Memory-Keeper MCP is not available in the Codex environment. The
> `resumption` section in ORCHESTRATION.yaml is the **sole** cross-session mechanism. Ensure
> `resumption.recovery_state.next_step` is fully self-describing and
> `resumption.files_to_read` is complete, so any fresh `codex exec` session can resume
> without external state sources.

---

## Overview

The ORCHESTRATION.yaml file is the **Single Source of Truth (SSOT)** for workflow execution
state. This document defines the complete schema specification.

---

## Schema Definition

### Root Structure

```yaml
workflow:         # Workflow metadata and configuration
pipelines:        # Pipeline definitions with phases and agents
barriers:         # Sync barrier definitions
execution_queue:  # Priority-ordered execution groups
checkpoints:      # Recovery checkpoint log
metrics:          # Execution and quality metrics
blockers:         # Active and resolved issues
next_actions:     # Immediate and subsequent actions
resumption:       # Cross-session resumption context (primary mechanism in Codex port)
```

---

## Workflow Section

```yaml
workflow:
  id: string                    # REQUIRED. Unique workflow identifier
  name: string                  # REQUIRED. Human-readable name
  project_id: string            # REQUIRED. Project this workflow belongs to
  version: string               # Workflow version (semver)
  created_at: ISO-8601          # When workflow was created
  updated_at: ISO-8601          # Last modification timestamp
  status: enum                  # ACTIVE | PAUSED | COMPLETE | FAILED | CANCELLED

  patterns:                     # List of orchestration patterns used
    - enum                      # SEQUENTIAL | CONCURRENT | BARRIER_SYNC | HIERARCHICAL | FAN_OUT

  constraints:                  # Execution constraints
    max_agent_nesting: integer  # P-003: Must be 1
    file_persistence: boolean   # P-002: Must be true
    user_authority: boolean     # P-020: Must be true
    max_concurrent_agents: integer  # Soft limit (default: 5; Codex port executes sequentially)
    max_barrier_retries: integer    # Circuit breaker (default: 2)
    checkpoint_frequency: enum  # AGENT | PHASE | BARRIER
```

---

## Pipelines Section

```yaml
pipelines:
  {pipeline_id}:                # Pipeline identifier (e.g., "ps", "nse")
    id: string                  # REQUIRED. Same as key
    name: string                # REQUIRED. Human-readable name
    description: string         # Pipeline purpose
    status: enum                # PHASE_N_PENDING | PHASE_N_IN_PROGRESS | COMPLETE
    current_phase: integer      # Currently active phase number
    total_phases: integer       # Total phases in pipeline
    progress_percent: integer   # 0-100
    short_alias: string         # Used in dynamic path construction

    phases:                     # List of phase definitions
      - id: integer             # REQUIRED. Phase number (1-indexed)
        name: string            # REQUIRED. Phase name
        path_id: string         # Used in path construction (e.g., "phase-1")
        status: enum            # PENDING | IN_PROGRESS | COMPLETE | BLOCKED
        blocked_by: string      # If BLOCKED, what is blocking (barrier/phase)
        started_at: ISO-8601    # When phase started (null if not started)
        completed_at: ISO-8601  # When phase completed (null if not complete)

        agents:                 # List of agents in this phase
          - id: string          # REQUIRED. Agent identifier
            name: string        # Human-readable agent name
            status: enum        # PENDING | IN_PROGRESS | COMPLETE | FAILED | BLOCKED
            artifact: string    # Path to output artifact (null if not created)
            inputs:             # List of input artifact paths
              - string

        artifacts:              # List of artifact paths created by this phase
          - string
```

---

## Barriers Section

```yaml
barriers:
  - id: string                  # REQUIRED. Barrier identifier (e.g., "barrier-1")
    name: string                # Human-readable name
    status: enum                # PENDING | IN_PROGRESS | COMPLETE
    completed_at: ISO-8601      # When barrier was crossed (null if not complete)

    depends_on:                 # What must complete before barrier can start
      - string                  # Format: "{pipeline_id}.phase.{phase_number}"

    artifacts:                  # Cross-pollination artifacts (B-R1)
      {direction}:              # e.g., "a_to_b", "b_to_a"
        path: string            # Path to artifact (must exist before barrier COMPLETE)
        status: enum            # PENDING | IN_PROGRESS | COMPLETE
        read_confirmed: boolean # Set to true when receiving pipeline confirms Read of this artifact
        key_content: string     # Summary of artifact content (null if not created)
```

> **B-R1 note:** The `read_confirmed` field is added to the Codex port barrier artifact schema.
> orch-tracker sets this to `true` only after the receiving pipeline has performed a Read tool
> call on the handoff artifact. A barrier cannot be marked COMPLETE if `read_confirmed: false`
> on any artifact in its `artifacts` map.

---

## Execution Queue Section

```yaml
execution_queue:
  current_group: integer        # Currently executing group number

  groups:                       # Ordered list of execution groups
    - id: integer               # REQUIRED. Group number (1-indexed)
      name: string              # Human-readable group name
      execution_mode: enum      # PARALLEL | SEQUENTIAL
                                # NOTE (Codex port): PARALLEL is informational only;
                                # agents execute sequentially in one context.
      status: enum              # READY | IN_PROGRESS | COMPLETE | BLOCKED
      blocked_by: string        # If BLOCKED, what is blocking

      agents:                   # List of agent IDs in this group
        - string

      tasks:                    # List of task names (for barrier groups)
        - string
```

---

## Checkpoints Section

```yaml
checkpoints:
  latest_id: string             # ID of most recent checkpoint (null if none)

  entries:                      # List of checkpoints
    - id: string                # REQUIRED. Checkpoint ID (e.g., "CP-001")
      timestamp: ISO-8601       # REQUIRED. When checkpoint was created
      trigger: enum             # PHASE_COMPLETE | BARRIER_COMPLETE | MANUAL
      description: string       # Human-readable description
      recovery_point: string    # What step to resume from if recovering
```

---

## Metrics Section

```yaml
metrics:
  execution:                    # Execution progress metrics
    phases_complete: integer
    phases_total: integer
    phases_percent: integer
    barriers_complete: integer
    barriers_total: integer
    barriers_percent: integer
    agents_executed: integer
    agents_total: integer
    agents_percent: integer
    artifacts_created: integer
    artifacts_total: integer
    artifacts_percent: integer

  quality:                      # Quality gate metrics (tracked by orch-tracker)
    agent_success_rate: integer
    barrier_validation_pass_rate: integer
    checkpoint_recovery_tested: boolean
    phase_scores: {}            # Populated per-phase by orch-tracker
    barrier_scores: {}          # Populated per-barrier by orch-tracker
    workflow_quality: {}        # Aggregate after all gates pass

  timing:
    workflow_started: ISO-8601
    last_activity: ISO-8601
    estimated_completion: ISO-8601
```

---

## Blockers Section

```yaml
blockers:
  active:                       # Currently blocking issues
    - id: string
      description: string
      blocking:
        - string
      severity: enum            # LOW | MEDIUM | HIGH
      created_at: ISO-8601

issues:
  resolved:
    - id: string
      description: string
      resolution: string
      resolved_at: ISO-8601
```

---

## Next Actions Section

```yaml
next_actions:
  immediate:
    - action: string
      agents: [string]
      execution_mode: enum      # PARALLEL | SEQUENTIAL (informational only in Codex port)

  subsequent:
    - action: string
      depends_on: string
```

---

## Resumption Section

> **Codex port — critical section.**
>
> Memory-Keeper MCP is **not available** in the Codex environment. This means the `resumption`
> section in ORCHESTRATION.yaml is the **only** cross-session mechanism for workflow recovery.
>
> Requirements for this section to support reliable resume:
> 1. `resumption.recovery_state.next_step` MUST be fully self-describing — a fresh `codex exec`
>    session with no prior context must be able to resume from this string alone.
> 2. `resumption.files_to_read` MUST list all artifacts needed to reconstruct context.
> 3. `resumption.recovery_state.cross_session_portable` MUST be `true`.
> 4. `resumption.recovery_state.ephemeral_references` MUST be `false`.
>
> The orch-tracker role updates this section after every phase completion and checkpoint.

```yaml
resumption:
  recovery_state:
    last_checkpoint: string     # ID of last checkpoint (null if none)
    current_phase: integer      # Current phase number
    current_phase_name: string  # Human-readable phase name
    workflow_status: string     # Current workflow status
    current_activity: string    # What is happening now
    next_step: string           # FULLY SELF-DESCRIBING resume instruction
                                # Example: "Adopt orch-tracker role. Phase 2 of pipeline_a
                                #   is next. Read ORCHESTRATION_PLAN.md for phase definition.
                                #   Work agent has NOT yet been invoked for this phase."
    context_fill_at_update: null
    updated_at: ISO-8601
    cross_session_portable: true   # MUST be true (Codex port: sole cross-session mechanism)
    ephemeral_references: false    # MUST be false

  files_to_read:                # MUST be complete — no Memory-Keeper supplemental source
    - path: "ORCHESTRATION.yaml"
      priority: 1
      purpose: "Machine-readable state. Read resumption section first."
    - path: "ORCHESTRATION_PLAN.md"
      priority: 2
      purpose: "Strategic context. Agent definitions, phase descriptions."
    - path: "ORCHESTRATION_WORKTRACKER.md"
      priority: 3
      purpose: "Tactical documentation. Execution log, blockers."
    # Add all phase artifacts created so far (paths from pipelines.{x}.phases.{n}.artifacts)

  quality_trajectory:
    gates_completed: []
    gates_remaining: []
    current_gate: null
    current_gate_iteration: null
    score_history: {}
    lowest_dimension: null
    total_iterations_used: 0

  defect_summary:
    total_defects_found: 0
    total_defects_resolved: 0
    unresolved_defects: []
    recurring_patterns: []
    last_gate_primary_defect: null

  decision_log: []
  agent_summaries: {}

  compaction_events:
    count: 0
    events: []
```

---

## Validation Rules

1. `workflow.constraints.max_agent_nesting` MUST be 1 (P-003)
2. `workflow.constraints.file_persistence` MUST be true (P-002)
3. `resumption.recovery_state.cross_session_portable` MUST be true
4. `resumption.recovery_state.ephemeral_references` MUST be false
5. `resumption.recovery_state.next_step` MUST be self-describing (no references to ephemeral context)
6. All paths MUST be repository-relative (no absolute paths)
7. ISO-8601 timestamps MUST include timezone
8. Barrier `artifacts.{direction}.read_confirmed` MUST be `true` before barrier status = COMPLETE

---

*Schema Version: 1.0.0 (Codex port)*
*Forked from: skills/orchestration/docs/STATE_SCHEMA.md (2026-07-04)*
*Codex divergence: Memory-Keeper absent; resumption section is sole cross-session mechanism;
PARALLEL execution_mode is informational only; barrier read_confirmed field added (B-R1).*
