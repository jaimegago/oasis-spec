# OASIS

**Open Assessment Standard for Intelligent Systems**

OASIS is an open standard for evaluating AI agents that interface with external systems. It defines how to declare what an agent should and should not do, and how to verify that the agent's actual behavior matches those declarations.

## Why OASIS

Every existing evaluation framework treats safety as a scored dimension — one metric among many. OASIS treats safety as a gate. One safety failure and the evaluation verdict is FAIL. No partial credit, no capability score. An agent that is capable but unsafe receives nothing.

This is the defining design decision. Safety is not alongside capability — it is a prerequisite.

## Scope

OASIS applies to AI agents that interface with **external systems** — stateful systems with defined interfaces that exist independently of the agent. Kubernetes clusters, databases, trading platforms, robotic controllers, CI/CD pipelines, building management systems.

It does not apply to AI systems that only produce text, images, or other content without interacting with external systems.

## How it works

An OASIS evaluation runs in two phases:

**Phase 1 — Safety gate.** Binary pass/fail scenarios test whether the agent respects declared boundaries. Prompt injection resistance, authority escalation, blast radius containment, declared mode verification. Any failure stops the evaluation.

**Phase 2 — Capability scoring.** Weighted scoring across four dimensions: task completion, reliability, reasoning quality, and observability. Only reached if Phase 1 passes.

Evaluations are structured around **stimuli** — not just prompts, but environmental conditions, injected tool outputs, and temporal events that test how the agent behaves under realistic and adversarial conditions.

## Repo structure

```
oasis-spec/
├── README.md
├── oasis-spec.md                          # Core standard (normative)
├── domains/
│   └── infrastructure/
│       └── oasis-profile-infrastructure.md # First domain profile
├── examples/
│   ├── scenarios/                          # Annotated scenario files
│   └── suites/                             # Annotated suite files
```

## Reading order

1. **This README** — you're here
2. **[oasis-spec.md](oasis-spec.md)** — the core standard. Start with sections 1-4 for the architecture, then section 6 for the scenario format.
3. **[domains/infrastructure/](domains/infrastructure/)** — the first domain profile, showing what a concrete implementation of the standard looks like
4. **[examples/](examples/)** — annotated YAML files illustrating the scenario and suite schemas

## Status

v0.1.0-draft — structure and definitions. The spec is complete enough to read and critique, not yet complete enough to implement against.

## Contributing

Open an issue. The spec needs pressure-testing — edge cases in the external system definition, multi-agent scenarios, continuous evaluation, scenario versioning. See Appendix B of the spec for the full list of open questions.

## License

This work is licensed under [Creative Commons Attribution 4.0 International (CC BY 4.0)](https://creativecommons.org/licenses/by/4.0/).

You are free to share, adapt, and build implementations against this standard. Attribution is required.
