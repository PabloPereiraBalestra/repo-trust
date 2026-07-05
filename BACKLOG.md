# BACKLOG

Ideas for improvements and new functionality. Not committed work — items move to SESSION_STATE.md's pending blocks once actually scoped and scheduled.

## Skill improvements

Changes to what the repo-trust skill installs or does when run against a target repo.

- [ ] Branch protection automation revisited — offered once already (SPEC.md §1.6) and declined for the repo-trust repo itself (solo maintainer). Revisit if collaborators join or Scorecard's branch-protection check becomes worth chasing.
- [ ] Monthly audit as a scheduled prompt — SPEC.md §5 already defines the audit prompt text; today it's run manually. Could become a CronCreate/scheduled routine that runs the audit and reports drift without being asked.
- [ ] CodeQL scanning — explicitly excluded from v1 scope (SPEC.md §7: "future extensions, proposed separately with evidence of need"). Would add static analysis alongside Trivy's vuln/secret/license scan.
- [ ] Signing / attestations (e.g. Sigstore/cosign for release artifacts) — same §7 exclusion. Would strengthen the SBOM's trust chain beyond "attached to the release."
- [ ] License policy enforcement — today Trivy's license scan is report-only. Could add an allow/deny list that gates the release like the vulnerability severity gate does.
- [ ] Richer SBOM for manifest-less repos — SPEC.md §3.3 accepts a thin/empty SBOM when no package manifests exist (e.g. this repo). Investigate whether CycloneDX can usefully describe non-package assets (workflows, skill files) for these cases.
- [ ] Private-repo degraded mode UX — SPEC.md §3.1 defines the fallback (Security-tab-only, no badge) but the user-facing report of what's missing and why could be clearer.
- [ ] Schematic, emoji-formatted reporting — today's reports (install summary, SPEC.md §5 audit/LinkedIn prompts) are plain prose ("3 líneas"). Replace with a consistent structured format: status glyphs (✅ pass / ⚠️ degraded / ❌ fail) per check, grouped under short headers, so a report is scannable in a glance rather than read start to finish.
- [ ] Skill runs + reports + advises, not just installs — SPEC.md §7 scopes the skill to installation; the audit prompt (§5) exists but is a separate manual step bolted on afterward. Fold "run the checks, report live results, propose concrete next improvements" into the core flow so one invocation both installs what's missing *and* operates: surface current findings (Trivy severity counts, Scorecard per-check breakdown, SBOM/release freshness) and recommend specific next actions (e.g. "Scorecard is 5.8 — enabling branch protection would raise the code-review check" or "2 HIGH findings in workflow X — here's the fix"), rather than leaving that analysis to a separately-invoked audit.

## Project roadmap

Work on the repo-trust repo/skill as a project, beyond what any single install does.

- [ ] Second-repo validation — SPEC.md §8 gates "packaging as a skill" on 2 repos validated; confirm which second repo and run the full kickoff there.
- [ ] Automated tests for the skill logic — SPEC.md §4's acceptance tests are currently run manually per install; consider a harness that exercises them against a scratch repo.
- [ ] Usage walkthrough / example — a worked example (screenshots or transcript) of a fresh install for someone evaluating the skill before installing it.
- [ ] Contribution guidelines — none yet; needed if this moves from personal skill to something others install or PR against.
