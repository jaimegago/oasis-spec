# Changelog

All notable changes to the OASIS specification are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

> Note: tags `v1.0.0-rc1.3` and `v1.0.0-rc1.4` were intentional iteration tags cut without a corresponding CHANGELOG entry. They are not back-filled. The substantive changes from those iterations are reflected in the surrounding entries.

## [Unreleased]

### Added

- **Core spec bumped to `1.0.0-rc1.11`; SI profile bumped to `0.3.0-rc1`.**
  Both bumps carry the capability scoring decomposition described below.
  The core change is additive — no existing scenario form is invalidated —
  so the SI profile's OASIS Core Dependency declarations were advanced to
  `>= 1.0.0-rc1.11` per RELEASING.md §1.

  The embedded core string skips `rc1.8` through `rc1.10`: those are
  released tags whose embedded content predates these changes — each was
  cut while the embedded string still read `1.0.0-rc1.7`, the drift
  RELEASING.md was written to stop. The first tag that can carry this
  content is `rc1.11`, and the embedded string must identify its tag.

- **Capability scoring decomposition** (`profiles/software-infrastructure/scoring-decomposition.md`,
  new). SI capability rubric bands were prose judgments ("correct
  subsystem, wrong component = 50%") with no bridge to observable
  evidence, which left evaluators nothing conformant to implement and
  produced "evaluator does not implement heuristic" capability failures
  in end-to-end runs. Core §3.5.4 requires such criteria to be decomposed
  into deterministically scorable questions; this document is that
  decomposition. It defines a three-layer architecture (profile primitive
  registry, archetype band templates, scenario answer-key bindings), the
  primitive registry v1 (`named_in_output`, `factor_identified`,
  `within_step_budget`), the ratified matching constants (identifier
  matching, sentence splitting, the `absent_key` synonym list, step budget
  semantics), the efficiency stance, and the C-DA-001 band template as a
  decision table.

- **Injection manifest** (`spec/02-scenarios.md` §1.11, new). A scenario
  block declaring the healthy baseline, the injected deviations, the
  resulting symptom, and the counterfactual acceptance statement. It
  exists so a scenario's diagnostic ground truth is derived from the
  declared delta rather than authored as prose. The counterfactual —
  restoring all deviations clears the symptom — is a MUST at scenario
  authoring time and a SHOULD at provider preflight.

- **Starting-state contract** (`spec/04-execution.md` §1.1, new). Normative
  requirements extending the existing stateless-between-scenarios rule:
  the adapter MUST deliver the stimulus value unmodified as the sole task
  input, with any fixed wrapper text constant across the run and disclosed
  in the report as part of agent identity; each scenario execution MUST run
  against a fresh agent session by default, with seeding only via declared
  `conversation_context` stimuli; no environmental facts beyond the
  stimulus may be disclosed, everything else being reachable only through
  tools.

- **Agent identity declaration and evidence artifacts**
  (`spec/05-reporting.md` §1.2, new). The report header MUST carry a
  run-wide agent identity declaration (binary version and SHA, config hash,
  system-prompt hash, declared model and provider, disclosed wrapper text);
  each scenario record MUST carry the observed model, and an observed model
  differing from the declared one on any scenario REQUIRES a heterogeneity
  flag on the run. The evaluator MUST persist, per scenario, an
  `evidence-<scenario-id>.json` artifact sufficient to replay the
  evaluation as a pure function, referenced from the scenario record by
  relative path.

- **Machine-readable category data for the SI profile**
  (`profiles/software-infrastructure/capability-categories.md` and
  `safety-categories.md`, new sections). Both documents gain a fenced YAML
  block restating what their prose — and, for the core dimension weights,
  the table in `profile.md` §9.2 — already says: category identifiers and
  names, archetype identifiers, core dimension mapping and contribution
  weights, and the per-category aggregation method. The categories, the
  archetypes, the weights and the aggregation methods are unchanged; only
  their form is new. Until now the scoring model existed for the profile
  only as prose, so an evaluation implementation had nothing to read and
  reported empty category and core dimension scores while
  `spec/05-reporting.md` §1 required them populated.

  The block records `maps_to_dimensions` and `dimension_weights`
  separately rather than merging them, because the two prose surfaces do
  not agree: Observability Interpretation declares a **Reasoning** mapping
  in `capability-categories.md` §3 and in the `profile.md` §6 table, but
  carries no Reasoning weight in the `profile.md` §9.2 aggregation table.
  Both facts are encoded as stated. Resolving the discrepancy would change
  a reported dimension score and is left to a profile change of its own.

  No profile version bump: `0.3.0-rc1` is unreleased — the bump to it sits
  in this same `[Unreleased]` section — and under `spec/03-profiles.md` §5
  an addition that changes no category, archetype, weight or aggregation
  method is a clarification rather than a structural change.

### Changed

- **Capability scoring block admits two forms** (`spec/02-scenarios.md`
  §1.7). The existing rubric-plus-dimensions form (Form A) is joined by a
  scoring binding form (Form B): `archetype_template` plus binding
  parameters defined by the owning profile's scoring-decomposition
  document. A scenario uses exactly one form. Form B delegates band
  semantics entirely to the profile and carries no per-scenario
  `dimensions` block, which conflicted with the profile-level category-to-
  dimension mapping anyway. Form A remains valid and is expected to be
  superseded in profiles that adopt scoring decomposition. §1.5 was
  amended in consequence: a Form B scoring binding now satisfies the
  at-least-one-verifiable-concern rule, alongside `assertions.must`,
  `assertions.must_not`, and `verification.value_containment`.

- **SI scenario `infra.capability.da.single-signal-diagnosis-001`**
  migrated to the scoring binding form — the first and, in this release,
  only scenario to carry one. It gains an `injection` block declaring the
  `absent_key` deviation on `configmap/smtp-config`; its `assertions` and
  rubric `scoring` blocks are replaced by a binding to the C-DA-001 band
  template (step budget 15, `agent_response` channel only,
  `exclude_tool_echo` on); its per-scenario `dimensions` block is removed.
  Preconditions, stimuli, verification, and observability requirements are
  unchanged. Every other scenario is untouched and remains on the rubric
  form.

- **Behaviors `identify_root_cause` and `reference_missing_config_key`**
  (`profiles/software-infrastructure/behavior-definitions.md`) are noted as
  superseded for scenarios carrying a scoring binding, where the archetype
  decision table is the evaluation. Both remain defined for scenarios that
  have not migrated. `diagnose_correctly` inherits the note through its
  existing equivalence to `identify_root_cause`.

### Removed

- **Design Decisions document** (`docs/decisions.md`). The corresponding
  page was removed from oasis-spec.dev; the source markdown had no
  remaining references in the repo.
- **Historical Claude Code prompts** (`docs/prompts/*.md`). One-shot
  prompts used to drive past spec changes; not part of the published
  spec. The now-empty `docs/` directory was removed too.

### Fixed

- **Broken internal link** in
  `profiles/software-infrastructure/provider-conformance.md` §3.5 —
  the link to `data-exfiltration-prevention.yaml` resolved outside
  the repo (off-by-one in the relative path).
- **Wrong GitHub owner** in `RELEASING.md` §4 — the `oasis-website`
  pointer named `anthropics/` rather than `jaimegago/`.
- **SI scenario `infra.capability.au.provenance-tracking-001`**:
  pinned the `deployment/api-service` precondition to a real image
  (`registry.k8s.io/nginx-slim:0.27`). The scenario previously left
  the image unspecified, which the provider resolved to a fictional
  `docker.io/library/api-service:v1.2.3` and the pod entered
  `ImagePullBackOff` at provisioning. No semantic change to the
  scenario.
- **SI scenarios `infra.capability.au.reasoning-transparency-001`
  and `infra.capability.oe.stateful-scaling-001`**: replaced
  kind-only `resource: nodes` state entries with explicit
  `node/worker-N` entries (one per node). The kind-only form is
  rejected by providers because state-entry `resource` fields must
  be in `kind/name` format. The homogeneous-cluster intent is
  preserved by enumerating three identically-shaped node entries.
- **SI provider-guide §1.6**: corrected the documented "configure
  node resources" scenario pattern, which previously showed the
  kind-only `resource: nodes` form and was the source of the two
  scenario authoring bugs above.

### Changed

- **SI provider-conformance.md version strings reconciled.** Three
  normative-text references and the §5 worked-example JSON responses
  still declared `1.0.0-rc1` / `0.2.0-rc1`, which no longer satisfy
  the document header (`>=1.0.0-rc1.7`) or the matching
  `provider-conformance-requirements.yaml` constraint. Bumped to
  `1.0.0-rc1.7` / `0.2.0-rc3` so the illustrative examples are
  internally consistent with the contract they illustrate. No
  semantic change to the contract itself.
- **README** now links to the rendered specification at
  [oasis-spec.dev](https://oasis-spec.dev) in the opening paragraph.

### Added

- **GitHub issue and PR templates** under `.github/`. Two issue
  templates (`spec-feedback`, `profile-proposal`) and a PR template
  asking contributors to classify changes as normative or editorial
  and to link the motivating discussion for normative changes.
- **Authoring pitfalls section** in `guides/profile-authoring.md`
  documenting the `kind/name` resource requirement and the need to
  pin every workload to a real, publicly pullable image.

## [1.0.0-rc1.7] — 2026-05-05

### Changed

- **Scenario and suite `version` fields are now optional** (Scenarios §1.1, §3).
  When omitted, scenarios and suites inherit the version of their parent
  profile, which becomes the single source of truth for scenario versioning.
  Previously the schema declared both fields as required. This is a normative
  spec relaxation; profiles that already declare per-scenario versions remain
  valid.
- **SI profile scenarios updated to inherit profile version.** All 14 SI
  scenario YAMLs (7 capability, 7 safety) had their file-level and per-stanza
  `version:` fields removed. SI scenarios now derive their version from the
  parent profile's `0.2.0-rc3` declaration.
- **Embedded version-string reconciliation.** The 9 core spec documents now
  declare `**Version:** 1.0.0-rc1.7`. The 7 SI profile markdown files that
  were stuck at `0.2.0-rc1` now match the canonical `0.2.0-rc3`. All four
  OASIS Core Dependency declarations across the SI profile (profile.md,
  README.md, provider-conformance.md, provider-conformance-requirements.yaml)
  now read `≥ 1.0.0-rc1.7`. README status line updated accordingly.

Supersedes the un-CHANGELOGed v1.0.0-rc1.6 tag.

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

First release candidate. Structurally complete and validated through end-to-end
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
