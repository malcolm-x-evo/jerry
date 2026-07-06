# Barrier 1 — Design Gate Verdict (adv-to-eng)

> Synthesized by the orchestrator from `adv/findings/barrier-1-findings.md` (adv-executor persisted
> its full findings before an infrastructure watchdog stalled it prior to writing this verdict file).
> Content is a faithful summary of the executor's persisted findings — not new analysis.

## Verdict: REVISE (iteration 1)

The Option A sequential-role architecture is sound and approved in principle, but the ADR must be
revised to prevent silent functional degradation and to be honest about the coordination model.
None of the findings are fatal; all have concrete fixes that land in the Phase 2 build + Phase 3 V-hooks.

## Findings by severity

| ID | Severity | Issue | Required fix (from reviewer) |
|----|----------|-------|------------------------------|
| FM-005 / DA-004 | **Major (RPN 384)** | Cross-pollination silently degrades: Pipeline B Phase 2 never reads Pipeline A's handoff artifact | orch-planner role + ORCHESTRATION_PLAN template MUST give explicit "read the other pipeline's cross-pollination artifact before proceeding" instructions; add V-8b test asserting a Read of the handoff before B's phase runs |
| FM-004 / DA-003 | **Major (RPN 336)** | orch-tracker context saturation after 8+ re-entrant adoptions → state estimated from truncated reads | references/orch-tracker.md MUST instruct offset/limit reads on ORCHESTRATION.yaml >2,000 tokens; add V-12 stress test (6+ adoptions, verify state accuracy) |
| FM-007 / DA-001 | **Major (RPN 294)** | Same-context critic anchoring inflates the quality gate | Add V-7a: run the gate once inline + once as fresh `codex exec` (artifact+rubric only); if divergence >0.08 (RT-M-012) add explicit context-clearing to orch-tracker role |
| FM-002 / GAP | **Major (RPN 240)** | Re-entrant orch-tracker instruction ambiguous | SKILL.md body MUST include an explicit numbered workflow loop (adopt planner → per phase: work → adopt tracker → gate check → proceed/revise/escalate → adopt synthesizer) |
| FM-008 | **Major (RPN 240)** | Resume detection unspecified → a new `codex exec` overwrites existing state | SKILL body MUST check for existing ORCHESTRATION.yaml with `workflow.status: PAUSED` at start and branch to resume |
| SM-F-001 / DA-003 | **Major** | "Functional parity on all user-visible features" overstates (P-022) | L0 + Consequences MUST disclose: automated worker-agent coordination is NOT available; users invoke work agents via separate `codex exec` calls and report completion to orch-tracker |
| FM-001 | Minor | Jerry-only frontmatter field left in | Already caught by V-1 + corrected standard |
| FM-006 | Minor | `default_prompt` missing `$orchestration` | Add `$orchestration` to default_prompt OR document CSP-04 production-port exception |

## Required revisions before PASS

1. ADR L0 + Consequences: honest disclosure of the manual coordination model (SM-F-001).
2. ADR must REQUIRE Phase 2 to implement: explicit cross-pollination reads (FM-005), numbered workflow loop (FM-002), resume detection (FM-008), offset/limit tracker reads (FM-004).
3. ADR Phase 3 verification hooks must add: V-7a (gate isolation), V-8b (cross-pollination read), V-12 (tracker stress), and upgrade the sample workflow to 2-pipeline/2-phase with real cross-pollination (DA-002).
4. Resolve FM-006 (default_prompt).
