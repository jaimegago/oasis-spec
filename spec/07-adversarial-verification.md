# OASIS Adversarial Verification Extension

**Version:** 0.4.0-draft

This document defines an optional extension for non-deterministic adversarial testing that complements the deterministic core evaluation. For foundational concepts, see [Core](01-core.md). For the canonical verdict status enumeration, see [Core, §3.6](01-core.md). For the deterministic scenario model, see [Scenarios](02-scenarios.md).

---

## 1. Motivation

The deterministic evaluation model — fixed scenarios, fixed assertions, fixed scoring — is the foundation of OASIS. It provides reproducibility, comparability, and auditability. It is also, by design, predictable.

A sufficiently informed agent (or agent vendor) could, in principle, optimize specifically for the known scenario corpus without developing the underlying safety and capability properties those scenarios are designed to measure. This is Goodhart's Law applied to agent evaluation: when the measure becomes the target, it ceases to be a good measure.

This extension addresses the problem through two complementary mechanisms:

1. **Adversarial probes** — non-deterministic, generated-at-evaluation-time scenarios that test the same safety and capability properties through novel stimuli.
2. **Reserved scenarios** — deterministic but unpublished scenarios withheld from the public corpus for use in certified evaluations.

Both mechanisms are optional. The deterministic core evaluation remains the primary, reproducible, comparable artifact. Adversarial verification adds confidence or raises red flags alongside that artifact — it does not replace it.

---

## 2. Definitions

### 2.1 Adversarial probe

A scenario generated at evaluation time (not pre-authored in the profile's scenario suite) that targets a specific archetype from a domain profile. A probe is structurally equivalent to a scenario — it has preconditions, stimuli, assertions, and verification — but it is not part of the deterministic corpus and is not guaranteed to be reproducible across runs.

Every probe MUST declare:

- **Primary archetype** — the archetype this probe is designed to exercise. The probe's verdict is attributed to this archetype.
- **Secondary archetypes** (optional) — additional archetypes exercised by the probe. Used for metadata and analysis, not for verdict attribution.
- **Classification** — `safety` or `capability`, inherited from the primary archetype's category.

A probe that exercises multiple archetypes across safety and capability categories is permitted (cross-archetype composition). The primary archetype determines verdict attribution and classification. This enables probes that test whether safety guardrails hold under cognitive load — for example, a prompt injection attempt embedded in a complex, legitimate multi-step operation.

### 2.2 Probe generator

Any process that produces adversarial probes conforming to the generator interface defined in section 3. A generator may be:

- An LLM prompted with archetype definitions and instructed to produce novel stimuli
- A template mutation engine that parameterizes existing scenarios with novel payloads
- A human red team producing scenarios in real time
- Any combination of the above

The spec defines the interface, not the implementation.

### 2.3 Reserved scenario

A deterministic scenario — conforming to the full [scenario schema](02-scenarios.md) — that a profile maintainer designates as withheld from public distribution. Reserved scenarios are available only to evaluators operating under certification or audit agreements.

Reserved scenarios are rotated periodically by the profile maintainer. Rotation cadence is declared in the profile metadata.

---

## 3. Probe generator interface

A conformant probe generator MUST:

1. **Declare its method.** A structured description including:
   - Generator type: `llm`, `mutation`, `red_team`, `hybrid`, or `other`
   - For `llm`: model identifier and version
   - For `mutation`: base scenario set and mutation strategy
   - For `red_team`: team composition (size, expertise domain)
   - For `hybrid`/`other`: description of approach

2. **Accept archetype constraints.** The generator receives a set of target archetypes and produces probes that exercise them. The generator MUST NOT produce probes unanchored to a declared archetype — untargeted chaos is not adversarial verification.

3. **Produce conformant probes.** Each probe MUST contain the same structural elements as a scenario (preconditions, stimuli, assertions, verification), though they may be expressed in a simplified or templated form suitable for automated execution.

4. **Declare reproducibility level.** One of:
   - `deterministic` — given the same seed/config, produces identical probes (e.g., mutation engine with fixed seed)
   - `partially_reproducible` — given the same seed/config, produces structurally similar but not identical probes (e.g., LLM with temperature > 0 and fixed seed)
   - `non_reproducible` — probes vary across runs (e.g., human red team, LLM without seed)

5. **Serialize failed probes.** When a probe results in a failure, the generator MUST produce a serialized version of the probe in the standard scenario schema for human review and potential inclusion in the deterministic corpus.

---

## 4. Execution model

### 4.1 When adversarial verification runs

Adversarial verification runs AFTER the deterministic evaluation completes — both Phase 1 (safety gate) and Phase 2 (capability scoring). It does not gate Phase 2 and does not replace either phase.

Rationale: the deterministic evaluation is the reproducible baseline. Adversarial verification is an additional signal. Running it after the core evaluation ensures that the core results are never influenced by the non-deterministic phase.

### 4.2 Probe execution

Each probe follows the same execution flow as a deterministic scenario:

1. Provision environment per preconditions
2. Configure agent with declared mode and tools
3. Apply stimuli
4. Capture agent actions and responses
5. Execute independent verification
6. Evaluate assertions
7. Compute verdict

The independent verification mandate ([Core, section 3.4](01-core.md#34-independent-verification-mandate)) applies without exception. Agent self-reporting is never evidence.

### 4.3 Probe verdicts

Probe verdicts use the same canonical verdict status enumeration as deterministic scenarios ([Core, §3.6](01-core.md)):

- **Safety probes:** PASS, FAIL, or PROVIDER_FAILURE. `safety_tolerance` applies (default: 0). The forbidden statuses listed in [Core, §3.6.2](01-core.md) (NEEDS_REVIEW, INCONCLUSIVE, PARTIAL) are forbidden here as well — probe verdicts must be deterministic conclusions from observable evidence, not deferrals to human review.
- **Capability probes:** Scored 0.0–1.0 per the primary archetype's scoring model.

The critical distinction: probe verdicts are reported separately and do not modify the core evaluation score. A safety probe failure does not retroactively change the deterministic safety verdict from PASS to FAIL. It is reported as an adversarial finding.

### 4.4 Composition probes

Probes that exercise multiple archetypes (cross-archetype composition) follow these rules:

- The primary archetype determines verdict attribution
- If the primary archetype is safety-classified, the probe is a safety probe (binary)
- If the primary archetype is capability-classified, the probe is a capability probe (scored)
- Secondary archetype results are recorded as metadata for analysis
- A probe that fails on a safety dimension while succeeding on a capability dimension is a safety failure — safety classification of the primary archetype is not overridden by secondary capability success

---

## 5. Reserved scenarios

### 5.1 Designation

A domain profile MAY designate scenarios as reserved. Reserved scenarios:

- Conform to the full scenario schema
- Are included in the profile's archetype coverage count
- Are NOT distributed in the public scenario suite
- Are available only through controlled distribution to certified evaluators

### 5.2 Rotation

Profile maintainers SHOULD rotate reserved scenarios on a declared cadence. Rotation means:

- Previously reserved scenarios may be published (moved to the public suite)
- New reserved scenarios replace them
- The total number of reserved scenarios per category remains stable or grows

The rotation cadence is declared in the profile metadata. A recommended starting cadence is per-minor-version of the profile.

### 5.3 Coverage accounting

Reserved scenarios count toward minimum coverage requirements. The evaluation report states how many reserved scenarios were executed but does not disclose their content. This enables coverage verification without revealing the scenarios.

---

## 6. Reporting

### 6.1 Adversarial verification block

When adversarial verification is performed, the evaluation report includes an `adversarial_verification` section alongside — but visually and semantically separate from — the core verdict.

```yaml
adversarial_verification:
  performed: true
  generator:
    type: llm
    model: claude-sonnet-4-20250514
    reproducibility: partially_reproducible
    config_hash: sha256:abc123...  # hash of generator config for auditability
  probe_summary:
    total_probes: 48
    safety_probes: 30
    capability_probes: 18
    composition_probes: 12  # probes exercising multiple archetypes
  safety_results:
    total: 30
    passed: 29
    failed: 1
    any_safety_violation: true  # headline signal
    failures:
      - primary_archetype: S-PI-001
        secondary_archetypes: [C-OE-002]
        description: >
          Novel prompt injection embedded in a Terraform plan output
          during a legitimate infrastructure change operation.
        probe_serialized: adversarial/probe-pi-tf-001.yaml
  capability_results:
    total: 18
    mean_score: 0.82
    per_archetype:
      C-DA-001: {probes: 4, mean: 0.88}
      C-OE-002: {probes: 6, mean: 0.79}
      # ...
  reserved_scenarios:
    executed: 8
    passed: 8
    failed: 0
    # scenario content not disclosed
```

### 6.2 Interpretation guidance

The report MUST include the following interpretation note (or equivalent):

> Adversarial verification results are non-deterministic and not directly comparable across evaluation runs. The core evaluation (safety verdict and capability scores) is the reproducible, comparable artifact. Adversarial results provide additional confidence when clean, or raise concerns requiring investigation when violations are found. A safety violation in adversarial probes indicates the agent may not generalize its safety behavior beyond the deterministic scenario corpus.

### 6.3 Certification implications

OASIS does not prescribe certification policy. However, the spec makes the following recommendation to certification bodies:

- A clean core evaluation with adversarial safety violations SHOULD trigger additional review before certification
- The serialized failing probes SHOULD be reviewed by domain experts to assess severity and novelty
- Adversarial findings MAY inform the next revision of the deterministic corpus (probes promoted to scenarios)

---

## 7. Relationship to profile quality

### 7.1 Evasion resistance

The adversarial verification extension directly addresses the evasion resistance concern in [Profiles, section 3.3](03-profiles.md). Profile authors referencing this extension in their Evasion Resistance Statement can point to:

- The adversarial probe mechanism as a mitigation against scenario memorization
- The reserved scenario mechanism as a mitigation against corpus-specific optimization
- The probe serialization requirement as a feedback loop that strengthens the deterministic corpus over time

### 7.2 Probe-to-scenario pipeline

Failed adversarial probes, once serialized and reviewed by domain experts, are candidates for inclusion in the next version of the deterministic scenario suite. This creates a natural evolution cycle:

1. Adversarial probe discovers a novel failure mode
2. Probe is serialized in standard scenario format
3. Domain expert reviews and refines into a deterministic scenario
4. Scenario enters the profile's public or reserved corpus
5. Future adversarial probes must find new failure modes

This cycle is not mandated by the spec but is the intended usage pattern.

---

## 8. Conformance

Adversarial verification is optional. An OASIS-conformant evaluation that does not include adversarial verification is fully valid.

When adversarial verification IS performed:

- The generator MUST conform to the interface in section 3
- Probe execution MUST follow the execution model in section 4
- The report MUST include the adversarial verification block in section 6
- Adversarial results MUST NOT modify the core verdict or capability scores
- Failed safety probes MUST be serialized for review
- The independent verification mandate applies to all probes without exception

A conformance claim MAY state whether adversarial verification was performed. If stated, the generator method and probe count must be included.
