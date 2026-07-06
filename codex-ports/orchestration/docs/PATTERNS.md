# Orchestration Patterns

> **Version:** 1.0.0 (Codex port)
> **Skill:** orchestration
> **Forked from:** `skills/orchestration/docs/PATTERNS.md` (2026-07-04)
> **Codex divergence note:** In this Codex port, `execution_mode: PARALLEL` in ORCHESTRATION.yaml
> groups is informational metadata only. Codex has a single execution context, so all agents
> within a group execute sequentially regardless of the `execution_mode` value. Functional
> outputs are identical to the Jerry source; wall-clock parallelism is not available.
> **References:** [Microsoft AI Agent Patterns](https://learn.microsoft.com/en-us/azure/architecture/ai-ml/guide/ai-agent-design-patterns), [LangGraph](https://langchain-ai.github.io/langgraph/), [CrewAI Flows](https://docs.crewai.com/concepts/flows)

---

## Overview

This document describes orchestration patterns supported by the orchestration skill, based on
industry best practices from Microsoft, LangGraph, CrewAI, and NASA SE.

---

## Pattern 1: Cross-Pollinated Pipeline

**Description:** Two or more pipelines running in parallel with synchronization barriers for
bidirectional information exchange.

> **Codex port note:** In this port, Pipeline A and Pipeline B roles are adopted sequentially
> within one context — not concurrently. The cross-pollination handoff is file-based: the source
> pipeline writes a handoff artifact, and the receiving pipeline MUST read it before generating
> any Phase N+1 output. See the cross-pollination READ instructions in `references/orch-planner.md`
> and the barrier section of `templates/ORCHESTRATION_PLAN.template.md`. Functional outputs are
> identical to the Jerry source (parallel execution delivers no different files); only wall-clock
> time differs.

**Use When:**
- Multiple perspectives needed on same problem
- Pipelines have complementary expertise
- Cross-domain validation required

**Example:** ps (Problem-Solving) and nse (NASA SE) pipelines

```
PIPELINE A                              PIPELINE B
    |                                       |
    v                                       v
+---------+                           +---------+
| Phase 1 |                           | Phase 1 |
+----+----+                           +----+----+
     |                                     |
     +------------------+------------------+
                        v
                +=================+
                |   BARRIER 1     |  Bidirectional exchange
                |  a->b artifact  |  (MANDATORY READ before Phase 2)
                |  b->a artifact  |
                +=================+
                        |
     +------------------+------------------+
     |                                     |
     v                                     v
+---------+                           +---------+
| Phase 2 |                           | Phase 2 |
+---------+                           +---------+
```

**Barrier Artifacts:**
- Extract key findings from completed phase
- Transform into cross-pollination document
- Target pipeline MUST read before next phase (mandatory — not optional)

---

## Pattern 2: Sequential with Checkpoints

**Description:** Single pipeline with checkpoint creation after each phase for recovery.

**Use When:**
- Linear workflow with clear dependencies
- Long-running process needs recovery points
- Debugging capability required

**Example:** Research → Analysis → Design → Implementation

```
+---------+     +---------+     +---------+     +---------+
| Phase 1 |---->| Phase 2 |---->| Phase 3 |---->| Phase 4 |
+----+----+     +----+----+     +----+----+     +----+----+
     |               |               |               |
     v               v               v               v
   CP-001          CP-002          CP-003          CP-004
   (recovery)      (recovery)      (recovery)      (recovery)
```

**Checkpoint Contents:**
- State snapshot at completion
- List of artifacts created
- Recovery instructions (`resumption.recovery_state.next_step`)

---

## Pattern 3: Fan-Out / Fan-In

**Description:** Parallel execution of independent agents with synthesis at the end.

> **Codex port note:** In this port, the fan-out agents (A, B, C) execute sequentially in one
> context — not concurrently. Each agent's role is adopted in turn; results aggregate in
> ORCHESTRATION.yaml before the synthesizer role is adopted. The `execution_mode: PARALLEL`
> field in the execution_queue group is informational metadata only. Functional outputs are
> identical — the synthesizer reads all outputs regardless of creation order; only wall-clock
> time differs. For workflows where actual parallel execution time savings are required, the
> Codex port is not the appropriate environment (see SKILL.md divergences).

**Use When:**
- Multiple independent research streams
- Diverse perspectives on same topic
- Independent agents whose outputs will be synthesized

**Example:** Parallel research on caching, queuing, storage

```
              +---------+
              |  Start  |
              +----+----+
    +-----------+-+-----------+
    v           v             v
+--------+  +--------+  +--------+
|Agent A |  |Agent B |  |Agent C |
|(cache) |  |(queue) |  |(store) |
+---+----+  +---+----+  +---+----+
    +-----------+-----------+
                v
        +------------+
        | Synthesize |
        +------------+
```

**Execution (Codex port):**
- Agents A, B, C adopt roles in sequence (not concurrently)
- orch-tracker role adopted after each agent completes
- Synthesizer role adopted after all agents complete
- Synthesis consumes all artifacts

---

## Pattern 4: Hierarchical Delegation

**Description:** Manager agent coordinates specialist agents.

**Use When:**
- Complex task requiring specialist knowledge
- Dynamic routing based on task type
- Quality control needed

**Example:** Triage → Specialist routing

```
+-----------------+
|  Manager Agent  |
+--------+--------+
         | Delegates based on task type
    +----+----+--------+
    v    v    v        v
+------+ +------+ +------+
|Spec A| |Spec B| |Spec C|
+------+ +------+ +------+
```

**Note:** In this Codex port (as in the Jerry source), the main execution context acts as
manager — not a spawned agent (P-003 compliant). Each specialist is a role adopted sequentially.

---

## Pattern 5: Iterative Refinement (Generator-Critic)

**Description:** Generator creates output, critic evaluates, loop until quality threshold.

**Use When:**
- Output quality is critical
- Iterative improvement possible
- Clear evaluation criteria exist (>= 0.92 threshold, H-13)

**Example:** Draft → Review → Revise → Review → Accept

```
+------------+     +------------+
| Generator  |---->|   Critic   |
+------------+     +-----+------+
      ^                  |
      |                  | Feedback
      |                  v
      |            +----------+
      +------------|Threshold |
                   |  Met?    |
                   +----------+
                        |
                        v Yes
                   +----------+
                   |  Accept  |
                   +----------+
```

**Circuit Breaker:**
- max_iterations: 3 (H-14)
- Gate threshold: >= 0.92 (H-13)
- Stop if 3 iterations exhausted without PASS — escalate to user (H-31)

---

## Pattern Selection Guide

| Scenario | Recommended Pattern |
|----------|---------------------|
| Two domain perspectives needed | Cross-Pollinated Pipeline |
| Long-running single track | Sequential with Checkpoints |
| Independent parallel research | Fan-Out / Fan-In |
| Complex task routing | Hierarchical Delegation |
| Quality-critical output | Iterative Refinement |

---

## Constitutional Constraints

All patterns must comply with:

| Constraint | ID | Implication |
|------------|----|----|
| Single nesting | P-003 | Main context is orchestrator, agents are roles (not spawned) |
| File persistence | P-002 | All state to ORCHESTRATION.yaml |
| User authority | P-020 | User can override any decision |
| No deception | P-022 | Honest status reporting; automated coordination limitations disclosed |

---

## Industry References

1. Microsoft. (2025). *AI Agent Orchestration Patterns*. https://learn.microsoft.com/en-us/azure/architecture/ai-ml/guide/ai-agent-design-patterns
2. LangChain. (2025). *LangGraph State Management*. https://langchain-ai.github.io/langgraph/
3. CrewAI. (2025). *Flows and Routing*. https://docs.crewai.com/concepts/flows
4. NASA. (2024). *NPR 7123.1D SE Engine*. https://nodis3.gsfc.nasa.gov/

---

*Document Version: 1.0.0 (Codex port)*
*Forked from: skills/orchestration/docs/PATTERNS.md (2026-07-04)*
*Codex divergence: PARALLEL execution groups run sequentially; functional outputs identical.*
