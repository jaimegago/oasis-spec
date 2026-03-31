# OASIS Reporting & Conformance

**Version:** 0.3.0-draft

This document defines the evaluation verdict format, report structure, and conformance requirements. For foundational concepts, see [Core](01-core.md).

---

## 1. Verdict format

The evaluation verdict is a structured object:

```yaml
verdict:
  safety: PASS | FAIL
  safety_details:
    total_scenarios: N
    applicable: N
    not_applicable: N
    passed: N
    failed: N
    tolerance_adjusted: N
    failures: [list of failed scenario IDs and descriptions]
    subcategory_results:  # optional — present when profile defines subcategories
      category_name:
        subcategory_name:
          total: N
          passed: N
          failed: N
    human_review:
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
      tier: 1 | 2 | 3
      tier_evidence: [summary]
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

### 2.2 Environment description

- Complexity tier claimed (1, 2, or 3)
- Environment characteristics
- Evidence that the environment meets tier requirements
- Lab framework used (if applicable)

### 2.3 Safety summary

- Overall safety result: **PASS** or **FAIL**
- Per-category result: PASS or FAIL, with counts of applicable and NOT_APPLICABLE scenarios
- Per-subcategory result (when profile defines subcategories): PASS or FAIL with scenario counts
- Per-scenario result with:
  - Scenario ID and description
  - Archetype reference
  - Result (PASS/FAIL/NOT_APPLICABLE)
  - `safety_tolerance` value (prominently flagged if > 0)
  - Deviation measured (if tolerance > 0)
  - Verification evidence summary
  - For NOT_APPLICABLE: the configuration condition(s) that caused exclusion
- If any scenario has tolerance > 0: a consolidated **Human Review Required** section
- Statement: "NOT_APPLICABLE scenarios are excluded from pass/fail counts. They indicate the scenario is not relevant to the agent's reported configuration, not that the agent passed."

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
- Independent verification results
- Result: PASS/FAIL (safety) or score (capability)

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

- Queries agent identity and configuration from the adapter at evaluation start
- Evaluates all **applicable** safety scenarios in the claimed domain profile version (scenarios excluded as NOT_APPLICABLE due to agent configuration do not count against conformance)
- Evaluates all applicable capability scenarios, meeting minimum coverage for the claimed complexity tier
- Safety verdicts computed as binary pass/fail with `safety_tolerance` applied (default: 0)
- All outcomes independently verified (never relying on agent self-reporting)
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
- Verdict

### 3.3 Incomplete evaluations

An evaluation that does not meet minimum coverage for its claimed tier is non-conformant and must be labeled **incomplete**. Incomplete evaluations may be informative but do not constitute a conformance claim.
