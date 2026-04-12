# Why OASIS Exists

**Version:** 1.0.0-rc1

This document explains why OASIS was created, what gap it fills in the current AI agent evaluation landscape, and how it relates to existing benchmarks and frameworks. It is non-normative — readers looking for the technical specification should start with [Core](01-core.md).

---

## 1. The problem

AI agents are gaining access to production infrastructure, financial systems, databases, industrial control systems, and physical devices. The gap between what agents *can* do and what we can reliably *verify* about their behavior is widening — and it is widening inside systems where mistakes have consequences that don't roll back.

Recent empirical work makes the scale of the gap concrete. Carnegie Mellon and AI2's [OpenAgentSafety](https://arxiv.org/abs/2507.06134) benchmark, applied to five frontier models in realistic multi-turn workplace scenarios, found unsafe behavior rates between 51% and 73% on safety-vulnerable tasks. [SafeAgentBench](https://arxiv.org/abs/2412.13178) reports rejection rates as low as 5–10% for embodied agents handed explicitly hazardous instructions. [AgentHarm](https://arxiv.org/abs/2410.09024) shows that frontier models comply with malicious agent requests at high rates even without jailbreaking. These are not edge cases. They are the baseline.

The evaluation methodology has not kept pace. Most existing frameworks were designed when agents were research artifacts, not production systems. They measure capability with elaborate scoring rubrics and treat safety as one dimension among many — a score on a dashboard, a percentage to optimize, a number that trades off against task completion. This shape of evaluation is fine for measuring model progress on a leaderboard. It is not fine for deciding whether an agent is safe to connect to a Kubernetes cluster, a trading desk, or a robotic arm.

OASIS exists because the systems we are about to hand to agents require a different shape of evaluation — one where safety is a precondition for capability to be measured at all, where evaluation results are reproducible and independently verifiable, and where the methodology is a standard the whole ecosystem can align on rather than another benchmark from another team.

---

## 2. Why a standard, not another benchmark

The distinction matters. A benchmark is a fixed corpus of tasks published by a research team — valuable for comparing models on that specific corpus at a specific moment. A standard defines *how* evaluations are built, structured, and reported, so that different teams working on different domains can produce results that are comparable, auditable, and composable.

The current landscape is rich with benchmarks and thin on standards. OpenAgentSafety, AgentHarm, SafeAgentBench, AgentBench, GAIA, WebArena — each is a fixed task set bound to a specific environment and a specific domain (workplace productivity, embodied robotics, web browsing, general capability). None of them define a shape that a third party could use to evaluate an agent in a domain the benchmark authors never considered — a DNS server, a SCADA controller, a clinical ordering system, a payment rail.

OASIS takes the opposite approach. The core spec is domain-agnostic and defines grammar: what a scenario is, what a safety gate is, what independent verification means, how conformance is reported. Domain knowledge lives in versioned profiles that anyone can author. The first profile (Software Infrastructure) is the reference case, not the entire standard. Finance, robotics, and clinical profiles are planned but not bottlenecks — a team with domain expertise and no prior involvement with OASIS can write a conformant profile and publish it independently.

This is how HTTP, OpenTelemetry, OAuth, and SQL became ubiquitous. The spec is the product. Reference implementations are how you prove the spec works, not how you own the ecosystem.

---

## 3. Why safety must be a gate

OASIS treats safety as a binary gate with an explicit tolerance parameter, not a score. An agent that fails any applicable safety scenario receives no capability score — the evaluation simply fails.

This is a deliberate departure from scored safety evaluation and it exists because scored safety has a structural problem: optimization. A score that can be traded against other scores invites optimization that treats safety violations as acceptable losses for capability gains. In systems with rollback — chatbots, drafting assistants, code suggestions — this is defensible. In systems where the agent can delete the production database, transfer funds, or actuate a motor, the trade-off is not defensible, and the evaluation methodology should reflect that.

The gate model has a second property worth naming: it makes safety failures visible as failures. A 0.92 safety score on a dashboard reads as "mostly fine." A FAIL verdict reads as "do not deploy." Evaluation methodology is a communication channel to the humans making deployment decisions, and the channel should carry signals that match the stakes.

OASIS pairs the gate with a negative testing requirement (principle 10, *safety through precision, not paralysis*). An agent cannot pass OASIS by refusing everything — each safety refusal archetype must have a matching capability scenario that tests whether the agent correctly *performs* the legitimate version of what it was supposed to refuse. Safety without capability is not a pass; it is abstention, and the spec treats it as such.

---

## 4. Relationship to existing frameworks

The table below maps OASIS against the current landscape. The goal is not to dismiss these projects — several are excellent and OASIS borrows ideas from them. The goal is to make it clear what shape of gap OASIS is filling.

| Framework | Type | Scope | Safety model | What OASIS does differently |
|---|---|---|---|---|
| **[OpenAgentSafety](https://arxiv.org/abs/2507.06134)** (CMU / AI2, 2025) | Benchmark | Workplace tools (GitLab, file systems, browsers) | Rule-based + LLM-judge, reported as rates | Standard with pluggable profiles; binary gate with explicit tolerance; deterministic core separated from adversarial verification |
| **[AgentHarm](https://arxiv.org/abs/2410.09024)** (UK AISI / Gray Swan, 2024) | Benchmark | 110 malicious multi-step tasks across 11 harm categories | Harm score + refusal rate | Not a misuse benchmark — OASIS tests agents operating on behalf of legitimate operators in real systems, not adversarial users trying to weaponize them |
| **[SafeAgentBench](https://arxiv.org/abs/2412.13178)** (2024) | Benchmark | Embodied agents in AI2-THOR simulation | Rejection rate on 750 tasks (450 hazardous + 300 safe control) across 10 hazard types | Real external systems, not simulators; domain-extensible via profiles rather than bound to one environment |
| **[AgentBench](https://arxiv.org/abs/2308.03688)** | Benchmark | Multi-environment capability evaluation | Safety as one scored dimension | Safety is a gate, not a dimension; domain-specific rather than general-purpose |
| **[τ-bench / τ²-bench / τ³-bench](https://arxiv.org/abs/2406.12045)** (Sierra, 2024–2026) | Benchmark | Customer service domains (airline, retail, telecom, banking knowledge, voice); multi-turn tool-agent-user interaction; policy adherence; dual-control (τ²) and knowledge/voice (τ³) extensions | Goal-database state verification; pass^k reliability metric across repeated trials | Independent state verification is shared ground — but τ-bench is capability and reliability focused with fixed customer service domains, no safety gate, and no profile mechanism for new domains. OASIS could host a τ-bench-style customer service profile; the inverse is not true. |
| **[GAIA](https://arxiv.org/abs/2311.12983)** | Benchmark | General assistant capability | None — capability only | Adds safety-first architecture and domain scoping |
| **[WebArena / VisualWebArena](https://arxiv.org/abs/2307.13854)** | Benchmark | Web browsing tasks | None — task completion only | Scope extends beyond browsers; safety is primary |
| **[ToolEmu](https://arxiv.org/abs/2309.15817)** | Framework | LLM-emulated tool sandboxes | LLM grades accidental safety violations | Real external systems with independent state verification, not LLM-emulated tools or LLM-graded verdicts |
| **[IBM ARES](https://github.com/IBM/ares)** (AI Robustness Evaluation System) | Red-teaming framework | Attack orchestration against AI endpoints (models, agentic apps, REST APIs); plugin-based, with documentation citing tools such as Garak and Crescendo as example integration targets; mapped to OWASP LLM categories (mapping is explicitly work-in-progress upstream, with LLM03 unsupported, LLM08 deferred to LLM02, and several supported categories lacking example notebooks) | Intent field routes and maps attack runs to OWASP LLM categories; individual attack success or failure is determined by keyword-based and LLM-as-judge evaluators | Standard rather than an attack framework; deterministic scenario verdicts and independent state verification, with adversarial probing as an explicitly optional extension that does not modify the core verdict |
| **[MLCommons AIRR](https://mlcommons.org/working-groups/ai-risk-reliability/)** (AI Risk & Reliability working group) — components: [AILuminate v1.0](https://arxiv.org/abs/2503.05731) (2025), [Jailbreak v0.5](https://mlcommons.org/ailuminate/jailbreak/) benchmark, [Agentic Workstream](https://mlcommons.org/ailuminate/safety/), [AIL GAP](https://mlcommons.org/2026/02/ailuminate-global-assurance/) assurance program (2026) | Standards body with benchmarks, assurance program, and ISO SC 42 liaison; 501(c)(6) nonprofit governance with weekly open meetings and eight collaborative workstreams | AILuminate v1.0: single-turn chat model safety across 12 hazard categories; Jailbreak v0.5: adversarial text-to-text and text+image-to-text evaluation layered on AILuminate v1.0 with a Resilience Gap metric; Agentic Workstream: agentic reliability evaluation standard (design principles, benchmark factory — in progress); AIL GAP: assurance and labeling program backed by KPMG, Google, Microsoft, and Qualcomm, bridging ISO 42001 to empirical model performance | Hazard-category grades on a five-tier scale (Poor / Fair / Good / Very Good / Excellent) anchored to a reference model via a relative ratio; Jailbreak v0.5 adds resilience metrics | OASIS is differentiated on evaluation methodology, not on existence or governance: interface-type abstraction enabling tool-agnostic scenarios rather than prompt-level hazard categories; action-first safety assertion semantics (absence of forbidden actions, not presence of refusal phrases); independent verification mandate where evaluators check system state directly rather than trusting agent self-reports; the profile system for domain extensibility; the negative testing ratio pairing safety refusal archetypes with capability companions; and Goodhart mitigation via human-judged profile quality. AIRR's Agentic Workstream is parallel work toward agentic evaluation — OASIS and AIRR are converging peers in this space, not substitutes. Complementary on chat-model safety (AILuminate's domain); differentiated on agent-on-system evaluation methodology |
| **[MLflow 3 GenAI Eval / Mosaic AI Agent Evaluation](https://docs.databricks.com/aws/en/mlflow3/genai/)** (Databricks) | Evaluator framework | Trace-based scorers, built-in and custom LLM judges, Agent-as-a-Judge (MLflow 3.4.0 `make_judge` with trace template variable) | Scorers attach feedback to traces; boolean, numeric, or categorical per scorer | OASIS defines *what* to evaluate and the verdict shape; tooling like this can be used to implement conformant runners |

Two patterns are worth calling out.

**Benchmarks vs. standards.** Almost every entry above is a benchmark — a fixed task corpus. MLCommons AIRR is the closest structural comparison: a 501(c)(6) nonprofit standards body with formal governance, eight collaborative workstreams, ISO SC 42 liaison, and — as of February 2026 — the AILuminate Global Assurance Program (AIL GAP), an assurance and labeling layer backed by major vendors. AIRR has also launched an Agentic Workstream developing an agentic reliability evaluation standard, so OASIS is not the only effort moving toward agentic evaluation. The differentiation is methodological: OASIS evaluates agents through interface-type abstraction, action-first safety assertion semantics, independent system-state verification, and a profile mechanism for domain extensibility. AIRR's current evaluation artifacts (AILuminate v1.0, Jailbreak v0.5) operate at the prompt-and-response level with hazard-category grading; OASIS operates at the action-and-state level with binary safety gates. The two are complementary on chat-model safety and converging peers on agentic evaluation.

**LLM-as-judge vs. independent verification.** Many benchmarks rely on LLM-as-judge for final verdicts. OASIS permits LLM-as-judge for adversarial verification (which is optional and non-deterministic by design) but requires independent verification — direct inspection of external system state — for the deterministic core safety verdict. The agent's self-reporting is never used as evidence. τ-bench is a notable precedent in the same direction, evaluating against goal database state rather than conversation quality; OASIS generalizes this discipline across domains and elevates it from a design choice to a conformance requirement. This is not a criticism of LLM-as-judge; it is a recognition that the core safety verdict needs to be reproducible by a third party using the same scenario corpus, and LLM-as-judge alone is not reproducible enough to carry that weight.

---

## 5. What OASIS is not

To keep the scope clear:

- **OASIS is not a benchmark.** It does not publish a fixed task corpus as its primary artifact. The Software Infrastructure profile ships with a scenario library, but that library is one profile's implementation, not the standard itself.

- **OASIS is not a certification body.** The provider conformance spec ([08-provider-conformance.md](08-provider-conformance.md)) defines what makes an evaluation provider conformant. Certification, accreditation, and governance are deferred until there is operational experience with multiple independent providers.

- **OASIS is not a chat model safety evaluation.** Efforts like MLCommons AIRR (AILuminate v1.0, Jailbreak v0.5), HELM safety, and red-teaming benchmarks for chat models address a different problem: the model's textual outputs. AIRR's Agentic Workstream is extending into agent evaluation, and OASIS views that as parallel and complementary work. OASIS's specific contribution is evaluation of what the agent *does* when it has hands — when it can call APIs, modify state, and affect systems that exist independently of the conversation — using interface-type scenarios, action-first assertions, and independent state verification.

- **OASIS is not opposed to existing benchmarks.** Benchmarks prove problems exist (the OpenAgentSafety numbers are cited above precisely because they are useful). Standards give the ecosystem a common shape for solving those problems. Both are needed.

- **OASIS is not a replacement for human judgment.** Profile quality is assessed by domain experts, not scored automatically. The spec is explicit about this ([03-profiles.md, §2.9](03-profiles.md)) because safety-critical evaluation is a Goodhart trap waiting to happen.

---

## 6. Design lineage

OASIS draws on four traditions.

**Software testing discipline.** The binary gate, isolation by default, deterministic assertions, and negative testing requirements come from decades of practice in safety-critical software (aviation, medical devices, nuclear control). The field already knows how to separate "it might work" from "it will work" — OASIS imports that discipline rather than reinventing it.

**Standards ecosystems.** The profile mechanism, the conformance spec, the separation of normative and non-normative content, and the CC BY 4.0 license borrow from how HTTP, OAuth, OpenTelemetry, and SQL grew. The goal is to make it easy for anyone to author a conformant profile or implementation without permission.

**Empirical AI safety research.** The adversarial verification extension ([07-adversarial-verification.md](07-adversarial-verification.md)) draws on red-teaming practice and on the recognition — from OpenAgentSafety, AgentHarm, and others — that deterministic corpora alone can be gamed. The open question of probe generator trust is a direct descendant of that research tradition.

**Operational production experience.** The blast radius, declared mode verification, authority escalation, and state corruption categories come from incident retrospectives in real infrastructure environments. They are not theoretical risk categories. They are the failures platform engineers have watched happen.

---

## 7. What success looks like

OASIS succeeds if, within two to three years:

- A handful of domain profiles exist beyond Software Infrastructure, authored by teams with no prior OASIS involvement.
- At least one independent evaluation provider publishes conformance reports that a third party can reproduce.
- Agent builders cite OASIS verdicts in the same way software vendors cite SOC 2 or ISO 27001 — as baseline evidence of due diligence, not as a marketing score.
- Regulators, auditors, and procurement teams can point to a document and say "evaluate against this" without having to specify the methodology themselves.

Success is not adoption of a specific benchmark. It is the emergence of a common vocabulary for "is this agent safe to deploy in this context?" — a vocabulary the field currently lacks.

---

## References

1. Vijayvargiya, S., Soni, A. B., Zhou, X., Wang, Z. Z., Dziri, N., Neubig, G., & Sap, M. (2025). "OpenAgentSafety: A Comprehensive Framework for Evaluating Real-World AI Agent Safety." *arXiv preprint arXiv:2507.06134*. https://arxiv.org/abs/2507.06134

2. Yin, S. et al. (2024). "SafeAgentBench: A Benchmark for Safe Task Planning of Embodied LLM Agents." *arXiv preprint arXiv:2412.13178*. https://arxiv.org/abs/2412.13178

3. Andriushchenko, M., Souly, A. et al. (2024). "AgentHarm: A Benchmark for Measuring Harmfulness of LLM Agents." *ICLR 2025*. *arXiv preprint arXiv:2410.09024*. https://arxiv.org/abs/2410.09024

4. Vidgen, B., Agrawal, A. et al. (2025). "AILuminate v1.0: A Comprehensive Industry-Standard Benchmark for Evaluating LLM Safety." *arXiv preprint arXiv:2503.05731v2*. https://arxiv.org/abs/2503.05731

5. MLCommons AI Risk & Reliability (AIRR) Working Group. https://mlcommons.org/working-groups/ai-risk-reliability/

6. MLCommons AILuminate Safety. https://mlcommons.org/ailuminate/safety/

7. MLCommons AILuminate Jailbreak. https://mlcommons.org/ailuminate/jailbreak/

8. MLCommons (2026). "AILuminate Global Assurance Program (AIL GAP)." https://mlcommons.org/2026/02/ailuminate-global-assurance/
