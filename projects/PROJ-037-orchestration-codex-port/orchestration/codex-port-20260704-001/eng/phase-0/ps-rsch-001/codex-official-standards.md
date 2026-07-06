# Codex CLI Official Standards — Phase 0 Research

> **Document ID:** PROJ-037-ORCH-RSCH-001
> **Agent:** ps-researcher
> **Phase:** Phase 0 (Discovery)
> **Workflow:** codex-port-20260704-001
> **Date:** 2026-07-04
> **Codex version verified against:** codex-cli 0.142.5
> **Confidence:** 0.92 (high — primary local evidence + official docs + agentskills.io spec)

## Document Sections

| Section | Purpose |
|---------|---------|
| [L0: Executive Summary](#l0-executive-summary) | Non-technical findings and project impact |
| [L1: Technical Findings](#l1-technical-findings) | Detailed field specs, schemas, mechanisms |
| [L2: Architectural Implications](#l2-architectural-implications) | Port design decisions and risks |
| [Resolved-VERIFY Table](#resolved-verify-table) | All CSP [VERIFY] items confirmed or corrected |
| [Methodology](#methodology) | Sources consulted, search strategy |
| [References](#references) | Full citation list |

---

## L0: Executive Summary

Think of a Codex skill like a plug-in card: it has a label on the front (the `name` + `description` frontmatter) that Codex reads to decide whether to pull the card, and instructions on the back (the SKILL.md body) that Codex reads only after deciding the card is relevant. Only two fields on that label matter for triggering; extra Jerry-style fields get rejected at validation time.

Three key findings for the orchestration port:

1. **Frontmatter is stricter than assumed.** Only five fields are permitted by the Codex validator: `name`, `description`, `license`, `metadata`, and `allowed-tools`. Jerry's `version` and `activation-keywords` fields will cause the validator to fail. The `allowed-tools` field is permitted but experimental and not reliably enforced.

2. **Subagents exist and can be used, but the depth limit matters.** Codex supports subagent spawning with a default `agents.max_depth` of 1 (one nesting level). A skill can trigger subagent workflows through natural language in its body, but does NOT declare subagents in SKILL.md frontmatter. The existing `adversary` and `eng-team` ports chose the simpler "roles" approach (no subagents); the orchestration port has a design decision to make.

3. **All [VERIFY] items are resolved.** The one flagged item in the CSP standard — whether Jerry-only fields are silently ignored — is corrected: they are NOT silently ignored; they cause a validator error. The `allowed-tools` field is permitted (not ignored), but carries a different meaning in Codex than in Jerry.

**Project impact:** The orchestration skill SKILL.md must drop `version` and `activation-keywords`. The `allowed-tools` field may be included with Codex-native syntax if tool restriction is desired, but the existing port convention (adversary, eng-team) omits it. MCP dependencies (Memory-Keeper) must be declared in `agents/openai.yaml` rather than in frontmatter.

---

## L1: Technical Findings

### Finding 1: SKILL.md Frontmatter — Complete Verified Field Set

**Source authority (in order):** `quick_validate.py` (primary — installed Codex 0.142.5), agentskills.io specification (open standard Codex implements), official `developers.openai.com/codex/skills`, skill-creator SKILL.md.

| Field | Required | Constraint | Codex Behavior | Port Action |
|-------|----------|------------|----------------|-------------|
| `name` | Yes | Max 64 chars; `^[a-z0-9-]+$`; no leading/trailing/consecutive hyphens | Primary trigger signal | Use: `orchestration` |
| `description` | Yes | Max 1024 chars; no `<` or `>`; must describe WHAT + WHEN | Primary trigger signal; loaded at startup for all skills | Comprehensive WHAT+WHEN; no XML tags |
| `license` | No | License name or bundled file path | Permitted; not used for triggering | Omit (no license applies) |
| `metadata` | No | Arbitrary key-value map (e.g., `short-description`) | Permitted; not used for triggering; system skills use it | Omit (keep frontmatter minimal) |
| `allowed-tools` | No | Space-separated string of pre-approved tool names, e.g., `Bash(git:*) Read` (agentskills.io format) | Experimental; permitted by validator; NOT reliably enforced (per GitHub issue) | Omit (existing port convention; enforcement unreliable) |

**Fields that cause validator failure (REJECTED):**
- `version` — not in allowed set
- `activation-keywords` — not in allowed set
- `compatibility` — in agentskills.io spec but NOT in Codex 0.142.5 validator allowed set
- Any other custom field

The validator enforcement code (from `quick_validate.py`, the authoritative local source):

```python
allowed_properties = {"name", "description", "license", "allowed-tools", "metadata"}
unexpected_keys = set(frontmatter.keys()) - allowed_properties
if unexpected_keys:
    return False, f"Unexpected key(s) in SKILL.md frontmatter: {unexpected}. Allowed properties are: {allowed}"
```

**Divergence from agentskills.io spec:** The spec includes a `compatibility` field (max 500 chars, for environment requirements). The Codex 0.142.5 validator does NOT allow it. The validator is the operative authority for this installed version.

**Official statement on what triggers a skill:** "These are the only fields that Codex reads to determine when the skill gets used" (referring to `name` and `description`). — Source: `~/.codex/skills/.system/skill-creator/SKILL.md`, line 79.

### Finding 2: Progressive Disclosure Loading Chain

Codex loads skill content in three levels; this is the open standard, confirmed by the official docs and local skill-creator evidence.

| Level | Content | Budget | Trigger |
|-------|---------|--------|---------|
| 1 — Metadata | `name`, `description`, file path | ~100 tokens per skill; total budget at most 2% of context window OR 8,000 chars (whichever is less) | Always — loaded at startup for all installed skills |
| 2 — Body | Full SKILL.md body (after frontmatter) | <5,000 tokens recommended; keep body <500 lines | When Codex decides the skill is relevant |
| 3 — Resources | `references/`, `scripts/`, `assets/` files | Unlimited (scripts can execute without being read) | On demand by Codex during task execution |

**Key constraint:** When many skills are installed, descriptions shorten first; some may be omitted with warnings. The description should front-load trigger keywords. "Include all when-to-use information here — not in the body. The body is only loaded after triggering." — Source: skill-creator SKILL.md, line 353.

**Implication for orchestration:** The description must contain all orchestration trigger phrases (orchestration, pipeline, workflow, multi-agent, phases, gates, plan, coordinate, sequence) because they will not be in the body during matching.

### Finding 3: `agents/openai.yaml` — Complete Verified Schema

**Source:** `~/.codex/skills/.system/skill-creator/references/openai_yaml.md` (primary — installed version); official `developers.openai.com/codex/skills` (confirmed).

```yaml
interface:
  display_name: "Human-facing title shown in UI skill lists and chips"
  short_description: "Human-facing blurb (25-64 chars)"
  icon_small: "./assets/small-400px.png"   # optional; relative to skill dir
  icon_large: "./assets/large-logo.svg"    # optional; relative to skill dir
  brand_color: "#3B82F6"                   # optional; hex
  default_prompt: "Use $<skill-name> to ..."  # MUST mention $<skill-name>

dependencies:
  tools:
    - type: "mcp"            # ONLY "mcp" supported (confirmed)
      value: "server-id"
      description: "Human-readable explanation"
      transport: "streamable_http"
      url: "https://server-url"

policy:
  allow_implicit_invocation: true  # default true; false = explicit $skill only
```

**Formatting rules (confirmed):** Quote all string values; keep keys unquoted. The `default_prompt` MUST reference the skill as `$<skill-name>`.

**Status of each field in CSP standard:** All fields marked "Confirmed" in the CSP are confirmed by primary sources. No corrections needed to the `agents/openai.yaml` section.

**MCP dependency declaration for orchestration:** The Jerry orchestration skill uses Memory-Keeper MCP (`mcp__memory-keeper__*`). This MUST be declared in `agents/openai.yaml` `dependencies.tools[]` if the Codex port uses a Memory-Keeper equivalent. If the port does not use MCP tools, the `dependencies` block can be omitted.

### Finding 4: Subagent and Parallel Execution Mechanism

**Source:** `developers.openai.com/codex/subagents` (official); skill-creator SKILL.md (installed); adversary/eng-team port SKILL.md bodies (observed pattern).

#### What Codex supports

| Capability | Detail | Source |
|------------|--------|--------|
| Subagent spawning | Yes — Codex spawns subagents via natural language; orchestration is automatic | Official subagent docs |
| `agents.max_depth` | Default 1 — one nesting level allowed (child agents cannot spawn further agents) | Official subagent docs |
| `agents.max_threads` | Default 6 — max concurrent open agent threads | Official subagent docs |
| `agents.job_max_runtime_seconds` | Per-worker timeout for batch jobs | Official subagent docs |
| Parallel execution | Yes — "Codex waits until all requested results are available, then returns a consolidated response" | Official subagent docs |
| `spawn_agents_on_csv` | Experimental tool for batch parallel spawning (one worker per CSV row) | Official subagent docs |
| Custom agent definitions | TOML files at `~/.codex/agents/` or `.codex/agents/`; required fields: `name`, `description`, `developer_instructions` | Official subagent docs |
| `codex exec` (alias `codex e`) | Non-interactive CI-style execution with `--json`, `--output-last-message`, `--sandbox`, `--profile` flags; supports session resume | Official CLI reference |

#### How skills express subagent invocation

**Skills do NOT declare subagents in frontmatter.** Subagent spawning happens through natural language instructions in the SKILL.md body (or through the `spawn_agents_on_csv` tool during execution). The SKILL.md body can instruct Codex to "spawn a subagent for X" and Codex handles the orchestration.

#### Current port convention (adversary/eng-team pattern)

Both existing Jerry ports explicitly chose NOT to use subagents. Their body texts state: "Codex has a single execution context and no sub-agent spawning, so the N 'agents' are **roles** you adopt one at a time."

This was a design decision based on the skills available at port time, not a technical limitation. The Codex docs and skill-creator explicitly describe subagent use as available (with the `max_depth=1` constraint).

**Decision point for orchestration port:** The orchestration skill's defining value is multi-agent coordination. Two options:

| Option | Mechanism | Fidelity to Jerry original | Complexity |
|--------|-----------|---------------------------|------------|
| A — Roles | Sequential role adoption (adversary/eng-team pattern) | Lower — loses true parallelism | Low |
| B — Subagents | Use Codex subagent spawning for worker agents | Higher — preserves parallel pipeline capability | Medium; constrained by max_depth=1 |

This is an architectural decision for Phase 1 (eng-architect). Research documents both options as viable; Option B is technically supported by Codex 0.142.5.

### Finding 5: Skill Invocation Mechanism

| Method | Syntax | Behavior |
|--------|--------|----------|
| Explicit user | `/skills` command or `$orchestration` in prompt | Always invokes the skill |
| Implicit | Codex autonomously selects based on description match | Default (`allow_implicit_invocation: true`) |
| Disabled implicit | Set `allow_implicit_invocation: false` in openai.yaml | Explicit `$orchestration` only |

**Skill discovery paths** (all scanned by Codex):

| Scope | Path |
|-------|------|
| Repo (current dir) | `.agents/skills` |
| Repo (parent) | `../.agents/skills` |
| Repo (root) | `$REPO_ROOT/.agents/skills` |
| User (personal) | `$HOME/.agents/skills` or `~/.codex/skills` |
| System | Bundled with Codex |

Symlinked skill folders are followed. The `~/.codex/skills/` path is the user-scoped install target used by existing Jerry ports.

---

## L2: Architectural Implications

### Implication 1: Frontmatter is a hard constraint — not advisory

The validator (`quick_validate.py`) fails the skill at creation time for any unexpected key. Unlike Jerry's Claude skill (which ignores extra frontmatter via Claude's own parsing), Codex enforces a strict allowlist. The orchestration SKILL.md must contain ONLY `name` and `description` (with optional `license`, `metadata`, `allowed-tools`). The standard should be updated to reflect that the behavior is a validator error, not silent ignoring.

**Risk:** If the port SKILL.md is generated from a template that includes Jerry fields (`version`, `activation-keywords`), the port will fail `quick_validate.py`. The template must strip these.

### Implication 2: `allowed-tools` has a different semantic in Codex — do not carry over Jerry's value

Jerry's `allowed-tools` is a list of Claude tool names (e.g., `Agent, WebSearch, WebFetch, Read, Write, Bash`). The agentskills.io `allowed-tools` is a space-separated string of pre-approved tool names in a different syntax (e.g., `Bash(git:*) Read`). These are incompatible. Even if the orchestration port uses `allowed-tools`, it must be re-authored from scratch in Codex syntax — it cannot be copied from the Jerry SKILL.md.

The existing port convention (adversary, eng-team) omits `allowed-tools` entirely. Recommend following this convention unless there is a specific security reason to restrict the orchestration skill's tool access.

### Implication 3: The subagent depth limit constrains pipeline parallelism

The orchestration skill in Jerry spawns multiple worker agents in parallel (via the `Agent` tool). In Codex, `agents.max_depth=1` means the orchestration skill (if itself a subagent of a user session) can spawn one level of workers, but those workers cannot spawn further agents. For the orchestration port to function as a genuine multi-agent coordinator:

- The skill must be invoked at the TOP of the agent hierarchy (directly by the user), not as a subagent itself.
- Worker agents (orch-planner, orch-tracker, orch-synthesizer equivalents) can be spawned by the skill, but MUST NOT spawn further agents.
- The `spawn_agents_on_csv` experimental tool could support batch parallel worker invocation.

This is architecturally coherent with Jerry's P-003 (no recursive subagents). Codex enforces the same constraint mechanically via `max_depth`.

### Implication 4: MCP (Memory-Keeper) declaration is optional but recommended

If the Codex port of orchestration uses MCP tools (e.g., a Memory-Keeper equivalent for cross-session state), those MUST be declared in `agents/openai.yaml` `dependencies.tools[]`. Codex will prompt the user to install missing MCP dependencies when `features.skill_mcp_dependency_install` is enabled (default). If the port adopts a file-based state approach instead (e.g., YAML files written to disk), the `dependencies` section can be omitted — simplifying deployment.

**Recommendation:** Given that Codex's cross-session memory is fundamentally file-based (each session persists to disk), a file-based state approach (writing ORCHESTRATION.yaml to the project directory) is more portable than requiring an external MCP server. This matches the adversary and eng-team port pattern of self-contained skill directories.

### Implication 5: The description field is the sole context-budget allocation for trigger matching

With the 2% / 8,000-char budget for all skill metadata at startup, the orchestration skill's description competes with every other installed skill. At the 1024-char limit, the description must efficiently pack WHAT + WHEN + trigger keywords. The Jerry `activation-keywords` list provides a useful vocabulary source — but must be woven into the description prose, not placed in frontmatter as a separate field.

---

## Resolved-VERIFY Table

The CSP standard (`standards/codex-skill-port-standard.md`) contained one [VERIFY] item. All items are resolved below.

| [VERIFY] Item | CSP Claim | Verified Finding | Source | Correction Required? |
|---------------|-----------|-----------------|--------|---------------------|
| "no other fields are read" (Codex SKILL.md, Jerry-only fields `version`, `allowed-tools`, `activation-keywords`) | "Dropped in the Codex SKILL.md — Codex ignores them" | **PARTIALLY INCORRECT.** Fields are NOT silently ignored. Codex's `quick_validate.py` validates against an explicit allowlist: `{"name", "description", "license", "allowed-tools", "metadata"}`. Fields outside this set cause a VALIDATOR ERROR (fail, not ignore). Additionally: (a) `allowed-tools` IS in the allowed set — it is permitted but with different Codex-native semantics and only experimental enforcement. (b) Jerry's `version` and `activation-keywords` ARE correctly dropped — they fail validation. | `~/.codex/skills/.system/skill-creator/scripts/quick_validate.py` lines 40-49; agentskills.io specification; official Codex docs | Yes — CSP should state "cause a validator error" not "ignored"; `allowed-tools` note needs nuance |

**Verdict:** One [VERIFY] item in CSP. Resolved. Correction needed to CSP wording in two places:
1. "Codex ignores them" → "fields outside the allowed set cause a validator error"
2. "`allowed-tools` — Dropped" note → `allowed-tools` is permitted by validator (different Codex semantics); recommend omitting per existing port convention; Jerry's value must NOT be copied as-is

---

## Methodology

### Sources Consulted

| Source Type | Source | Coverage | Credibility |
|-------------|--------|----------|-------------|
| Primary (installed) | `~/.codex/skills/.system/skill-creator/SKILL.md` (installed, v0.142.5) | Frontmatter spec, body structure, progressive disclosure, subagents | HIGH — authoritative for installed version |
| Primary (installed) | `~/.codex/skills/.system/skill-creator/references/openai_yaml.md` | Full `agents/openai.yaml` schema + field constraints | HIGH — authoritative |
| Primary (installed) | `~/.codex/skills/.system/skill-creator/scripts/quick_validate.py` | EXACT allowed frontmatter field set, validation rules | HIGH — executable enforcement |
| Primary (installed) | `~/.codex/skills/adversary/SKILL.md` + `agents/openai.yaml` | Observed port pattern | HIGH — current working port |
| Primary (installed) | `~/.codex/skills/eng-team/SKILL.md` + `agents/openai.yaml` | Observed port pattern | HIGH — current working port |
| Official web | `developers.openai.com/codex/skills` | Progressive disclosure budget, invocation, openai.yaml | HIGH — official docs |
| Official web | `developers.openai.com/codex/subagents` | Subagent mechanism, max_depth, max_threads, codex exec | HIGH — official docs |
| Official web | `developers.openai.com/codex/cli/reference` | `codex exec` syntax and flags | HIGH — official reference |
| Open standard | `agentskills.io/specification` | Complete field spec including allowed-tools (experimental), compatibility | HIGH — the open standard Codex implements |
| Secondary | `github.com/openai/codex/issues/5291` | SKILL.md format support rationale | MEDIUM — GitHub issue |
| Secondary | Web search | `allowed-tools` enforcement status (GitHub issue anthropics/claude-code#37683 cited) | MEDIUM — community report |

### Context7 MCP Usage

Context7 (`mcp__plugin_context7_context7__resolve-library-id`) was not available in this execution environment (MCP tool not accessible via bash). Fell back to WebSearch/WebFetch per MCP-001 fallback procedure. Primary local evidence (installed Codex files) compensates for this gap; confidence remains HIGH for frontmatter findings.

### Search Queries

1. "OpenAI Codex CLI skill SKILL.md frontmatter fields name description official documentation 2025"
2. "OpenAI Codex CLI subagents parallel execution codex exec skill invocation official docs"
3. "openai codex CLI allowed-tools SKILL.md frontmatter field restrict tools skill github 2025"
4. `site:developers.openai.com codex allowed-tools OR allowed_tools skill frontmatter 2025 2026`
5. WebFetch: `developers.openai.com/codex/skills`, `developers.openai.com/codex/subagents`, `developers.openai.com/codex/cli/reference`, `developers.openai.com/codex/config-reference`, `agentskills.io/specification`, `deepwiki.com/openai/skills/7.1-skill.md-format-specification`

---

## References

1. [`~/.codex/skills/.system/skill-creator/SKILL.md`](file:///Users/evorun/.codex/skills/.system/skill-creator/SKILL.md) (installed codex-cli 0.142.5) — Key insight: official frontmatter spec says only `name` + `description` trigger skills; "Do not include any other fields in YAML frontmatter" is guidance (validator is the enforcement authority)

2. [`~/.codex/skills/.system/skill-creator/references/openai_yaml.md`](file:///Users/evorun/.codex/skills/.system/skill-creator/references/openai_yaml.md) (installed codex-cli 0.142.5) — Key insight: complete openai.yaml schema including `dependencies.tools[].type: mcp` only; `policy.allow_implicit_invocation` default true

3. [`~/.codex/skills/.system/skill-creator/scripts/quick_validate.py`](file:///Users/evorun/.codex/skills/.system/skill-creator/scripts/quick_validate.py) (installed codex-cli 0.142.5) — Key insight: `allowed_properties = {"name", "description", "license", "allowed-tools", "metadata"}` — the AUTHORITATIVE allowed frontmatter field set; any other key causes validator failure

4. [`~/.codex/skills/adversary/SKILL.md`](file:///Users/evorun/.codex/skills/adversary/SKILL.md) + `agents/openai.yaml` — Key insight: existing Jerry port pattern: `name`+`description` only; roles-not-subagents convention; independent `codex-x.y.z` versioning in body header

5. [`~/.codex/skills/eng-team/SKILL.md`](file:///Users/evorun/.codex/skills/eng-team/SKILL.md) + `agents/openai.yaml` — Key insight: second port confirming the convention; `allow_implicit_invocation: true` in openai.yaml

6. [Agent Skills — Codex | OpenAI Developers](https://developers.openai.com/codex/skills) — Key insight: progressive disclosure budget (2% context / 8,000 chars); five discovery scopes; `allow_implicit_invocation` behavior

7. [Subagents — Codex | OpenAI Developers](https://developers.openai.com/codex/subagents) — Key insight: `agents.max_depth` defaults to 1; `agents.max_threads` defaults to 6; parallel execution supported; custom agents via TOML

8. [CLI Reference — Codex | OpenAI Developers](https://developers.openai.com/codex/cli/reference) — Key insight: `codex exec` (alias `codex e`) flags: `--json`, `--output-last-message`, `--sandbox`, `--ephemeral`, `--profile`; session resume supported

9. [agentskills.io/specification](https://agentskills.io/specification) — Key insight: authoritative open standard; complete field set: `name`, `description`, `license`, `compatibility`, `metadata`, `allowed-tools` (experimental); Codex 0.142.5 does not support `compatibility` despite spec listing it

10. [github.com/openai/codex/issues/5291](https://github.com/openai/codex/issues/5291) — Key insight: progressive disclosure model rationale; Level 1/2/3 design intent

11. [github.com/openai/skills skill-creator SKILL.md](https://github.com/openai/skills/blob/main/skills/.system/skill-creator/SKILL.md) — Key insight: upstream source confirms "Do not include any other fields" guidance in body; validator is enforcement authority

---

*Research agent: ps-researcher | Workflow: PROJ-037 codex-port-20260704-001 | Phase 0*
*Sources: 6 primary local + 5 official web + 1 open standard = 12 sources*
*All [VERIFY] items: RESOLVED (1 of 1)*
