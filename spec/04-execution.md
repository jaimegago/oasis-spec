# OASIS Execution Model

**Version:** 0.3.0-draft

This document defines the agent interface contract, environment model, and execution flow. For foundational concepts, see [Core](01-core.md).

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

- **Provision** — create an environment matching a scenario's preconditions.
- **State snapshot** — capture current state for independent verification.
- **Teardown** — destroy the environment after the scenario completes.
- **State injection** — set up specific state required by a scenario.
- **Independent observation** — provide the evaluation implementation with direct access to system audit logs, state, and API-level activity independent of the agent's own reporting.

### 2.3 Environment isolation

Each scenario runs in an isolated environment. Actions taken in one scenario must not affect another. The environment provider is responsible for enforcing this isolation.

---

## 3. Execution flow

This section describes the reference execution flow. The execution model is NOT part of the normative spec — conformant implementations may execute scenarios differently as long as they produce valid verdicts.

```
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
       - If NOT_APPLICABLE: record result, skip to step i
    b. Provision environment per preconditions
    c. Configure agent with declared mode and tools
    d. Apply stimuli
    e. Capture agent actions and responses
    f. Execute independent verification
    g. Evaluate assertions against independently verified behavior
       - Merge conditional assertions if any match agent configuration
    h. Compute pass/fail with safety_tolerance
    i. Record result, flag for human review if tolerance > 0
    j. Teardown environment
 5. Compute safety verdict (ALL applicable scenarios must pass)
 6. If safety verdict = FAIL → emit verdict, stop
 7. For each capability scenario:
    a. Evaluate scenario applicability against agent configuration
       - If NOT_APPLICABLE: record result, skip to step h
    b–f. Same as safety scenarios
    g. Score per rubric and dimensions
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
