---
name: {skill-name}
description: >-
  {WHAT the skill does} Use when {WHEN to use it — trigger conditions and phrases}.
  {Key capabilities.} Do NOT use for {scope boundary}. NOTE: Codex reads only `name` and
  `description` to trigger this skill — keep this comprehensive. No XML angle brackets (H-26 parity).
---

# {Skill Title} — {one-line purpose} (Codex port)

> **Codex port version:** `codex-1.0.0` — versioned independently of the Jerry source skill.
> **Forked from:** Jerry `/{skill-name}` v{X.Y.Z} (divergence point: {YYYY-MM-DD}). This Codex port is
> a separate running version: it is **not** kept in lockstep with the Jerry original and may
> intentionally diverge. Bump this `codex-x.y.z` version on its own cadence when this port or its
> bundled references change; do not assume parity with `skills/{skill-name}/`.

> Ported from the Jerry Framework `/{skill-name}` Claude skill (`skills/{skill-name}/SKILL.md`).
> {One or two sentences on how the multi-agent model maps to Codex — e.g. subagent spawning, or roles
> adopted one at a time. Keep methodology identical to the Claude original as of the divergence point.}

## What this skill does

{Body — Codex-facing instructions. Loaded only after the skill triggers.}

## When to use / When NOT to use

{Trigger conditions and anti-patterns.}

## Workflow

{Step-by-step. Reference `references/<role>.md` files for role/subagent briefs.}

## References

- `references/{role}.md` — {purpose}
