# Software Infrastructure — Capability Scoring Decomposition

**Profile version:** 0.3.0-rc1

This document defines how SI capability rubric bands are decomposed into deterministic predicates over recorded evidence. It is the bridge between the percentage bands stated as prose in [Capability Categories](capability-categories.md) and an evaluator that is a pure function of the transcript.

[Core §3.5.4](/docs/v1.0/spec/core/) requires profile authors who want graded capability criteria to decompose them "into deterministically scorable questions grounded in tool call order, system state, or structured output fields." Criteria that cannot be decomposed belong in the Adversarial Verification Extension, not in the core scenario corpus. This document is the SI profile's discharge of that requirement.

Nothing here introduces a judge, a heuristic, or a human-review path. Every band in this document resolves to a verdict from the recorded evidence alone.

---

## 1. Three-layer architecture

Scoring is split across three layers. The precedent is the core value-containment mechanism ([Core §3.5.5](/docs/v1.0/spec/core/)): scenarios declare value references, and the evaluator runs one generic deterministic matcher. This design generalizes that pattern.

| Layer | Owner | Contents |
|---|---|---|
| 1 | Profile | Scoring-primitive registry — reusable, parameterized, deterministic predicate types |
| 2 | Archetype | Band templates — each archetype's percentage bands expressed as a decision table over primitive roles |
| 3 | Scenario | Answer-key bindings — the scenario YAML binds the template's roles to concrete values |

### 1.1 Layer 1 — profile primitive registry

A small library of predicate types. Each primitive is a pure function of captured evidence (agent output channels, tool call records, audit log, state snapshots) and its declared parameters. The registry is defined in §2. An evaluation implementation implements the registry once; scenarios are data.

Identifier matching is structural — resource names, keys, component identifiers — never vocabulary or phrasing. This respects the anti-vocabulary-coupling guidance for action assertions ([Core §3.5.3](/docs/v1.0/spec/core/)) while staying a pure function of transcript plus declared identifier set. Diagnosis output is a detection-type assertion ([Core §3.5.2](/docs/v1.0/spec/core/)), and [Core §3.5.4](/docs/v1.0/spec/core/) explicitly blesses "pattern matching against fixed expected-phrase lists" for this class.

### 1.2 Layer 2 — archetype band templates

Each archetype expresses its existing percentage bands as a decision table over primitive roles. Percentages and table shape live at the archetype, not the scenario.

This is the layer that keeps scores comparable across scenarios of the same archetype. Without it, every scenario's table is free-standing and the bespoke-heuristic problem returns in data form. Templates are defined in §5.

### 1.3 Layer 3 — scenario answer-key bindings

Each scenario YAML carries a `scoring` block that binds the archetype template's roles to concrete values: ground-truth identifier sets, decoy sets, expected operation sequences, step budgets. This is Form B of the capability scoring block, defined in [Scenarios §1.7](/docs/v1.0/spec/scenarios/); the core spec defines the shape of the binding, and this document defines the role vocabulary each template expects. A Form B scenario declares no `scoring.type`, no `scoring.rubric`, and no per-scenario `dimensions` block.

### 1.4 Escape valve

Scenario-local composite predicates are permitted where an archetype template genuinely does not fit. They MUST still be declared decision tables over registry primitives. Free-form evaluator code is never an acceptable form for a scenario-local composite — a scenario that cannot be expressed as a decision table over registry primitives is not conformant under this document, and the correct response is to extend the registry (§2.4) or to relocate the criterion to the Adversarial Verification Extension.

---

## 2. Primitive registry v1

Registry v1 contains exactly the primitives required by the archetype templates defined in §5. It is deliberately minimal and explicitly extensible (§2.4).

Every primitive is evaluated against the evidence artifact persisted for the scenario ([Reporting §1.2](/docs/v1.0/spec/reporting/)) and MUST return the same verdict on every evaluation of the same artifact.

### 2.1 `named_in_output(identifier_set, channels, exclude_tool_echo)`

**Evidence source:** agent output channels.

**Question:** does the agent's output name a member of `identifier_set`?

**Parameters:**

- `identifier_set` (array of strings, required) — the identifiers to match.
- `channels` (array of strings, required) — the output channels to search. Channel identifiers are those defined by this profile; diagnosis bands use `agent_response` only (§2.5).
- `exclude_tool_echo` (boolean, optional, default `false`) — when `true`, answer lines that appear verbatim in captured tool response bodies are excluded from match scope before matching.

**Result:** true if at least one member of `identifier_set` appears in the in-scope text of any listed channel, under the identifier matching rule in §3.1. Membership, not conjunction — a predicate requiring every member of a set to appear is expressed as a conjunction of `named_in_output` calls over singleton sets, as `factor_identified` does (§2.2).

**`exclude_tool_echo` semantics.** The evaluator partitions the agent's output into lines. A line that appears verbatim in the body of any tool response captured for this scenario is removed from match scope. Matching then runs over the remaining lines only. This kills quotation gaming — an agent that dumps a deployment manifest containing `SMTP_PORT` has not named `SMTP_PORT` as a finding. The predicate remains a pure function of the transcript plus the tool call records, both of which are in the evidence artifact.

### 2.2 `factor_identified(factor, channels)`

**Evidence source:** agent output channels, plus the scenario's injection manifest for the factor's `deviation_type`.

**Question:** did the agent name a contributing factor *and* characterize the deviation correctly?

**Definition.** `factor_identified(f)` is true iff both hold:

1. **Identifier conjunction** — every member of `f.required_identifiers` satisfies `named_in_output([identifier], channels, exclude_tool_echo)`.
2. **Deviation-type co-occurrence** — a member of the synonym list for `f.deviation_type` (§3.3) occurs in the same sentence as one of `f.required_identifiers`, under the sentence splitting rule in §3.2.

`f.ref` binds the factor to a deviation `id` in the scenario's `injection.deviations` block ([Scenarios §1.11](/docs/v1.0/spec/scenarios/)); the `deviation_type` is read from that deviation, not restated in the scoring block.

The sentence-scoped co-occurrence window is the weakest deterministic link in this design. It is deterministic and reproducible, but its adequacy has been validated only against the C-DA-001 template. It is flagged for stress under the C-DA-002 and C-DA-003 decompositions.

### 2.3 `within_step_budget(n)`

**Evidence source:** tool call records in the transcript.

**Question:** did the agent conclude within `n` steps?

**Step definition:** one step is one tool invocation recorded in the transcript (§3.4).

**Result:** true if the agent produced a final answer and the count of recorded tool invocations up to and including that answer is less than or equal to `n`. An agent that exhausts the budget without producing a final answer yields false.

### 2.4 Extensibility

The registry is expected to grow as further archetypes are decomposed. Additional primitive types are added here, not in scenario files. Candidates identified during design but not required by any landed template — and therefore **not** part of registry v1 — include `ops_against`, `ops_precede`, `sequence_matches`, `state_delta_matches`, and `trail_matches_oracle`. They are recorded so that scenario authors do not invent divergent local versions; a scenario MUST NOT reference a primitive that is not registered here.

### 2.5 Channel restriction for diagnosis bands

Diagnosis bands match `agent_response` only, never the reasoning trace. Diagnosis is a communicative act; matching the trace would credit an agent that considered the correct answer and discarded it. Templates that score diagnosis declare `channels: [agent_response]` and scenario bindings MUST NOT widen it.

---

## 3. Ratified scoring constants

These constants are normative for every primitive in this document. They exist so that two conformant evaluators produce identical verdicts from identical evidence.

### 3.1 Identifier matching

Case-insensitive, NFC-normalized, maximal-token. The identifier MUST appear delimited by characters outside the class `[A-Za-z0-9_-]` — that is, the character immediately before and the character immediately after the match must fall outside that class, with start-of-text and end-of-text counting as delimiters.

Consequence: `smtp_port` matches `SMTP_PORT`; `SMTP_PORT_OLD` does not match `SMTP_PORT`.

### 3.2 Sentence splitting

Split on newline, and on `.`, `!`, or `?` followed by whitespace or end of text. Nothing cleverer. Abbreviations, decimal points, and version strings are deliberately not special-cased; echo exclusion (§2.1) handles dumped output, which is where such text overwhelmingly originates.

### 3.3 `absent_key` synonym list v1

Profile-maintained, case-insensitive, matched as literal phrases:

`missing`, `absent`, `not present`, `not set`, `not defined`, `undefined`, `not found`, `does not exist`, `doesn't exist`, `no key`, `lacks`, `lacking`, `omitted`, `unset`.

Synonym lists are a Layer-1 profile asset. They are maintained centrally in this document, never as per-scenario ad hoc lists. A scenario MUST NOT declare its own synonyms.

### 3.4 Step budget semantics

A step is one tool invocation recorded in the transcript. Not one reasoning turn, not one message, not one agent-internal planning step. Tool invocations are the unit because they are the unit the evidence artifact records unambiguously.

Budgets are declared per scenario, not per archetype. The budget ratified for `infra.capability.da.single-signal-diagnosis-001` is 15.

---

## 4. Efficiency stance

Three distinct things, with different conformance and comparability properties. Conflating them is how efficiency pressure corrupts a capability score.

1. **Path shape — legitimate scoring input.** Some bands are path predicates in disguise: "identified then self-corrected" is "the agent issued operations against the declared decoy before the ground truth," checkable over the audit log. Ordering relative to declared decoys and checkpoints is a legitimate band input.

2. **Step budget — a legitimate ceiling gate.** Every scenario already has an implicit budget in the form of loop limits and timeouts. A scenario-declared `step_budget`, gated by `within_step_budget`, makes that budget honest: an agent that exhausts it without concluding lands in the bottom band by construction.

3. **Raw step counts and wall-clock time — never a scoring input.** They are reported as **non-scoring metadata** per scenario in the verdict (steps, duration, tokens). Organizations set their own thresholds over that metadata, exactly as they set their own capability thresholds ([Profile §9.2](profile.md)). The exclusion has three reasons:
   - Wall-clock measures LLM latency and provider load, not capability, and breaks run-to-run comparability against the determinism mandate ([Core §3.5.4](/docs/v1.0/spec/core/)).
   - Step counts are incomparable across agents with different tool granularity, and this profile is deliberately tool-agnostic via abstract [interface types](interface-types.md).
   - Grading step economy Goodharts directly against what the rubrics reward: C-OE-002 pays for reading state first and C-OE-003 pays for verification steps, and efficiency pressure punishes exactly those behaviors.

---

## 5. Archetype band templates

### 5.1 C-DA-001 — Single-Signal Diagnosis

**Prose bands being decomposed** ([Capability Categories §C-DA-001](capability-categories.md)): correct root cause = 100%; correct subsystem, wrong component = 50%; incorrect = 0%.

**Roles the scenario binding must supply:**

| Role | Type | Meaning |
|---|---|---|
| `factor` | object | `ref` (binds to an `injection.deviations` entry) plus `required_identifiers` (array of strings) |
| `subsystem_set` | array of strings | Identifiers that place the answer in the correct subsystem without naming the factor |
| `channels` | array of strings | Output channels to match; `[agent_response]` per §2.5 |
| `step_budget` | integer | Ceiling in tool invocations, per §3.4 |
| `exclude_tool_echo` | boolean | Passed through to `named_in_output`; expected `true` |

**Decision table — first match wins:**

| # | Predicate | Score |
|---|---|---|
| 1 | ¬ `within_step_budget(step_budget)` — no final answer within budget | 0.0 |
| 2 | `factor_identified(factor, channels)` | 1.0 |
| 3 | `named_in_output(subsystem_set, channels, exclude_tool_echo)` | 0.5 |
| 4 | else | 0.0 |

Row 1 precedes row 2 deliberately: an agent that never concluded has not diagnosed anything, regardless of what its intermediate output contains.

**Emergent grading behavior.** An agent that says "something's wrong with the SMTP_PORT config" — right locus, no deviation characterization — fails row 2 for want of the deviation-type co-occurrence and lands at 0.5 via row 3. The identifier hierarchy grades partial understanding without any semantic judgment.

**Ground truth is the injection manifest.** The scenario's answer key is derived from the declared delta between the healthy baseline and the injected state ([Scenarios §1.11](/docs/v1.0/spec/scenarios/)), not from prose authorship of "the answer." The counterfactual rule — restoring all deviations clears the declared symptom — is what makes the factor set the contributing-factor set rather than an assertion of the author's opinion.

---

## 6. Scope limit

This mechanism measures recovery of a **constructed** fault set from observable evidence. It does not measure diagnosis of emergent, unknown-unknown failures in real socio-technical systems, which include organizational contributors that no lab injects. Scores produced by these tables require context; see [Principles](/docs/v1.0/spec/principles/).
