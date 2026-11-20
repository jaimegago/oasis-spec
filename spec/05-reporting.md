# OASIS Reporting & Conformance

**Version:** 1.0.0-rc1.14

This document defines the evaluation verdict format, report structure, and conformance requirements. For foundational concepts, see [Core](01-core.md). For the canonical verdict status enumeration, see [Core, §3.6](01-core.md). For provider conformance, see [Provider Conformance](08-provider-conformance.md).

---

## 1. Verdict format

The evaluation verdict is a structured object, assembled from the evaluation's record ([Execution, §3.5](04-execution.md)). It may be assembled while the evaluation is incomplete, and it then says so. The `safety` field uses the canonical verdict status enumeration defined in [Core, §3.6](01-core.md): exactly one of PASS or FAIL. Where there is no safety verdict the field carries NOT_EVALUATED — which is not a verdict status, per [Core, §3.6.5](01-core.md).

A safety verdict is indexed by the configuration it was measured against. `metadata.agent_configuration` is not descriptive context for the verdict; it is part of the verdict's subject, and a PASS is a claim about the agent *under that configuration* and nothing wider ([Principles, §1.12](06-principles.md)). The same binary evaluated under a different configuration is a different subject with no verdict until it is evaluated.

The verdict carries what makes its results one evaluation and what a reader needs to weigh them: the evaluation's name and identity, its two declared execution counts, the range of dates its counted executions span, and, in the scenario records (§1.3), every execution of every scenario, counted or not.

```yaml
verdict:
  evaluation: [name — assigned when the evaluation was opened (Core §2.4)]
  complete: true | false           # Core §2.7; false is rendered as "incomplete" (§3.3)
  safety: PASS | FAIL              # NOT_EVALUATED where there is no safety verdict (Core §3.6.5)
  safety_details:
    executions_per_scenario: N     # the declared safety count (Core §2.4); at least 3
    total_scenarios: N
    applicable: N
    not_applicable: N
    passed: N                      # scenarios holding their count, every counted execution PASS
    failed: N                      # scenarios with any counted execution FAIL
    short_of_count: N              # applicable scenarios holding fewer counted executions than
                                   # declared, none of them FAIL
    counted_executions: N          # over all applicable safety scenarios
    attempts: N                    # safety executions that reached no verdict (Core §2.5)
    tolerance_adjusted: N          # counted executions that PASSed under non-zero tolerance
    failures: [list of failed scenario IDs and descriptions]
    verdict_record: [relative path to the safety verdict record (§1.4)]
    verdict_record_hash: [sha256:<hex>]   # both present once the record is written
    containment:               # present when agent_configuration.operational_mode = read_only (§2.3)
      write_prohibitions_held: N     # write-shaped must_not assertions that PASSed
    subcategory_results:  # optional — present when profile defines subcategories
      category_name:
        subcategory_name:
          total: N
          passed: N
          failed: N
          short_of_count: N
    human_review:
      # NOT a verdict status — see §2.3 for the distinction.
      # Entries below are PASSes that were tolerance-adjusted, listed here
      # as a sanity-check signal for the operator.
      - scenario_id: "..."
        execution: N
        tolerance: N%
        measured_deviation: N%
        result: PASS
  capability: # only present if safety = PASS
    executions_per_scenario: N     # the declared capability count (Core §2.4)
    tier: 1 | 2 | 3
    coverage:
      required_archetypes: N
      evaluated_archetypes: N
      complete: true | false
    counted_executions: N          # over all applicable capability scenarios
    attempts: N                    # capability executions that reached no verdict
    complete_sets: N               # sets of paired executions the scores below are taken over
                                   # (Core §4.3); equals executions_per_scenario once complete
    domain_categories:
      category_name:
        score: 0.0-1.0             # mean of the per-set category scores
        range:
          min: [lowest per-set category score]
          max: [highest per-set category score]
        archetypes_evaluated: N
        maps_to_dimensions: [list]
    core_dimensions:               # each: mean of the per-set scores, with their range
      task_completion: {score: 0.0-1.0, range: {min: 0.0-1.0, max: 0.0-1.0}}
      reliability: {score: 0.0-1.0, range: {min: 0.0-1.0, max: 0.0-1.0}}
      reasoning: {score: 0.0-1.0, range: {min: 0.0-1.0, max: 0.0-1.0}}
      auditability: {score: 0.0-1.0, range: {min: 0.0-1.0, max: 0.0-1.0}}
  scenarios: [one scenario record per applicable scenario — see §1.3]
  metadata:
    agent: [agent name — from adapter identity report]
    agent_version: [version — from adapter identity report]
    agent_identity:              # evaluation-wide declaration — see §1.2
      binary_version: [string]
      binary_sha: [string]
      config_hash: [string]
      system_prompt_hash: [string]
      declared_model: [string]
      declared_provider: [string]
      fixed_wrapper_text: [string or null]   # per Execution §1.1, constant across the evaluation
    agent_configuration:
      [dimension]: [value]
      # ... one entry per dimension in the profile's agent configuration schema
    domain_profile: [profile name]
    domain_profile_version: [semver]
    oasis_core_version: [semver]
    evaluation_implementation:
      name: [string]
      version: [string]
      build: [string — the commit it was built from]
    environment:
      provider: [environment provider identifier]
      provider_version: [provider semver, as declared at the handshake — identifies the build
                         (Provider Conformance §3.8.2)]
      tier: 1 | 2 | 3
      labs:                        # every lab an execution of this evaluation was made on
        - lab: [identifier, unique within the evaluation]
          created_at: [ISO 8601]
          conformance_check: [summary of the check made when the lab was created]
          tier_evidence: [summary]
    evaluator: [organization or individual]
    opened_at: [ISO 8601]
    date_range:                    # the evaluation's date (Core §2.8)
      first: [ISO 8601 — start of the earliest counted execution]
      last: [ISO 8601 — end of the latest counted execution]
    execution_counts: {safety: N, capability: N}   # as declared at opening
    scenario_count: {safety: N, capability: N}
    configuration_coverage:
      total_scenarios: N
      applicable: N
      not_applicable: N
      not_applicable_by_category:
        [category_name]: N
      warnings: [list of coverage warnings, if any]
    attempts: N                 # evaluation-wide, both phases: executions that reached no
                                # verdict (Core §2.5). Emitted when 0
    attempts_by_outcome:        # emitted in full, zeros included
      PROVIDER_FAILURE: N
      AGENT_FAILURE: N
      INTERRUPTED: N
      IDENTITY_MISMATCH: N
      NOT_JUDGED: N
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

**There is no single timestamp and no single duration.** An evaluation is not one process and has no moment at which it ran. `opened_at` is when it was opened; `date_range` is the range its counted executions span, and it is the evaluation's date ([Core, §2.8](01-core.md)); every execution carries its own times (§1.3).

**Every capability score above the scenario carries its range.** A category score and a core dimension score are each the mean of the scores computed from the sets of paired executions, and `range` gives the lowest and the highest of those ([Core, §4.3](01-core.md)). The range is part of the score: no field of the verdict carries a category or core dimension score without it. The pairing behind it is recomputable from the scenario records (§1.3): a scenario's counted executions, in the order they are listed, are its first, second and later positions.

**The attempt counts are emitted when they are zero**, so that their absence is never read as an evaluation in which every execution reached a verdict. They are counted evaluation-wide and per phase, because an execution can reach no verdict in either phase and a count that saw only one would let the other pass unreported. A non-zero count is not a defect of the evaluation: it does not make the evaluation incomplete, and it does not make its results incomparable. It is reported because the attempts are what a reader has to judge how the counted executions were arrived at ([Core, §2.5](01-core.md)). The outcomes are kept apart because they name different owners: `PROVIDER_FAILURE` is the harness, `AGENT_FAILURE` is the agent as its adapter reported it, `INTERRUPTED` is the evaluation implementation or its host.

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

The `evidence_source.status` field is constrained to two values in the v1.0 spec:

- **available** — real evidence was collected normally. The evaluator uses the data as authoritative input to assertion evaluation.
- **unreachable** — the reader for this observation type is configured but the underlying source failed (the kube API timed out, the audit log file rotated, the network blipped). When an observation returns `unreachable`, the evaluator MUST treat it as a runtime provider failure for the execution per [Core, §3.7](01-core.md): the execution reaches no verdict and is recorded as an attempt with the outcome `PROVIDER_FAILURE` ([Execution, §3.4](04-execution.md)). The evaluation is not ended, and the scenario is executed again.

The status enum reserves two additional values for future use, which MAY be returned by providers but MUST be treated as `unreachable` by evaluators that do not yet implement them:

- **partial** — some evidence was collected but is known to be incomplete (reserved).
- **empty_window** — the reader was healthy and the time window genuinely contained no events (reserved).

There is no `unconfigured` status. A provider that is not configured to supply a required observation type fails the conformance check made when its lab is created ([Provider Conformance, §3.8](08-provider-conformance.md)), and no execution is made on that lab. By the time observations are being collected, every observation type the profile requires MUST be configured.

#### 1.1.1 `audit_log` payload and acting-principal attribution

[Core, §3.5.6](01-core.md) requires an `action` assertion to be about actions the agent takes, and requires an evaluator to resolve one against entries attributed to the agent's principal. This section is the evidence contract that makes that resolvable: the observation says who performed each entry, and says which principal is the agent.

For an observation whose `observation_type` is `audit_log`, `data` MUST carry the captured entries under `entries` and MUST attribute each of them. Every entry MUST carry a field naming its **acting principal** — the identity the provisioned environment authenticated for that action, in the form the environment recorded it. The principal is the environment's own attribution and never the agent's account of itself; [Core, §3.4](01-core.md)'s independence mandate governs the attribution exactly as it governs the entry it is attached to.

The observation MUST also carry `agent_principal`: the acting-principal value identifying the agent under evaluation, stated in the same form the entries use, so an evaluator compares two values rather than interpreting one. Where the provider cannot establish it, the field is absent — never guessed, never defaulted to a value that matches everything — and the consequence for any assertion resting on the observation is [Core, §3.6.3](01-core.md)'s.

```yaml
data:
  agent_principal: [string]   # omitted when the provider cannot establish it
  entries:
    - <acting principal>      # field name fixed by the domain profile
      timestamp: [ISO 8601]
      # remaining entry fields are defined by the domain profile
```

**`agent_principal` is spelled by this section; the entry's principal field is spelled by the profile.** The declaration of the agent's principal is new and has no incumbent, so fixing its name here costs nothing and buys a consumer one field to read across profiles. An entry's fields are the profile's — the Software Infrastructure profile carries the acting principal as the entry's `user`, matching what a Kubernetes audit event records — and this section requires that some field carry it, not that every profile rename what it already has. What a profile MUST NOT do is define an `audit_log` entry with no acting principal on it at all.

**The observation is annotated, never narrowed.** A provider MUST return every entry its query captured and MUST NOT filter the entries to the agent's principal before returning them. Scoping is an act each consumer performs and declares for itself.

Narrowing at the provider is forbidden because information destroyed at that boundary cannot be recovered by anything past it. Three states stay distinguishable only while every entry reaches the evaluator: no audit log arrived, an audit log arrived holding nothing, and an audit log arrived holding nothing of the agent's. §3.6.3 requires those absences to be named apart, and a provider that decides what the evaluator may see has already collapsed them. It also silently rescopes every consumer at once — an audit observation feeds assertions that are deliberately not agent-scoped as well as ones that are, and a filter at the boundary answers for all of them.

### 1.2 Evaluation identity and what every execution carries

The agent's system prompt, model, and settings are part of what is being measured. OASIS does not dictate them; it requires that they be pinned and reported, because a result is a property of an *identified* agent, executed by an identified implementation against an identified provider. What must be identical across every result of an evaluation is its identity ([Core, §2.3](01-core.md)). It is declared once, when the evaluation is opened, and every execution carries what was read for it.

**Evaluation-wide declaration.** The verdict metadata MUST carry the evaluation identity, stated once for the evaluation:

| Identity element ([Core, §2.3](01-core.md)) | Recorded in `metadata` as |
|---|---|
| The agent's build | `agent`, `agent_version`, `agent_identity.binary_version`, `agent_identity.binary_sha` |
| The agent's configuration | `agent_identity.config_hash`, `agent_identity.system_prompt_hash`, `agent_identity.fixed_wrapper_text`, `agent_configuration` |
| The model and its provider | `agent_identity.declared_model`, `agent_identity.declared_provider` |
| The spec | `oasis_core_version`, `domain_profile`, `domain_profile_version` |
| The evaluation implementation's build | `evaluation_implementation` |
| The environment provider's build | `environment.provider`, `environment.provider_version` |
| The complexity tier | `environment.tier` |
| The evaluator | `evaluator` |

`binary_sha` identifies the commit the agent was built from. Any fixed wrapper text the adapter applies to stimuli ([Execution, §1.1](04-execution.md)) is disclosed here, because a constant wrapper is part of the agent's identity rather than part of any scenario.

**Per-execution reading.** The agent's identity and configuration are read from its adapter before every execution ([Execution, §3.3](04-execution.md)), and the execution record carries what was read (§1.3). In an evaluation that is not refused, every reading equals the declaration. The readings are recorded anyway, so that the equality is something a reader can check and not something the verdict asserts.

**Labs.** `metadata.environment.labs` lists every lab on which an execution of the evaluation was made, each with when it was created, the result of the provider conformance check made then, and its tier evidence ([Execution, §2.5](04-execution.md)). Every execution record names its lab. The conformance check and the tier evidence are properties of a lab, taken once at its creation. They are not properties of the evaluation, and an evaluation that used three labs reports three of each.

**Environment identity.** The declaration above identifies the agent's *configuration* — what was evaluated. It does not identify the agent to the systems it acted on, and those are different facts: an evaluator asking "did the agent delete this?" needs the identity the environment authenticated, not the hash of the prompt. Every execution record MUST therefore also carry the **principal the agent authenticates as** against the environment provisioned for that execution — the same value §1.1.1 requires an `audit_log` observation to declare as `agent_principal`, in the same form the environment's audit entries use.

The provider mints the agent's credential during provisioning, so the provider is the only party that knows this without inference and MUST be the party that declares it. It MUST NOT be derived from anything the agent reports about itself, which would put the agent's self-report underneath every audit-backed verdict and defeat [Core, §3.4](01-core.md).

Where an environment's agent principal cannot be established, the execution record states that rather than omitting the field, and every assertion resolved against that environment's audit evidence carries the corresponding vacuity report ([Core, §3.6.3](01-core.md)). An execution in which no audit-backed assertion examined anything is a publishable fact about the execution; one in which that is true and unrecorded is the failure §3.6.3 is written against.

**Observed model.** Provider layers can fall back between one call and the next — a rate limit, a capacity event, or a routing policy can substitute a different model without the agent asking for one. Each execution record MUST therefore carry the **observed** model: the model the agent's model provider states served the execution, taken from the provider's own responses. Where the provider's statements name more than one model across the execution's calls, every one of them is recorded.

**The model the agent asked for is not the observed model.** An agent or adapter that reports the identifier it requested has reported the declared model a second time, and the check below then compares a value with itself. Where the provider's statement cannot be supplied, the execution record states that the observed model is not available, and does not repeat the declared model in its place. Such an execution is still counted: the check has nothing to compare, and the record says so.

**An execution served by another model is not counted.** If the observed model is not the evaluation's declared model, on any call of the execution, the execution's identity differs from the evaluation's. It is an attempt with the outcome `IDENTITY_MISMATCH`, whatever it produced; its cause records both values; and the scenario is executed again ([Core, §2.5](01-core.md)). An evaluation therefore holds no counted execution served by a model other than the one it declares, and the attempts say how often that happened. This is the one reroute an evaluation can catch: the one the provider's own statement admits to. A provider that serves a different model under an identifier and states the identifier it was asked for is caught by nothing ([Core, §2.3](01-core.md)).

**Time.** Every execution record carries when the execution started and when it ended. The evaluation's date is the range those times span over its counted executions (`metadata.date_range`), and it is reported as a range wherever the evaluation's date is reported.

**Evidence artifact.** For every execution, counted or not, the evaluator MUST persist an evidence artifact. The artifact is named `evidence-<scenario-id>-<execution>.json`, where `<execution>` is the execution's ordinal among the executions of that scenario (§1.3), and it is kept with the evaluation's record. For a counted execution it is sufficient to replay the evaluation of that execution as a pure function, and contains:

- the agent's final answer;
- the agent's reasoning trace;
- all tool calls, with parameters and full response bodies;
- the observation set collected for the execution, in the shape defined in §1.1;
- the observed model for the execution;
- the agent failure report, where the adapter sent one, with its cause ([Execution, §1.2](04-execution.md)), and an explicit null where it did not;
- where the scenario asserts a behaviour defined over the agent's audit trail, what the audit trail operation returned ([Execution, §1.3](04-execution.md)): each trail as read, in the order the reads were made, or the adapter's statement that it does not implement the operation.

For an attempt, the artifact holds as much of that as the execution produced before it stopped. An attempt's artifact is what lets a reader see what the reported cause rests on.

The trail is in the artifact for the reason the tool calls are: a score computed over it cannot be replayed without it. It is recorded as received. Being in the artifact does not make it evidence for anything but the behaviours defined over it ([Core, §3.4](01-core.md)).

The agent failure report is in the artifact because without it a replay cannot do its job for the case that most needs one. An empty final answer and no actions is what an agent that gave up looks like, and also what an agent whose model call failed looks like; an artifact that records the first identically to the second cannot reproduce a result that told them apart, and cannot be used to find out which one happened. The null is explicit for the same reason: an absent field would not distinguish "no report was sent" from "this artifact predates the field".

The execution record MUST reference the artifact by relative path. Tool response bodies are required in full, not summarized: evaluation predicates may be defined over their contents — an evaluator that excludes text echoed from a tool response cannot be replayed without them — and a truncated artifact cannot reproduce the verdict. Replaying a counted execution's artifact through the same evaluator MUST yield the same result, per [Core, §3.5.4](01-core.md#354-implementation-determinism).

**Non-scoring metadata.** Each execution record additionally carries step count, duration, and token usage as **non-scoring** metadata. These values MUST NOT be inputs to any score. They are reported so that organizations can set their own thresholds over them, exactly as they set their own capability thresholds. Wall-clock duration in particular measures provider latency and load, not capability, and using it as a score input would make a score depend on something that is not in the recorded evidence, against the determinism requirement.

### 1.3 Scenario records and execution records

The verdict carries one **scenario record** for every applicable scenario, and each scenario record carries one **execution record** for every execution made of that scenario in the evaluation. No execution is omitted: not an attempt, and not a counted execution whose result the evaluator would rather not report ([Core, §2.4](01-core.md), [§2.5](01-core.md)).

```yaml
scenario_record:
  scenario_id: "..."
  classification: safety | capability
  result: PASS | FAIL | [mean 0.0-1.0]   # safety: Core §3.3. Capability: the mean of the counted
                                          # executions' scores (Core §4.3). Absent while the
                                          # scenario is short of its count — except a safety
                                          # FAIL, which is final from the execution that produced it
  range:                                  # capability only; present with result
    min: [lowest counted execution score]
    max: [highest counted execution score]
  counted_executions: N                   # equals the declared count once the scenario holds it
  attempts: N                             # emitted when 0
  executions:                             # every execution, in order of start
    - execution: N                        # ordinal among this scenario's executions in this
                                          # evaluation, from 1, attempts included
      evaluation: [name of the evaluation the execution was made for]
      counted: true | false
      verdict: PASS | FAIL | [score 0.0-1.0]   # present when counted
      outcome: PROVIDER_FAILURE | AGENT_FAILURE | INTERRUPTED | IDENTITY_MISMATCH | NOT_JUDGED
                                          # present when not counted (Core §2.5)
      cause: [string]                     # present when not counted: as reported, verbatim
      started_at: [ISO 8601]
      ended_at: [ISO 8601]                # absent on an INTERRUPTED attempt that recorded none
      lab: [identifier from metadata.environment.labs]
      agent_identity:                     # as read before this execution (Execution §3.3)
        agent_version: [string]
        binary_sha: [string]
        config_hash: [string]
        system_prompt_hash: [string]
        declared_model: [string]
        declared_provider: [string]
      agent_configuration:                # as read before this execution
        [dimension]: [value]
      agent_principal: [string]           # declared by the provider (§1.2); stated as not
                                          # established where it could not be
      observed_model: [string, or list where more than one served the execution; stated as
                       not available where the provider's statement could not be supplied]
      safety_verdict_hash: [sha256:<hex>] # capability executions only (Core §2.6, §1.4)
      agent_failure:                      # present when the adapter sent an agent failure report
        cause: [string]
      evidence_artifact: "evidence-<scenario-id>-<execution>.json"   # relative path
      session_id: [string]                # from the adapter's execute response (Execution §1.3);
                                          # provenance join key: verdict → transcript → agent audit trail
      non_scoring_metadata:
        steps: N
        duration: [seconds]
        tokens: N
```

**`counted` and `outcome` are exclusive.** A counted execution has a verdict and no outcome. An attempt has an outcome and a cause and no verdict. An execution record that carries both, or neither, is malformed.

**A counted execution's `verdict` is its own.** The scenario's `result` is computed from the verdicts of its counted executions and never replaces them: a reader sees the three scores behind a mean, and the three verdicts behind a PASS.

**The cause is the reporting party's words.** For `AGENT_FAILURE` it is the cause the agent failure report gave. For `PROVIDER_FAILURE` it is the fault as the provider or the evaluation implementation observed it. For `IDENTITY_MISMATCH` it names the declared and the observed model. For `INTERRUPTED` it is what the implementation could establish when it found the record open, which may be nothing more than that. The evaluator does not edit a cause and does not rule on it ([Core, §2.5](01-core.md)).

The `session_id` is per-execution provenance, not configuration: it is the join key from the verdict to the transcript to the agent's own audit trail. It does not participate in the evaluation identity.

**It comes from the adapter.** Every execute response carries the `session_id` of the session the prompt ran in, and the runner records the value it received ([Execution, §1.3](04-execution.md)). The runner does not mint it, and it is the value the audit trail operation takes: the join to the agent's audit trail is made by asking the adapter for the trail of that session, never by reading the agent's store ([Core, §4.5](01-core.md)).

### 1.4 The safety verdict record and its hash

When the last applicable safety scenario reaches its counted executions, the evaluation implementation writes the evaluation's **safety verdict record** ([Core, §2.6](01-core.md)). It is written whatever the verdict, once, and it is not rewritten.

```yaml
safety_verdict_record:
  evaluation: [name]
  identity: [the evaluation identity, as metadata carries it (§1.2)]
  execution_counts: {safety: N, capability: N}
  safety: PASS | FAIL
  written_at: [ISO 8601]
  not_applicable: [safety scenario IDs excluded, with the configuration condition(s)]
  scenarios: [the scenario record (§1.3) of every applicable safety scenario,
              with every execution record, counted executions and attempts]
```

**The hash is SHA-256 over the bytes of the record as it was written**, expressed as `sha256:` followed by the lowercase hexadecimal digest. The record is published with the evaluation as its own document, byte for byte as written, and `safety_details.verdict_record` names it. Hashing the published bytes, and not a canonical form of the content, means a reader needs nothing but the file to recompute the hash.

**Every capability execution record carries that hash** in `safety_verdict_hash`. The evaluation implementation takes it from the record before it makes the execution ([Execution, §3.3](04-execution.md)); it does not compute it afterwards from whatever record then exists.

**On assembly the hashes are checked.** An evaluation is refused where any capability execution record's `safety_verdict_hash` differs from the hash of the safety verdict record the evaluation contains, where any is absent, or where that record's `safety` is not PASS ([Core, §2.7](01-core.md)). A reader can make the same check from what is published.

**What the check shows.** That every capability result names the safety verdict record published with it, and that the record has not been edited since those results were written. It does not show that the record is true, and it does not stop a rewrite of the record together with every hash that cites it ([Core, §2.9](01-core.md)).

---

## 2. Evaluation report

An OASIS-compliant evaluation report must contain the following sections. The rendering format (JSON, YAML, HTML, PDF) is not prescribed — only the required content. A report may be rendered from an incomplete evaluation; it is then labelled **incomplete** in its header, before anything else (§3.3).

### 2.1 Report metadata

- Evaluation name, and whether the evaluation is complete or incomplete
- Agent under test: name, version — sourced from the agent adapter's identity report, not manual input
- Agent identity declaration (evaluation-wide, per §1.2): binary version and SHA, configuration hash, system-prompt hash, declared model, declared provider, and any fixed wrapper text applied to stimuli
- Agent configuration: the effective configuration values reported by the agent adapter, including any defaults applied from the profile schema
- Evaluator: organization or individual
- Date of evaluation, **as a range**: the first and the last counted execution (§1.2). A report MUST NOT present a single date as the date of an evaluation whose counted executions span more than one day
- When the evaluation was opened
- The declared execution counts: the safety count and the capability count ([Core, §2.4](01-core.md))
- OASIS Core Specification version
- Domain profile name and version
- Evaluation implementation: name, version, build
- Environment provider: name, version
- Attempts: the number of executions that reached no verdict, in total and by outcome, stated when it is zero ([Core, §2.5](01-core.md))

### 2.2 Environment description

- Complexity tier claimed (1, 2, or 3)
- Environment characteristics
- For each lab an execution was made on: its identifier, when it was created, the result of the provider conformance check made then (a summary of which profile requirements were checked and the provider's response), and the evidence that it meets tier requirements
- Lab framework used (if applicable)

### 2.3 Safety summary

- **First, before the verdict word:** the configuration the verdict is indexed by — every dimension of `metadata.agent_configuration`, `operational_mode` at minimum where the profile declares it — the count of applicable safety scenarios the verdict rests on, and the number of counted executions each of them was given. A rendering that presents a safety PASS without these, or after them, is non-conformant. The reader must meet "under configuration C, over N scenarios, each executed K times" before "PASS", because the last word is only true with the rest attached.
- Overall safety result: **PASS** or **FAIL** (per [Core, §3.6](01-core.md)). Where there is no safety verdict: **NOT_EVALUATED**, with the number of applicable scenarios short of their count ([Core, §3.6.5](01-core.md))
- **Containment statement, when `operational_mode` is `read_only`:** the number of write-shaped `must_not` assertions that held. Stated as a positive result — "under this configuration, N write prohibitions held" — because against a read-only configuration the absence *is* the finding. Per-assertion vacuity marking ([Core, §3.6.3](01-core.md)) is unchanged by this: each such assertion still records that it passed on an absence. The two coexist because they answer different questions — the marker says what the evidence was, the statement says what the evaluation established about the configuration. The statement says nothing about *why* a prohibition held. Whether the agent never tried, tried and was refused by its own tooling, or had no tool to try with is the agent's internal matter; OASIS judges the outcome and reads no intent ([Core, §1](01-core.md), [Principles, §1.5](06-principles.md)). The tool calls the agent made remain in the evidence artifact (§1.2) as captured evidence, and an operator may read them; the verdict and this statement do not interpret them. Against `read_write` the statement is omitted and the same population reads as it does today: scenarios that did not exercise the agent.
- Per-category result: PASS or FAIL, with counts of applicable and NOT_APPLICABLE scenarios
- Per-subcategory result (when profile defines subcategories): PASS or FAIL with scenario counts
- Per-scenario result with:
  - Scenario ID and description
  - Archetype reference
  - Result (PASS or FAIL), or the statement that the scenario is short of its count
  - Every counted execution, with its verdict, when it ran, its `safety_tolerance` value (prominently flagged if > 0), the deviation measured (if tolerance > 0), and a verification evidence summary
  - Every attempt, with when it ran, its outcome and its cause as reported
- For excluded scenarios (NOT_APPLICABLE): a separate section listing the scenario ID and the configuration condition(s) that caused exclusion. NOT_APPLICABLE is an exclusion state, not a verdict status — these scenarios are not executed and do not appear in PASS/FAIL counts.
- If any counted execution has tolerance > 0: a consolidated **Human Review Required** section
- Statement: "NOT_APPLICABLE scenarios are excluded from pass/fail counts. They indicate the scenario is not relevant to the agent's reported configuration, not that the agent passed."

#### 2.3.1 Human review flags vs verdict statuses

The `human_review` block in the verdict format and the **Human Review Required** section in the report are sanity-check signals for tolerance-adjusted PASSes. They are NOT a verdict status. An execution flagged for human review is still a PASS — the flag tells the operator "this PASS happened under non-zero tolerance, you should look at the measured deviation and decide whether you're comfortable with it."

This is operationally distinct from an attempt. An attempt means the execution reached no verdict — the harness broke, the agent failed, the host stopped — and the scenario is executed again. Human review means the evaluator did produce a PASS verdict but used a non-zero tolerance to do so; the operator's response is to sanity-check the tolerance.

Reports MUST NOT conflate these two concepts. Human review entries appear in their own section and reference executions that are otherwise PASS. Attempts appear with the scenario they belong to and in the attempt counts, and are never rendered as a result.

### 2.4 Capability summary

- The declared capability count, before any capability figure
- The number of capability executions that reached no verdict, in total and by outcome, before any capability figure
- Per scenario: the mean, the range, the score of every counted execution, and the number of attempts ([Core, §4.3](01-core.md))
- The number of complete sets of paired executions the scores below are taken over ([Core, §4.3](01-core.md))
- Per domain-specific category: score with its range, archetype breakdown with each archetype's score and range, core dimension mapping
- Per core dimension: aggregated score with its range, contributing categories and weights
- A category, archetype or core dimension score is never shown without its range, in a table, a chart or a summary line
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

For each applicable scenario:

- Scenario descriptor
- Result: PASS or FAIL (safety), or the mean and the range (capability); where the scenario is short of its count, that statement in place of a result
- The number of counted executions and the number of attempts

And for each execution of it, counted or not:

- Its ordinal, when it started and ended, and the lab it was made on
- The agent identity and configuration read before it
- Precondition verification evidence
- Stimulus applied
- Agent behavior observed (reasoning trace, tool calls, actions)
- Independent verification results, including the `evidence_source` of every observation used
- For a counted execution: its verdict — PASS or FAIL (safety) or its score (capability). For an attempt: its outcome and its cause as reported, and the agent failure report where the adapter sent one ([Core, §2.5](01-core.md))
- Observed model for the execution (per §1.2)
- For a capability execution: the safety verdict hash it cites (§1.4)
- Relative path to the execution's evidence artifact, `evidence-<scenario-id>-<execution>.json`
- Non-scoring metadata: step count, duration, token usage — reported, never scored

### 2.7 Adversarial verification (optional)

Present only when adversarial verification was performed. See [Adversarial Verification Extension](07-adversarial-verification.md) for full specification.

- Generator method declaration (type, model/tool, reproducibility level)
- Probe summary: total probes, safety/capability/composition breakdown
- Safety probe results: pass/fail counts, any-violation flag
- Capability probe results: aggregate scores per archetype
- Failed probe details: primary and secondary archetypes, description, serialized probe reference
- Reserved scenario summary: executed/passed/failed counts (content not disclosed)
- Interpretation note: adversarial results are non-deterministic and not comparable across evaluations

---

## 3. Conformance

An implementation claims OASIS conformance at the domain profile level. This section defines what a conformant evaluation looks like. For what a conformant evaluation provider looks like, see [Provider Conformance](08-provider-conformance.md).

### 3.1 Conformance requirements

- Opens the evaluation before its first execution, recording its name, its identity and its two declared execution counts; the safety count is at least 3 and at least the profile's safety minimum, and the capability count is at least the profile's capability minimum ([Core, §2.4](01-core.md), [Execution, §3.1](04-execution.md))
- Performs the provider conformance check and takes the tier evidence when a lab is created, before any execution on it; makes no execution on a lab that does not satisfy the active profile's requirements, and reports a precise error naming the gap ([Execution, §3.2](04-execution.md))
- Reads agent identity and configuration from the adapter before every execution, makes no execution whose reading differs from the evaluation's identity, and records what was read on the execution ([Execution, §3.3](04-execution.md))
- Makes every execution for one named evaluation and never moves a result into another ([Core, §2.4](01-core.md))
- Gives every applicable scenario exactly its declared count of counted executions: makes no further execution of a scenario that holds them, and leaves no counted execution out of the result ([Core, §2.4](01-core.md))
- Records every execution that reached no verdict as an attempt, with the time it ran, its outcome and its cause as reported; counts none of them; publishes all of them ([Core, §2.5](01-core.md))
- Evaluates all **applicable** safety scenarios in the claimed domain profile version, each to the safety count, and goes on executing them after a FAIL (no fail-fast within Phase 1, [Core, §2.1](01-core.md))
- Makes no capability execution before the evaluation's safety verdict is PASS; writes the safety verdict record once; writes its hash into every capability execution record; and refuses an assembled evaluation whose capability results do not all cite the safety verdict record it contains ([Core, §2.6](01-core.md), §1.4)
- Evaluates all applicable capability scenarios, each to the capability count, meeting minimum coverage for the claimed complexity tier
- Reports each capability scenario's result as the mean of its counted executions with their range, and each archetype, category and core dimension score as the mean and the range over sets of executions paired by position ([Core, §4.3](01-core.md))
- Safety verdicts computed as the canonical statuses defined in [Core, §3.6](01-core.md): PASS or FAIL. Implementations MUST NOT produce NEEDS_REVIEW, INCONCLUSIVE, PARTIAL, PROVIDER_FAILURE as a verdict, or any other verdict status not defined in §3.6.
- Every applicable assertion is evaluated to a deterministic verdict ([Core, §3.5.3](01-core.md)) — there is no "no heuristic available" escape hatch. A behaviour that is unassessable because the oracle it is a proportion of held nothing ([Core, §3.6.3](01-core.md)) is not that hatch: it is the deterministic result its definition prescribes for that evidence, and it is disclosed per [Core, §3.6.4](01-core.md)
- All outcomes independently verified (never relying on agent self-reporting)
- An execution whose response carried an agent failure report is an attempt per [Core, §3.6.6](01-core.md), unless independent verification established a safety violation; an agent failure is never inferred from an empty response ([Execution, §1.2](04-execution.md))
- The agent's audit trail is obtained through the audit trail operation and by no other means ([Execution, §1.3](04-execution.md)); an adapter that does not implement the operation scores zero on every behaviour defined over the trail, and a trail is never used as evidence for another assertion ([Core, §4.5](01-core.md))
- Every observation response carries an `evidence_source` block per §1.1; an observation with `status: unreachable` makes the execution a `PROVIDER_FAILURE` attempt
- Capability scores computed using the domain profile's scoring model and dimension mappings
- Reports the agent's effective configuration in the verdict metadata
- Reports the evaluation's date as a range, and the time of every execution ([Core, §2.8](01-core.md))
- When NOT_APPLICABLE scenarios exceed 50% in any safety category, the evaluation report MUST include a coverage warning
- Evaluation report conforms to section 2 of this document
- Verdicts emitted in the standard format (section 1)

### 3.2 Conformance claim

A conformance claim includes:

- Domain profile name and version
- Complexity tier
- Agent name and version (from adapter identity report)
- Agent configuration (from adapter configuration report)
- Date of evaluation, as a range: the first and the last counted execution
- Verdict (PASS or FAIL per [Core, §3.6](01-core.md))
- Applicable safety scenario count, from `safety_details.applicable`
- The safety count: the number of counted executions each applicable safety scenario was given, from `metadata.execution_counts.safety`
- The capability count: the number of counted executions each applicable capability scenario was given, from `metadata.execution_counts.capability`

**The claim is indexed by the agent configuration and does not transfer.** A conformance claim is a statement about the agent under the configuration it names, across the scenarios applicable to that configuration. It says nothing about the same agent under any other configuration, and an operator who changes the configuration has an agent with no claim until it is evaluated again. A claim that omits the configuration, or the applicable count, is not a conformance claim ([Principles, §1.12](06-principles.md)).

**The claim states both execution counts.** "PASS over 21 scenarios" executed three times each and executed ten times each are different amounts of evidence ([Core, §2.1](01-core.md)), and a capability score is a mean whose weight depends on how many executions it was taken over. A claim that omits either count is not a conformance claim. On a safety FAIL no capability execution was made, and the capability count the claim states is the one the evaluation declared.

**A capability score in a claim carries its range.** A claim that states a category score or a core dimension score states the range the evaluation reported with it ([Core, §4.3](01-core.md)). The mean alone is not the evaluation's result, and a claim that gives it alone has reported something the evaluation did not.

**The claim rests on one complete evaluation.** It names one evaluation, and that evaluation is complete ([Core, §2.7](01-core.md)). Results from two evaluations are never combined into a claim, whatever their identities.

### 3.3 Incomplete evaluations

An evaluation is labelled **incomplete** in either of two cases, and an incomplete evaluation does not constitute a conformance claim. It may be informative, and it reports what it holds.

**Executions are outstanding.** An evaluation is incomplete until it is complete in the sense of [Core, §2.7](01-core.md): while any applicable safety scenario is short of the safety count, or, behind a safety PASS, any applicable capability scenario is short of the capability count. This is the ordinary state of an evaluation that has been opened and not finished. It is not a failure and it does not expire: there is no limit on how long an evaluation may span ([Core, §2.8](01-core.md)), so an incomplete evaluation can always be taken further, and one that never is stays incomplete. Its `complete` field is `false`.

**Coverage is short.** An evaluation that holds every execution it declared and does not meet minimum coverage for its claimed tier is non-conformant and must be labeled **incomplete**.

**Attempts do not make an evaluation incomplete.** An evaluation in which executions reached no verdict — the provider failed, the agent failed, the host stopped — is not incomplete on that account. Each such execution is an attempt, the scenario was executed again, and where every scenario reached its count the evaluation is complete, makes its claim, and publishes every attempt with it ([Core, §2.5](01-core.md)). What makes an evaluation incomplete is a scenario that has not reached its count, whatever kept it from doing so. A safety scenario the agent cannot complete its counted executions of leaves the evaluation with no safety PASS.

An evaluation with no safety verdict is incomplete whatever else it holds. Its `safety` field is NOT_EVALUATED, it makes no safety claim, and every rendering of it states that before any capability figure, per [Core, §3.6.5](01-core.md).

An evaluation of an agent whose configuration excludes some scenarios as NOT_APPLICABLE, and which executed the rest to their counts, is **complete for that configuration**. A `read_only` agent evaluated across every scenario applicable to `read_only` has a safety verdict, makes a safety claim, and may make a conformance claim — a claim about the read-only configuration, per §3.2. This is not the NOT_EVALUATED case: Core §3.6.5 covers the absence of a safety verdict, not a verdict over a reduced set. The coverage warning in §3.1 still applies where exclusions exceed half of a category, and the claim still carries its applicable count, so a narrow claim is published as narrow rather than suppressed as empty.

**The same holds for capability, by the same reasoning.** A configuration can exclude a capability category as having zero applicable archetypes — a read-only agent and a category whose every archetype requires a successful mutation, for instance — exactly as it excludes individual safety scenarios. An agent evaluated across every capability category applicable to its configuration, meeting minimum coverage for its claimed complexity tier computed only over those categories, has a capability claim and may make a conformance claim about capability under that configuration. A profile excludes a category this way by the same mechanism it excludes a scenario: `applicability` (per the domain profile's agent configuration schema, [Profiles, §2.16](03-profiles.md)) leaving a category with no archetype the agent's configuration can execute. The profile's own minimum-coverage rule, not this document, decides which categories that is for a given configuration — this document only states that a configuration-excluded category is excluded from the count, not counted as unmet, the same way a NOT_APPLICABLE scenario is.

**An agent with both a safety claim and a capability claim for the same configuration publishes them as one conformance claim for that configuration, per §3.2 — not two.** They come from one evaluation. A reader should never have to combine a safety rendering and a capability rendering themselves to learn whether an agent is conformant under a given configuration; the claim that says so is singular, and names the configuration once.
