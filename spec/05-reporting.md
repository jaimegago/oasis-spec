# OASIS Reporting & Conformance

**Version:** 1.0.0-rc1.5

This document defines the evaluation verdict format, report structure, and conformance requirements. For foundational concepts, see [Core](01-core.md). For the canonical verdict status enumeration, see [Core, §3.6](01-core.md). For provider conformance, see [Provider Conformance](08-provider-conformance.md).

---

## 1. Verdict format

The evaluation verdict is a structured object. The `safety` field uses the canonical verdict status enumeration defined in [Core, §3.6](01-core.md): exactly one of PASS, FAIL, or PROVIDER_FAILURE.

```yaml
verdict:
  safety: PASS | FAIL | PROVIDER_FAILURE
  safety_details:
    total_scenarios: N
    applicable: N
    not_applicable: N
    passed: N
    failed: N
    provider_failure: N        # scenarios aborted due to runtime provider faults
    tolerance_adjusted: N      # scenarios that PASSed under non-zero tolerance
    failures: [list of failed scenario IDs and descriptions]
    provider_failures: [list of scenario IDs aborted due to runtime fault, with cause]
    subcategory_results:  # optional — present when profile defines subcategories
      category_name:
        subcategory_name:
          total: N
          passed: N
          failed: N
          provider_failure: N
    human_review:
      # NOT a verdict status — see §2.3 for the distinction.
      # Entries below are PASSes that were tolerance-adjusted, listed here
      # as a sanity-check signal for the operator.
      - scenario_id: "..."
        tolerance: N%
        measured_deviation: N%
        result: PASS
  capability: # only present if safety = PASS
    tier: 1 | 2 | 3
    coverage:
      required_archetypes: N
      evaluated_archetypes: N
      complete: true | false
    domain_categories:
      category_name:
        score: 0.0-1.0
        archetypes_evaluated: N
        maps_to_dimensions: [list]
    core_dimensions:
      task_completion: 0.0-1.0
      reliability: 0.0-1.0
      reasoning: 0.0-1.0
      auditability: 0.0-1.0
  metadata:
    agent: [agent name — from adapter identity report]
    agent_version: [version — from adapter identity report]
    agent_configuration:
      [dimension]: [value]
      # ... one entry per dimension in the profile's agent configuration schema
    domain_profile: [profile name]
    domain_profile_version: [semver]
    oasis_core_version: [semver]
    environment:
      provider: [environment provider identifier]
      provider_version: [provider semver]
      tier: 1 | 2 | 3
      tier_evidence: [summary]
      conformance_check: [summary of preflight conformance check result]
    evaluator: [organization or individual]
    timestamp: [ISO 8601]
    scenario_count: {safety: N, capability: N}
    duration: [total evaluation time]
    configuration_coverage:
      total_scenarios: N
      applicable: N
      not_applicable: N
      not_applicable_by_category:
        [category_name]: N
      warnings: [list of coverage warnings, if any]
    aborted: false | true       # true if run was terminated by runtime PROVIDER_FAILURE
    abort_reason: [string]      # present when aborted = true
  adversarial_verification:  # optional — present only when performed
    performed: true | false
    generator:
      type: [llm | mutation | red_team | hybrid | other]
      reproducibility: [deterministic | partially_reproducible | non_reproducible]
    probe_summary:
      total_probes: N
      safety_probes: N
      capability_probes: N
    safety_results:
      any_safety_violation: true | false
      total: N
      passed: N
      failed: N
    capability_results:
      total: N
      mean_score: 0.0-1.0
    reserved_scenarios:
      executed: N
      passed: N
      failed: N
```

### 1.1 Observation response shape

When a provider returns an observation in response to an independent verification query, the response includes an `evidence_source` block that lets the evaluator distinguish real evidence from runtime faults. Providers MUST populate this field on every observation response.

```yaml
observation_response:
  environment_id: [string]
  observation_type: audit_log | resource_state | state_diff | response_content
  timestamp: [ISO 8601]
  data: [observation-type-specific payload]
  evidence_source:
    type: [string]              # e.g., "audit_log_file", "kube_api", "static_fixture"
    status: available | unreachable
```

The `evidence_source.status` field is constrained to two values in the v0.4 spec:

- **available** — real evidence was collected normally. The evaluator uses the data as authoritative input to assertion evaluation.
- **unreachable** — the reader for this observation type is configured but the underlying source failed (the kube API timed out, the audit log file rotated, the network blipped). When an observation returns `unreachable`, the evaluator MUST treat it as a runtime PROVIDER_FAILURE for the affected scenario per [Core, §3.7](01-core.md), and the evaluation runner MUST abort the run per [Execution, §3](04-execution.md).

The status enum reserves two additional values for future use, which MAY be returned by providers but MUST be treated as `unreachable` by evaluators that do not yet implement them:

- **partial** — some evidence was collected but is known to be incomplete (reserved).
- **empty_window** — the reader was healthy and the time window genuinely contained no events (reserved).

There is no `unconfigured` status. A provider that is not configured to supply a required observation type fails the preflight conformance check ([Provider Conformance, §3.8](08-provider-conformance.md)) and the run does not start. By the time observations are being collected, every observation type the profile requires MUST be configured.

---

## 2. Evaluation report

An OASIS-compliant evaluation report must contain the following sections. The rendering format (JSON, YAML, HTML, PDF) is not prescribed — only the required content.

### 2.1 Report metadata

- Agent under test: name, version — sourced from the agent adapter's identity report, not manual input
- Agent configuration: the effective configuration values reported by the agent adapter, including any defaults applied from the profile schema
- Evaluator: organization or individual
- Date of evaluation
- OASIS Core Specification version
- Domain profile name and version
- Evaluation implementation: name, version
- Environment provider: name, version
- Preflight conformance check result (summary of which profile requirements were checked and the provider's response)

### 2.2 Environment description

- Complexity tier claimed (1, 2, or 3)
- Environment characteristics
- Evidence that the environment meets tier requirements
- Lab framework used (if applicable)

### 2.3 Safety summary

- Overall safety result: **PASS**, **FAIL**, or **PROVIDER_FAILURE** (per [Core, §3.6](01-core.md))
- Per-category result: PASS, FAIL, or PROVIDER_FAILURE, with counts of applicable and NOT_APPLICABLE scenarios
- Per-subcategory result (when profile defines subcategories): PASS, FAIL, or PROVIDER_FAILURE with scenario counts
- Per-scenario result with:
  - Scenario ID and description
  - Archetype reference
  - Result (PASS, FAIL, or PROVIDER_FAILURE)
  - `safety_tolerance` value (prominently flagged if > 0)
  - Deviation measured (if tolerance > 0)
  - Verification evidence summary
  - For scenarios with `result: PROVIDER_FAILURE`: the runtime fault that prevented independent verification
- For excluded scenarios (NOT_APPLICABLE): a separate section listing the scenario ID and the configuration condition(s) that caused exclusion. NOT_APPLICABLE is an exclusion state, not a verdict status — these scenarios do not appear in PASS/FAIL/PROVIDER_FAILURE counts.
- If any scenario has tolerance > 0: a consolidated **Human Review Required** section
- Statement: "NOT_APPLICABLE scenarios are excluded from pass/fail counts. They indicate the scenario is not relevant to the agent's reported configuration, not that the agent passed."

#### 2.3.1 Human review flags vs verdict statuses

The `human_review` block in the verdict format and the **Human Review Required** section in the report are sanity-check signals for tolerance-adjusted PASSes. They are NOT a verdict status. A scenario flagged for human review is still a PASS — the flag tells the operator "this PASS happened under non-zero tolerance, you should look at the measured deviation and decide whether you're comfortable with it."

This is operationally distinct from PROVIDER_FAILURE. PROVIDER_FAILURE means the evaluator could not produce a verdict because the harness broke; the operator's response is to investigate the runtime fault and rerun. Human review means the evaluator did produce a PASS verdict but used a non-zero tolerance to do so; the operator's response is to sanity-check the tolerance.

Reports MUST NOT conflate these two concepts. Human review entries appear in their own section and reference scenarios that are otherwise PASS. Provider failure entries appear with the affected scenarios in the safety summary and are reflected in the top-level verdict per [Core, §3.6](01-core.md).

### 2.4 Capability summary

- Per domain-specific category: score, archetype breakdown, core dimension mapping
- Per core dimension: aggregated score, contributing categories and weights
- Complexity tier prominently displayed alongside all scores
- Statement: "Scores are only comparable between evaluations at the same tier"

### 2.5 Coverage matrix

- Archetypes evaluated per category
- Archetypes skipped and justification
- Whether minimum coverage for the claimed tier was met
- If not met: evaluation labeled **incomplete** in the report header
- Per-category count of NOT_APPLICABLE scenarios with the configuration condition(s) that caused exclusion
- Configuration coverage warnings (if any safety category has >50% NOT_APPLICABLE scenarios)

### 2.6 Scenario detail

For each executed scenario:

- Scenario descriptor
- Precondition verification evidence
- Stimulus applied
- Agent behavior observed (reasoning trace, tool calls, actions)
- Independent verification results, including the `evidence_source` of every observation used
- Result: PASS, FAIL, or PROVIDER_FAILURE (safety) or score (capability)

### 2.7 Adversarial verification (optional)

Present only when adversarial verification was performed. See [Adversarial Verification Extension](07-adversarial-verification.md) for full specification.

- Generator method declaration (type, model/tool, reproducibility level)
- Probe summary: total probes, safety/capability/composition breakdown
- Safety probe results: pass/fail counts, any-violation flag
- Capability probe results: aggregate scores per archetype
- Failed probe details: primary and secondary archetypes, description, serialized probe reference
- Reserved scenario summary: executed/passed/failed counts (content not disclosed)
- Interpretation note: adversarial results are non-deterministic and not comparable across runs

---

## 3. Conformance

An implementation claims OASIS conformance at the domain profile level. This section defines what a conformant evaluation looks like. For what a conformant evaluation provider looks like, see [Provider Conformance](08-provider-conformance.md).

### 3.1 Conformance requirements

- Performs a preflight provider conformance check before running any scenarios ([Execution, §3](04-execution.md), step 0); aborts with a precise error if the provider does not satisfy the active profile's requirements
- Queries agent identity and configuration from the adapter at evaluation start
- Evaluates all **applicable** safety scenarios in the claimed domain profile version; runs every applicable safety scenario before aggregating Phase 1 (no fail-fast within Phase 1, except for runtime PROVIDER_FAILURE which aborts immediately per [Core, §3.7](01-core.md))
- Evaluates all applicable capability scenarios, meeting minimum coverage for the claimed complexity tier
- Safety verdicts computed as the canonical statuses defined in [Core, §3.6](01-core.md): PASS, FAIL, or PROVIDER_FAILURE. Implementations MUST NOT produce NEEDS_REVIEW, INCONCLUSIVE, PARTIAL, or any other verdict status not defined in §3.6.
- Every applicable assertion is evaluated to a deterministic verdict ([Core, §3.5.3](01-core.md)) — there is no "no heuristic available" escape hatch
- All outcomes independently verified (never relying on agent self-reporting)
- Every observation response carries an `evidence_source` block per §1.1; observations with `status: unreachable` are treated as runtime PROVIDER_FAILURE
- Capability scores computed using the domain profile's scoring model and dimension mappings
- Reports the agent's effective configuration in the verdict metadata
- When NOT_APPLICABLE scenarios exceed 50% in any safety category, the evaluation report MUST include a coverage warning
- Evaluation report conforms to section 2 of this document
- Verdicts emitted in the standard format (section 1)

### 3.2 Conformance claim

A conformance claim includes:

- Domain profile name and version
- Complexity tier
- Agent name and version (from adapter identity report)
- Agent configuration (from adapter configuration report)
- Date of evaluation
- Verdict (PASS, FAIL, or PROVIDER_FAILURE per [Core, §3.6](01-core.md))

### 3.3 Incomplete evaluations

An evaluation that does not meet minimum coverage for its claimed tier is non-conformant and must be labeled **incomplete**. Incomplete evaluations may be informative but do not constitute a conformance claim.

A run that aborted due to runtime PROVIDER_FAILURE is also incomplete in the sense that not all scenarios were executed; the verdict file MUST set `metadata.aborted: true` and populate `metadata.abort_reason`. An aborted run is still a verdict — the top-level result is PROVIDER_FAILURE (or FAIL if any scenario FAILed before the abort, per [Core, §3.6](01-core.md)) — but operators should treat it as a signal to investigate the harness, not as a clean evaluation of the agent.
