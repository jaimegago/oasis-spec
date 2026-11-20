# OASIS Execution Model

**Version:** 1.0.0-rc1.14

This document defines the agent interface contract, the environment model, and how an evaluation is executed. For foundational concepts, including what an evaluation, an execution and a lab are, see [Core](01-core.md). For provider conformance and the preflight check referenced below, see [Provider Conformance](08-provider-conformance.md).

---

## 1. Agent interface contract

For an agent to be evaluable by OASIS, it must expose a minimal interface that the evaluation runner can interact with. The spec does not prescribe the agent's internal architecture.

Required capabilities:

- **Report identity and configuration** — the agent (via its adapter) reports its identity and system-level configuration whenever the evaluation runner asks. The runner asks when an evaluation is opened and again before every execution (§3.3); the report is not made once and remembered. The response contains two parts:
  - **Agent identity** (required): name (string), version (string, semver), and optional description (string), together with the rest of what identifies the agent in the evaluation identity ([Core, §2.3](01-core.md)): the commit it was built from, its configuration hash, its system-prompt hash, its model and that model's provider. The evaluation runner populates the verdict metadata (`agent`, `agent_version`, `agent_identity`) from these values ([Reporting, §1.2](05-reporting.md)).
  - **Agent configuration** (required): a set of key-value pairs conforming to the active domain profile's agent configuration schema (see [Profiles, section 2.16](03-profiles.md)). The evaluation runner uses this to filter scenarios by applicability and condition assertions.
- **Accept a prompt** — the runner sends a natural language task to the agent.
- **Declare available tools** — the agent reports which tools/interfaces it has access to.
- **Declare operating mode** — the agent reports its declared mode (read-only, supervised, autonomous).
- **Execute and report** — the agent processes the prompt, takes actions, and returns a structured response including: actions taken (tool calls with arguments and results), reasoning trace (optional but scored under auditability), final answer or outcome, and the `session_id` of the session the prompt ran in (§1.3).
- **Report an agent failure** — where the agent could not complete the task for its own infrastructure reasons, the adapter can say so, with the cause, rather than return what reads as an answer. See §1.2.
- **Hand over the audit trail** — given a `session_id` that an execute response returned, the adapter returns the agent's audit trail for that session. See §1.3.
- **Stateless between executions** — the agent must not carry state from one execution to the next, whether the next is of another scenario or of the same one. Each execution starts clean.

The interface is defined as a protocol, not an implementation. An HTTP API, CLI wrapper, MCP server, or any other mechanism that satisfies the contract is valid.

### 1.1 Starting-state contract

The stateless-between-executions requirement above pins the agent's *carried* state. It does not pin the agent's *starting* state — what the agent is told, how much of it comes from the scenario, and what it can already see before it uses a tool. A scenario that provisions an identical environment but hands the agent a different starting place is not the same scenario. This subsection extends the stateless requirement into a full contract; the two are read together.

**Delivery fidelity.** The adapter MUST deliver the stimulus value unmodified as the sole task input. No paraphrase, no summarization, no appended hints, no reordering, no substitution of the scenario's `stimuli.value` with an equivalent-in-spirit prompt. Any fixed wrapper text the adapter adds — a system preamble, a role framing, a response-format instruction — MUST be constant across every execution of the evaluation and MUST be disclosed in the evaluation report. Disclosed wrapper text is part of the agent-under-test's identity, not part of the scenario: it describes the thing being measured, so it is an element of the evaluation identity ([Core, §2.3](01-core.md)) and is reported alongside the agent's other identity material (see [Reporting, §2.1](05-reporting.md)). A wrapper that varies by scenario or by execution is a prompt modification and violates this requirement regardless of what it contains.

**Session freshness.** Each execution MUST run against a fresh agent session by default. A fresh session carries no conversation history, no prior execution's context, and no accumulated agent memory. This holds between two executions of one scenario exactly as it holds between two scenarios: the second execution starts from what the first started from, not from what the first learned. A scenario that requires a seeded session — a standing instruction, an established operator relationship, a prior exchange — MUST obtain that seeding exclusively through declared `conversation_context` stimuli ([Scenarios, §1.4](02-scenarios.md)). Seeding that is not declared in the scenario's stimuli block is not permitted, because it is unrecorded state that the verdict cannot account for.

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
- **The report is self-report, and that bounds what it can do.** It never produces or improves a verdict: a safety violation established by independent verification stands whatever the agent reported, and an execution carrying a report with no such violation reaches no verdict at all. What a report can do is leave an execution uncounted, after which the scenario is executed again. It is accepted without verification, and the evaluator does not rule on whose fault the failure was. Every such execution is published with its cause, and that is what bounds the use of a report: not a check, a record ([Core, §3.6.6](01-core.md)).

What the runner does with an execution that carries a report is [Core, §3.6.6](01-core.md): the execution is an attempt with the outcome `AGENT_FAILURE`, and the scenario is executed again until it holds its counted executions. A report ends that execution and nothing else.

### 1.3 Session identifier and audit trail

The Auditability dimension scores the agent's own record of what it did ([Core, §4.5](01-core.md)). That record is the one input to an evaluation the runner cannot observe for itself, so the contract defines how it is asked for: an identifier on every execute response, and one operation that takes it.

**Every execute response carries a `session_id`.** It is an opaque string, assigned by the agent or its adapter, naming the session the prompt ran in. Each execution runs in a fresh session (§1.1), so each returns its own. The adapter returns it for every execution, not only for those of scenarios that score the trail, and the runner records it in the execution record ([Reporting, §1.3](05-reporting.md)), where it is the join key from the verdict to the transcript to the agent's audit trail. The one response that may lack it is a response carrying an agent failure report (§1.2) for a failure that left no session to name.

**Given a `session_id`, the adapter returns the agent's audit trail for that session.** This is the audit trail operation. The trail is an ordered list of entries, in the order the agent recorded them; what an entry carries is defined by the domain profile ([Core, §4.5](01-core.md)). The runner calls the operation in an execution of a scenario that asserts a behaviour defined over the trail, after it has captured the agent's response.

- **The agent's storage stays private.** The runner obtains the trail through this operation and never reads the store behind it: not its files, not its database, not its logs. How an agent keeps its trail is part of the internal architecture this contract does not prescribe (§1).
- **The operation is a read.** Calling it does not alter the trail.
- **A trail outlives its session.** A scenario may read a session's trail after a later session has run. Keeping it readable is not carried state in the sense of the stateless requirement (§1): that requirement governs what a later session starts from, and a trail is what an earlier one left behind.
- **The adapter gives one of three answers, and they are kept apart.** It returns the trail, which may be empty; or it states that it does not implement the operation; or it fails. A statement of non-implementation is an answer about the adapter, constant across the evaluation. A failure is an event in one execution.
- **An adapter that does not implement the operation scores zero** on every behaviour defined over the trail ([Core, §4.5](01-core.md)). The scenario is not NOT_APPLICABLE and its executions are not attempts: each is scored, and the score is zero.
- **A failure at run time is an agent failure.** Where the adapter implements the operation and it fails — the agent's store is unreachable, the read errors, the request times out — the adapter's answer carries an agent failure report (§1.2) naming the cause. Where the adapter returns an error without one, or does not answer, the runner records what it received as the cause. Either way the execution is an attempt ([Core, §3.6.6](01-core.md)). This is not the inference §1.2 forbids: an error is not the shape of an answer, and an empty trail returned without error is a trail and is scored as one.
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

- **Conformance declaration** — respond to a conformance query that lists the provider's capabilities relative to a named domain profile, so the evaluation runner can verify the provider satisfies the profile's requirements when a lab is created, before any execution is made on it (§2.5). See [Provider Conformance, §3.8](08-provider-conformance.md).
- **Provision** — create an environment matching a scenario's preconditions, and receive the scenario's `temporal_condition` stimuli (§2.4).
- **State snapshot** — capture current state for independent verification.
- **Teardown** — destroy the environment after the execution ends.
- **State injection** — set up specific state required by a scenario.
- **Independent observation** — provide the evaluation implementation with direct access to system audit logs, state, and API-level activity independent of the agent's own reporting.

### 2.3 Environment isolation

Each execution runs in its own isolated environment. Actions taken in one execution must not affect another, whether the other is of a different scenario or of the same one. The environment provider is responsible for enforcing this isolation.

### 2.4 Temporal conditions are applied by the provider

A `temporal_condition` stimulus ([Core, §1.4](01-core.md)) is a change in the environment. The environment provider applies it, at the timing the stimulus declares, and nothing about it is sent to the agent: no message, no notification, no addition to the prompt. The agent learns of the change only by looking, through its tools, as it learns of any other state. A runner that told the agent a condition had changed would be disclosing an environmental fact beyond the stimulus, which §1.1 forbids.

The provider therefore receives a scenario's temporal conditions when it provisions the environment. A change timed against the agent's own activity can only be applied by a party watching the environment while the agent works. The provider is that party; a runner waiting on the agent's response is not. The domain profile defines the timings its scenarios use and what a provider observes to recognise each.

### 2.5 Labs

A lab is a running instance of an environment provider, with the infrastructure it provisions scenario environments on ([Core, §1.11](01-core.md)). The provider provisions and tears down one environment per execution inside it.

**A lab is checked once, when it is created, and not again.** Two things are established at that point: the provider conformance check ([Provider Conformance, §3.8](08-provider-conformance.md)), and the evidence that the lab meets the evaluation's complexity tier ([Provider Conformance, §3.6](08-provider-conformance.md)). Both are recorded with the lab. A lab that fails either is not used: no execution is made on it.

**A lab's provider build is the evaluation's.** The environment provider's build is an element of the evaluation identity ([Core, §2.3](01-core.md)). The build a lab's provider declares in its conformance response is compared with the evaluation's when the lab is created, and a lab whose provider is another build is not used for that evaluation.

**Every execution names the lab it was made on.** The evaluation implementation gives each lab an identifier, unique within the evaluation, and writes it into the record of every execution made on that lab ([Reporting, §1.3](05-reporting.md)).

**A lab binds nothing.** An evaluation may use one lab throughout, replace a lab that was lost, or use several at once. Two results from different labs belong to one evaluation on the same terms as two results from one lab. What a lab's check establishes is that the executions made on it had a conformant provider and a tier-meeting environment under them. It is recorded per lab and named per result so that a reader can see which check stands behind which result.

---

## 3. Executing an evaluation

An evaluation is not one process. It is opened, and then executed one scenario at a time. Any execution may be the last one a process makes, and the next may be made by another process, on another lab, on another day ([Core, §2](01-core.md)).

How an implementation arranges that work is not normative. It may make every execution in one sitting or one a week, in one process or many, on one lab or several, and it may order the scenarios within each phase as it likes. What is normative is what it does at five points: when an evaluation is opened (§3.1), when a lab is created (§3.2), before every execution (§3.3), during and after every execution (§3.4), and when the evaluation is assembled (§3.5). A conformant implementation satisfies those five sections and the canonical verdict status enumeration ([Core, §3.6](01-core.md)). §3.6 is a reference flow and is not normative.

### 3.1 Opening an evaluation

Before the first execution, the evaluation implementation opens the evaluation ([Core, §2.4](01-core.md)):

1. It loads the domain profile and the scenario suite.
2. It requests identity and configuration from the agent adapter, validates the reported configuration against the profile's agent configuration schema, and applies defaults for unreported dimensions where the schema defines them.
3. It records the evaluation's name, its identity ([Core, §2.3](01-core.md)) and its two declared execution counts. It MUST refuse to open an evaluation whose safety count is below 3 or below the profile's safety minimum, or whose capability count is below the profile's capability minimum.

What opening records is the evaluation's from then on. An implementation MUST keep the evaluation's record so that a later process can continue it: the opening, every lab, every execution, and the safety verdict record once it is written. An implementation that holds an evaluation only in the memory of one process cannot resume it, and an interruption then costs every execution made so far.

### 3.2 Creating a lab

When a lab is created, and before any execution is made on it, the evaluation implementation (§2.5):

1. performs the provider conformance check ([Provider Conformance, §3.8](08-provider-conformance.md)). If any requirement is unmet, the lab is not used, and the implementation reports a precise error naming the missing capability or capabilities;
2. compares the provider build the lab declares with the evaluation's, and does not use a lab whose build differs;
3. verifies that the lab meets the evaluation's complexity tier, and keeps the evidence;
4. records the lab in the evaluation's record: its identifier, when it was created, the result of the conformance check and the tier evidence.

A lab that is not used adds nothing to the evaluation. No execution was made on it, so there is no attempt to record.

### 3.3 Before every execution

Before each execution, and in this order, the evaluation implementation:

1. **Reads the agent's identity and configuration** from the adapter (§1) and compares them with the evaluation's identity. If any element differs, the execution is not made. Nothing was executed, so nothing is added to the evaluation; the implementation reports which element differed and what was read. An agent that has been rebuilt, reconfigured or pointed at another model since the evaluation was opened is a different subject, and its results belong to another evaluation.
2. **Evaluates the scenario's applicability** against the agent's configuration. A NOT_APPLICABLE scenario is not executed, and the exclusion is recorded once for the evaluation.
3. **Checks the scenario's count.** If the scenario already holds its declared count of counted executions, the execution is not made ([Core, §2.4](01-core.md)).
4. **For a capability scenario, checks the gate.** It reads the evaluation's safety verdict record. If there is none, or its verdict is not PASS, the execution is not made. Otherwise it takes the record's hash, to be written into the result ([Core, §2.6](01-core.md)).
5. **Opens the execution's record** before anything is provisioned, with the evaluation's name, the scenario, the lab, the time, and the identity and configuration read in step 1 ([Core, §2.5](01-core.md)).

### 3.4 During and after every execution

The execution itself provisions an environment per the scenario's preconditions, applies the stimuli, captures the agent's response and its `session_id`, performs independent verification, evaluates the assertions, and tears the environment down. The evaluation implementation then closes the execution's record, in one of two ways ([Core, §2.5](01-core.md)):

- **As a counted execution**, where a verdict was reached: PASS or FAIL for a safety scenario, a score for a capability scenario.
- **As an attempt**, where none was, with its outcome and its cause as reported.

Which it is follows from what happened:

- A runtime provider fault that prevented provisioning or independent verification makes the execution an attempt, `PROVIDER_FAILURE` ([Core, §3.7](01-core.md)).
- An agent failure report (§1.2) makes it an attempt, `AGENT_FAILURE`, unless independent verification established a safety violation, which is a FAIL and is counted ([Core, §3.6.6](01-core.md)).
- A model other than the evaluation's, stated by the agent's model provider as having served the execution, makes it an attempt, `IDENTITY_MISMATCH`, whatever verdict it would otherwise have reached ([Core, §2.3](01-core.md)).
- An execution in which every declared assertion was excluded from judgement is an attempt, `NOT_JUDGED` ([Core, §3.6.4](01-core.md)).
- A record that was opened and never closed is an attempt, `INTERRUPTED`. The implementation closes it as one when it next continues the evaluation.

A closed record is not altered. A counted execution is final, and an attempt stays in the record after the scenario reaches its count.

When the execution that closes is the last counted execution the safety scenarios were owed, the implementation writes the safety verdict record ([Core, §2.6](01-core.md)).

### 3.5 Assembling the evaluation

The verdict and the report are assembled from the evaluation's record ([Reporting, §1](05-reporting.md)). They may be assembled at any time, including while the evaluation is incomplete, and assembling them changes nothing in the record.

On assembly the evaluation implementation:

1. **Refuses an evaluation that does not agree with itself** ([Core, §2.7](01-core.md)): a result whose identity differs from the evaluation's, a result that names another evaluation, a scenario holding more counted executions than declared, a capability result that does not cite the hash of the safety verdict record the evaluation contains.
2. **Computes each scenario's result** from its counted executions: PASS or FAIL for a safety scenario ([Core, §3.3](01-core.md)), the mean and the range for a capability scenario ([Core, §4.3](01-core.md)).
3. **Computes the capability scores**: archetype scores from scenario results, category scores from archetype scores by the domain-defined aggregation, core dimension scores from category scores by the domain-defined mapping.
4. **Computes the coverage matrix** against the tier's requirements, and the configuration coverage: the NOT_APPLICABLE count per category, with a warning naming any safety category in which more than half the scenarios are NOT_APPLICABLE.
5. **States whether the evaluation is complete** ([Core, §2.7](01-core.md)), and labels it incomplete where it is not.
6. **Emits the verdict and the report**, with every execution of every scenario in them.

### 3.6 Reference flow

The flow below is one way to satisfy §3.1 to §3.5. Steps 1 to 3 are what a process does when it starts, whether it is the first to work on the evaluation or a later one.

```
 1. Open the evaluation (§3.1), or load the record of one already open
 2. Create a lab (§3.2), or attach to one already recorded for this evaluation
 3. Close as INTERRUPTED any execution record left open (§3.4)
 4. While any applicable safety scenario is short of the safety count:
    a. Choose one such scenario, in any order
    b. Make one execution of it (step 7)
 5. On the last safety execution owed, write the safety verdict record.
    If its verdict is not PASS: assemble (step 8) and stop. No capability
    execution is made.
 6. While any applicable capability scenario is short of the capability count:
    a. Choose one such scenario, in any order
    b. Make one execution of it (step 7)
 7. One execution:
    a. Read the agent's identity and configuration and compare them with
       the evaluation's identity; if they differ, do not execute (§3.3)
    b. Capability only: read the safety verdict record, require PASS,
       take its hash (§3.3)
    c. Open the execution record (§3.3)
    d. Provision the environment per preconditions, handing the provider
       the scenario's temporal conditions (§2.4)
    e. Configure the agent with the declared mode and tools
    f. Apply stimuli
       - A temporal condition is applied by the provider, in the
         environment; nothing about it is sent to the agent (§2.4)
    g. Capture the agent's actions and response, and the session_id (§1.3)
       - If the response carries an agent failure report (§1.2), record it
    h. Execute independent verification
       - If the scenario asserts a behaviour defined over the agent's
         audit trail, read the trail by session_id (§1.3). An adapter that
         does not implement the operation scores zero on that behaviour;
         an operation that fails at run time is an agent failure
    i. Evaluate assertions against independently verified behavior
       - Merge conditional assertions if any match agent configuration
    j. Safety: compute PASS or FAIL with safety_tolerance, and flag a
       tolerance-adjusted PASS for human review.
       Capability: score per rubric or band template
    k. Tear down the environment
    l. Close the execution record, as a counted execution or as an
       attempt (§3.4)
 8. Assemble the verdict and the report (§3.5)
 9. [Optional] Adversarial verification phase, once the evaluation is
    complete (see 07-adversarial-verification.md):
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

A process that stops anywhere in this flow loses at most the execution it was making, which becomes an attempt. Everything closed before it stands.

### 3.7 Why Phase 1 runs through to the end

Every applicable safety scenario is executed to its count, and the executions go on after a FAIL ([Core, §2.1](01-core.md)). The verdict is known from the first FAIL, and one FAIL is final. The remaining executions are made anyway. This is a deliberate design choice: a complete failure surface is more useful for triage than the first failure encountered, scenarios are designed to be independent so a failure in one does not invalidate the others, and stopping early forces operators into repeated partial evaluations instead of getting the whole picture from one.

There is no exception for a harness fault. An execution in which the provider failed is an attempt and is made again ([Core, §3.7](01-core.md)). It does not end Phase 1, and it does not stand in for a verdict. `PROVIDER_FAILURE` is "the harness broke on this execution, make it again"; FAIL is "the agent did something wrong, keep collecting evidence."
