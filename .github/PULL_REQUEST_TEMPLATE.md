<!--
Thanks for contributing to OASIS. Please fill in the sections below before requesting review. Delete sections that don't apply (e.g. the release checklist on non-release PRs).

For background on what spec vs. profile contributions are reviewed against, see CONTRIBUTING.md (if present) or spec/03-profiles.md §3 for profile quality criteria.
-->

## Scope

<!-- Spec change, SI profile change, new profile, tooling, docs-only, or release commit. -->

## What changed

<!-- A short summary of the diff. Don't restate file names — describe the substantive change. -->

## Normative or editorial?

<!--
- Normative: adds, removes, or modifies a MUST / MUST NOT / SHOULD requirement, an enum value, a verdict status, an interface, or the meaning of a defined term. Normative changes require prior design discussion (issue or GitHub Discussion).
- Editorial: typo fix, clarifying rewrite that does not change meaning, broken-link fix, formatting.
- N/A: PR doesn't touch spec/ or profiles/.

If normative, link the motivating issue or discussion below.
-->

- [ ] Normative
- [ ] Editorial
- [ ] N/A (does not touch spec or profiles)

## Motivating issue or discussion

<!-- For normative changes, link the prior discussion. Required for normative spec/ or profile/ changes. -->

Closes #

## Checklist

- [ ] I have read the contribution guidelines.
- [ ] For spec changes: every affected `spec/*.md` file's `**Version:**` header is bumped if this is a normative change.
- [ ] For SI profile changes: ripple updates per `RELEASING.md` are included if this is a normative change.
- [ ] CHANGELOG entry added under `[Unreleased]` (or under the new version heading for release commits).

## Release commits only

<!-- Skip this section on non-release PRs. -->

If this PR is the version-bump commit, the release checklist from `RELEASING.md` §6 applies:

- [ ] Decided what's being released (core / profile / both)
- [ ] Updated embedded version strings per `RELEASING.md` §1 (core) and/or §2 (profile)
- [ ] Updated README surface references (status line for core, profile table row for SI)
- [ ] Added CHANGELOG entry
- [ ] grep for the old version string returns only acceptable hits
