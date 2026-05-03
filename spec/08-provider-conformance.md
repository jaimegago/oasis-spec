# OASIS Provider Conformance

**Version:** 1.0.0-rc1.5

This document defines what it means for an evaluation provider to be OASIS-conformant. For what a conformant evaluation looks like, see [Reporting & Conformance](05-reporting.md). For the execution model, see [Execution](04-execution.md). For the canonical verdict status enumeration, see [Core, §3.6](01-core.md).

---

## 1. Purpose

The OASIS core spec defines conformance for evaluations (05-reporting.md §3) and for domain profiles (03-profiles.md §§2–3). This document closes the third leg: conformance for the entity that executes the evaluation.

An evaluation provider is any system, organization, or individual that executes OASIS evaluations and produces verdicts. The provider is distinct from the domain profile (which defines what to test) and from the agent under evaluation (which is being tested).

Without provider conformance criteria, two providers could execute the same profile against the same agent and produce different verdicts — not because the agent behaved differently, but because the providers interpreted the evaluation model differently.

### 1.1 Mechanism here, content in profiles

This document defines the **mechanism** of provider conformance: how providers declare what they support, how the evaluation runner checks them, what happens when a provider does not satisfy a profile's requirements, and what happens when a conformant provider fails at runtime. The mechanism is intentionally abstract — it does not enumerate specific capabilities like "supports audit log capture" or "supports Kubernetes provisioning."

The **content** of provider conformance — the specific requirements a provider must satisfy to be considered conformant for a particular profile — lives in each domain profile. Different profiles test different things and need different capabilities from their providers; the spec avoids prescribing a universal provider concept and instead lets profiles declare their own requirements.

This separation has three consequences worth naming explicitly:

- The core spec does not enumerate observation types, environment types, capability flags, or any other provider concept. All such concepts are profile-defined.
- A conformance claim is always scoped to a specific domain profile. "I am a conformant OASIS provider" is not a meaningful statement; "I am a conformant Software Infrastructure provider" is.
- When a new profile is added to the OASIS ecosystem, it brings its own conformance contract. Providers that wish to support the new profile implement that contract; existing providers are unaffected.

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

A statement by a provider that it meets the requirements in this document for a specific domain profile. A conformance claim is scoped to a profile — a provider conformant for the Software Infrastructure profile is not implicitly conformant for any other profile, even one in an adjacent domain.

A conformance claim is also implicitly scoped to a complexity tier (see [Core, §5](01-core.md)): a provider may declare support for some tiers and not others, and the profile's conformance requirements MAY differentiate by tier. Whether to differentiate is a profile-defined decision.

---

## 3. Conformance requirements

A conformant evaluation provider MUST satisfy all of the following.

### 3.1 Scenario execution

The provider MUST execute scenarios according to the evaluation model defined in the core spec:

- **Phase ordering.** Safety scenarios execute before capability scenarios. Phase 1 runs every applicable safety scenario before aggregating; the provider does not stop at the first FAIL within Phase 1 ([Core, §2.1](01-core.md)). If the aggregated Phase 1 verdict is anything other than PASS, capability scenarios are not executed.
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
- **Value containment verification.** When a scenario includes `verification.value_containment` ([Scenarios, §1.6](02-scenarios.md)), the provider MUST resolve each declared `value_ref` against the provisioned environment state during scenario setup and supply the resolved literal value to the evaluator for substring matching. The provider MUST also capture the agent's output for every channel listed in `output_channels` across the scenario window and make each captured channel available to the evaluator as an input to the containment match. This is a distinct evidence source from system state and audit logs; a profile that uses value containment declares the required channel identifiers in its `provider-conformance-requirements.yaml` and the preflight check (§3.8) verifies the provider can capture them. The plaintext handling of registered values is scoped to the test environment per [Core, §3.5.5](01-core.md#355-value-containment-assertions); the provider MUST NOT persist registered values beyond the scenario window.
- **No agent self-reporting as evidence.** The provider never uses the agent's claimed actions, reasoning, or outcomes as evidence for pass/fail or scoring verdicts. Agent self-reporting may be evaluated as a capability (auditability) but is never used as verification evidence.
- **Evidence source declaration.** Every observation response the provider returns MUST include an `evidence_source` block per [Reporting, §1.1](05-reporting.md). The provider MUST set `evidence_source.status` to `available` only when the evidence is real and complete; runtime faults that prevent evidence collection MUST be reported as `unreachable`, which the evaluation runner treats as a runtime PROVIDER_FAILURE.

### 3.3 Behavior evaluation

The provider MUST evaluate named behaviors in assertions using the verification methods defined in the domain profile's behavior definitions ([Profiles, §2.3](03-profiles.md)).

- For each `must` assertion with a `behavior` field, the provider applies the behavior's defined verification method to determine whether the behavior was exhibited.
- For each `must_not` assertion with a `behavior` field, the provider applies the behavior's defined verification method to determine whether the behavior was exhibited, and fails the assertion if it was.
- The provider MUST NOT invent its own interpretation of a named behavior. The profile's behavior definition is authoritative.
- The provider MUST evaluate every applicable assertion to a deterministic verdict ([Core, §3.5.3](01-core.md)) and MUST be deterministic with respect to its inputs ([Core, §3.5.4](01-core.md)). A missing heuristic for a behavior is a bug in the provider, not a verdict; the provider implementation must be fixed. Producing NEEDS_REVIEW or any other "I don't know" verdict for safety assertions is non-conformant. Using LLM-as-judge or any non-deterministic verification heuristic for core assertions is non-conformant.

### 3.4 Observability capture

The provider MUST capture the observability artifacts declared in each scenario's `observability_requirements` field. At minimum, every scenario requires ([Scenarios, §1.8](02-scenarios.md)):

- Agent reasoning trace (if the agent provides one)
- All tool calls with parameters and responses
- System-level audit log for the evaluation window

These artifacts serve two purposes: they are inputs to independent verification, and they are evidence supporting the verdict in the evaluation report.

If a required observability artifact cannot be captured for runtime reasons (the audit log file is unreachable, the kube API timed out), the provider MUST report this via `evidence_source.status: unreachable` on the affected observation. This triggers a runtime PROVIDER_FAILURE for the affected scenario per [Core, §3.7](01-core.md). A provider that simply omits a required artifact, or returns an empty payload without flagging the source as unreachable, is non-conformant — the spec explicitly forbids the Goodhart-style failure mode in which empty audit data is treated as positive safety evidence.

If a profile's required observation types are not configured on the provider at all (as opposed to being temporarily unreachable), the provider fails the preflight conformance check (§3.8) and the evaluation does not start.

### 3.5 Scoring computation

The provider MUST compute scores according to the models defined in the core spec and domain profile:

- **Safety scoring.** Per-scenario verdict drawn from the canonical enumeration in [Core, §3.6](01-core.md): PASS, FAIL, or PROVIDER_FAILURE. Phase 1 verdict is the conjunction of all safety scenario results, with FAIL winning over PROVIDER_FAILURE ([Core, §3.3](01-core.md)).
- **Capability scoring.** Per the domain profile's declared aggregation methods, dimension mappings, and weights.
- **Tier labeling.** All capability scores are reported with the complexity tier at which they were measured.

### 3.6 Environment fidelity

The provider MUST provision environments that meet the domain profile's environment specification ([Profiles, §2.9](03-profiles.md)) and complexity tier requirements ([Profiles, §2.7](03-profiles.md)).

- The provider MUST verify that the environment meets the claimed tier's requirements before beginning evaluation.
- The provider MUST include tier evidence in the evaluation report.
- If the environment does not meet the claimed tier's requirements, the evaluation is labeled **incomplete**.
- The provider MUST support all state injection, verification, and stimulus operations enumerated in the domain profile's provider implementation guide ([Profiles, §2.13](03-profiles.md)). A provider that does not support an operation required by a scenario cannot execute that scenario; the preflight conformance check (§3.8) MUST detect such gaps and abort the run before scenarios begin.

### 3.7 Report production

The provider MUST produce an evaluation report conforming to [Reporting, §2](05-reporting.md). The report is the primary artifact of the evaluation and the basis for any conformance claim.

### 3.8 Preflight conformance handshake

The provider MUST expose a conformance endpoint that the evaluation runner queries before any scenarios are executed. The handshake exists so configuration mismatches are caught immediately, with a clear error, instead of surfacing as opaque failures partway through a run.

#### 3.8.1 Endpoint

The endpoint is HTTP `GET` at the path `/v1/conformance` on the provider's API server. The endpoint accepts a single required query parameter:

- `profile` — the identifier of the domain profile the runner intends to evaluate against (e.g., `oasis-profile-software-infrastructure`).

The provider responds with a structured document describing its capabilities relative to the requested profile. A provider MAY support multiple profiles; the runner asks for one profile per query.

#### 3.8.2 Response shape

The response is a JSON object with the following structure:

```json
{
  "provider": "string",
  "provider_version": "semver string",
  "oasis_core_spec_versions": ["semver string", ...],
  "profile": "profile identifier",
  "profile_version": "semver string",
  "supported": true | false,
  "requirements": {
    "<profile-defined-key>": <profile-defined-value>,
    ...
  },
  "unmet_requirements": [
    {
      "requirement": "string",
      "reason": "string"
    },
    ...
  ]
}
```

Field semantics:

- `provider` — the provider's name (e.g., `petri`).
- `provider_version` — the provider's own version, in semver.
- `oasis_core_spec_versions` — the list of OASIS core spec versions this provider implements. The runner uses this to verify spec compatibility before checking profile requirements.
- `profile` — echoes the profile identifier from the query, confirming which profile the response describes.
- `profile_version` — the version of that profile the provider was built against. Used by the runner to verify that the provider's profile reference matches the profile being evaluated.
- `supported` — a single boolean summarizing whether the provider claims conformance to the requested profile. `true` means every requirement the profile declares is satisfied; `false` means at least one requirement is not satisfied. The runner uses this as a fast-path check.
- `requirements` — a map whose keys and values are defined by the profile, not by the core spec. The provider populates this map with the values it can supply for each requirement the profile declares. The shape is opaque to the core spec; the runner validates it against the profile's conformance schema.
- `unmet_requirements` — when `supported` is `false`, a list of the specific requirements that are not satisfied, each with a human-readable reason. When `supported` is `true`, this list is empty or omitted.

#### 3.8.3 Runner behavior

The evaluation runner MUST perform the preflight check as the first step of an evaluation run, before any provision, observe, or scenario execution traffic. Specifically, the runner:

1. Issues `GET /v1/conformance?profile=<profile_id>` to the provider.
2. Verifies that the response's `oasis_core_spec_versions` includes a version compatible with the profile's declared core spec dependency.
3. Verifies that the response's `profile` and `profile_version` match the profile being evaluated.
4. Validates the `requirements` map against the profile's conformance schema.
5. Checks the `supported` flag and the `unmet_requirements` list.
6. If any of the above checks fails, aborts the run with a precise error naming the specific gap. The error message MUST be actionable: "provider does not satisfy SI requirement `audit_policy_installation`: audit policy file not present at expected path" is conformant; "provider not conformant" is not.

If the conformance endpoint itself is unreachable or returns a non-200 status, the runner MUST treat this as a preflight failure and abort with an error describing the network or HTTP problem. A provider that does not implement `/v1/conformance` is not conformant under this spec.

#### 3.8.4 Distinction from runtime PROVIDER_FAILURE

The preflight check catches **configuration gaps**: the provider was not set up to do what the profile requires. The operator's response is to fix the provider configuration and rerun. No verdict file is produced, because no scenarios ran.

A runtime PROVIDER_FAILURE catches **runtime faults**: the provider was conformant at startup but something failed during execution. The operator's response is to investigate the transient cause and rerun. A verdict file IS produced, with `safety: PROVIDER_FAILURE` (or `safety: FAIL` if any scenario FAILed before the abort), and `metadata.aborted: true`.

The two checks exist for different reasons and have different operator responses. They are not interchangeable, and a provider implementation MUST NOT collapse them into a single concept.

---

## 4. Provider conformance claim

A provider conformance claim includes:

- Provider name and version (for software systems) or organization name
- Domain profile name and version (the conformance claim is always scoped to a specific profile per §2.2)
- Date of conformance claim
- Attestation that all requirements in section 3 are met, including the preflight conformance handshake in §3.8
- The complexity tier(s) the provider supports for the claimed profile (a profile MAY differentiate conformance requirements by tier; consult the profile's conformance contract)

A provider MAY claim conformance for multiple domain profiles. Each profile is a separate claim; conformance for one profile does not imply conformance for any other.

---

## 5. Profile reference management

A provider implementation MUST reference a specific version of the domain profile it claims conformance to. The profile contains normative artifacts — scenario definitions, behavior definitions, the provider implementation guide, the conformance contract — that the provider depends on. Copying these artifacts into the provider's codebase creates drift risk: the profile evolves, the copy does not, and the provider silently falls out of conformance.

### 5.1 Recommended approach

Provider implementations SHOULD include the domain profile's repository as a version-pinned dependency — for example, a git submodule pointing at a specific tagged release or commit. This ensures:

- The provider always references a specific, immutable version of the profile
- Updates are deliberate: the implementer bumps the pin, verifies compatibility, and commits the change
- The pinned version is visible in the provider's version control history, providing an auditable record of which profile version the provider was built against
- Automated tooling (CI, evaluation runners) can verify that the provider's pinned profile version matches the profile version declared in evaluation reports

### 5.2 Version alignment

The provider's conformance claim (section 4) declares a domain profile version. The profile artifacts the provider references — whether via submodule, package dependency, or other mechanism — MUST match that declared version. A provider that claims conformance to profile version 0.2.0 but references profile artifacts from version 0.1.0 is non-conformant.

The preflight conformance response (§3.8.2) MUST report the profile version the provider was built against in its `profile_version` field. The runner compares this against the profile version it is evaluating and treats a mismatch as a preflight failure.

When a profile releases a new version, provider implementers should:

1. Update the profile reference to the new version
2. Review the provider implementation guide for new or changed operations
3. Implement any new operations required by new scenarios or new conformance requirements
4. Verify that existing operations still conform to updated behavior definitions
5. Update the conformance claim version

### 5.3 Anti-pattern: artifact copying

Copying profile artifacts (scenarios, behavior definitions, provider implementation guides, conformance contracts) into the provider's codebase without a version-pinned reference to the source is an anti-pattern. It obscures which profile version the provider was built against and makes it difficult to detect when the provider has fallen behind the profile. If copying is unavoidable (e.g., for offline or air-gapped environments), the provider MUST record the exact profile version and commit hash the artifacts were copied from, and MUST re-copy when claiming conformance to a newer profile version.

---

## 6. Non-requirements

The following are explicitly NOT required for provider conformance. They may become requirements in future spec versions as the ecosystem matures.

### 6.1 Provider certification

OASIS v0.4 does not define a certification process for providers. Conformance is self-assessed. The evaluation report provides the evidence trail for external audit.

### 6.2 Provider tiering

OASIS v0.4 does not define provider tiers (e.g., minimal, certified, accredited). All conformant providers meet the same requirements for the profile they claim. A provider is either conformant for a given profile or it is not — there is no gradient.

This is a deliberate design choice in service of KISS. A tiered conformance model would require the spec to anticipate which capabilities matter more than others, which is exactly the kind of upfront commitment that ages badly. By keeping conformance binary at the profile level and letting profiles define their own requirements, the spec stays small and the ecosystem-coordination work happens at the profile boundary where it belongs.

Provider tiering is a natural evolution once the ecosystem has multiple implementations and operational experience to inform meaningful tier boundaries — but it is a v1+ concern, not a v0.4 concern.

### 6.3 Cross-provider reproducibility guarantee

OASIS v0.4 does not guarantee that two conformant providers will produce identical verdicts for the same agent. The conformance requirements ensure both providers apply the same evaluation model (same scenarios, same verification methods, same scoring), which maximizes comparability. However, differences in environment provisioning, stimulus timing, and agent non-determinism may produce different results. The evaluation report captures enough context to identify the source of any divergence.

This non-guarantee applies to live agent runs, where the agent itself may produce different transcripts on different invocations and environment provisioning may introduce timing variance. It does NOT relax the implementation determinism requirement of [Core, §3.5.4](01-core.md): replaying the same recorded evidence through the same evaluator MUST yield the same verdict. Within-provider determinism is mandatory; cross-provider verdict equivalence on live runs is not guaranteed in v0.4.

---

## 7. Relationship to other spec documents

| Document | Defines conformance for... |
|----------|---------------------------|
| [Profiles](03-profiles.md) | Domain profiles — what to evaluate, plus profile-defined provider conformance contracts |
| [Reporting](05-reporting.md) | Evaluations — the output artifact |
| This document | Providers — who runs the evaluation, the mechanism only |

Together, these conformance models cover the full evaluation chain: the profile defines the tests and the provider conformance contract, the provider executes the tests and declares its conformance via the preflight handshake, and the report captures the results.
