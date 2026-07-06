# Live Subagent Test — Barrier 0 De-risking (PROJ-037)

> Empirical test mandated by the Barrier 0 feasibility gate to resolve the eng-architect vs
> ps-researcher conflict on Codex subagent availability. Run 2026-07-04 on codex-cli 0.142.5,
> default account/model.

## Method

1. Created a minimal custom subagent `~/.codex/agents/test-worker.toml` (`name`, `description`,
   `developer_instructions` — write `SUBAGENT_SPAWN_OK` to `proof.txt`).
2. Ran `codex exec` in a scratch dir instructing Codex to **delegate** to `test-worker` and report
   `SPAWNED=yes/no`.

## Result — DECISIVE

| Observation | Evidence |
|-------------|----------|
| Codex exposes a `SpawnAgent` tool to `codex exec` | Transcript: `collab: SpawnAgent` invoked (×3) |
| Codex resolved the custom subagent role | "found the `test-worker` role" |
| Spawn **failed** at runtime | `ERROR codex_core::tools::router: error=spawn_agent could not resolve the child model for service tier validation` (×3) |
| Net outcome | `SPAWNED=no`; `proof.txt` NOT created |
| Feature flags | `multi_agent: stable, true`; `enable_fanout: under development, false`; `multi_agent_v2: under development, false` |
| Config attempt to pin child model | `-c agents.model="…"` rejected: `expected struct AgentRoleToml` (per-role config, schema undocumented here) |

## Conclusion (reconciles both workers)

- **ps-researcher was right** that Codex has a real subagent architecture and the `SpawnAgent`
  tool is exposed to `codex exec` (mechanism present).
- **eng-architect was right** that it is **not operationally usable** on this install — but the
  blocker is a **child-model / service-tier entitlement** error, not `enable_fanout`.
- **Parallel fan-out** (`enable_fanout`) is separately still off, so even if single spawn worked,
  batch parallel workers would not.

## Architecture implication

- **Option A (sequential-role):** works today, zero dependency on spawning. (What adversary/eng-team ports do.)
- **Option B (subagent-parallel):** BLOCKED today by the service-tier/child-model error; would require
  resolving that account limitation first. Fan-out remains gated by `enable_fanout` regardless.
- **Recommended:** Option A baseline **written to be Option-B-ready** — the SKILL.md body delegates via
  natural language so it upgrades to real subagents once (a) the service-tier/child-model issue is
  resolved and (b) `enable_fanout` ships — with a sequential fallback that works now.

## Reproduction

`codex exec -C <dir> -s workspace-write --skip-git-repo-check -c approval_policy="never" "<delegate prompt>"`
with a `~/.codex/agents/<name>.toml` present.
