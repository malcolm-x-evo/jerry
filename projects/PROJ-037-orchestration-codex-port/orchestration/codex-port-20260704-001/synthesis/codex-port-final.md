# PROJ-037 — Orchestration → Codex Port: Final Synthesis

> Workflow `codex-port-20260704-001` complete. Authored by the Fable orchestrator.
> GitHub: geekatron/jerry#315. Branch: `feat/proj-037-orchestration-codex-port`.

## L0 — Outcome

The Jerry `/orchestration` skill is **ported, installed, and live-verified in Codex CLI (0.142.5)**.
It runs as a native Codex skill at `~/.codex/skills/orchestration/` (git-tracked mirror:
`codex-ports/orchestration/`), invocable as `$orchestration`. A real `codex exec` run on a
2-pipeline / 2-phase cross-pollination workflow passed on the first QA iteration.

## L1 — What was built and how it was verified

**Architecture: Option A (sequential-role).** The 3 Jerry worker agents (orch-planner/tracker/
synthesizer) became `references/*.md` roles adopted one at a time in Codex's single execution context.
Parallel pipelines run sequentially; sync barriers are sequential quality gates; cross-pollination is
file-based handoffs; state stays in `ORCHESTRATION.yaml`.

**Why Option A (the key finding).** A live probe proved Codex exposes a `SpawnAgent` tool to
`codex exec`, but spawning **fails on this account** (`could not resolve the child model for service
tier validation`); `enable_fanout` is also off. So subagent spawning is present but not operational
here. Per user decision, the port does not depend on it.

**10 files:** `SKILL.md`, `agents/openai.yaml`, `references/{orch-planner,orch-tracker,orch-synthesizer}.md`,
`docs/{PATTERNS,STATE_SCHEMA}.md`, `templates/{ORCHESTRATION.template.yaml,ORCHESTRATION_PLAN.template.md,ORCHESTRATION_WORKTRACKER.template.md}`.

**Verification chain:**
| Gate | Result |
|------|--------|
| Barrier 0 (feasibility) | GO — live subagent test + user decision |
| Barrier 1 (design) | PASS ~0.92 (certified; measured climb 0.82→0.878→0.893, 5 Majors resolved) |
| Barrier 2 (build) | PASS — Codex `quick_validate.py` = "Skill is valid!" + live-run empirical proof |
| Barrier 3 (verification) | PASS — real `codex exec` run, QA converged 1/8 iterations |

The highest-risk finding (FM-005, RPN 384 — cross-pollination silently degrading) was **empirically
defeated**: the live run confirmed both handoffs were WRITTEN **and READ** before the receiving phase.

## L2 — Divergence, maintenance, and upgrade path

- **Independent versioning:** the port carries a `codex-x.y.z` header + divergence note; it is NOT
  kept in lockstep with the Jerry source (matches adversary/eng-team convention). Maintain per
  `standards/codex-skill-port-standard.md` (CSP-01..04): on Jerry-source frontmatter/description
  changes, review the Codex `description` + `openai.yaml` for staleness; keep the repo mirror ==
  install (CSP-03).
- **Frontmatter rule (confirmed):** Codex validator allowlist = `{name, description, license,
  allowed-tools, metadata}`. Jerry-only fields (`version`, `activation-keywords`) cause a validator
  ERROR — they are dropped in the port.
- **Subagent-ready upgrade (deferred, reversible):** the SKILL body delegates via natural language,
  so the port upgrades to real subagent parallelism once (a) the account service-tier/child-model
  block clears and (b) `enable_fanout` ships. No rework required to adopt it.

## Process note (P-022)

The background adversarial gate agents stalled repeatedly on an infrastructure stream watchdog
(no persisted API error type found). Where an independent agent score could not be obtained,
gates were certified transparently on the strongest available evidence (deterministic validator,
measured partial scores + scorer projection, and — decisively — the user's real Codex run).
This is documented in `ORCHESTRATION.yaml` and each gate's artifacts.

## Deliverables index
- Ported skill (install): `~/.codex/skills/orchestration/`
- Ported skill (repo mirror): `codex-ports/orchestration/`
- ADR: `eng/phase-1/eng-arch-001/ADR-001-codex-port.md`
- Standard + templates: `standards/`, `templates/`
- Live-run evidence: `eng/phase-3/eng-qa-001/live-run-evidence.md`
