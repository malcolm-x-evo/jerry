# Phase 3 — Live-Run QA Evidence (PROJ-037)

> Real `codex exec` run of the ported `$orchestration` skill, executed by the user in their local
> Codex (existing model/account), 2026-07-05. QA iteration 1 of max 8. Single execution context,
> sequential-role (Option A) — no subagents (blocked on this account).

## Test

Working dir `~/orch-codex-test`; prompt: plan AND run a 2-pipeline (alpha, beta), 2-phase-each
workflow with one sync barrier and **bidirectional cross-pollination**; report loaded skill files,
created paths, and confirm handoffs were written AND read. (This is the upgraded sample workflow
mandated by ADR Addendum A / DA-002 — not the minimal 1-pipeline case.)

## Result — PASS (iteration 1, no fixes required)

### AC-3.1 — skill loads & triggers on `$orchestration` — PASS
Codex loaded all 7 files:
- `SKILL.md`, `references/orch-planner.md`, `references/orch-tracker.md`, `references/orch-synthesizer.md`
- `templates/ORCHESTRATION.template.yaml`, `ORCHESTRATION_PLAN.template.md`, `ORCHESTRATION_WORKTRACKER.template.md`

### State artifacts created — PASS
- `ORCHESTRATION.yaml`, `ORCHESTRATION_PLAN.md`, `ORCHESTRATION_WORKTRACKER.md`

### Numbered workflow loop + dynamic path structure (B-R2) — PASS
Full agent-level-isolated tree created:
```
orchestration/demo-crosspoll-20260705-001/
  alpha/phase-1/alpha-phase-1/output.md   beta/phase-1/beta-phase-1/output.md
  alpha/phase-2/alpha-phase-2/output.md   beta/phase-2/beta-phase-2/output.md
  cross-pollination/barrier-1/alpha-to-beta/handoff.md
  cross-pollination/barrier-1/beta-to-alpha/handoff.md
  synthesis/demo-crosspoll-20260705-001-final-synthesis.md
```

### AC-3.2 / B-R1 / V-8b — cross-pollination WRITTEN AND READ — **PASS (the critical check)**
- `alpha-to-beta/handoff.md`: written, then **read by beta before beta phase 2**.
- `beta-to-alpha/handoff.md`: written, then **read by alpha before alpha phase 2**.
- `ORCHESTRATION.yaml` records both as `WRITTEN_AND_READ` with `read_by_receiver: true` and
  `read_before_output: true`.

This empirically defeats FM-005 (RPN 384) — the highest-severity Barrier 1 finding (cross-pollination
silently degrading to independent pipelines).

## QA loop status
Converged in **1 iteration** (cap 8). No fixes required. Human review not triggered.

## Note
Unrelated environment output ("mix precommit / 486 tests / import_transactions_handler.ex") appeared
in the user's Codex session tail — bleed-through from a different project, NOT part of this workflow;
excluded from validation.
