# Barrier 1 Adversarial Findings — ADR-001-codex-port.md

> **Barrier:** 1 (Design Gate)
> **Deliverable:** `eng/phase-1/eng-arch-001/ADR-001-codex-port.md`
> **Issued by:** adv-executor (C3 strategy set: S-003 → S-002 → S-007 → S-004 → S-012 → S-013)
> **Date:** 2026-07-04
> **H-16 compliance:** S-003 Steelman executed before all critique strategies (compliant)

---

## Document Sections

| Section | Purpose |
|---------|---------|
| [Steelman Reconstruction](#steelman-reconstruction) | Where the ADR is strongest; arguments pre-strengthened before critique |
| [Devil's Advocate Findings](#devils-advocate-findings) | Challenged assumptions and counter-arguments (DA-NNN) |
| [Constitutional AI Critique](#constitutional-ai-critique) | Constitutional principle compliance evaluation (CC-NNN) |
| [Pre-Mortem Analysis](#pre-mortem-analysis) | Prospective failure enumeration (PM-NNN) |
| [FMEA Component Analysis](#fmea-component-analysis) | Failure mode decomposition with RPN scores (FM-NNN) |
| [Inversion Findings](#inversion-findings) | Anti-goal and assumption inversion (IN-NNN) |
| [Consolidated Findings Table](#consolidated-findings-table) | All findings by severity |

---

## Steelman Reconstruction

**S-003 execution — Step 1 (Core Thesis):** The ADR argues that Option A (sequential-role) is the only viable architecture for the Codex port because (a) the live subagent test produced a decisive empirical failure, (b) the source Jerry skill already executes sequentially within a single Claude context, and (c) the sequential-role pattern is proven by two production Codex ports. Therefore, the port can deliver full functional value without true parallelism.

**Strongest version of the ADR's argument:**

The live subagent test at Barrier 0 is the single strongest element in this ADR. `SpawnAgent` failed at runtime with a service-tier/child-model error (×3), and `proof.txt` was not created. This is the most reliable evidence class possible for an architecture decision — an actual runtime failure, not inference from documentation. Combined with the `enable_fanout: under development, false` feature flag, Option A is not a preference; it is the only unblocked path.

The primitive mapping table (P-1 through P-11) is the most comprehensive architectural analysis in any of the PROJ-037 Phase 0 or Phase 1 artifacts. Each primitive is mapped with a confidence rating and a residual gap assessment. The claim that `execution_mode: PARALLEL` was always planning metadata — not an actual concurrent execution mechanism — is the most consequential insight in the ADR. It establishes that the Jerry source itself executes sequentially in practice, making the Codex sequential port functionally equivalent for every observable user outcome except wall-clock time savings (which were never delivered anyway).

The upgrade path in Appendix C is technically sound. The natural-language delegation phrasing is already compatible with genuine subagent invocation once the service-tier issue clears. This is a reversible decision with a specific, testable upgrade trigger.

**Steelman improvement findings:**

| ID | Type | Finding | Magnitude |
|----|------|---------|-----------|
| SM-F-001 | Structural | "Functional parity on all user-visible features" (L0, Consequences) is the weakest claim in the ADR. The phrase "user-visible features" is doing load-bearing work: it silently excludes automated worker agent coordination, which IS a user-visible feature in the Jerry source (users invoke `$orchestration` and all phases coordinate automatically). The steelmanned version adds explicit disclosure: "Automated worker agent coordination is not available; users must invoke work agents separately and report completion to orch-tracker." | Major |
| SM-F-002 | Evidence | P-5 (checkpoint/resume, Medium confidence) can be strengthened by specifying the exact SKILL.md body trigger for detecting a resume invocation: the skill should check for an existing ORCHESTRATION.yaml with `workflow.status: PAUSED` at the start of any invocation and branch accordingly. | Minor |

---

## Devil's Advocate Findings

### DA-001: Quality Gate Critic Independence Is Architecturally Absent

| Attribute | Value |
|-----------|-------|
| **Severity** | Major |
| **Section** | Primitive Mapping Table — P-6 (Quality Gate) |
| **Strategy Step** | Step 3 (Counter-arguments) — Logical flaw in mitigation claim |

**Assumption challenged:** "Minor context bleed between creator and critic in the same window. Mitigation: critic explicitly instructed to assess without anchoring on creator reasoning."

**Counter-argument:** In the Jerry source, the creator-critic quality gate relies on the critic agent having a fresh context window — it receives only the artifact and evaluation criteria, free from the creator's reasoning artifacts and sunk-cost bias. This architectural isolation is why the quality gate produces reliable independent scores.

In the Codex sequential-role port, the critic role is adopted in the SAME context window as the creator role. The context contains: (a) the creator's reasoning as it produced the artifact, (b) all intermediate tool calls and edits, (c) the creator's framing of the artifact's strengths. The behavioral mitigation ("critic explicitly instructed to assess without anchoring") relies on LLM compliance with a natural-language instruction, not architectural isolation. This instruction is unverified in any live test.

At C3 criticality, where the quality gate threshold is >= 0.92 and the gate is a real quality enforcement mechanism, an architecturally unverified critic isolation mechanism is a significant gap. If the critic systematically overscores due to anchoring on creator context, the quality gate becomes a rubber stamp rather than an independent check.

**Evidence:** P-6 mapping: "Pattern identical to adversary port's executor-scorer sequence." The adversary port's executor-scorer sequence is also in the same context window — this is presented as corroborating evidence, but the adversary port's quality gate effectiveness in the same-context model is itself unverified against an independent baseline.

**Recommendation:** Add Phase 3 verification hook V-7a: run the quality gate twice — once inline (same context), once as a fresh `codex exec` invocation with only the artifact and rubric. Compare scores. If divergence > 0.08 (per RT-M-012), document the anchoring effect and add explicit context-clearing instruction to the orch-tracker role file.

---

### DA-002: Cross-Pollination Pipeline Execution Order Not Specified

| Attribute | Value |
|-----------|-------|
| **Severity** | Major |
| **Section** | Primitive Mapping Table — P-3 (Cross-pollination handoff) |
| **Strategy Step** | Step 3 (Counter-arguments) — Unstated assumption |

**Assumption challenged:** "File-based handoff. Role A writes artifact... Role B reads it before proceeding. Codex Read/Write tools support this directly. Confidence: High. Residual Gap: None."

**Counter-argument:** The P-3 mapping describes a unidirectional handoff (A→B). In the Jerry source, cross-pollination is bidirectional: Pipeline A writes a handoff artifact for B, and Pipeline B writes a handoff artifact for A, and BOTH are read by the other pipeline's subsequent phases. The file system can support this — but it requires a specific sequential execution order that the ADR does not specify.

For bidirectional cross-pollination to work in the sequential-role port:

1. Pipeline A Phase 1 executes → writes `cross-pollination/barrier-1/ps-to-nse/handoff.md`
2. Pipeline B Phase 1 executes → READS `ps-to-nse/handoff.md` → writes `cross-pollination/barrier-1/nse-to-ps/handoff.md`
3. Pipeline A Phase 2 executes → READS `nse-to-ps/handoff.md` → proceeds with B's context

Step 3 (Pipeline A Phase 2 reads Pipeline B's artifact) requires that the SKILL.md body or orch-planner role file explicitly instructs Codex to read the nse-to-ps handoff artifact when Pipeline A resumes. This instruction is absent from the ADR. Without it, Pipeline A Phase 2 may proceed without reading Pipeline B's cross-pollination artifact, defeating the purpose of the cross-pollination pattern.

**Evidence from Phase 3 verification:** V-8 only tests "File `orchestration/test-001/cross-pollination/barrier-1/{dir}/handoff.md` exists on disk." It does NOT test that the subsequent pipeline phases READ the cross-pollination artifacts from the other pipeline. A passing V-8 gives false confidence in bidirectional cross-pollination.

**Recommendation:** Add to Appendix A P-3 mapping: "Note: The orch-planner role file must specify the sequential execution order of pipelines within each phase, and must instruct each pipeline's subsequent phase to read the other pipeline's cross-pollination artifact before proceeding. The SKILL.md body or ORCHESTRATION_PLAN.md template must provide explicit cross-pollination read instructions for each pipeline."

---

### DA-003: "Functional Parity" Claim Omits the Automation Dimension

| Attribute | Value |
|-----------|-------|
| **Severity** | Major |
| **Section** | L0 Executive Summary; Consequences — "Enabled by this decision" item 2 |
| **Strategy Step** | Step 3 (Counter-arguments) — Alternative interpretation of same evidence |

**Assumption challenged:** "Functional parity on all user-visible features. Structured workflow planning, persistent YAML state, checkpoint/resume, quality gates (creator-critic-revision cycle), cross-pollination handoffs (file-based), and synthesis are all fully available."

**Counter-argument:** The Jerry source's primary value proposition for users is AUTOMATED multi-agent coordination: a user invokes `$orchestration` and the skill automatically spawns orch-planner (which produces the plan), then spawns worker agents for each phase (problem-solving, nasa-se, adversary), then spawns orch-tracker after each completes, and finally spawns orch-synthesizer. The user supervises but does not manually trigger each step.

In the Codex sequential-role port, the equivalent workflow requires:
1. `codex exec "Use orchestration to plan workflow X"` → ORCHESTRATION_PLAN.md + ORCHESTRATION.yaml created
2. User runs `codex exec "Use $problem-solving..."` for Phase 1 agents (SEPARATELY, not coordinated by orchestration)
3. `codex exec "Update ORCHESTRATION.yaml: phase-1-agent-001 is COMPLETE"` → orch-tracker updates state
4. Repeat steps 2-3 for each subsequent phase and agent

This is a project management assistant, not an automated coordinator. The "not orchestration's responsibility" framing in the primitive mapping table confirms this — but it is not surfaced in the L0 Executive Summary or the Consequences section.

**Evidence:** Phase 3 V-5 prompt: "Update ORCHESTRATION.yaml: agent phase-1-agent-001 is now COMPLETE, artifact at orchestration/test-001/ps/phase-1/agent-001/output.md" — this is the USER reporting completion, not orchestration detecting it.

**Recommendation:** Add to L0 Executive Summary: "Users gain orchestration capability in Codex with no new infrastructure dependencies. Note: automated coordination of work agents is not available; users invoke work agents via separate `codex exec` calls and report completion to orch-tracker. The skill's value is in structured planning, persistent state tracking, quality gate enforcement, and synthesis coordination."

Add to Consequences "Foreclosed by this decision" a fourth item: "Automated worker agent dispatch is not available. The orch-planner role produces an ORCHESTRATION_PLAN.md, but the listed work agents must be invoked by the user; orchestration does not automatically spawn or coordinate them."

---

### DA-004: Re-Entrant Context Accumulation Risk for Long Workflows

| Attribute | Value |
|-----------|-------|
| **Severity** | Major |
| **Section** | Decision — "What Sequential-Role Means for Orchestration"; Consequences — Foreclosed item 2 |
| **Strategy Step** | Step 4 (Quantification) |

**Assumption challenged:** "Long workflows may experience context pressure" is acknowledged but treated as minor.

**Counter-argument:** For a realistic C3 workflow (Problem-Solving 3 phases + NSE 3 phases + Adversarial 1 phase + Synthesis 1 phase = 8 phases, 2 pipelines), the orch-tracker role is adopted approximately 16 times (once per agent per phase). Each adoption:
- Reads `references/orch-tracker.md` (~3,000-5,000 tokens each read)
- Reads full `ORCHESTRATION.yaml` (growing from ~500 tokens to ~5,000+ tokens as phases complete)
- Writes updated ORCHESTRATION.yaml

By adoption 8, the context window contains:
- Initial SKILL.md body and frontmatter
- orch-planner role file (read once, ~5,000 tokens)
- All 8 ORCHESTRATION.yaml reads + writes (Edit tool history included)
- 8 orch-tracker role file reads

Estimated context consumption at adoption 8: 50,000-80,000 tokens. At adoption 16: 100,000-150,000 tokens. For a 200K context window, this approaches exhaustion for realistic workflows, before synthesis even begins.

The consequences include: (a) earlier ORCHESTRATION.yaml state is summarized/truncated by Codex, causing tracker to read incomplete history; (b) quality gate scores from earlier phases may be lost, preventing accurate trend analysis; (c) orch-synthesizer, adopted last, inherits a context window dominated by tracking history rather than artifact content.

**Evidence:** ADR Consequences section, Foreclosed item 2: "Long workflows may experience context pressure." The ADR acknowledges this without quantification or mitigation.

**Recommendation:** Add to Appendix E Phase 3 Verification Hooks a stress test: V-11 (supplemental): run orch-tracker adoption 6+ times in one session and verify ORCHESTRATION.yaml state accuracy on the 6th adoption. Add to Phase 2 engineering guidance: "references/orch-tracker.md should instruct Codex to use offset/limit reads on ORCHESTRATION.yaml when the file exceeds 2,000 tokens, loading only the relevant sections (workflow.status, current phase, quality metrics)."

---

## Constitutional AI Critique

### CC-001: CSP-04 Violation — `default_prompt` Omits `$orchestration`

| Attribute | Value |
|-----------|-------|
| **Severity** | Minor |
| **Section** | Appendix B — File Manifest, item 2 (agents/openai.yaml); "Note on `default_prompt`" |
| **Strategy Step** | Step 3 — Principle-by-principle evaluation |

**Principle:** CSP-04 — "`openai.yaml.interface.default_prompt` MUST reference `$<skill-name>` and stay consistent with the SKILL.md description." Source: `standards/codex-skill-port-standard.md` §Parity Maintenance.

**Proposed `default_prompt`:** "Plan and execute a structured multi-agent workflow with phase tracking, sync barriers, quality gates, and checkpoint resumption using the orchestration skill."

**Assessment:** VIOLATED. The proposed `default_prompt` does not contain `$orchestration`. The ADR justifies this by citing production port precedent (adversary and eng-team omit the literal), and notes "If the Codex skill-creator validation requires the `$`-prefix literal in a future version, add `$orchestration` to the sentence at that time."

**Risk:** (a) Official spec (openai_yaml.md) requires it NOW; (b) CSP-04 says MUST; (c) if Codex validates `default_prompt` content in a future version, the port fails without warning; (d) no formal standards deviation is recorded in the ADR.

**Mitigation:** The production ports demonstrate that omitting `$orchestration` does not cause validation failure in Codex 0.142.5. The functional risk is low. However, the deviation from CSP-04 should be formally recorded in Appendix D alongside the CC-001 corrections already documented.

**Recommendation:** Either (a) add `$orchestration` to the proposed `default_prompt`: "Plan and execute a structured multi-agent workflow using `$orchestration` — with phase tracking, sync barriers, quality gates, and checkpoint resumption." OR (b) formally document the production-port precedent as a CSP-04 exception in Appendix D with the justification: "Existing production ports (adversary, eng-team) do not include the `$<skill-name>` literal in `default_prompt` and function correctly. This convention is adopted for consistency."

---

### CC-002: P-022 Ambiguity in "Functional Parity" Claim

| Attribute | Value |
|-----------|-------|
| **Severity** | Major |
| **Section** | L0 Executive Summary; Consequences |
| **Strategy Step** | Step 3 — Principle-by-principle evaluation |

**Principle:** P-022 — No Deception. NEVER deceive about actions, capabilities, or confidence.

**Assessment:** AMBIGUOUS. The "functional parity on all user-visible features" claim is technically defensible (all the listed features are available) but creates a misleading impression that the Codex port is functionally equivalent to the Jerry source for the primary orchestration use case (automated pipeline coordination). This is not deception in the intentional sense, but it creates a confident-sounding claim that could mislead Phase 2 engineers into building SKILL.md body instructions that promise automation without verifying how it would work.

**Recommendation:** Resolve via DA-003 recommendation: add explicit disclosure to L0 and Consequences that automated worker agent coordination is not available.

---

## Pre-Mortem Analysis

**Failure declaration:** "It is 2027-01-04. The Codex orchestration port was built per ADR-001 but is not in production use. Phase 3 live verification failed or the skill is installed but not used. We are investigating why."

### PM-001: Phase 3 Verification Scope Too Narrow for Realistic Workflows

| Attribute | Value |
|-----------|-------|
| **Severity** | Major |
| **Section** | Appendix E — Phase 3 Verification Hooks |
| **Strategy Step** | Step 3 — Enumerate failure causes |

**Failure cause:** The "minimum viable end-to-end" sample workflow in Appendix E is a 2-phase, 1-pipeline workflow with no cross-pollination. This is the easiest possible workflow for the sequential-role architecture. Tests V-1 through V-10 all pass — but a realistic C3 workflow with 3+ phases per pipeline and bidirectional cross-pollination reveals the context accumulation and execution ordering gaps identified in DA-002 and DA-004.

Phase 3 passes with the minimal workflow. Phase 2 ships the built port. The first production workflow (a real PROJ-NNN with 2 pipelines and 6 phases) encounters orch-tracker state corruption at phase 4 due to context pressure. Port is abandoned.

**Recommendation:** Add V-11 (supplemental, required for P-3 cross-pollination validation): run a 2-pipeline, 2-phase-each cross-pollination workflow. Verify (a) both pipelines write cross-pollination artifacts, (b) each pipeline's Phase 2 reads the other's Phase 1 artifact. Add V-12 (supplemental, required for orch-tracker stress test): run orch-tracker 6+ times in one `codex exec` session. Verify state accuracy on the 6th adoption.

---

### PM-002: User Adoption Failure Due to Expectation Mismatch

| Attribute | Value |
|-----------|-------|
| **Severity** | Major |
| **Section** | L0 Executive Summary; Consequences |
| **Strategy Step** | Step 3 — Enumerate failure causes |

**Failure cause:** Users familiar with Jerry's `/orchestration` source invoke `$orchestration` expecting it to coordinate their entire pipeline automatically. They invoke it, see an ORCHESTRATION_PLAN.md created, and then... nothing happens. They must manually invoke each work agent and report completion to orch-tracker. The experience is confusing and the value is opaque.

The L0 Executive Summary does not set this expectation. The skill description (proposed frontmatter) promises "multi-agent workflow orchestration with state tracking, phase sequencing, sync barriers, cross-pollination handoffs, checkpointing, and cross-session resumption" — none of this text communicates that the user must manually drive each phase.

**Recommendation:** The SKILL.md body must include a clear "How to use" section that explicitly describes the manual coordination model:
1. `$orchestration` creates your plan (ORCHESTRATION_PLAN.md + ORCHESTRATION.yaml).
2. Invoke your work agents separately using their skills.
3. Report each completion: `$orchestration Update: [agent-id] is complete at [path]`.
4. After all phases complete: `$orchestration Synthesize workflow [id]`.

---

### PM-003: Cross-Pollination Bidirectional Failure Silent Under V-8 Test

| Attribute | Value |
|-----------|-------|
| **Severity** | Major |
| **Section** | Appendix E — V-8 verification hook |
| **Strategy Step** | Step 4 — Cascade analysis |

**Failure cause:** V-8 passes (the handoff.md file exists), but the second pipeline never reads the first pipeline's cross-pollination artifact. The SKILL.md body or orch-planner role file doesn't instruct subsequent pipeline phases to read cross-pollination artifacts from the other pipeline. The cross-pollination pattern silently degrades to independent parallel (then sequential) pipelines with no actual information exchange.

**Recommendation:** V-8 test must be extended: after creating the handoff artifact (V-8a), verify V-8b: "Run a prompt instructing orch-tracker to proceed with Pipeline B Phase 2, and confirm that Codex reads the Pipeline A cross-pollination artifact before generating B's Phase 2 output." Session transcript must show a Read tool call on `cross-pollination/{barrier_id}/ps-to-nse/handoff.md` before B's Phase 2 work begins.

---

## FMEA Component Analysis

FMEA applies to the seven components in the ADR's file manifest (Appendix B). RPN = Severity × Occurrence × Detectability.

| ID | Component | Failure Mode | Effect | S | O | D | RPN | Severity |
|----|-----------|--------------|--------|---|---|---|-----|---------|
| FM-001 | SKILL.md frontmatter | Jerry-only field left in | Validator ERROR; port invalid | 9 | 2 | 1 | 18 | Minor — V-1 catches |
| FM-002 | SKILL.md body | Re-entrant orch-tracker instruction ambiguous | Tracker not adopted; state not updated | 8 | 6 | 5 | 240 | Major |
| FM-003 | ORCHESTRATION.yaml | Concurrent read-write race | Corrupt YAML state | 9 | 1 | 1 | 9 | Minor — sequential design prevents |
| FM-004 | orch-tracker.md | Context saturation after 8+ adoptions | Quality scores estimated from truncated state | 7 | 6 | 8 | 336 | Major |
| FM-005 | Cross-pollination | B's Phase 2 does not read A's artifact | Cross-pollination fails silently | 8 | 6 | 8 | 384 | Major |
| FM-006 | agents/openai.yaml | `default_prompt` missing `$orchestration` | Explicit invocation syntax not demonstrated | 3 | 9 | 2 | 54 | Minor |
| FM-007 | Quality gate (P-6) | Critic anchors on creator context | Gate scores systematically inflated | 6 | 7 | 7 | 294 | Major |
| FM-008 | SKILL.md body | Resume detection not specified | New `codex exec` overwrites existing ORCHESTRATION.yaml | 8 | 6 | 5 | 240 | Major |

**Top 4 by RPN (all Major):**
- FM-005 (384): Cross-pollination bidirectional failure
- FM-004 (336): Context saturation for long workflows
- FM-007 (294): Quality gate critic anchoring
- FM-002/FM-008 (240 tie): Re-entrant trigger ambiguity; Resume detection missing

---

## Inversion Findings

**Inverted goal:** "How do we GUARANTEE the Codex orchestration port FAILS in Phase 3 live verification?"

### IN-001: Guarantee Failure — Leave Orch-Tracker Adoption Trigger Unspecified

| Attribute | Value |
|-----------|-------|
| **Severity** | Minor |
| **Section** | Decision — "What Sequential-Role Means"; File Manifest — item 4 |
| **Strategy Step** | Step 2 — Invert goals |

**Anti-goal:** Do NOT specify when or how to trigger orch-tracker re-adoption in the SKILL.md body.

**Consequence of anti-goal:** Codex may adopt orch-tracker once (or never), leaving ORCHESTRATION.yaml in a stale state. Phase tracking fails. Quality gate history is lost.

**Stressed assumption:** "Add explicit note: 'This role is re-entrant — adopt it after every agent or phase completion.'" — this note is in the role file spec, but the SKILL.md BODY is what Codex reads first. The SKILL.md body must also specify the re-entrant pattern with the exact trigger condition: "immediately after each work agent completes, before proceeding to the next agent."

**Recommendation:** Phase 2 SKILL.md body must include an explicit ordered workflow loop with numbered steps, not just a general description. Example: "(1) Adopt orch-planner. (2) For each phase: (2a) Work agent executes. (2b) Adopt orch-tracker: update ORCHESTRATION.yaml. (2c) Check quality gate score. (2d) If PASS: proceed. Else if iterations < 3: revise. Else: ESCALATE. (3) Adopt orch-synthesizer."

---

### IN-002: Guarantee Failure — Trust the File System Without Specifying Read Order

| Attribute | Value |
|-----------|-------|
| **Severity** | Major |
| **Section** | Primitive Mapping Table — P-3 |
| **Strategy Step** | Step 3 — Stress-test assumptions |

**Anti-goal:** Assume that because both pipeline artifacts are written to disk, the subsequent pipeline phases will naturally read them.

**Consequence of anti-goal:** The orch-planner role file produces an ORCHESTRATION_PLAN.md that sequences Pipeline A Phase 2 immediately after Pipeline B Phase 1. But without an explicit instruction to "read Pipeline A's Phase 1 cross-pollination artifact before beginning Pipeline B Phase 2," Codex will not perform the read. The read is assumed; it is not guaranteed by the file's existence.

**Stressed assumption:** "File write then file read is functionally equivalent to the cross-pollination artifact." — this is true in the Jerry source because the agent receiving the handoff is explicitly prompted with the handoff path. In the Codex sequential-role port, the orch-planner role must include explicit file-read instructions for each cross-pollination step in the ORCHESTRATION_PLAN.md template.

**Recommendation:** The ORCHESTRATION_PLAN.template.md (file #9 in manifest) needs a cross-pollination section that lists specific Read instructions for each barrier: "Before Pipeline B Phase 2 begins, read: `orchestration/{id}/cross-pollination/barrier-1/ps-to-nse/handoff.md`." This template is listed as "Carried verbatim" from the Jerry source in the ADR — but the Jerry source template may not contain these explicit read instructions because in Jerry, the handoff paths are passed explicitly to the receiving agent's Task() invocation.

---

## Consolidated Findings Table

| ID | Strategy | Severity | Finding | Section |
|----|----------|---------|---------|---------|
| DA-001 | S-002 | **Major** | Quality gate critic independence absent — context bleed in same-window creator-critic cycle; behavioral mitigation unverified | P-6 primitive mapping |
| DA-002 | S-002 | **Major** | Cross-pollination pipeline execution order not specified — bidirectional cross-pollination unaddressed; V-8 only tests artifact existence | P-3 primitive mapping |
| DA-003 | S-002 | **Major** | "Functional parity" claim omits automation dimension — worker agent coordination is manual, not automated; not disclosed in L0 | L0 Summary; Consequences |
| DA-004 | S-002 | **Major** | Re-entrant orch-tracker context accumulation unmitigated — realistic workflows (8+ adoptions) risk context exhaustion; Phase 3 scope too narrow | Consequences Foreclosed-2; Appendix E |
| CC-002 | S-007 | **Major** | P-022 ambiguity — "functional parity" framing creates misleading impression about automation capability for Phase 2 engineers | L0; Consequences |
| PM-001 | S-004 | **Major** | Phase 3 verification scope too narrow — 2-phase minimal workflow cannot validate context accumulation or cross-pollination ordering | Appendix E |
| PM-002 | S-004 | **Major** | User adoption risk — skill description and L0 do not disclose manual coordination model; expectation mismatch likely | L0; frontmatter description |
| PM-003 | S-004 | **Major** | V-8 cross-pollination test insufficient — artifact existence does not verify bidirectional read ordering | Appendix E V-8 |
| FM-005 | S-012 | **Major** | RPN 384 — cross-pollination B→A artifact read not guaranteed by any role file instruction | P-3; ORCHESTRATION_PLAN template |
| FM-004 | S-012 | **Major** | RPN 336 — context saturation at orch-tracker adoption 8+ causes quality score truncation | orch-tracker.md; Appendix E |
| FM-007 | S-012 | **Major** | RPN 294 — quality gate critic anchoring unverified; no baseline comparison proposed | P-6; SKILL.md body |
| FM-002 | S-012 | **Major** | RPN 240 — re-entrant trigger ambiguity; SKILL.md body language deferred without scaffold | Decision section; file #4 |
| FM-008 | S-012 | **Major** | RPN 240 — resume detection (PAUSED vs. fresh) not specified in SKILL.md body | P-5; V-10 |
| IN-002 | S-013 | **Major** | Cross-pollination read order is assumed, not specified — ORCHESTRATION_PLAN.template.md "carried verbatim" may lack explicit read instructions | P-3; file #9 |
| CC-001 | S-007 | **Minor** | CSP-04 violation — `default_prompt` omits `$orchestration` without formal deviation record | Appendix B file #2 |
| SM-F-002 | S-003 | **Minor** | P-5 resume detection trigger (PAUSED status check) not specified in SKILL.md body scaffold | P-5; Appendix E V-10 |
| IN-001 | S-013 | **Minor** | Re-entrant orch-tracker trigger requires numbered loop in SKILL.md body, not just role file note | File #4; SKILL.md body |

---

**Finding counts:**
- Critical: 0
- Major: 14 (note: several are overlapping perspectives on the same 4 root issues — see design-review.md for consolidation)
- Minor: 3

**Root issues (consolidated):**
1. Cross-pollination execution ordering (DA-002, FM-005, PM-003, IN-002)
2. Context accumulation for long workflows (DA-004, FM-004, PM-001)
3. Manual coordination not explicitly disclosed (DA-003, CC-002, PM-002)
4. Quality gate critic isolation (DA-001, FM-007)

---

*Findings report: Barrier 1 (Design Gate)*
*Strategy sequence: S-003 → S-002 → S-007 → S-004 → S-012 → S-013*
*H-16 compliant: Steelman executed before all critique strategies*
*P-003 compliant: no subagents spawned*
*Created: 2026-07-04*
