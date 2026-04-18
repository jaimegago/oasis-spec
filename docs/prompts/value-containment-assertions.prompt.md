# Prompt: Add value containment assertions as a new verification primitive

Generated: 2026-04-18
Model: claude-opus-4-7 (1M context)
Target:
- spec/01-core.md (§3.5.5 added)
- spec/02-scenarios.md (§1.6 extended)
- spec/08-provider-conformance.md (§3.2 extended)
- spec/00-motivation.md, spec/03-profiles.md, spec/04-execution.md, spec/05-reporting.md, spec/06-principles.md, spec/07-adversarial-verification.md (Version header bumps only)
- CHANGELOG.md ([Unreleased] promoted)

## Specification

You are modifying the OASIS specification to add a new verification primitive called **value containment constraints**.

### Context

OASIS evaluates AI agents operating on external systems. Safety scenarios verify that agents respect boundaries. The spec currently supports three assertion verification mechanisms in Core §3.5:

1. Action-based assertions (§3.5.1) — verified via system state, audit logs, tool call records.
2. Detection-based assertions (§3.5.2) — verified via natural language analysis of agent output.
3. Implementation determinism (§3.5.4) — all verification must be deterministic.

There is a gap: scenarios where the safety threat is the agent leaking a value through its output channel cannot be verified by system state or audit logs alone, because the harmful action IS the output. The previous workaround was lexical behavior assertions on agent prose, which were removed because they violated the Goodhart principle (Principles §3.27). Five SI profile scenarios now have empty assertions blocks because of this gap.

Value containment fills this gap. It is **not** about sensitive data specifically — it is about values that are bound to a context boundary and must not cross the output channel when the agent operates outside that context.

### Changes required

**1. Core (spec/01-core.md) — new subsection §3.5.5 "Value containment assertions"**, placed immediately after §3.5.4 (Implementation determinism) and before §3.6. The section must define, in normative prose consistent with the surrounding spec:

- A value containment constraint is a declaration that a specific value, present in the scenario's preconditions, MUST NOT appear in any agent output channel during the scenario.
- Two modes:
  - **Absolute containment** — the value must never appear in any agent output regardless of scope.
  - **Contextual containment** — the value must not appear in agent output when the agent is operating outside the value's declared boundary context (a scope, zone, namespace, tenant, or other domain-defined boundary).
- Absolute containment is the degenerate case of contextual containment where the restricted scope is "all scopes." Both modes share a single verification mechanism; mode only gates whether the scope check precedes the substring match.
- The evaluation provider registers containment values during scenario setup by resolving declared value references against the provisioned environment state. Because the provider is a test environment (not production), the registered values are synthetic and the evaluator may hold them in plaintext for output-channel matching. Plaintext handling carries no confidentiality implication.
- Verification is deterministic substring matching of the registered literal value against captured output-channel contents. No semantic interpretation, paraphrase detection, or vocabulary matching. This satisfies §3.5.4 implementation determinism.
- Value containment satisfies the independent-verification mandate (§3.4) because the evaluator performs the match directly against captured output, not against agent self-report.
- Value containment assertions are a form of action-based assertion per §3.5.1 — the "action" being verified is whether the agent emitted the bounded value through any declared output channel during the scenario window.
- Containment constraints are domain-agnostic. Any domain profile may use them for any scenario where the threat model involves data crossing an output boundary (credentials, PII, cross-tenant data, compliance-regulated values, zone-restricted data). The core spec does not enumerate qualifying value classes; the profile's threat model determines when containment is appropriate.

Do not renumber any existing sections. Do not modify the substance of §3.5.1–§3.5.4.

**2. Scenarios (spec/02-scenarios.md) — extend §1.6 verification schema** with one new optional verification method. Keep all existing methods unchanged. Add:

- `verification.value_containment` (array, optional). Each entry declares a value from the scenario's preconditions that must not appear in agent output. Fields per entry:
  - `value_ref` (string, required) — a reference path to the value inside `preconditions.environment.state`, e.g. `secret/db-credentials.data.DB_PASSWORD`. The provider resolves this during setup and registers the resolved literal.
  - `scope` (string, required) — either the literal `absolute` or a boundary reference drawn from `preconditions.agent.scope`. `absolute` means the value must never appear in any output. A boundary reference means the value must not appear when the agent operates outside that boundary.
  - `output_channels` (array of strings, required) — which channels the evaluator searches. Channel identifiers are domain-profile-defined. Mention `agent_response`, `tool_call_parameters`, and `reasoning_trace` as common examples.

Cross-reference Core §3.5.5 from the new schema entry.

Do not add example YAML scenarios — examples belong in profile files.

**3. Provider Conformance (spec/08-provider-conformance.md) — extend §3.2 (Independent verification)** with one additional bullet for value containment support:

- When a scenario includes `verification.value_containment`, the provider MUST resolve each declared `value_ref` against the provisioned environment state during setup and supply the resolved literal value to the evaluator for substring matching. The provider MUST also capture the agent's output across every channel listed in `output_channels` for the scenario window and make each captured channel available to the evaluator.
- This is a distinct evidence-source type. Profiles that use value containment declare the required channel identifiers in `provider-conformance-requirements.yaml`, and the preflight check verifies the provider can capture them.
- Plaintext handling of registered values is scoped to the test environment per Core §3.5.5; the provider MUST NOT persist registered values beyond the scenario window.

**4. Version header bumps.** Bump the `**Version:**` header in every spec document (spec/00 through spec/08) from `1.0.0-rc1` to `1.0.0-rc1.2`. Do not touch any other content in the files that only require a version bump.

**5. CHANGELOG.md.** Promote the existing `[Unreleased]` section to `[1.0.0-rc1.2] — 2026-04-18`. Keep the existing `### Changed` entry about behavior-assertion removal verbatim (the five-scenario backfill is a separate profile commit). Prepend a new `### Added` section above `### Changed` with three bullets:

- Value containment assertions (Core §3.5.5): a new verification primitive for scenarios where the safety threat is data crossing an output boundary. Supports absolute and contextual containment modes. Deterministic substring matching, domain-agnostic, satisfies independent verification mandate.
- Value containment verification method (Scenarios §1.6): `verification.value_containment` schema for declaring containment constraints in scenarios.
- Provider capability for value containment (Provider Conformance §3.2): providers must capture containment values and agent output channels when scenarios declare `value_containment` verification.

### Constraints

- Do not modify any profile files. Profile-level adoption is a separate commit.
- Do not add example YAML scenarios to the spec.
- Writing style must match existing spec prose: precise, normative (MUST/SHOULD/MAY per RFC 2119 usage already established in the spec), no marketing language, no emojis.
- Do not renumber existing sections. §3.5.5 is a new subsection under existing §3.5.
- Do not modify the substance of any existing section beyond the version header bumps and the three additions above.
- Cross-references between the new additions must be wired consistently:
  - Scenarios §1.6 `value_containment` entry → Core §3.5.5.
  - Provider Conformance §3.2 new bullet → Scenarios §1.6 and Core §3.5.5.
  - Core §3.5.5 → §3.4 (independent verification), §3.5.1 (action-based classification), §3.5.4 (implementation determinism).

### Acceptance criteria

- Core §3.5.5 is present, placed between §3.5.4 and §3.6, and defines both containment modes, the setup/plaintext story, substring-matching determinism, §3.4 alignment, §3.5.1 classification, and domain-agnostic scope.
- Scenarios §1.6 lists `verification.value_containment` alongside the existing four verification methods with the three required fields fully described.
- Provider Conformance §3.2 contains a new bullet covering value registration, channel capture, profile-declared identifiers, and plaintext-handling scope.
- All nine spec documents carry `**Version:** 1.0.0-rc1.2`.
- CHANGELOG.md has `[1.0.0-rc1.2] — 2026-04-18` with the three new `### Added` bullets and the original `### Changed` entry preserved. No `[Unreleased]` heading remains.
