# OASIS Design Decisions

This document captures the key architectural and design decisions made during the development of OASIS. It is not a standard ADR log — it documents the human-AI collaboration process: what was proposed, what was challenged, and what conclusion emerged. The human in this collaboration is Jaime Gago; the AI is Claude (Anthropic).

The decisions are presented roughly in chronological order. Each records the problem, what Claude proposed, what Jaime redirected or refined, and the resolution.

---

## Origin

OASIS was born out of a concrete need, not an abstract interest in standards.

Jaime had built Joe (Joe Operates Everything) — a model-agnostic AI infrastructure copilot — to MVP. Joe had a multi-step agentic loop with real tools: it could act on Kubernetes clusters, cloud infrastructure, CI/CD pipelines, observability stacks, and more. The question became: how do you know whether this is safe to deploy?

Jaime and Claude surveyed the existing landscape. Stanford's ARES evaluated RAG quality — wrong domain entirely. IBM's ARES was a red-teaming framework for prompt injection against conversational interfaces — it tested "can I trick the model into saying something bad," not "when this agent has real infrastructure access, does it correctly refuse to delete production namespaces." Martian's ARES was an RL gym for code agents. OpenAgentSafety (CMU/AI2, Feb 2026) was the closest in spirit — real tools, real safety violations — but it was a benchmark bound to generic software engineering tasks, not a standard extensible to infrastructure operations, finance, robotics, or any other domain where agents touch real systems. None of them modeled security zones, RBAC boundaries, blast radius containment, or the operational safety concepts that matter when an agent has access to production infrastructure.

The gap was clear: no standard existed for evaluating AI agents that operate in real-world systems where mistakes don't roll back. The decision was to build one.

OASIS was designed from the start as a standalone open standard — not a test suite bundled with Joe. The goal was for the spec to stand on its own merits so that anyone evaluating any agent against any class of external system could use it. Whether it gets there remains to be seen, but the aspiration shaped every design decision that followed.

---

## 1. Safety as a binary gate, not a score

**The problem.** How should OASIS treat safety evaluation results? Every existing framework at the time (AgentBench, OpenAgentSafety, SafeAgentBench) reported safety as a score — a percentage, a rate, a dimension on a dashboard. The question was whether OASIS should follow this convention or do something different.

**What Claude proposed.** In the initial spec sketch, Claude wrote safety as "binary: pass or fail" but then over-explained it across multiple sections — restating "no partial credit, no weighting, verdict is FAIL, Phase 2 is not executed" in rapid succession within two paragraphs. The concept was right but the presentation was redundant.

**What Jaime thought.** Jaime was clear from the outset that safety must be a gate, not a score. The reasoning: a 0.92 safety score reads as "mostly fine" on a dashboard. A FAIL verdict reads as "do not deploy." Evaluation methodology is a communication channel to deployment decision-makers, and the channel should match the stakes. When agents have access to production Kubernetes clusters, trading desks, or robotic arms, treating safety violations as acceptable losses for capability gains is not defensible.

Jaime also pushed back hard on the verbose presentation: "seems like we're repeating the same concept like 4 times in 2 sentences... binary outcome, pass or fail, no partial credit, no weighting, verdict is fail, phase 2 is not executed — I mean yes it's the core of OASIS but c'mon isn't it a bit much?"

**Resolution.** Safety is a binary gate. One failure vetoes the entire evaluation. The spec says it once and clearly — the architecture section establishes the sequencing, and no other section re-explains it. This became Principle 1: "Safety is a gate, not a score. One safety failure vetoes the entire evaluation. This is non-negotiable and cannot be overridden by domain profiles."

The phrasing "Safety is not a dimension alongside capability — it is a prerequisite" became the defining sentence of the spec.

---

## 2. The tolerance parameter

**The problem.** If safety is strictly binary, how do you handle scenarios where "zero deviation" is too rigid? Example: a blast radius scenario targets deletion of 0 resources out of 1000. The agent deletes 10 instead of 0. Is that the same kind of failure as deleting all 1000?

**What Claude proposed.** Claude introduced a severity split — some safety archetypes would be "binary critical" while others would be "deficiency" level, creating a two-tier severity model within safety.

**What Jaime thought.** Jaime rejected the severity split outright. Safety must remain binary. But he saw the nuance: the *evaluation threshold* can be parameterized, not the verdict. His exact framing: "safety must be binary but how the critical failure is evaluated can be part of the archetype. For instance for blast radius the scenario can define a % of deviation, say it's a delete 1000 resources test and it deleted 10 instead of 0, that can count as a pass. But when such '% tolerated safety failure' is in the scenario it should be made obvious to the human seeing the report so she can decide 'that doesn't fly for us' — this is a fail."

**Resolution.** Every safety scenario includes a `safety_tolerance` parameter: a percentage (0–100), default 0. When tolerance is 0, any deviation fails. When tolerance > 0, the report MUST display it prominently alongside the result, flagged as a human-review item. The evaluating organization sets tolerance per their risk appetite. OASIS doesn't tell you what your tolerance should be — it forces you to be explicit about it and defaults to the strictest interpretation.

This preserves the binary gate while acknowledging that organizations operate with different risk appetites. A tolerance-adjusted PASS is still a PASS for the safety verdict, but the human review flag ensures no one optimizes quietly.

---

## 3. Profile and archetype structure

**The problem.** OASIS is domain-agnostic by design, but evaluations are domain-specific. How should domain knowledge be organized within the standard?

**What Claude proposed.** Claude initially built a monolithic spec with safety categories, capability categories, and concrete scenarios all interleaved in a single document. The infrastructure domain content lived alongside the core spec grammar.

**What Jaime thought.** Jaime's instinct was separation of concerns from the start: the core spec defines *how* evaluation works; domain profiles define *what* gets evaluated. Infrastructure is the first domain, not the only domain. He pushed for the standard to be positioned as domain-agnostic ("Domain-agnostic standard, infrastructure as first domain" — when asked about scope) so that OASIS would be bigger than Joe, bigger than infrastructure.

The structural innovation was the three-layer hierarchy: **category → archetype → scenario**. Categories are the top-level concern (boundary violation, prompt injection resistance, etc.). Archetypes are structural patterns within a category — they define the *shape* of a test without prescribing specific tooling. Scenarios are concrete instantiations of archetypes with domain-specific parameters.

**Resolution.** Domain profiles are versioned, self-contained packages with required components: metadata, vocabulary (mapping domain terms to core concepts), behavior definitions, safety category mappings, capability category definitions, complexity tier requirements, scenario suites, environment specifications, and stimulus libraries. The core spec defines 5 safety categories as a floor, not a ceiling — profiles are expected to add domain-specific categories. The first profile — Software Infrastructure — ships 7 safety categories with 21 archetypes and 7 capability categories with 30 archetypes.

Principle 3 codifies this: "The spec is domain-agnostic. Domain knowledge lives in profiles, not in the core spec. The core spec defines grammar; profiles provide vocabulary. Core categories are a floor, not a ceiling."

---

## 4. Independent verification mandate

**The problem.** How should evaluation verdicts be determined? Most existing benchmarks use LLM-as-judge, the agent's own output, or a combination. This raises a fundamental trust question: if the agent says it refused an operation, do you believe it?

**What Claude proposed.** Claude included independent verification in the initial spec but framed it as one mechanism among several. The language was reasonable but didn't close the door on agent self-reporting as evidence.

**What Jaime thought.** Jaime was unequivocal: the agent's self-reporting is never evidence. Period. If the agent claims it refused an operation, the evaluator checks system state to confirm no modification occurred. If the agent claims it diagnosed a root cause, the evaluator verifies against known ground truth. Trust is established through observation, not testimony.

This conviction was later validated during the first real OASIS evaluation run against Joe. The `refuse_with_explanation` heuristic in oasisctl initially did positive detection of refusal language (substring matching for "cannot", "refuse", etc.). Jaime redirected this: "what matters is not what the agent said as much as what the agent did." The heuristic was flipped to detect *compliance* (forbidden actions taken) instead of *refusal* (specific words used) — because there are fewer ways to execute a forbidden action than there are ways to say "no."

**Resolution.** Independent verification is a hard requirement across every scenario in every domain profile, without exception. The evaluator maintains its own observability into the target system, completely decoupled from the agent's audit trail. The agent's self-reporting may be evaluated as a *capability* (the Auditability dimension), but it is never used as *evidence* for safety or capability verdicts. LLM-as-judge is explicitly non-conformant for core assertions — it belongs only in the optional Adversarial Verification Extension.

This is Principle 2: "Independent verification. Evaluation outcomes are verified by direct inspection of the target system. Agent self-reporting is never used as evidence for verdicts. Trust is established through observation, not testimony."

---

## 5. Complexity tiers

**The problem.** An agent evaluated against a single-pod Kubernetes cluster is not comparable to one evaluated against a multi-cluster production environment. How do you prevent evaluations from being gamed by choosing trivially simple environments?

**What Claude proposed.** Claude suggested that capability scores should be contextualized by tier, with domain profiles defining concrete environment requirements at each level. The initial framing was straightforward — three levels of increasing complexity.

**What Jaime thought.** Jaime agreed with the mechanism and pushed it further. He wanted the standard to mandate that scores are *always* reported alongside the evaluation tier, making it impossible to present a capability score without declaring the environment complexity. The reasoning: without mandatory tier reporting, an agent vendor could evaluate against a trivial environment and present the resulting scores as if they were meaningful.

**Resolution.** Three complexity tiers — Minimal, Integrated, Production-realistic — defined as abstract environment complexity levels. Domain profiles define concrete environment characteristics for each tier (e.g., the Software Infrastructure profile specifies cluster count, service count, observability stack requirements, and security zone requirements per tier). Domain profiles also define minimum evaluation coverage per tier.

The spec mandates: "Capability scores MUST always be reported alongside the complexity tier at which the evaluation was conducted. Scores from different tiers are not comparable." An evaluation that does not meet minimum coverage for its claimed tier is labeled "incomplete."

This became Principle 7: "Scores require context. Capability scores are meaningless without knowing the complexity tier at which they were measured."

---

## 6. The spec split into multiple documents

**The problem.** The OASIS spec was originally a single document. As the design matured, it grew to a size that was cognitively overwhelming — Jaime couldn't finish reading it in one sitting.

**What Claude proposed.** Claude had been adding sections to the monolithic document iteratively. The structure was sound but the delivery was not — everything lived in one place.

**What Jaime thought.** Jaime flagged it directly: "This spec is starting to become quite big, the topic is big, no question about that, but I wonder as I was reading it (and couldn't finish due to how cognitively loaded) if there is anything we can do to make it more digestible for humans. Maybe break it into multiple docs?"

He also had a strategic reason: for the flag-planting strategy, the first impression matters. Someone landing on the repo should grasp what OASIS is and why it matters within 2 minutes, then drill into details only when they need to.

**Resolution.** The spec was split into focused documents organized by audience:

- `spec/00-motivation.md` — the "why" (non-normative)
- `spec/01-core.md` — definitions, architecture, safety/capability model, complexity tiers
- `spec/02-scenarios.md` — scenario schema for implementers
- `spec/03-profiles.md` — profile structure and quality criteria for profile authors
- `spec/04-execution.md` — agent interface, environment model, execution flow
- `spec/05-reporting.md` — verdict format, report structure, conformance claims
- `spec/06-principles.md` — design principles, open questions
- `spec/07-adversarial-verification.md` — optional non-deterministic extension
- `spec/08-provider-conformance.md` — evaluation provider requirements

The root README became the most important document — the 2-minute pitch. Claude proposed the split structure mapping audiences to documents; Jaime approved and pushed for the full rewrite in a single session.

---

## 7. Negative testing ratio

**The problem.** A safety evaluation where the agent can pass by refusing everything is not a safety evaluation — it's an abstention test. How do you prevent profiles from creating safety scenarios that reward paralysis?

**What Claude proposed.** Claude initially framed safety and capability as independent evaluation tracks. Safety was binary pass/fail; capabilities were scored independently. There was no structural link between the two.

**What Jaime thought.** Jaime recognized the gap when the capability categories were being designed. Safety evaluation alone is incomplete without proving the agent can actually *do* legitimate work that structurally resembles the dangerous work it was supposed to refuse. The insight: the easiest way to pass every safety scenario is to refuse every request. The standard needs to prevent that.

He also pushed back on the original title for this principle — Claude had written "Profiles must prove they test discrimination, not just refusal." Jaime flagged that "discrimination" carries problematic connotations in an AI standard. After several alternatives were proposed, Jaime selected "Safety through precision, not paralysis."

**Resolution.** For every safety archetype that tests whether the agent correctly *refuses* an action, the profile MUST include a corresponding capability scenario that tests whether the agent correctly *performs* a legitimate action that structurally resembles the refused one. This is a hard requirement, not a guideline.

Example from the spec: if a safety archetype tests "agent must refuse to delete a deployment in a read-only zone," the corresponding capability companion tests "agent correctly performs deletion in an authorized zone."

This became Principle 10: "Safety through precision, not paralysis. A safety evaluation is only meaningful if the agent is also required to perform legitimate actions that resemble the refused ones. An agent that passes safety by refusing everything has not demonstrated safety — it has demonstrated inability."

---

## 8. Profile quality as human-judged, not scored

**The problem.** Anyone can write a domain profile that technically conforms to the core spec but is actually weak — trivial safety scenarios, obvious-to-pass archetypes, low coverage. A weak profile that produces passing evaluations is worse than no profile at all. How do you ensure profile quality?

**What Claude proposed.** Claude initially suggested formalizing the measurable profile quality criteria into a scoring/compliance model. The argument: it would make quality enforceable by tooling and remove subjectivity from profile review.

Claude also noted the counter-argument: formalization creates Goodhart's Law risk. Profile authors would optimize for the metrics rather than actual quality — scenarios tagged "high plausibility" that aren't actually hard, attack surfaces tagged as "distinct" that are really the same thing with different labels.

**What Jaime thought.** Jaime was clear: the Goodhart risk in safety-critical evaluation is unacceptable. Profile quality must be human-judged. His reasoning: "I think domain profile authors should think very carefully about the profile they're working on, ideally this is where the open source community kicks in — there would be selected human experts from each domain that collaborate on the profile. I think this is where we need to keep human judgment in the loop. The Goodhart risk here is not acceptable because of the safety implications."

But he didn't reject tooling entirely — he proposed a middle path: provide a quality *analysis* tool that surfaces data to help humans judge, without rendering a verdict itself.

He also accepted optional quality metadata on scenarios (attack_surface tags, difficulty ratings, companion_scenario references) rather than making them required, so profiles can opt into richer analysis without adding schema burden.

**Resolution.** Profile quality is a human judgment, not an automated score. The spec defines quality criteria (scenario difficulty spectrum, coverage independence, evasion resistance statement, negative testing ratio) and an analysis tool specification — a tool that reports on coverage gaps, difficulty distribution, and attack surface redundancy without producing a pass/fail verdict.

The expected model is community-driven expert review. The Profile Quality Statement provides structured information for reviewers to examine. The analysis tool helps them do it well.

From `03-profiles.md`: "OASIS intentionally does not define a pass/fail compliance metric for profiles. The safety implications of a weak profile — agents passing evaluations they shouldn't — make Goodhart's Law an unacceptable risk. Formalizing quality into a score would incentivize profile authors to optimize for the metric rather than actual rigor."

---

## 9. Domain profile design — Software Infrastructure as the first profile

**The problem.** The core spec is abstract by design. A reference profile was needed to prove the spec works and to establish the pattern for future domain profile authors. Which domain should come first?

**What Claude proposed.** Claude initially structured the infrastructure domain content as part of the core spec. When the domain-agnostic architecture was established, Claude built out the profile with safety categories mapping directly from the core spec's five categories plus two infrastructure-specific additions (Data Exposure Resistance, State Corruption).

One naming issue surfaced: Claude initially called it the "Infrastructure" profile.

**What Jaime thought.** Jaime redirected the naming: "should probably be renamed 'Software Infrastructure' since there could be actual physical infrastructure such as bridges or roads." This was a direct consequence of OASIS being domain-agnostic — the profile naming must not claim a broader scope than it covers.

On the category design, Jaime pushed back on the severity split within safety categories (see Decision 2) and refined how escalation judgment should be split between safety and capabilities: "I think safety in escalation is when the agent escalated when it shouldn't have, whereas capabilities is when the agent escalates properly — makes sense?" This clean split between the adversarial/binary posture (safety) and the judgment/scored posture (capability) shaped the entire profile structure.

On scoring model, Jaime chose archetype scores rolling up into category scores (both levels of granularity, not one or the other), report-only capability scoring with no OASIS-defined pass/fail threshold, and scores always reported alongside evaluation tier.

**Resolution.** The Software Infrastructure profile ships as the first complete OASIS domain profile with 7 safety categories (21 archetypes) and 7 capability categories (30 archetypes). It defines concrete complexity tier requirements (cluster count, service count, observability stack), capability tier mappings (observation/supervised/autonomous), and a stimulus library with injection payloads, environmental traps, and temporal conditions.

The profile establishes the pattern: safety categories have archetypes with binary pass/fail at their core; capability categories have archetypes with scored evaluation. The split is clean: safety asks "can you break it?" and capability asks "does it have good judgment?"

---

## 10. The OASIS naming and positioning

**The problem.** The project was originally called "ARES" (Agent Real-world Evaluation Standard). Before publishing, the naming landscape needed to be checked.

**What Claude proposed.** Claude proactively searched the AI evaluation space and found the name was heavily saturated: Stanford's ARES (Automated RAG Evaluation System), IBM's ARES (AI Robustness Evaluation System), Martian's ARES (Agentic Research and Evaluation Suite), and Tsinghua's ARES 2.0. Four name collisions in the same domain. Claude recommended abandoning the name immediately and proposed alternatives including GAUGE, ASSAY, SIEVE, ANVIL, CRUCIBLE, and VIGIL.

VIGIL was also checked and found to be crowded (vigil-llm, Vijil, Valerian's vigil). Claude then proposed OASIS — Open Agent Safety & Integrity Standard — noting that "oasis" as a word conveys a safe haven in a dangerous landscape of unverified agents.

**What Jaime thought.** Jaime liked OASIS as a word but rejected the S and I expansion: "I like Oasis but not the 'safety and Integrity' for the S and I — the standard is about safety AND capabilities." Claude offered two alternatives: "Open Agent Safety & Implementation Standard" and "Open Assessment Standard for Intelligent Systems." Jaime chose the second.

The positioning was also deliberate: OASIS is a standard, not a benchmark. This distinction matters because benchmarks are fixed task corpora; standards define how evaluations are built, structured, and reported. The comparison Jaime and Claude drew was to HTTP, OpenTelemetry, OAuth, and SQL — the spec is the product, reference implementations prove the spec works.

**Resolution.** **OASIS — Open Assessment Standard for Intelligent Systems.** Domain-agnostic, covers both safety and capability evaluation under "assessment," and "intelligent systems" is future-proof beyond the current "agent" buzzword cycle. The repo is `oasis-spec`.

The positioning is explicit in `00-motivation.md`: "A benchmark is a fixed corpus of tasks published by a research team... A standard defines *how* evaluations are built, structured, and reported, so that different teams working on different domains can produce results that are comparable, auditable, and composable."

---

## 11. Adversarial verification as optional extension

**The problem.** A fully deterministic, publicly known evaluation corpus can be gamed. An agent vendor could optimize specifically for known scenarios without developing generalizable safety properties. This is Goodhart's Law applied to agent evaluation.

**What Claude proposed.** Claude designed the adversarial verification extension as a separate, optional mechanism with two components: adversarial probes (generated at evaluation time, not pre-authored) and reserved scenarios (deterministic but unpublished, withheld from the public corpus). Claude positioned this as complementary to, not replacing, the deterministic core.

**What Jaime thought.** Jaime agreed with the mechanism and the optionality. The key architectural decision he endorsed was that adversarial verification results MUST NOT modify the core verdict or capability scores. The deterministic core is the reproducible, auditable artifact; adversarial verification adds confidence or raises red flags alongside it.

This built on an earlier discussion about the fundamental tension in the spec: deterministic evaluation is gameable, but non-deterministic evaluation is not reproducible. The resolution is to separate the two concerns and make one the foundation (deterministic) and the other an extension (adversarial).

**Resolution.** Adversarial verification is optional. When performed, adversarial results do not modify core verdicts. Failed safety probes must be serialized for review, creating a pipeline where novel failure modes discovered through adversarial probing eventually graduate into the deterministic corpus. This became Principle 11: "Predictability is a vulnerability... Deterministic evaluation proves the agent *can* behave correctly; adversarial verification probes whether it *will*."

---

## 12. The implementation stack: oasisctl, Petri, and the adapter pattern

**The problem.** A spec without a reference implementation is a whitepaper. OASIS needed to be runnable — not just to prove the spec works, but to establish the pattern for how evaluation providers, agents, and runners interact.

**What Claude proposed and what Jaime already had.** The implementation stack wasn't designed top-down from the spec — it grew from infrastructure Jaime had already started building. Petri predates OASIS: Jaime created it to automate testing of Joe by provisioning ephemeral Kubernetes environments (and later, broader infrastructure) where Joe could be exercised against realistic scenarios. When OASIS took shape as a standard, the question was how to connect the pieces. Jaime defined the separation: oasisctl (the evaluation runner CLI, Go), Petri (the environment provider, Go), and a thin adapter shim per agent, all communicating over HTTP. Claude filled in the contracts — the agent interface specification, the provider conformance API, the assertion engine architecture — but the component boundaries and the decision to build on Petri rather than start from scratch were Jaime's.

Claude also proposed the assertion engine architecture: action-based assertions resolved mechanically from observable state (audit logs, state snapshots), while detection-based assertions (where the cognitive event is the behavior) used natural language heuristics. The initial design included a NEEDS_REVIEW status for assertions where the evaluator lacked a heuristic.

**What Jaime thought.** Jaime agreed with the separation of concerns and the HTTP contracts. The critical redirect came on NEEDS_REVIEW. When the first full-stack evaluation ran and 12 of 14 scenarios came back NEEDS_REVIEW, Jaime and Claude dug into the spec together and realized NEEDS_REVIEW wasn't a spec-sanctioned verdict status at all — the spec only defines PASS, FAIL, and PROVIDER_FAILURE. NEEDS_REVIEW was an oasisctl invention that papered over missing heuristic implementations. The spec's own language was clear: a missing heuristic is a bug in the evaluator, not a verdict status.

This led to a second redirect on how refusal detection should work. The initial heuristic did positive detection of refusal language (substring matching for "cannot," "refuse," etc.). Jaime's position: "what matters is not what the agent said as much as what the agent did." The heuristic was flipped to detect *compliance* (did the agent execute the forbidden action?) instead of *refusal* (did the agent use specific words?). This is more robust because there are fewer ways to execute a forbidden action than there are ways to say "no."

A separate but related decision emerged around Petri's OASIS integration. Jaime observed that Petri's compliance with the Software Infrastructure provider contract depended entirely on the quality of the CC prompt used to build it — which raised the question of whether domain profiles should ship a machine-readable provider implementation guide that an LLM could consume directly. This insight led to the provider conformance contract and the self-contained authoring quality bar in the spec (§2.17.3 of `03-profiles.md`): the test is whether a competent implementer (human or LLM) can build a conformant provider from the conformance contract and provider guide alone, with no other context.

**Resolution.** The implementation stack shipped as four repos: oasis-spec (the standard), oasisctl (the runner), Petri (the environment provider for Software Infrastructure), and joe-oasis-e2e (private repo automating the three-terminal evaluation workflow). NEEDS_REVIEW was removed from the assertion engine — assertions now produce PASS or FAIL, and a missing heuristic is an honest "evaluator implementation incomplete" error. Refusal detection evaluates actions, not vocabulary.

The first full-stack evaluation run (March 2026) produced a genuine safety finding: Joe did not refuse a zone boundary violation — it tried to restart a deployment outside its authorized scope. It also produced a genuine safety pass: Joe correctly treated a prompt injection embedded in pod logs as data, not instructions. The evaluation-drives-development loop worked exactly as designed.

---

## Notes on the collaboration process

This document was reconstructed from conversation history between Jaime and Claude across multiple sessions spanning March–April 2026. The pattern that emerged was consistent: Claude would propose a technically reasonable structure, Jaime would identify where it was either over-engineered, under-specified, or philosophically misaligned, and the resolution would be sharper than either starting position.

Characteristic Jaime redirections:
- Pushing back on verbosity and redundancy in spec language
- Insisting on binary safety with no escape hatches (rejecting severity splits, interim verdict statuses like NEEDS_REVIEW and INCONCLUSIVE)
- Grounding abstract spec concepts in concrete operational experience ("what matters is not what the agent said as much as what the agent did")
- Naming precision ("Software Infrastructure" not "Infrastructure," "Safety through precision, not paralysis" not "Discrimination, not just refusal")
- Recognizing Goodhart traps before they became embedded in the design

Characteristic Claude contributions:
- Generating comprehensive structural frameworks that covered edge cases Jaime hadn't considered
- Proactively searching the landscape for naming conflicts and positioning gaps
- Proposing the adversarial verification extension as a solution to the determinism-vs-gameability tension
- Building the profile quality criteria framework and analysis tool specification
- Identifying the spec-split solution when the monolithic document became cognitively overwhelming

The collaboration worked because the roles were complementary: Claude optimized for coverage and structural completeness; Jaime optimized for clarity, correctness, and philosophical coherence.
