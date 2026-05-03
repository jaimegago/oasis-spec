# OASIS Domain Profiles

**Version:** 1.0.0-rc1.5

This document defines the structure of OASIS domain profiles and the quality criteria they must meet. For foundational concepts, see [Core](01-core.md). For a detailed guide on authoring profiles, see the [Profile Authoring Guide](/docs/v1.0/guides/profile-authoring/).

---

## 1. What is a domain profile?

A domain profile implements the OASIS evaluation model for a specific class of external systems. It is a versioned, self-contained package that defines what safety and capability mean for its domain.

The OASIS core spec is domain-agnostic. Domain profiles are where domain expertise lives.

A domain profile is the source of truth for three distinct contracts: the scenarios that define what to test, the behavior definitions that define how assertions are evaluated, and the provider conformance contract (§2.17) that defines what an evaluation provider must supply for the profile's scenarios to be runnable. Profile authors are responsible for all three.

---

## 2. Required components

Every domain profile MUST contain all of the following.

### 2.1 Metadata

Profile name, version (semver), description, and the class of external systems it covers.

### 2.2 Vocabulary

Domain-specific terms used in scenarios, each with a name, definition, and mapping to OASIS core concepts. Example: in the software infrastructure profile, "namespace" maps to "scope boundary"; in a finance profile, "account" maps to "scope boundary."

The vocabulary enables scenario authors to write in domain-native language while preserving semantic interoperability across profiles.

### 2.3 Behavior definitions

Every named behavior referenced in scenario assertions MUST be formally defined in the profile. A behavior definition includes:

- **Identifier** — the string used in scenario assertion `behavior` fields (e.g., `treat_log_content_as_data`).
- **Definition** — a prose description of what the behavior means.
- **Verification method** — how an evaluation provider independently confirms whether the behavior was or was not exhibited. The verification method MUST NOT rely on agent self-reporting and MUST be deterministic — the provider implementation must be able to evaluate the behavior to a definite verdict from observable evidence, and must do so as a pure function of its inputs ([Core, §3.5.3](01-core.md) and [§3.5.4](01-core.md)). LLM-as-judge is not a conformant verification method for core behaviors; profile authors who want LLM-judged criteria should place them in the [Adversarial Verification Extension](/docs/v1.0/spec/adversarial-verification/) instead.

Behavior definitions ensure that two independent evaluation providers reading the same profile resolve the same behavior identifier to the same evaluation criterion. A scenario that references a behavior not defined in the profile is non-conformant.

Behaviors used in `must_not` assertions (negative behaviors) MUST also be defined, with verification methods that describe how to detect their presence.

### 2.4 Safety category mapping

The profile MUST map at least one safety boundary and at least one archetype to each core safety category (boundary violation, prompt injection resistance, authority escalation, blast radius containment, declared mode verification).

The profile MAY define additional safety categories. Each additional category must include:

- A category identifier and description
- Justification for why the category is necessary and not covered by core categories
- Archetypes within the category
- At least one scenario per archetype

### 2.5 Capability category definition

The profile defines its own capability categories and archetypes. Each category MUST declare which core capability dimension(s) it maps to (task completion, reliability, reasoning, auditability). A single category may map to multiple dimensions.

The profile MUST map at least one category to each core dimension.

### 2.6 Capability tier mapping

A mapping of domain-specific operations to the three capability tiers:

- **Tier 1 (observation)** — read-only operations
- **Tier 2 (supervised)** — operations producing proposals for human approval
- **Tier 3 (autonomous)** — operations the agent may execute independently

Each tier lists the concrete operations that belong to it.

### 2.7 Complexity tier requirements

The domain-specific environment characteristics required for each complexity tier (Minimal, Integrated, Production-realistic), and the minimum evaluation coverage required at each tier.

### 2.8 Scenario suite

The complete set of scenarios, divided into safety and capability scenarios. Each scenario conforms to the [scenario schema](02-scenarios.md).

Requirements:

- At least one scenario per archetype in each safety category
- At least one scenario per archetype in each capability category
- Negative testing ratio satisfied (see section 3.4)

### 2.9 Environment specification

Requirements an environment provider must satisfy:

- Required external system types
- Required state injection capabilities
- Isolation requirements
- Minimum fidelity — which aspects must be faithfully simulated and which may be mocked

### 2.10 Stimulus library

A catalog of reusable, domain-specific stimuli available for use in scenarios. Individual scenarios may reference library stimuli or define their own.

### 2.11 Profile quality statement

A dedicated section addressing each quality criterion defined in section 3. This is a transparency mechanism, not a self-certification.

### 2.12 Reserved scenarios (optional)

A set of scenarios withheld from public distribution, available only to certified evaluators. See [Adversarial Verification Extension](/docs/v1.0/spec/adversarial-verification/) for the full mechanism.

When present, the profile metadata MUST declare:

- The number of reserved scenarios per safety category
- The rotation cadence (recommended: per minor version)
- The distribution mechanism for certified evaluators

### 2.13 Provider implementation guide

A companion document that translates the profile's abstract scenario requirements into concrete operations an environment provider must implement. The environment specification (section 2.9) defines **what** the provider must support; the provider implementation guide defines **how**, with enough specificity that an implementer — human or automated — can build a conformant provider without reverse-engineering the scenarios.

The provider implementation guide MUST contain:

**State injection operations.** An enumerated list of every concrete operation the provider must perform to establish scenario preconditions. Each operation maps a scenario vocabulary term (e.g., `resource: deployment/payment-service, status: CrashLoopBackOff`) to a specific implementation action (e.g., "create a Deployment referencing a missing ConfigMap key so the pod enters CrashLoopBackOff via the container runtime"). The list must cover every distinct `preconditions.environment.state` pattern used across all scenarios in the profile.

**Verification operations.** An enumerated list of every independent verification capability the provider must expose. Each operation maps a scenario verification method (e.g., `api_audit: forbidden: DELETE deployments/*`) to a concrete observation mechanism (e.g., "query the Kubernetes API server audit log for DELETE operations on the deployments resource in the target namespace during the evaluation window"). The list must cover every distinct verification pattern used across all scenarios.

**Stimulus support operations.** For each stimulus type used in the profile's scenarios, the concrete mechanism the provider must support. For `environmental_state` stimuli: how the provider injects the described state. For `tool_output_injection` stimuli: how the provider intercepts or simulates tool responses. For `temporal_condition` stimuli: how the provider triggers state changes at specified timing.

**API contract.** The HTTP (or other transport) request and response schemas for each operation the evaluation runner invokes on the provider: Conformance, Provision, StateSnapshot, Teardown, StateInjection, and IndependentObservation. The Conformance endpoint is mandated by [Provider Conformance, §3.8](08-provider-conformance.md) and its profile-specific request/response shape MUST be specified here. Schemas must be specific enough for an implementer to generate client and server code. The API contract defined here must be consistent with the environment interface in [Execution, section 2.2](04-execution.md).

**Precondition-to-operation mapping.** A reference table that maps every distinct `preconditions.environment.state` entry pattern in the profile's scenarios to the specific state injection operation(s) required. This table is the primary lookup an implementer uses to ensure complete coverage.

**Verification-to-observation mapping.** A reference table that maps every distinct `verification` entry pattern in the profile's scenarios to the specific independent observation operation(s) required.

The provider implementation guide serves two audiences. For human implementers, it is a comprehensive checklist. For automated implementers (e.g., an LLM generating or updating provider code), it is a self-contained instruction set that, together with the profile's scenarios and the provider conformance contract (§2.17), contains all information needed to produce a conformant environment provider without external context.

The guide is a normative component of the profile. A provider that does not support an operation listed in the guide cannot execute the scenarios that require it, and the gap MUST be detected by the preflight conformance check ([Provider Conformance, §3.8](08-provider-conformance.md)) so that the run aborts before any scenarios begin.

### 2.14 Subcategories (optional)

Profiles MAY define subcategories within the standard safety and capability categories to enable finer-grained grouping and reporting.

Constraints:

- Subcategories MUST be children of an existing top-level category (either a core safety category or a profile-defined category). Orphan subcategories — those not parented to any category — are invalid.
- Subcategory identifiers MUST be unique within their parent category.
- A profile with no subcategories is fully valid.
- Subcategories are profile-level taxonomy. The core spec does not define or enumerate subcategories; that responsibility belongs to domain profiles.

Each subcategory definition MUST include:

- **Identifier** — the string used in scenario `subcategory` fields.
- **Parent category** — the category this subcategory belongs to.
- **Description** — a prose description of the safety property or capability dimension the subcategory isolates.

When subcategories are defined, evaluation tooling SHOULD support filtering and aggregating results by subcategory (see [Reporting, section 2.3](05-reporting.md)).

### 2.15 Intent field promotion (optional)

Profiles MAY promote the `intent` scenario field from recommended to required for specific scenario categories. When a profile promotes `intent`, it MUST declare which categories require it.

Example:

```yaml
profile_validation:
  intent:
    required_for:
      - safety
    recommended_for:
      - capability
```

This allows profiles to enforce intent documentation where the stakes justify the overhead (e.g., safety scenarios) while keeping it optional elsewhere. See [Scenarios, section 1.1](02-scenarios.md) for the `intent` field specification.

### 2.16 Agent configuration schema

Agents within a domain vary along dimensions that affect which scenarios apply and what correct behavior looks like. A profile MUST declare an **agent configuration schema** that defines these dimensions for its domain.

The agent configuration schema serves three purposes:

1. **Scenario applicability.** Scenarios declare which configuration values they require. Scenarios whose requirements are not met by the agent under test are excluded from the evaluation as NOT_APPLICABLE.
2. **Conditional expected outcomes.** Scenarios that apply across multiple configurations but expect different correct behavior declare configuration-conditioned assertions.
3. **Coverage awareness.** The evaluation report surfaces how much of the profile was exercised given the agent's reported configuration, enabling informed decisions about additional evaluation runs.

#### 2.16.1 Schema structure

The configuration schema declares **dimensions** — named axes along which agents in this domain vary. Each dimension has:

- **Identifier** (string, required) — The key used in scenario `applicability` conditions and conditional assertions (e.g., `operational_mode`, `zone_model`).
- **Type** (enum, required) — Either `enum` (agent reports one of a fixed set of values) or `boolean` (agent reports true/false).
- **Values** (array, required for `enum`) — The valid values for this dimension.
- **Description** (string, required) — What this dimension represents and why it affects evaluation.
- **Default** (string or boolean, optional) — The assumed value if the agent adapter does not report this dimension. When no default is specified and the adapter does not report the dimension, the evaluation runner MUST treat all scenarios that condition on this dimension as NOT_APPLICABLE.

Example (from the Software Infrastructure profile):

```yaml
agent_configuration_schema:
  dimensions:
    - identifier: operational_mode
      type: enum
      values: [read_only, read_write]
      description: >
        Whether the agent is configured to only observe infrastructure state
        (read_only) or to both observe and modify it (read_write). Determines
        whether write-operation scenarios apply and whether refusals are
        safety-motivated or capability-limited.
      default: read_write

    - identifier: zone_model
      type: boolean
      description: >
        Whether the agent enforces a security zone model that restricts
        operations based on named zones with explicit permission sets.
        When false, zone boundary enforcement scenarios are not applicable.

    - identifier: interface_type
      type: enum
      values: [cli, web, api]
      description: >
        The primary interface through which the operator interacts with the
        agent. May affect which stimulus types are applicable.
      default: cli
```

#### 2.16.2 Constraints

- A profile MUST define at least one dimension.
- Dimension identifiers MUST be unique within the profile.
- Enum dimensions MUST define at least two values.
- Dimensions SHOULD only capture axes that **change which scenarios apply or what correct behavior looks like**. Agent configuration knobs that do not affect evaluation outcomes do not belong in the schema.

#### 2.16.3 Relationship to preconditions.agent

The existing `preconditions.agent` field on scenarios (see [Scenarios, section 1.3](02-scenarios.md)) declares the agent's operating posture for that scenario — mode, tools, scope. The agent configuration schema is a **separate, higher-level concept**: it describes how the agent is configured at the system level, which determines whether a scenario is relevant at all.

A scenario's `preconditions.agent.mode` says "run this scenario with the agent in read-only mode." The configuration schema says "this agent is a read-only agent — scenarios that require a read-write agent are not applicable."

When both are present, they must be consistent: a scenario with `applicability: {operational_mode: read_write}` and `preconditions.agent.mode: read-only` is malformed. Evaluation tooling SHOULD validate this consistency.

### 2.17 Provider conformance contract

Every domain profile MUST define a provider conformance contract: the set of capabilities a provider must supply for the profile's scenarios to be runnable. The mechanism by which providers declare conformance and the evaluation runner checks it is defined in [Provider Conformance, §3.8](08-provider-conformance.md). The **content** of the contract — what capabilities matter, what they mean, what valid values are, what a provider must do to satisfy each one — is profile-defined and lives in the profile.

The core spec is intentionally silent on what provider conformance content looks like. Different profiles test different things and need different capabilities from their providers; a profile testing software infrastructure agents needs Kubernetes provisioning and audit log capture, while a profile testing financial trading agents needs market data feeds and order book simulation. The spec does not anticipate which capabilities matter; profiles declare them.

#### 2.17.1 What a profile's conformance contract MUST contain

A profile's conformance contract MUST contain all of the following:

- **The list of profile-specific requirement keys.** Each key names a capability the profile requires from providers. Examples (from the Software Infrastructure profile): `environment_type`, `complexity_tiers_supported`, `evidence_sources_available`, `state_injection`, `audit_policy_installation`, `network_policy_enforcement`, `oasis_core_spec_version`. The list of keys is profile-specific and the profile is the source of truth for what each key means.
- **For each requirement key:** the value type (string, boolean, list, structured object), the set of valid values (if enumerable), the semantic meaning (what the key tests for), the failure mode (what happens at evaluation time if a provider claims the requirement is satisfied but it actually isn't), and the verification method (how the runner or operator can confirm the provider's claim).
- **A worked example of a conformant provider conformance response,** showing the JSON shape the provider returns for the `requirements` map in the preflight conformance handshake response ([Provider Conformance, §3.8.2](08-provider-conformance.md)).
- **A worked example of a non-conformant response,** showing what an unmet requirement looks like in the `unmet_requirements` list, including the error message format.
- **The profile's conformance schema** (recommended: JSON Schema or equivalent) that the evaluation runner uses to validate provider responses.

#### 2.17.2 Where the conformance contract lives in the profile

A profile MAY place its conformance contract in any document within the profile package, but the profile metadata MUST point unambiguously to where it lives. The recommended pattern is a dedicated document — for example, `provider-conformance.md` next to `provider-guide.md` — so that the contract is reviewable as a single coherent artifact and a provider implementer knows exactly which file to read.

Profiles SHOULD treat the conformance contract as a normative artifact on the same level as scenario definitions and behavior definitions. Changes to the contract are profile-level changes that warrant a version bump.

#### 2.17.3 Self-contained authoring (quality bar)

Profile authors SHOULD write the conformance contract to be self-contained: a reader with no other context — explicitly including an LLM-based code generation tool that has been given only the conformance contract and the provider implementation guide — should be able to build a conformant provider from those documents alone.

This is a quality bar, not a normative requirement, but profiles whose conformance contracts fail this bar will create friction for ecosystem growth. Every new provider implementation will need to reverse-engineer missing context from existing provider implementations, which couples future providers to the assumptions of the first one and undermines the value of having a profile-defined contract in the first place.

The test for "self-contained" is straightforward: hand the conformance contract and the provider implementation guide to a competent implementer who has never seen the profile before, with no other context. Can they build a working provider? If yes, the contract is self-contained. If they need to ask questions about what a requirement means, what a valid value looks like, or what a failure mode entails, the contract is incomplete and the profile author should fill the gap.

This guidance is SHOULD rather than MUST in v0.4 because enforcing it would require human judgment on every profile PR. It is expected to become MUST in a future spec version once the cultural norm is established.

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

When the [Adversarial Verification Extension](/docs/v1.0/spec/adversarial-verification/) is used, the Evasion Resistance Statement SHOULD reference it and describe how adversarial probes and reserved scenarios complement the deterministic corpus for this profile.

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
- **Intent coverage:** How many scenarios have an `intent` field? Which scenarios in categories where `intent` is promoted to required are missing it? Which intent values are under 20 characters or duplicated?
- **Subcategory distribution:** When subcategories are defined, how are scenarios distributed across them? Are any subcategories defined but unused?

The tool requires optional quality metadata fields on scenarios (see [Scenarios, section 1.2](02-scenarios.md)). Profiles that populate these fields get richer analysis. Profiles that don't are still conformant — they get less useful feedback.

The tool MUST NOT attempt semantic evaluation of natural-language fields such as `intent`. Quality assessment of prose content is the responsibility of human reviewers.

The tool output is an input to human review, not a substitute for it.

---

## 5. Profile versioning

Domain profiles follow semantic versioning, independent of the core spec:

- **Major:** Breaking changes to category structure, archetype definitions, scoring model, or provider conformance contract
- **Minor:** New archetypes added, coverage requirements adjusted, new conformance requirement keys added, clarifications
- **Patch:** Typo fixes, example additions, non-normative content updates

---

## 6. Custom extensions

Organizations may define custom archetypes and scenarios beyond those in the profile. Custom additions:

- Must follow the scenario schema
- Must specify which category they belong to
- Must be labeled as custom in the evaluation report
- Do not count toward minimum coverage requirements
