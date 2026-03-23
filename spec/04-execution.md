# OASIS Execution Model

**Version:** 0.3.0-draft

This document defines the agent interface contract, environment model, and execution flow. For foundational concepts, see [Core](01-core.md).

---

## 1. Agent interface contract

For an agent to be evaluable by OASIS, it must expose a minimal interface that the evaluation runner can interact with. The spec does not prescribe the agent's internal architecture.

Required capabilities:

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
 2. Verify environment meets claimed complexity tier requirements
 3. For each safety scenario:
    a. Provision environment per preconditions
    b. Configure agent with declared mode and tools
    c. Apply stimuli
    d. Capture agent actions and responses
    e. Execute independent verification
    f. Evaluate assertions against independently verified behavior
    g. Compute pass/fail with safety_tolerance
    h. Record result, flag for human review if tolerance > 0
    i. Teardown environment
 4. Compute safety verdict (ALL must pass)
 5. If safety verdict = FAIL → emit verdict, stop
 6. For each capability scenario:
    a–e. Same as safety scenarios
    f. Score per rubric and dimensions
 7. Compute capability scores:
    a. Archetype scores (from scenario scores)
    b. Category scores (from archetype scores, domain-defined aggregation)
    c. Core dimension scores (from category scores, domain-defined mapping)
 8. Compute coverage matrix against tier requirements
 9. Emit final verdict with full report
```
