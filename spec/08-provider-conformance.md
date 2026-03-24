# OASIS Provider Conformance

**Version:** 0.3.0-draft

This document defines what it means for an evaluation provider to be OASIS-conformant. For what a conformant evaluation looks like, see [Reporting & Conformance](05-reporting.md). For the execution model, see [Execution](04-execution.md).

---

## 1. Purpose

The OASIS core spec defines conformance for evaluations (05-reporting.md §3) and for domain profiles (03-profiles.md §§2–3). This document closes the third leg: conformance for the entity that executes the evaluation.

An evaluation provider is any system, organization, or individual that executes OASIS evaluations and produces verdicts. The provider is distinct from the domain profile (which defines what to test) and from the agent under evaluation (which is being tested).

Without provider conformance criteria, two providers could execute the same profile against the same agent and produce different verdicts — not because the agent behaved differently, but because the providers interpreted the evaluation model differently.

---

## 2. Definitions

### 2.1 Evaluation provider

The system and/or organization responsible for:

- Provisioning evaluation environments
- Executing scenarios against the agent under test
- Capturing observability artifacts
- Performing independent verification
- Computing verdicts
- Producing the evaluation report

A provider may be a software system (an evaluation runner), an organization operating that system, or both. The conformance requirements apply regardless of form.

### 2.2 Provider conformance claim

A statement by a provider that it meets the requirements in this document for a specific domain profile and complexity tier. A conformance claim is scoped — a provider conformant for the Software Infrastructure profile at Tier 1 is not implicitly conformant for the same profile at Tier 2.

---

## 3. Conformance requirements

A conformant evaluation provider MUST satisfy all of the following.

### 3.1 Scenario execution

The provider MUST execute scenarios according to the evaluation model defined in the core spec:

- **Phase ordering.** Safety scenarios execute before capability scenarios. If the safety verdict is FAIL, capability scenarios are not executed ([Core, §2.1](01-core.md)).
- **Isolation.** Each scenario runs in an isolated environment. No shared state between scenarios ([Execution, §2.3](04-execution.md)).
- **Precondition establishment.** The provider provisions the environment to match each scenario's declared preconditions before applying stimuli. If a precondition cannot be established, the scenario is marked as skipped with justification — it is not silently omitted.
- **Stimulus application.** All declared stimuli are applied as specified. For `environmental_state` stimuli, the provider injects the described state. For `operator_prompt` stimuli, the provider delivers the prompt to the agent. For `tool_output_injection`, the provider ensures the agent receives the manipulated output. For `temporal_condition`, the provider triggers the condition at the specified timing.
- **Agent interface.** The provider interacts with the agent through the agent interface contract ([Execution, §1](04-execution.md)). The provider does not inspect agent internals.

### 3.2 Independent verification

The provider MUST independently verify all evaluation outcomes through direct inspection of the target system ([Core, §3.4](01-core.md)). Specifically:

- **State verification.** When a scenario includes `verification.state_assertions`, the provider queries system state directly (not through the agent) and evaluates each assertion.
- **Audit log verification.** When a scenario includes `verification.api_audit`, the provider inspects system-level audit logs (not agent-reported logs) for the specified operations.
- **Negative verification.** When a scenario includes `verification.negative_verification`, the provider confirms through system audit logs that the specified actions did not occur.
- **State diff verification.** When a scenario includes `verification.state_diff`, the provider captures before and after state snapshots and computes the diff.
- **No agent self-reporting as evidence.** The provider never uses the agent's claimed actions, reasoning, or outcomes as evidence for pass/fail or scoring verdicts. Agent self-reporting may be evaluated as a capability (auditability) but is never used as verification evidence.

### 3.3 Behavior evaluation

The provider MUST evaluate named behaviors in assertions using the verification methods defined in the domain profile's behavior definitions ([Profiles, §2.3](03-profiles.md)).

- For each `must` assertion with a `behavior` field, the provider applies the behavior's defined verification method to determine whether the behavior was exhibited.
- For each `must_not` assertion with a `behavior` field, the provider applies the behavior's defined verification method to determine whether the behavior was exhibited, and fails the assertion if it was.
- The provider MUST NOT invent its own interpretation of a named behavior. The profile's behavior definition is authoritative.

### 3.4 Observability capture

The provider MUST capture the observability artifacts declared in each scenario's `observability_requirements` field. At minimum, every scenario requires ([Scenarios, §1.8](02-scenarios.md)):

- Agent reasoning trace (if the agent provides one)
- All tool calls with parameters and responses
- System-level audit log for the evaluation window

These artifacts serve two purposes: they are inputs to independent verification, and they are evidence supporting the verdict in the evaluation report.

If a required observability artifact cannot be captured (e.g., the agent does not produce a reasoning trace), the provider MUST note this in the scenario detail of the evaluation report. A missing artifact does not automatically fail the scenario — it reduces the evidence available for verification and scoring.

### 3.5 Scoring computation

The provider MUST compute scores according to the models defined in the core spec and domain profile:

- **Safety scoring.** Binary pass/fail per scenario, with `safety_tolerance` applied. Safety verdict is the conjunction of all safety scenario results ([Core, §3.3](01-core.md)).
- **Capability scoring.** Per the domain profile's declared aggregation methods, dimension mappings, and weights.
- **Tier labeling.** All capability scores are reported with the complexity tier at which they were measured.

### 3.6 Environment fidelity

The provider MUST provision environments that meet the domain profile's environment specification ([Profiles, §2.9](03-profiles.md)) and complexity tier requirements ([Profiles, §2.7](03-profiles.md)).

- The provider MUST verify that the environment meets the claimed tier's requirements before beginning evaluation.
- The provider MUST include tier evidence in the evaluation report.
- If the environment does not meet the claimed tier's requirements, the evaluation is labeled **incomplete**.

### 3.7 Report production

The provider MUST produce an evaluation report conforming to [Reporting, §2](05-reporting.md). The report is the primary artifact of the evaluation and the basis for any conformance claim.

---

## 4. Provider conformance claim

A provider conformance claim includes:

- Provider name and version (for software systems) or organization name
- Domain profile name and version
- Complexity tier supported
- Date of conformance claim
- Attestation that all requirements in section 3 are met

A provider MAY claim conformance for multiple domain profiles and multiple tiers. Each combination is a separate claim.

---

## 5. Non-requirements

The following are explicitly NOT required for provider conformance. They may become requirements in future spec versions as the ecosystem matures.

### 5.1 Provider certification

OASIS v0.3 does not define a certification process for providers. Conformance is self-assessed. The evaluation report provides the evidence trail for external audit.

### 5.2 Provider tiering

OASIS v0.3 does not define provider tiers (e.g., minimal, certified, accredited). All conformant providers meet the same requirements. Provider tiering is a natural evolution once the ecosystem has multiple implementations and operational experience to inform meaningful tier boundaries.

### 5.3 Cross-provider reproducibility guarantee

OASIS v0.3 does not guarantee that two conformant providers will produce identical verdicts for the same agent. The conformance requirements ensure both providers apply the same evaluation model (same scenarios, same verification methods, same scoring), which maximizes comparability. However, differences in environment provisioning, stimulus timing, and agent non-determinism may produce different results. The evaluation report captures enough context to identify the source of any divergence.

---

## 6. Relationship to other spec documents

| Document | Defines conformance for... |
|----------|---------------------------|
| [Profiles](03-profiles.md) | Domain profiles — what to evaluate |
| [Reporting](05-reporting.md) | Evaluations — the output artifact |
| This document | Providers — who runs the evaluation |

Together, these three conformance models cover the full evaluation chain: the profile defines the tests, the provider executes them, and the report captures the results.
