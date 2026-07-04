# Codex Skill Port Standard (CSP)

> Project-scoped standard for porting a Jerry skill to the OpenAI Codex CLI skill format and
> **maintaining frontmatter across both surfaces**. Grounded in observed Codex ports
> (`~/.codex/skills/adversary`, `eng-team`) and Codex's own `skill-creator` reference
> (`~/.codex/skills/.system/skill-creator/`). Codex-specific field constraints in this document
> are marked **[VERIFY]** where they MUST be confirmed against official Codex documentation during
> Phase 0 (`/problem-solving` research, MCP-001). Promote to `.context/rules/` as a framework
> standard only after adversary review (would be auto-C3 per AE-002).

## Document Sections

| Section | Purpose |
|---------|---------|
| [Scope](#scope) | What this standard governs |
| [Frontmatter Standards](#frontmatter-standards) | Jerry vs Codex frontmatter, maintenance rules |
| [Port Structure](#port-structure) | Required Codex skill file layout |
| [Divergence & Versioning](#divergence--versioning) | How the port version is tracked independently |
| [Parity Maintenance](#parity-maintenance) | Keeping the two frontmatters valid over time |
| [Verification](#verification) | How compliance is checked (incl. official-doc verification) |

---

## Scope

Applies to any Jerry `skills/<name>/` ported to `~/.codex/skills/<name>/`. Governs the porting
structure, the two frontmatter surfaces (Jerry SKILL.md + Codex SKILL.md / `agents/openai.yaml`),
independent version tracking, and ongoing maintenance. Does NOT change the Jerry source skill.

---

## Frontmatter Standards

### Jerry source frontmatter (unchanged, authoritative for the Claude skill)

| Rule | Source | Requirement |
|------|--------|-------------|
| H-25 | skill-standards.md | `SKILL.md` exact case; folder kebab-case matching `name`; no `README.md` |
| H-26 | skill-standards.md | `description` = WHAT + WHEN + trigger phrases, < 1024 chars, no XML `< >`; repo-relative paths; registered in CLAUDE.md/AGENTS.md/mandatory-skill-usage.md |
| Jerry fields | skill-standards.md | `version`, `allowed-tools`, `activation-keywords` (Jerry-required) |

### Codex port frontmatter (Codex SKILL.md)

| Field | Required | Constraint | Status |
|-------|----------|------------|--------|
| `name` | Yes | Matches Codex skill folder | Confirmed (skill-creator) |
| `description` | Yes | Only `name`+`description` are read by Codex to trigger the skill; make it comprehensive (WHAT + WHEN). No `< >` (safe-frontmatter parity with H-26). | Confirmed (skill-creator) |
| Jerry-only fields (`version`, `allowed-tools`, `activation-keywords`) | No | **Dropped** in the Codex SKILL.md — Codex ignores them. Port version tracked in the body header instead (see Divergence). | **[VERIFY]** no other fields are read |

### Codex `agents/openai.yaml` (UI/harness metadata — NOT read by the agent)

| Field | Constraint | Status |
|-------|------------|--------|
| `interface.display_name` | Human title for UI lists/chips | Confirmed |
| `interface.short_description` | 25–64 chars | Confirmed |
| `interface.default_prompt` | 1 sentence; MUST mention the skill as `$<skill-name>` | Confirmed |
| `interface.icon_small`/`icon_large`/`brand_color` | Optional; include only if provided | Confirmed |
| `dependencies.tools[]` | Only `type: mcp` supported | Confirmed |
| `policy.allow_implicit_invocation` | `true` = injected by default; `false` = explicit `$skill` only. Default `true`. | Confirmed |

**Formatting rules (Codex):** quote all string values; keep keys unquoted. Prefer generating via the
Codex `scripts/generate_openai_yaml.py` / `init_skill.py --interface key=value` for determinism.

---

## Port Structure

```
~/.codex/skills/<name>/
├── SKILL.md              # Codex frontmatter (name, description) + body (Codex port)
├── agents/
│   └── openai.yaml       # UI/harness metadata (interface, dependencies, policy)
├── references/           # Jerry sub-agents flattened OR mapped to Codex subagents
│   └── <role>.md
├── scripts/              # optional deterministic helpers
└── assets/               # optional icons
```

A **git-tracked mirror** MUST be kept in the Jerry repo at `codex-ports/<name>/` (source of truth;
`~/.codex/` is the install target).

---

## Divergence & Versioning

- The Codex port is versioned **independently** as `codex-x.y.z` in the SKILL.md body header, and is
  NOT kept in lockstep with the Jerry source (matches the `adversary`/`eng-team` port convention).
- The body header MUST state: forked-from Jerry version, divergence date, and "may intentionally
  diverge; bump `codex-x.y.z` on its own cadence."
- Bump `codex-x.y.z` whenever the port or its bundled references change.

---

## Parity Maintenance

| Rule | Requirement |
|------|-------------|
| CSP-01 | On any change to the Jerry source frontmatter/description, review the Codex `description` and `openai.yaml` for staleness; regenerate `openai.yaml` if stale. |
| CSP-02 | Codex `SKILL.md` `description` MUST remain free of XML `< >` (H-26 parity — Codex frontmatter is also injected). |
| CSP-03 | The repo mirror `codex-ports/<name>/` MUST match `~/.codex/skills/<name>/` after each change (drift check). |
| CSP-04 | `openai.yaml.interface.default_prompt` MUST reference `$<skill-name>` and stay consistent with the SKILL.md description. |

---

## Verification

| Check | Method |
|-------|--------|
| Official-doc verification of Codex fields ([VERIFY] items) | **Phase 0 `/problem-solving` (ps-researcher)** analyzes official Codex documentation via Context7 / official sources (MCP-001); WebSearch fallback. Findings resolve every [VERIFY]. |
| Codex loads the ported skill | `codex` loads without error; skill triggers on its description |
| Frontmatter valid | `name`/`description` present; no `< >`; `openai.yaml` quoting rules satisfied |
| Divergence header present | `codex-x.y.z` + forked-from + divergence note in body |
| Mirror parity | `codex-ports/<name>/` == `~/.codex/skills/<name>/` |

---

*Standard: project-scoped (PROJ-037). Grounded in observed Codex ports + Codex skill-creator reference.*
*[VERIFY] items pending Phase 0 official-documentation analysis via /problem-solving.*
