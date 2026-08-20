# Releasing OASIS

This document is the procedure for bumping versions of the OASIS specification and Software Infrastructure (SI) profile in this repository. It applies whenever the core spec version is bumped (any new `v1.0.0-rcN` tag, or eventually `v1.0.0`, `v1.0.1`, etc.) and has a parallel section for SI profile bumps.

Until this document existed, the only documented release procedure (in [.claude/CLAUDE.md](.claude/CLAUDE.md), under "Post-edit workflow: prompting for website release") covered the git tag and the website's `versions.yaml` bump but said nothing about the version strings embedded inside the spec and profile documents — which is why the embedded strings drifted from the tagged versions and a reconciliation pass (commit `0b4829b`) was needed. RELEASING.md closes that gap. It documents the *content* changes that must precede a tag; the tag itself and the website propagation are still covered by [.claude/CLAUDE.md](.claude/CLAUDE.md) and are referenced from §5 below.

## 1. Core spec release

When the core spec version is bumped, update every embedded string below.

| Location | What to update |
|----------|----------------|
| `spec/00-motivation.md` line 3 | `**Version:** <new>` |
| `spec/01-core.md` line 3 | `**Version:** <new>` |
| `spec/02-scenarios.md` line 3 | `**Version:** <new>` |
| `spec/03-profiles.md` line 3 | `**Version:** <new>` |
| `spec/04-execution.md` line 3 | `**Version:** <new>` |
| `spec/05-reporting.md` line 3 | `**Version:** <new>` |
| `spec/06-principles.md` line 3 | `**Version:** <new>` |
| `spec/07-adversarial-verification.md` line 3 | `**Version:** <new>` |
| `spec/08-provider-conformance.md` line 3 | `**Version:** <new>` |
| `README.md` status line (currently around line 106) | `(latest: rcN)` parenthetical in the `## Status: Release Candidate` section. **Note this string does not contain the full version**, so a grep for the previous full version misses it. |
| `CHANGELOG.md` | Add a new dated entry for the new version, listing what changed |

### Should the core bump propagate into the SI profile?

The convention: bump the SI profile's **OASIS Core Dependency** declarations to track the latest core version when no breaking changes have occurred. If the bump is non-breaking, also update:

| Location | What to update |
|----------|----------------|
| `profiles/software-infrastructure/profile.md` line 5 | `**OASIS Core Dependency:** ≥ <new>` |
| `profiles/software-infrastructure/README.md` line 5 | `**OASIS Core Dependency:** ≥ <new>` |
| `profiles/software-infrastructure/provider-conformance.md` line 5 | `**OASIS Core Dependency:** ≥ <new>` |
| `profiles/software-infrastructure/provider-conformance.md`, rest of file | **Every other occurrence of the core version**, not only line 5: the requirement statements in §3 (`must include a version compatible with >=<new>`, the `simplest conformant declaration`, and the runner's abort message) and the `oasis_core_spec_version(s)` values in the JSON examples. Leaving these behind makes the file assert two different required versions at once. `grep -n '<previous>' ` the file and confirm zero hits before committing. |
| `profiles/software-infrastructure/provider-conformance-requirements.yaml` | `# OASIS Core Dependency: >= <new>` comment near the top, and the `oasis_core_dependency: ">=<new>"` field |

If the core bump is breaking and the SI profile has not been adapted yet, leave the SI dependency declarations alone and note this in the CHANGELOG entry.

## 2. SI profile release

When the SI profile version is bumped, update every embedded string below.

| Location | What to update |
|----------|----------------|
| `profiles/software-infrastructure/profile.md` line 3 | `**Version:** <new>` |
| `profiles/software-infrastructure/README.md` line 3 | `**Version:** <new>` |
| `profiles/software-infrastructure/safety-categories.md` line 3 | `**Profile version:** <new>` |
| `profiles/software-infrastructure/capability-categories.md` line 3 | `**Profile version:** <new>` |
| `profiles/software-infrastructure/behavior-definitions.md` line 3 | `**Profile version:** <new>` |
| `profiles/software-infrastructure/interface-types.md` line 3 | `**Profile version:** <new>` |
| `profiles/software-infrastructure/stimulus-library.md` line 3 | `**Profile version:** <new>` |
| `profiles/software-infrastructure/provider-guide.md` line 3 | `**Profile version:** <new>` |
| `profiles/software-infrastructure/provider-conformance.md` line 3 | `**Profile version:** <new>` |
| `profiles/software-infrastructure/provider-conformance-requirements.yaml` | `# Profile version: <new>` comment and the `profile_version: <new>` field near the top |
| `CHANGELOG.md` | Add an entry describing the SI profile bump |
| `README.md` profile table row (currently around line 82) | `RC (v<new>)` cell for the Software Infrastructure row |

## 3. Scenario versioning

Scenario YAMLs **do not** carry a `version` field. The parent profile's version is the single source of truth for scenario versioning.

This was relaxed from a previously-required field in commit `0b4829b` via a schema change in [spec/02-scenarios.md](spec/02-scenarios.md): the file-level `version` is now optional in §1.1 (scenarios) and §3 (suites). Anyone editing a scenario file should **not** reintroduce a `version:` line. Substantive scenario edits motivate a profile version bump rather than a per-scenario bump.

## 4. Tagging and website propagation

The git tag step and the bump of `versions.yaml` in the [`oasis-website`](https://github.com/jaimegago/oasis-website) repo are covered by the **Post-edit workflow: prompting for website release** section in [.claude/CLAUDE.md](.claude/CLAUDE.md). Follow that procedure after the content changes in §1 and §2 are committed.

In short: RELEASING.md is about the *content* changes that must precede a tag; .claude/CLAUDE.md is about the tag itself plus the website's `versions.yaml` bump.

## 5. Verification

After the website deploy completes, run these checks to confirm the bump landed:

- `curl -s https://oasis-spec.dev/spec.md | head -10` — the new version should appear in both the generation comment and the first document's `**Version:**` body line.
- The footer build timestamp on https://oasis-spec.dev should reflect the recent deploy.
- `grep -rn "<old-version>" --include="*.md" --include="*.yaml" .` in this repo should return only acceptable hits: CHANGELOG entry headings, illustrative code blocks, and intentional historical references. No genuinely stale strings.

If `grep` turns up an unexpected hit, fix it in a follow-up commit and consider whether this document needs another row.

## 6. Checklist

Copy this into the release commit message, PR description, or a scratch buffer and tick boxes as you go.

```markdown
- [ ] Decide what's being released (core / profile / both)
- [ ] Update embedded version strings per RELEASING.md §1 (core) and/or §2 (profile)
- [ ] Update README.md surface references (status line for core, profile table row for SI)
- [ ] Add CHANGELOG entry
- [ ] grep for the old version string — only acceptable hits remain
- [ ] Commit with a conventional commit message
- [ ] Cut and push the tag per .claude/CLAUDE.md
- [ ] Bump versions.yaml in oasis-website per .claude/CLAUDE.md
- [ ] Push the website commit
- [ ] Run post-deploy verification (RELEASING.md §5)
```
