# PROJ-037 — Orchestration Skill → Codex CLI Port

> **Status:** ACTIVE (plan authored, awaiting approval)
> **Created:** 2026-07-04
> **Active Workflow:** `orchestration/codex-port-20260704-001/`

## Goal

Port the Jerry `/orchestration` skill to run natively in **Codex CLI** (v0.142.5) under
`~/.codex/skills/orchestration/`, preserving multi-agent coordination (subagent spawning,
parallel pipelines, sync barriers, cross-pollination, state tracking) via Codex's own subagent
mechanism. The port is **live-run verified** inside Codex, built by `/eng-team` and reviewed by
`/adversary`.

## Constraints

- Planning by Fable; execution by Sonnet-tier agents.
- Ported skill is model-agnostic at run time.
- Follow the existing Codex-port convention (`~/.codex/skills/adversary`, `eng-team`):
  `SKILL.md` + `references/*.md` + `agents/openai.yaml`, independent `codex-x.y.z` version, divergence note.
- Keep a git-tracked mirror in `codex-ports/orchestration/`.
- No destructive writes; back up any existing `~/.codex/skills/orchestration/`.

## Reference

- Strategic plan: `orchestration/codex-port-20260704-001/ORCHESTRATION_PLAN.md`
- State (SSOT): `orchestration/codex-port-20260704-001/ORCHESTRATION.yaml`
- Execution tracking: `orchestration/codex-port-20260704-001/ORCHESTRATION_WORKTRACKER.md`
- Source skill: `skills/orchestration/`
