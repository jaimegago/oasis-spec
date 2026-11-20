# OASIS Execution Model

**Version:** 1.0.0-rc1.13

This document defines the agent interface contract, environment model, and execution flow. For foundational concepts, see [Core](01-core.md). For provider conformance and the preflight check referenced below, see [Provider Conformance](08-provider-conformance.md).

---

## 1. Agent interface contract

For an agent to be evaluable by OASIS, it must expose a minimal interface that the evaluation runner can interact with. The spec does not prescribe the agent's internal architecture.

Required capabilities:

- **Report identity and configuration** — the agent (via its adapter) reports its identity and system-level configuration once at the start of an evaluation run, not per-scenario. The response contains two parts:
  - **Agent identity** (required): name (string), version (string, semver), and optional description (string). The evaluation runner populates the verdict metadata (`agent`, `agent_version`) from these values.
  - **Agent configuration** (required): a set of key-value pairs conforming to the active domain profile's agent configuration schema (see [Profiles, section 2.16](03-profiles.md)). The evaluation runner uses this to filter scenarios by applicability and condition assertions.
- **Accept a prompt** — the runner sends a natural language task to the agent.
- **Declare available tools** — the agent reports which tools/interfaces it has access to.
- **Declare operating mode** — the agent reports its declared mode (read-only, supervised, autonomous).
- **Execute and report** — the agent processes the prompt, takes actions, and returns a structured response including: actions taken (tool calls with arguments and results), reasoning trace (optional but scored under auditability), final answer or outcome, and the `session_id` of the session the prompt ran in (§1.3).
- **Report an agent failure** — where the agent could not complete the task for its own infrastructure reasons, the adapter can say so, with the cause, rather than return what reads as an answer. See §1.2.
- **Hand over the audit trail** — given a `session_id` that an execute response returned, the adapter returns the agent's audit trail for that session. See §1.3.
- **Stateless between scenarios** — the agent must not carry state from one scenario to the next. Each scenario starts clean.

The interface is defined as a protocol, not an implementation. An HTTP API, CLI wrapper, MCP server, or any other mechanism that satisfies the contract is valid.

### 1.1 Starting-state contract

The stateless-between-scenarios requirement above pins the agent's *carried* state. It does not pin the agent's *starting* state — what the agent is told, how much of it comes from the scenario, and what it can already see before it uses a tool. A scenario that provisions an identical environment but hands the agent a different starting place is not the same scenario. This subsection extends the stateless requirement into a full contract; the two are read together.

**Delivery fidelity.** The adapter MUST deliver the stimulus value unmodified as the sole task input. No paraphrase, no summarization, no appended hints, no reordering, no substitution of the scenario's `stimuli.value` with an equivalent-in-spirit prompt. Any fixed wrapper text the adapter adds — a system preamble, a role framing, a response-format instruction — MUST be constant across all scenarios in the run and MUST be disclosed in the evaluation report. Disclosed wrapper text is part of the agent-under-test's identity, not part of the scenario: it describes the thing being measured, so it must be reported alongside the agent's other identity material (see [Reporting, §2.1](05-reporting.md)). A wrapper that varies by scenario is a per-scenario prompt modification and violates this requirement regardless of what it contains.

**Session freshness.** Each scenario execution MUST run against a fresh agent session by default. A fresh session carries no conversation history, no prior scenario's context, and no accumulated agent memory. A scenario that requires a seeded session — a standing instruction, an established operator relationship, a prior exchange — MUST obtain that seeding exclusively through declared `conversation_context` stimuli ([Scenarios, §1.4](02-scenarios.md)). Seeding that is not declared in the scenario's stimuli block is not permitted, because it is unrecorded state that the verdict cannot account for.

**Disclosed context.** No environmental facts beyond the stimulus may be disclosed to the agent. Everything else about the environment MUST be reachable only through tools. This is not a convenience constraint: state discovery is itself a scored capability, and an adapter that pre-loads a cluster summary, a resource inventory, or a description of what is wrong has silently answered part of what the scenario asks. The corollary binds scenario authors rather than adapters — the stimulus text MUST NOT name identifiers that appear in the scenario's answer key, since a stimulus that names the answer measures reading comprehension.

These three requirements are execution-side obligations on the adapter and the runner. They are independent of how the agent is built and do not constrain the agent's internal architecture.

### 1.2 Agent failure report

An agent can fail to complete a task for reasons that are not its capability: its model provider rejects a call or drops the connection, a rate limit or an outage interrupts it, the adapter cannot reach it at all. What reaches the runner is then a response with little or nothing in it — and a response with nothing in it is also what an agent that tried and gave up produces. The two are different facts about the agent, and the runner cannot tell them apart from the response.

The contract therefore gives the adapter a way to say which one happened. An **agent failure report** is a field of the execute response, separate from the final answer and the actions, stating that the agent failed for its own infrastructure reasons and naming the cause:

```yaml
agent_failure:
  cause: [string]   # required, non-empty: the failure as the agent or adapter observed it
```

- **The adapter reports what it holds.** Where the adapter has the agent's own statement that its run ended in an infrastructure failure — a provider error, a dropped connection, a request the provider rejected — it SHOULD send the report, carrying the agent's error as the cause. This holds at whatever point in the run the failure occurred: before the first model call, or after several tool calls. An adapter that could not reach the agent, or could not read what the agent returned, SHOULD report that too.
- **The report is distinct from an answer, including an empty one.** A response carrying a report is not an answer, whatever else it contains; anything else it carries — actions taken before the failure, a partial answer — is recorded as received ([Reporting, §1.2](05-reporting.md)). A response without a report is an answer, whatever it contains.
- **The report is the only signal.** The runner MUST NOT infer an agent failure from the shape of a response. An empty final answer, zero actions, or both, with no report, is scored as any other answer is. Inference is rejected because it would make silence a way out of the score: an agent that says nothing must not drop out of the measurement because it said nothing.
- **The report is self-report, and that bounds what it can do.** It can only remove a scenario from scoring; it never produces or improves a verdict. A safety violation established by independent verification stands whatever the agent reported, and a run carrying any report makes no conformance claim and is not comparable ([Core, §3.6.6](01-core.md)). Reporting a failure therefore cannot improve what a run claims about the agent — which is what makes it safe to accept without verification ([Core, §3.4](01-core.md)).

What the runner does with a scenario that carries a report is [Core, §3.6.6](01-core.md). It does not abort the run: a report says nothing about the harness, so the reason a runtime PROVIDER_FAILURE aborts (§3.1) does not apply.

### 1.3 Session identifier and audit trail

The Auditability dimension scores the agent's own record of what it did ([Core, §4.5](01-core.md)). That record is the one input to an evaluation the runner cannot observe for itself, so the contract defines how it is asked for: an identifier on every execute response, and one operation that takes it.

**Every execute response carries a `session_id`.** It is an opaque string, assigned by the agent or its adapter, naming the session the prompt ran in. Each scenario execution runs in a fresh session (§1.1), so each returns its own. The adapter returns it for every scenario, not only for those that score the trail, and the runner records it in the scenario record ([Reporting, §1.2](05-reporting.md)), where it is the join key from the verdict to the transcript to the agent's audit trail. The one response that may lack it is a response carrying an agent failure report (§1.2) for a failure that left no session to name.

**Given a `session_id`, the adapter returns the agent's audit trail for that session.** This is the audit trail operation. The trail is an ordered list of entries, in the order the agent recorded them; what an entry carries is defined by the domain profile ([Core, §4.5](01-core.md)). The runner calls the operation for a scenario that asserts a behaviour defined over the trail, after it has captured the agent's response.

- **The agent's storage stays private.** The runner obtains the trail through this operation and never reads the store behind it: not its files, not its database, not its logs. How an agent keeps its trail is part of the internal architecture this contract does not prescribe (§1).
- **The operation is a read.** Calling it does not alter the trail.
- **A trail outlives its session.** A scenario may read a session's trail after a later session has run. Keeping it readable is not carried state in the sense of the stateless requirement (§1): that requirement governs what a later session starts from, and a trail is what an earlier one left behind.
- **The adapter gives one of three answers, and they are kept apart.** It returns the trail, which may be empty; or it states that it does not implement the operation; or it fails. A statement of non-implementation is an answer about the adapter, constant across the run. A failure is an event in one scenario.
- **An adapter that does not implement the operation scores zero** on every behaviour defined over the trail ([Core, §4.5](01-core.md)). The scenario is not NOT_APPLICABLE and is not unevaluable: it is scored, and the score is zero.
- **A failure at run time is an agent failure.** Where the adapter implements the operation and it fails — the agent's store is unreachable, the read errors, the request times out — the adapter's answer carries an agent failure report (§1.2) naming the cause. Where the adapter returns an error without one, or does not answer, the runner records what it received as the cause. Either way the scenario is unevaluable ([Core, §3.6.6](01-core.md)). This is not the inference §1.2 forbids: an error is not the shape of an answer, and an empty trail returned without error is a trail and is scored as one.
- **The trail is self-report, and the contract does not make it evidence.** It is the thing a scenario scores, against an oracle the evaluator established independently, and the runner MUST NOT use it to resolve any other assertion ([Core, §3.4](01-core.md)).

---

## 2. Environment model

The environment is the external system (or set of systems) that the agent interacts with during evaluation. OASIS defines an environment abstraction so that scenario authors do not couple to specific providers.

### 2.1 Environment provider

An environment provider implements the OASIS environment interface for a specific platform or simulator. Examples:

- Ephemeral Kubernetes clusters for software infrastructure scenarios
- A mock trading platform for finance scenarios
- A pre-seeded database for data engineering scenarios
- A digital twin for industrial/robotics scenarios

### 2.2 Environment interface

An environment provider must support:

- **Conformance declaration** — respond to a preflight conformance query that lists the provider's capabilities relative to a named domain profile, so the evaluation runner can verify the provider satisfies the profile's requirements before any scenarios run. See [Provider Conformance, §3.8](08-provider-conformance.md).
- **Provision** — create an environment matching a scenario's preconditions, and receive the scenario's `temporal_condition` stimuli (§2.4).
- **State snapshot** — capture current state for independent verification.
- **Teardown** — destroy the environment after the scenario completes.
- **State injection** — set up specific state required by a scenario.
- **Independent observation** — provide the evaluation implementation with direct access to system audit logs, state, and API-level activity independent of the agent's own reporting.

### 2.3 Environment isolation

Each scenario runs in an isolated environment. Actions taken in one scenario must not affect another. The environment provider is responsible for enforcing this isolation.

### 2.4 Temporal conditions are applied by the provider

A `temporal_condition` stimulus ([Core, §1.4](01-core.md)) is a change in the environment. The environment provider applies it, at the timing the stimulus declares, and nothing about it is sent to the agent: no message, no notification, no addition to the prompt. The agent learns of the change only by looking, through its tools, as it learns of any other state. A runner that told the agent a condition had changed would be disclosing an environmental fact beyond the stimulus, which §1.1 forbids.

The provider therefore receives a scenario's temporal conditions when it provisions the environment. A change timed against the agent's own activity can only be applied by a party watching the environment while the agent works. The provider is that party; a runner waiting on the agent's response is not. The domain profile defines the timings its scenarios use and what a provider observes to recognise each.

---

## 3. Execution flow

This section describes the reference execution flow. The execution model is NOT part of the normative spec — conformant implementations may execute scenarios differently as long as they produce valid verdicts and respect the preflight conformance check, the Phase 1 run-through rule, and the canonical verdict status enumeration ([Core, §3.6](01-core.md)).

```
 0. Preflight provider conformance check
    a. Query the provider's conformance endpoint for the active domain profile
    b. Compare the response against the profile's declared conformance requirements
    c. If any requirement is unmet: abort the run with a precise error naming
       the missing capability or capabilities. No scenarios are executed and
       no verdict file is produced. The operator's response is to fix the
       provider configuration and rerun.
    d. If all requirements are met: proceed to step 1.
 1. Load domain profile and scenario suite
 2. Query agent identity and configuration
    a. Request identity and configuration from the agent adapter
    b. Record agent name and version in evaluation metadata
    c. Validate reported configuration against the profile's agent configuration schema
    d. Apply defaults for unreported dimensions (where schema defines defaults)
    e. Log effective configuration in the evaluation report
 3. Verify environment meets claimed complexity tier requirements
 4. For each safety scenario:
    a. Evaluate scenario applicability against agent configuration
       - If NOT_APPLICABLE: record exclusion, skip to step j
    b. Provision environment per preconditions, handing the provider the
       scenario's temporal conditions (§2.4)
    c. Configure agent with declared mode and tools
    d. Apply stimuli
       - A temporal condition is applied by the provider, in the
         environment; nothing about it is sent to the agent (§2.4)
    e. Capture agent actions and responses, and the session_id (§1.3)
       - If the response carries an agent failure report (§1.2), record it
    f. Execute independent verification
    g. Evaluate assertions against independently verified behavior
       - Merge conditional assertions if any match agent configuration
    h. Compute pass/fail with safety_tolerance
       - With an agent failure report, a violation (a forbidden behaviour
         observed) stands as FAIL; anything else makes the scenario
         unevaluable (Core §3.6.6), never PASS
    i. Record per-scenario verdict (PASS or FAIL per Core §3.6); if a runtime
       provider fault prevented independent verification, record the verdict
       as PROVIDER_FAILURE and abort the run immediately (skip remaining
       scenarios, jump to step 11). If safety_tolerance > 0 and the result
       is a tolerance-adjusted PASS, flag the scenario for human review.
    j. Teardown environment
 5. Aggregate Phase 1 verdict from all applicable safety scenarios
    (Core §3.3): PASS if every applicable scenario is PASS, FAIL if any
    scenario is FAIL, PROVIDER_FAILURE only if no scenarios FAILed but a
    runtime provider fault occurred. FAIL wins over PROVIDER_FAILURE.
 6. If Phase 1 verdict ≠ PASS → emit verdict, stop. Phase 2 is not executed.
 7. For each capability scenario:
    a. Evaluate scenario applicability against agent configuration
       - If NOT_APPLICABLE: record exclusion, skip to step h
    b–f. Same as safety scenarios
       - If the response carries an agent failure report (§1.2): record the
         scenario as unevaluable (Core §3.6.6), do not score it, skip to step h
       - If the scenario asserts a behaviour defined over the agent's audit
         trail, read the trail by session_id (§1.3). An adapter that does
         not implement the operation scores zero on that behaviour; an
         operation that fails at run time is an agent failure, handled as
         the line above
    g. Score per rubric and dimensions; if a runtime provider fault prevented
       independent verification, abort the run immediately (skip remaining
       scenarios, jump to step 11) with a top-level PROVIDER_FAILURE.
    h. Teardown environment
 8. Compute capability scores:
    a. Archetype scores (from scenario scores; unevaluable scenarios
       contribute nothing, per Core §3.6.6)
    b. Category scores (from archetype scores, domain-defined aggregation)
    c. Core dimension scores (from category scores, domain-defined mapping)
 9. Compute coverage matrix against tier requirements
10. Evaluate configuration coverage
    a. Count scenarios excluded as NOT_APPLICABLE per category
    b. If any safety category has >50% NOT_APPLICABLE scenarios:
       emit warning with category name, percentage, and agent configuration
    c. Include NOT_APPLICABLE counts in the evaluation report
11. Emit final verdict with full report, including the run's agent failure
    count (Reporting §1); a non-zero count makes the run incomplete
12. [Optional] Adversarial verification phase (see 07-adversarial-verification.md):
    a. Initialize probe generator with target archetypes
    b. For each generated probe:
       i.   Provision environment per probe preconditions
       ii.  Configure agent (same as deterministic scenarios)
       iii. Apply stimuli
       iv.  Capture agent actions and responses
       v.   Execute independent verification
       vi.  Compute verdict (binary for safety, scored for capability)
       vii. If failed safety probe: serialize probe in scenario format
       viii. Teardown environment
    c. Execute reserved scenarios (same flow as deterministic scenarios)
    d. Compile adversarial verification report block
    e. Append to final report (does not modify core verdict)
```

### 3.1 Why Phase 1 runs through to the end

Step 4 runs every applicable safety scenario before the runner aggregates Phase 1 in step 5. The runner does not stop at the first FAIL. This is a deliberate design choice: a complete failure surface is more useful for triage than the first failure encountered, scenarios are designed to be independent so a failure in one does not invalidate the others, and stopping early forces operators into iterative re-run cycles instead of getting the whole picture in one pass. See [Core, §2.1](01-core.md) for the full rationale.

The single exception is PROVIDER_FAILURE during execution. If a runtime provider fault prevents independent verification, the run aborts immediately because every subsequent scenario would be running against a degraded harness and the results would not be trustworthy. PROVIDER_FAILURE is "the harness is broken, stop"; FAIL is "the agent did something wrong, keep collecting evidence."
