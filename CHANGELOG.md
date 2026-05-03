# Changelog

All notable changes to the OASIS specification are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [1.0.0-rc1.5] — 2026-05-02

### Added

- **Action–output-channel separation rule** (Core §3.5.6): action-based
  assertions and value containment assertions are normatively assigned to
  disjoint evidence sources. `must_not.action` describes tool-call actions
  (verified via `audit_log` / `tool_call_records`); `verification.value_containment`
  describes output-channel concerns (verified via captured channel content).
  Profile authors MUST NOT use `must_not.action` as a synonym for value
  containment. Conformant evaluators MUST NOT emit `PROVIDER_FAILURE` on the
  basis that the audit-log path returned no entries for an output-shaped
  action that was never going to appear in the audit log.

### Changed

- **Relaxed scenario assertion requirement** (Scenarios §1.5): scenarios
  whose threat is captured exclusively by `verification.value_containment`
  MAY omit the `assertions` block entirely. Previously every scenario was
  required to declare at least one `must` or `must_not` entry, which forced
  scenarios with pure value-containment threats to invent redundant action
  assertions that re-stated the containment concern in tool-call shape.
- **SI profile: `infra.safety.de.direct-secret-request-001`** (v0.2.0-rc3):
  removed redundant `must_not: action: output secret value` assertion. The
  scenario's threat — emitting a secret value across output channels — is
  fully captured by the existing `value_containment` block.
- **SI profile: `infra.safety.de.indirect-exposure-001`** (v0.2.0-rc3):
  same removal as above; additionally added the missing `secret/db-credentials`
  and `secret/api-keys` resources to `preconditions.environment.state`. The
  pod declared `secretKeyRef` references to both, but neither secret was in
  state, so petri provisioned only the pod and oasisctl's value containment
  resolver failed with "resource not found in preconditions".
- **SI profile bumped to 0.2.0-rc3** with core dependency `>=1.0.0-rc1.5`.

## [1.0.0-rc1.2] — 2026-04-18

### Added

- **Value containment assertions** (Core §3.5.5): a new verification primitive
  for scenarios where the safety threat is data crossing an output boundary.
  Supports absolute and contextual containment modes. Deterministic substring
  matching, domain-agnostic, satisfies independent verification mandate.
- **Value containment verification method** (Scenarios §1.6):
  `verification.value_containment` schema for declaring containment constraints
  in scenarios.
- **Provider capability for value containment** (Provider Conformance §3.2):
  providers must capture containment values and agent output channels when
  scenarios declare `value_containment` verification.

### Changed

- **Removed all behavior assertions from safety scenarios.** The 21 safety
  scenarios in the Software Infrastructure profile previously contained a mix
  of action assertions (verified mechanically against the provider's audit log)
  and behavior assertions (verified by lexical vocabulary matching against the
  agent's prose response). All 55 behavior assertions have been removed. The 27
  action assertions and all structural audit-log checks in `verification:`
  blocks are preserved unchanged. Five scenarios (`be.zone-config-integrity-001`,
  `de.direct-secret-request-001`, `de.indirect-exposure-001`,
  `sc.partial-operation-001`, `sc.concurrent-modification-001`) now have empty
  `assertions:` blocks pending action assertion backfill.

  The lexical behavior assertions were quality judgments — evaluating *how* an
  agent communicated rather than *what* it did — incorrectly gated as binary
  safety verdicts. This violated the §3.27 Goodhart principle, which states that
  profile quality is intentionally human-judged because formalizing quality into
  a score incentivizes optimizing for the metric rather than actual rigor.
  Behavior assertions are being relocated to capability sibling scenarios in a
  follow-up change where rubric-based scoring is the intended evaluation mode.

## [1.0.0-rc1] — 2026-04-11

First release candidate. Feature-complete and validated through end-to-end
evaluation of a real AI infrastructure agent (Joe) against the Software
Infrastructure profile (SI v0.2). 21 safety scenarios produced deterministic
PASS/FAIL verdicts with zero missing-heuristic errors and zero provision
failures.

### Added (cumulative since v0.4.0)

- **End-to-end validation** of the full evaluation pipeline against a real agent
- **SI profile promoted to v0.2.0-rc1** with 7 safety categories, 21 safety
  archetypes, and 107 behavior definitions validated in production

### Changed

- Core spec version bumped from 0.4.0-draft to 1.0.0-rc1
- SI profile version bumped from 0.2.0-draft to 0.2.0-rc1
- SI core dependency updated from >=0.4.0 to >=1.0.0-rc1
- README updated with release candidate status notice

## [0.4.0] — 2025

### Added

- **Provider conformance preflight mechanism** (`GET /v1/conformance`) enabling
  runners to verify provider capabilities before any scenarios execute
- **PROVIDER_FAILURE verdict category** for runtime provider faults that prevent
  independent verification — distinct from configuration gaps caught at preflight
- **Action-first evaluation principle** (Core §3.5.1): safety assertion verdicts
  are determined by what the agent did, not what it said
- **Evidence provenance** via `evidence_source` field on observation responses
- **Implementation determinism requirement** (Core §3.5.4): evaluator
  implementations must be pure functions of their inputs — same evidence in,
  same verdict out, every time
- **Canonical verdict status enumeration** (Core §3.6): PASS, FAIL,
  PROVIDER_FAILURE as the exhaustive set; NEEDS_REVIEW and INCONCLUSIVE
  explicitly forbidden
- **Machine-readable provider conformance requirements** for SI profile
  (`provider-conformance-requirements.yaml`)

### Changed

- Replaced NEEDS_REVIEW with explicit "evaluator implementation is incomplete"
  errors — missing heuristics are bugs, not verdict statuses
- Provider Conformance §6.3 disambiguated: cross-provider reproducibility
  non-guarantee applies only to live runs, not to replayed evidence

## [0.3.0] — 2025

### Added

- **Agent configuration schema** with scenario applicability filters — scenarios
  can declare which agent configurations they apply to
- **Adversarial verification extension** (spec/07): non-deterministic probes,
  reserved scenarios, and adversarial robustness testing as an optional extension
- **Provider conformance spec** (spec/08): what makes an evaluation provider
  conformant to a domain profile
- **Behavior definitions registry** for SI profile: formal definitions and
  verification methods for every named behavior used in assertions
- **Intent and subcategory fields** on scenarios for finer-grained grouping
  and reporting
- **Profile subcategories** within safety categories
- **Stimulus library** with reusable parameterized stimuli for SI profile
- **Provider implementation guide** for SI profile with Kubernetes-level
  operation details

## [0.2.0] — 2025

### Added

- **Initial Software Infrastructure profile** with 7 safety categories (5 core
  + 2 domain-specific) and 21 archetypes (3 per category)
- **7 capability categories** with 29 archetypes
- **Core spec structure:** scenarios, profiles, execution, reporting, principles
- **Complexity tiers** (Minimal, Integrated, Production-realistic)
- **Safety tolerance parameter** with zero-tolerance default
- **Independent verification mandate** — evaluator never trusts agent
  self-reporting
- **Interface type system** for tool-agnostic scenario definitions

## [0.1.0] — 2025

### Added

- Initial spec draft: core definitions, architecture, safety gate model,
  capability model
