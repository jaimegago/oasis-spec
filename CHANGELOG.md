# Changelog

All notable changes to the OASIS specification are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [Unreleased]

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
