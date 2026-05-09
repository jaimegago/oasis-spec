# Contributing to OASIS

OASIS is an open standard. The specification and the Software Infrastructure profile are developed in this repository, and external contributions are welcome — but the contribution process is deliberately structured.

The reason is Goodhart's Law. A safety-evaluation specification only does its job if the bar it sets is meaningful. Vague requirements, untested categories, or hastily-added scenarios produce a standard that agents can pass without being safe — at which point the standard has become an attack surface, not a defense. We would rather move slowly and get this right than ship volume. The process below favors thoughtfulness over throughput on purpose.

## Filing issues

Two issue templates cover the common cases:

- **[Spec feedback](.github/ISSUE_TEMPLATE/spec-feedback.yml)** — for ambiguities, gaps, contradictions, or implementation friction in the spec or in a profile under `profiles/`. Use this if you read or implemented against the spec and something didn't fit. The RC window (see below) is the right time for this.
- **[Profile proposal](.github/ISSUE_TEMPLATE/profile-proposal.yml)** — for proposing a new domain profile (finance, robotics, healthcare, etc.). Scope the proposal here before drafting; this lets reviewers flag overlap, missing prerequisites, or scope concerns before you invest in a PR.

For anything else — tooling, build, repo-maintenance — open a generic issue.

## Proposing a spec change

Spec changes fall into two categories. The contributor self-classifies, and the [PR template](.github/PULL_REQUEST_TEMPLATE.md) asks for the classification explicitly.

**Normative.** A change is normative if it adds, removes, or modifies a MUST / MUST NOT / SHOULD requirement, an enum value, a verdict status, an interface, or the meaning of a defined term. Normative changes require design discussion **before** a PR is opened — typically a GitHub issue or a Discussions thread that works through the rationale, the tradeoffs, and the alignment with existing principles. PRs that introduce normative changes without prior discussion will be asked to start one. This is not gatekeeping; it is to make sure the design space is explored before the diff fixes it in place.

**Editorial.** A change is editorial if it does not alter meaning: typo fixes, clarifying rewrites, broken-link fixes, formatting. Editorial changes can come as direct PRs without prior discussion.

When in doubt, file an issue first. The cost of a quick triage exchange is much lower than the cost of a normative change landing without scrutiny.

A "design discussion" doesn't have to be heavyweight. For a small normative change, an issue that states the problem, the proposed change, the alternatives considered, and the tradeoff is enough. For larger changes — new verdict statuses, new interfaces, new safety categories — a Discussions thread is more appropriate because the back-and-forth tends to be longer.

## Proposing a domain profile

Profile contributions are the primary path for community involvement, and the bar is rigor, not throughput. Profiles define what safety and capability mean for a class of external systems — they encode domain expertise, and a weak profile produces weak verdicts in a domain where weak verdicts are dangerous.

Before drafting:

- Read [`spec/03-profiles.md`](spec/03-profiles.md), particularly §3 on profile quality criteria. Profile reviews check against these criteria.
- Read [`guides/profile-authoring.md`](guides/profile-authoring.md) for the technical shape of a profile — required components, archetype structure, scenario authoring, conformance contract.
- File a [profile proposal issue](.github/ISSUE_TEMPLATE/profile-proposal.yml) to scope the work.

Profiles are reviewed by domain experts. There is no review path that compensates for missing domain expertise, so the proposal asks who would author the profile and what their domain experience is. Profiles are versioned independently from the core spec.

The expected flow is: proposal issue → maintainer and community feedback on scope → drafting (often by a small group of domain experts collaborating on a branch or fork) → PR against this repository.

The first profile (Software Infrastructure) lives in this repository. Future profiles may live in their own repositories — guidance on multi-repo profile hosting will evolve as the second profile lands.

## PR conventions

- One logical change per PR. Bundling unrelated edits makes review slower and rollback harder.
- Reference the motivating issue or discussion. For normative spec or profile changes, this is required.
- Use the [PR template](.github/PULL_REQUEST_TEMPLATE.md). Tag the change as Normative, Editorial, or N/A.
- For normative changes touching `spec/`, bump the affected file's `**Version:**` header. For SI profile changes, follow the ripple-update guidance in [`RELEASING.md`](RELEASING.md).
- Add a [`CHANGELOG.md`](CHANGELOG.md) entry under `[Unreleased]`.

## The RC window

OASIS is currently at v1.0.0-rc1. The structure is complete and the spec has been validated end-to-end against a real agent under the Software Infrastructure profile, but the RC window exists for external feedback before v1.0.0 stability guarantees apply.

Feedback received during this window directly shapes v1.0.0. If you are implementing a provider, authoring a profile, or reading the spec carefully and something doesn't fit — file an issue now. After v1.0.0 the bar for normative change rises substantially.

## Licensing

This repository is licensed under [Creative Commons Attribution 4.0 International (CC BY 4.0)](https://creativecommons.org/licenses/by/4.0/). Contributions are accepted under the same terms: by opening a pull request, you agree to license your contribution under CC BY 4.0.
