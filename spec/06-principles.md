# OASIS Design Principles & Context

**Version:** 1.0.0-rc1

---

## 1. Design principles

1. **Safety is a gate, not a score.** One safety failure vetoes the entire evaluation. This is non-negotiable and cannot be overridden by domain profiles.

2. **Independent verification.** Evaluation outcomes are verified by direct inspection of the target system. Agent self-reporting is never used as evidence for verdicts. Trust is established through observation, not testimony.

3. **The spec is domain-agnostic.** Domain knowledge lives in profiles, not in the core spec. The core spec defines grammar; profiles provide vocabulary. Core categories are a floor, not a ceiling.

4. **Deterministic verdicts from deterministic evaluators.** Safety assertions are binary. Capability scoring uses defined rubrics, not vibes. Evaluator implementations are pure functions of their inputs: the same recorded evidence produces the same verdict on every run. LLM-as-judge and other non-deterministic heuristics belong in the optional Adversarial Verification Extension, not in the core evaluation.

5. **The agent is a black box.** OASIS evaluates behavior, not architecture. The spec does not care how the agent works internally — only what it does.

6. **Isolation by default.** Every scenario runs in a clean environment. No shared state between scenarios.

7. **Scores require context.** Capability scores are meaningless without knowing the complexity tier at which they were measured. The standard mandates that scores are always reported with their tier.

8. **Open standard, reference implementation.** The spec is the product. The runner, environment providers, and tools are one conformant implementation — not the only one.

9. **Explicit over implicit.** When safety tolerances are applied, they must be visible. When coverage is incomplete, it must be labeled. The standard forces transparency in evaluation methodology.

10. **Safety through precision, not paralysis.** A safety evaluation is only meaningful if the agent is also required to perform legitimate actions that resemble the refused ones. An agent that passes safety by refusing everything has not demonstrated safety — it has demonstrated inability.

11. **Predictability is a vulnerability.** A fully deterministic, publicly known evaluation corpus can be gamed. The standard provides extension points for adversarial verification — non-deterministic probes and reserved scenarios — to test whether safety properties generalize beyond the known test surface. Deterministic evaluation proves the agent *can* behave correctly; adversarial verification probes whether it *will*.

---

## 2. Relationship to existing frameworks

See [Motivation, §4](00-motivation.md) for an expanded comparison of OASIS against OpenAgentSafety, AgentHarm, SafeAgentBench, AgentBench, τ-bench, GAIA, WebArena, ToolEmu, IBM ARES, AILuminate, and other existing frameworks. The short version: OASIS is a standard, not a benchmark, and is the only entry in the landscape positioned explicitly as domain-extensible.

---

## 3. Open questions

### Definitions & scope

- **External system edge cases.** Does an agent writing to a shared file system count? (Probably yes.) Does an agent calling another AI model count? (Probably no — no persistent state independent of the call.) These need explicit rulings.

- **Multi-agent scenarios.** The current spec assumes a single agent under evaluation. How should OASIS handle agent-to-agent interactions?

### Scenario lifecycle

- **Scenario versioning and deprecation.** How are scenarios updated when external systems change? What happens to historical verdicts when a scenario is modified?

- **Partial environment fidelity.** When a mock environment doesn't perfectly simulate the real system, how should the verdict communicate this limitation?

### Governance

- **Certification vs. self-assessment.** Should OASIS define a certification process, or is it purely a self-assessment standard? [Provider Conformance](08-provider-conformance.md) defines the minimum requirements; certification and provider tiering are deferred to future versions pending ecosystem experience.

- **Provider tiering.** Should OASIS define maturity tiers for evaluation providers (e.g., self-assessed, audited, certified)? Deferred pending operational experience with multiple providers — see [Provider Conformance, §5.2](08-provider-conformance.md).

- **Core dimension evolution.** How are new core capability dimensions proposed, evaluated, and accepted? What governance model applies?

- **Profile quality enforcement.** Profile quality relies on human expert review, not automated scoring (to avoid Goodhart's Law in safety-critical evaluation). The quality analysis tool surfaces data to support reviewers. Open question: what community governance structure best supports expert profile review? Should there be a registry of reviewed profiles?

### Scoring & reporting

- **Continuous evaluation.** The current model is point-in-time. Should OASIS define a continuous monitoring mode for production agents?

- **Cross-domain reporting.** Core dimension scores enable cross-domain comparability, but how meaningful is "Reliability: 0.85" for an infrastructure agent vs. a finance agent?

- **Safety tolerance governance.** Should domain profiles define recommended tolerance ranges, or is tolerance purely organizational?

- **Negative testing completeness.** The negative testing ratio requires a companion capability scenario for each safety refusal archetype. Some archetypes may not have clean legitimate counterparts. How strictly should this be enforced?

### Adversarial robustness

- **Deterministic evaluation gaming.** A fully deterministic, publicly available scenario corpus is inherently gameable — an agent could be optimized to pass known scenarios without developing generalizable safety properties. The [Adversarial Verification Extension](07-adversarial-verification.md) addresses this through non-deterministic probes and reserved scenarios. Open question: should adversarial verification be required for certification-grade evaluations, or remain optional for all use cases?

- **Probe generator trust.** When an LLM generates adversarial probes, the probe quality depends on the generator's own capabilities and alignment. A weak generator produces weak probes. Should OASIS define minimum generator requirements, or is generator quality an evaluator responsibility?

- **Composition probe attribution.** Cross-archetype probes test emergent failure modes but make verdict attribution harder. The current rule (primary archetype determines attribution) is simple but may undercount safety failures discovered as secondary effects. Is this acceptable?
