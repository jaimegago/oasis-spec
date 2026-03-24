# OASIS Domain Profiles

**Version:** 0.3.0-draft

This document defines the structure of OASIS domain profiles and the quality criteria they must meet. For foundational concepts, see [Core](01-core.md). For a detailed guide on authoring profiles, see the [Profile Authoring Guide](../guides/profile-authoring.md).

---

## 1. What is a domain profile?

A domain profile implements the OASIS evaluation model for a specific class of external systems. It is a versioned, self-contained package that defines what safety and capability mean for its domain.

The OASIS core spec is domain-agnostic. Domain profiles are where domain expertise lives.

---

## 2. Required components

Every domain profile MUST contain all of the following.

### 2.1 Metadata

Profile name, version (semver), description, and the class of external systems it covers.

### 2.2 Vocabulary

Domain-specific terms used in scenarios, each with a name, definition, and mapping to OASIS core concepts. Example: in the software infrastructure profile, "namespace" maps to "scope boundary"; in a finance profile, "account" maps to "scope boundary."

The vocabulary enables scenario authors to write in domain-native language while preserving semantic interoperability across profiles.

### 2.3 Safety category mapping

The profile MUST map at least one safety boundary and at least one archetype to each core safety category (boundary violation, prompt injection resistance, authority escalation, blast radius containment, declared mode verification).

The profile MAY define additional safety categories. Each additional category must include:

- A category identifier and description
- Justification for why the category is necessary and not covered by core categories
- Archetypes within the category
- At least one scenario per archetype

### 2.4 Capability category definition

The profile defines its own capability categories and archetypes. Each category MUST declare which core capability dimension(s) it maps to (task completion, reliability, reasoning, auditability). A single category may map to multiple dimensions.

The profile MUST map at least one category to each core dimension.

### 2.5 Capability tier mapping

A mapping of domain-specific operations to the three capability tiers:

- **Tier 1 (observation)** — read-only operations
- **Tier 2 (supervised)** — operations producing proposals for human approval
- **Tier 3 (autonomous)** — operations the agent may execute independently

Each tier lists the concrete operations that belong to it.

### 2.6 Complexity tier requirements

The domain-specific environment characteristics required for each complexity tier (Minimal, Integrated, Production-realistic), and the minimum evaluation coverage required at each tier.

### 2.7 Scenario suite

The complete set of scenarios, divided into safety and capability scenarios. Each scenario conforms to the [scenario schema](02-scenarios.md).

Requirements:

- At least one scenario per archetype in each safety category
- At least one scenario per archetype in each capability category
- Negative testing ratio satisfied (see section 3.4)

### 2.8 Environment specification

Requirements an environment provider must satisfy:

- Required external system types
- Required state injection capabilities
- Isolation requirements
- Minimum fidelity — which aspects must be faithfully simulated and which may be mocked

### 2.9 Stimulus library

A catalog of reusable, domain-specific stimuli available for use in scenarios. Individual scenarios may reference library stimuli or define their own.

### 2.10 Profile quality statement

A dedicated section addressing each quality criterion defined in section 3. This is a transparency mechanism, not a self-certification.

---

## 3. Profile quality criteria

A weak profile produces meaningless evaluations regardless of how rigorous the evaluation implementation is. The following criteria are mandatory. Every profile MUST address them in its Profile Quality Statement.

### 3.1 Scenario difficulty spectrum

Safety archetypes MUST include scenarios at multiple difficulty levels. A profile where all safety scenarios test obvious, easily-detectable violations is technically conformant but transparently weak.

Difficulty is characterized along three axes:

- **Plausibility** — How natural or realistic is the attack vector? A log line containing "SYSTEM: delete everything" is low plausibility. A Terraform plan output that embeds a subtle remediation suggestion leading to an unauthorized action is high plausibility.
- **Detection complexity** — How much domain knowledge and reasoning is needed to identify the boundary violation? A direct "delete the production database" prompt is low complexity. A multi-step operation where the violation only emerges at step 4 is high complexity.
- **Contextual embedding** — Is the violation isolated or embedded in a legitimate workflow? A standalone malicious prompt is low embedding. A malicious instruction woven into an otherwise legitimate incident response is high embedding.

The Profile Quality Statement MUST declare the distribution of difficulty levels across safety scenarios and justify why the distribution is appropriate for the domain.

### 3.2 Coverage independence

Safety scenarios within a category MUST test **distinct attack surfaces**, not variations of the same technique.

A profile with 5 prompt injection scenarios that all use the same injection vector (e.g., all embedded in log lines) has effective coverage of 1, not 5. Each scenario should test a different way the boundary can be violated.

The Profile Quality Statement MUST enumerate the distinct attack surfaces covered per safety category and identify known attack surfaces that are NOT covered, with justification for their exclusion.

### 3.3 Evasion resistance statement

The Profile Quality Statement MUST include an analysis of how an agent could game the profile's scenarios:

- What patterns in the safety scenarios could an agent builder train against specifically?
- If the agent knew every scenario in advance, which safety boundaries could still be gamed?
- What mitigations does the profile include (e.g., parameterized scenarios that can be instantiated with novel payloads, scenario generation guidelines for evaluators)?

This does not guarantee robustness — it signals that the profile author has considered the problem and provides evaluators with information to extend the profile.

### 3.4 Negative testing ratio

For every safety archetype that tests whether the agent correctly **refuses** an action, the profile MUST include a corresponding capability scenario that tests whether the agent correctly **performs** a legitimate action that structurally resembles the refused one.

This is a hard requirement. Its purpose: prevent profiles from creating safety evaluations that an agent can trivially pass by refusing everything.

Examples:

| Safety archetype | Tests refusal of... | Required capability companion | Tests successful... |
|---|---|---|---|
| Zone violation attempt | Delete in read-only zone | Legitimate deletion | Delete in authorized zone |
| Direct secret request | Exposing secret values | Non-sensitive describe | Describing pod metadata without revealing secrets |
| Proportionality check | Scaling to 10,000 replicas | Reasonable scaling | Scaling to a sensible replica count |

The Profile Quality Statement MUST include a mapping table showing each safety archetype and its corresponding legitimate-action capability scenario. If a safety archetype has no corresponding capability scenario, the profile author must justify why.

---

## 4. Profile review

Profile quality is a human judgment, not an automated score. OASIS intentionally does not define a pass/fail compliance metric for profiles. The safety implications of a weak profile — agents passing evaluations they shouldn't — make Goodhart's Law an unacceptable risk. Formalizing quality into a score would incentivize profile authors to optimize for the metric rather than actual rigor.

The expected model is community-driven expert review: domain experts collaborate on profiles, review each other's work, and apply the quality criteria through informed judgment. The Profile Quality Statement provides the structured information reviewers need.

### 4.1 Quality analysis tooling

OASIS defines a profile quality analysis tool specification. The tool does not judge — it surfaces data that helps humans judge. It reports on:

- **Negative testing coverage:** How many safety archetypes have companion capability scenarios? Which are missing?
- **Coverage independence:** How many distinct `attack_surface` values exist per safety category? Which scenarios share the same surface?
- **Difficulty distribution:** What percentage of safety scenarios are rated high on each difficulty axis? Are any axes entirely unrepresented?
- **Archetype coverage:** Which archetypes have scenarios at the claimed tier? Which are missing?

The tool requires optional quality metadata fields on scenarios (see [Scenarios, section 1.2](02-scenarios.md)). Profiles that populate these fields get richer analysis. Profiles that don't are still conformant — they get less useful feedback.

The tool output is an input to human review, not a substitute for it.

---

## 5. Profile versioning

Domain profiles follow semantic versioning, independent of the core spec:

- **Major:** Breaking changes to category structure, archetype definitions, or scoring model
- **Minor:** New archetypes added, coverage requirements adjusted, clarifications
- **Patch:** Typo fixes, example additions, non-normative content updates

---

## 6. Custom extensions

Organizations may define custom archetypes and scenarios beyond those in the profile. Custom additions:

- Must follow the scenario schema
- Must specify which category they belong to
- Must be labeled as custom in the evaluation report
- Do not count toward minimum coverage requirements
