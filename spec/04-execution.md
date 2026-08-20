# OASIS Execution Model

**Version:** 1.0.0-rc1.12

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
- **Execute and report** — the agent processes the prompt, takes actions, and returns a structured response including: actions taken (tool calls with arguments and results), reasoning trace (optional but scored under auditability), and final answer or outcome.
- **Stateless between scenarios** — the agent must not carry state from one scenario to the next. Each scenario starts clean.

The interface is defined as a protocol, not an implementation. An HTTP API, CLI wrapper, MCP server, or any other mechanism that satisfies the contract is valid.

### 1.1 Starting-state contract

The stateless-between-scenarios requirement above pins the agent's *carried* state. It does not pin the agent's *starting* state — what the agent is told, how much of it comes from the scenario, and what it can already see before it uses a tool. A scenario that provisions an identical environment but hands the agent a different starting place is not the same scenario. This subsection extends the stateless requirement into a full contract; the two are read together.

**Delivery fidelity.** The adapter MUST deliver the stimulus value unmodified as the sole task input. No paraphrase, no summarization, no appended hints, no reordering, no substitution of the scenario's `stimuli.value` with an equivalent-in-spirit prompt. Any fixed wrapper text the adapter adds — a system preamble, a role framing, a response-format instruction — MUST be constant across all scenarios in the run and MUST be disclosed in the evaluation report. Disclosed wrapper text is part of the agent-under-test's identity, not part of the scenario: it describes the thing being measured, so it must be reported alongside the agent's other identity material (see [Reporting, §2.1](05-reporting.md)). A wrapper that varies by scenario is a per-scenario prompt modification and violates this requirement regardless of what it contains.

**Session freshness.** Each scenario execution MUST run against a fresh agent session by default. A fresh session carries no conversation history, no prior scenario's context, and no accumulated agent memory. A scenario that requires a seeded session — a standing instruction, an established operator relationship, a prior exchange — MUST obtain that seeding exclusively through declared `conversation_context` stimuli ([Scenarios, §1.4](02-scenarios.md)). Seeding that is not declared in the scenario's stimuli block is not permitted, because it is unrecorded state that the verdict cannot account for.

**Disclosed context.** No environmental facts beyond the stimulus may be disclosed to the agent. Everything else about the environment MUST be reachable only through tools. This is not a convenience constraint: state discovery is itself a scored capability, and an adapter that pre-loads a cluster summary, a resource inventory, or a description of what is wrong has silently answered part of what the scenario asks. The corollary binds scenario authors rather than adapters — the stimulus text MUST NOT name identifiers that appear in the scenario's answer key, since a stimulus that names the answer measures reading comprehension.

These three requirements are execution-side obligations on the adapter and the runner. They are independent of how the agent is built and do not constrain the agent's internal architecture.

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
- **Provision** — create an environment matching a scenario's preconditions.
- **State snapshot** — capture current state for independent verification.
- **Teardown** — destroy the environment after the scenario completes.
- **State injection** — set up specific state required by a scenario.
- **Independent observation** — provide the evaluation implementation with direct access to system audit logs, state, and API-level activity independent of the agent's own reporting.

### 2.3 Environment isolation

Each scenario runs in an isolated environment. Actions taken in one scenario must not affect another. The environment provider is responsible for enforcing this isolation.

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
    b. Provision environment per preconditions
    c. Configure agent with declared mode and tools
    d. Apply stimuli
    e. Capture agent actions and responses
    f. Execute independent verification
    g. Evaluate assertions against independently verified behavior
       - Merge conditional assertions if any match agent configuration
    h. Compute pass/fail with safety_tolerance
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
    g. Score per rubric and dimensions; if a runtime provider fault prevented
       independent verification, abort the run immediately (skip remaining
       scenarios, jump to step 11) with a top-level PROVIDER_FAILURE.
    h. Teardown environment
 8. Compute capability scores:
    a. Archetype scores (from scenario scores)
    b. Category scores (from archetype scores, domain-defined aggregation)
    c. Core dimension scores (from category scores, domain-defined mapping)
 9. Compute coverage matrix against tier requirements
10. Evaluate configuration coverage
    a. Count scenarios excluded as NOT_APPLICABLE per category
    b. If any safety category has >50% NOT_APPLICABLE scenarios:
       emit warning with category name, percentage, and agent configuration
    c. Include NOT_APPLICABLE counts in the evaluation report
11. Emit final verdict with full report
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
