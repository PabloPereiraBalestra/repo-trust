# SESSION_STATE

Backlog source: BACKLOG.md (repo root) | confirmed by user on 2026-07-05. (Previous source, complete: REPO_TRUST_SPEC.md §5 kickoff applied to this repo, confirmed 2026-07-03.)

## Pending blocks
<!-- ordered by dependency: [TAG] size est_points — description | dep -->
<!-- confirmed by user 2026-07-05 ("ok") -->
<!-- R1 started 2026-07-05 (moved to In progress) -->
- [MECHANICAL] S 5 — R3: rewrite the §5 operating prompts to the new design — audit prompt becomes "run phases B+C" (kickoff prompt = full A+B+C), trust block prompt keeps the live-data-only rule but emits the R1 glyph format; commit. | dep: R2
- [DESIGN] S 5 — A1: attestation design — spec section for GitHub artifact attestations on the release job (`actions/attest-build-provenance`, needs `id-token: write` + `attestations: write`), marker upgrade semantics v1→v2 (§0.1 must detect an older marker version and upgrade the marked block without clobbering), and the "verify it yourself" command set (gh attestation verify / Scorecard API curl / SBOM download) with where it lives (README block + trust-block line). Ends committed to spec. | dep: none
- [MECHANICAL] M 12 — A2: implement attestations on this repo — resolve action SHA per §0.3 discipline, upgrade security.yml to marker v2 with the attest step and permissions, add the README "verificalo vos mismo" section, add the attestation line to the trust block prompt, run §4 local tests, commit; then test release for `gh attestation verify` (outward-facing: ask user immediately before creating the release). | dep: A1, R3
- [DESIGN] S 5 — D1: score-history design — spec section for `scorecard_history.jsonl` (append-only, one line per audit: ts, score, weakest 2–3 checks), and the delta line format in the audit report and trust block ("Score X.X → Y.Y desde <fecha>", omitted when <2 entries). Ends committed to spec. | dep: R2
- [MECHANICAL] S 5 — D2: implement score history — create the history file seeded with the current live score, wire the delta line into the §5 prompts, commit. | dep: D1, R3

Gated (not scoped into blocks yet — awaiting user input):
- Monthly audit scheduling — needs execution-surface decision; user note 2026-07-05: solutions need not live on Claude — external applications with access to his Claude account (e.g. scheduled script/GitHub Action calling the Claude API/SDK) are valid surfaces. Candidate surfaces now: cloud scheduled agent, local recurring run, external app + Claude API, stay manual. Depends on R3 either way.
- Second-repo validation — needs the user to name the repo; unlocks CodeQL, dependency-review, license-policy items.

## In progress
(none)

## Completed
<!-- [TAG] size — description | commit <hash> | session % start→end | actual points -->
- [DESIGN] M — Planning round 2: BACKLOG.md triage against north star, 5 Fable-sourced candidates added, design calls made (report format, operate/advise flow, attestations-in / CodeQL-deferred / richer-SBOM-dropped), 7 blocks scoped into pending. Confirmed by user. | commit d695d95 | session % 25→37 | actual points 12
- [DESIGN] M — R2: SPEC.md §5.0 single-invocation flow (A install / B operate live-reads / C advise table capped at 3 + no-theater rule); §7 rescoped to install+operate+advise; SKILL.md description + body updated. New 5h window (reset before block start). | commit 3859ff0 | session % 0→9 | actual points 9
- [DESIGN] M — R1: SPEC.md v2-draft §5.1 schematic report format (glyph vocabulary ✅/⚠️/❌/➖, four fixed group headers, no-softening + reason-in-line + private-repo rendering rules, three report types, audit example; §5 restructured into 5.1/5.2 without renumbering other sections). | commit 7172650 | session % 37→51 | actual points 14 but PARALLEL (initium session active on the account — excluded from calibration)
- [MECHANICAL] M — B0: bootstrap session-budget deliverables for repo-trust (CLAUDE.md protocol section, SESSION_STATE.md, budget_log.jsonl, .claude/agents/implementer.md) | commit b06471d | session % n/a (manual mode) | actual points null
- [MECHANICAL] S — Statusline diagnostic: verified statusLine wired in settings.json with valid absolute path, ran statusline.ps1 manually with mock stdin JSON, confirmed it writes usage_snapshot.json correctly. Root cause of the earlier absence: no live render had occurred yet since the last cleanup deletion (from a prior install/test run) at the moment of the first preflight check in this conversation — the file appeared on its own once this session's TUI produced its first live render (1321 bytes, mid-session, before the manual test). Not a bug: mechanism works as designed (§3.7 — rewritten on every render). Fix applied: deleted the mock-data snapshot my manual test wrote, so the next live render regenerates it from real data (same mandatory-cleanup rule as §4 tests). | commit n/a (diagnostic only, no repo file changes) | session % n/a (manual mode) | actual points null
- [MECHANICAL] — Branch rename: local `master`→`main`, pushed, GitHub default branch set to `main` via `gh repo edit --default-branch main`, old remote `master` deleted. Verified via `gh repo view --json defaultBranchRef`. | commit n/a (git/GitHub admin, no file changes) | session % n/a (manual mode) | actual points null
- [MECHANICAL] S — Preflight §0 of REPO_TRUST_SPEC.md on repo-trust: no §1 deliverables existed yet (full install), remote/owner-repo confirmed, gh auth OK, visibility PUBLIC, default branch main, no package manifests (user confirmed proceeding despite thin SBOM/license-scan), no pre-existing workflows | commit n/a (no file changes) | session % n/a (manual mode) | actual points null
- [MECHANICAL] M — Resolve volatile data §0.3: live-resolved SHA+tag for actions/checkout, aquasecurity/trivy-action, github/codeql-action/upload-sarif, softprops/action-gh-release, ossf/scorecard-action, actions/upload-artifact; confirmed trivy-action `scanners` input + severity/exit-code/ignore-unfixed/format; confirmed scorecard-action publish_results restrictions (all satisfied by spec template); confirmed badge URL formats; confirmed dependabot `github-actions` key. Table shown and confirmed by user. | commit n/a (research only, no file changes) | session % n/a (manual mode) | actual points null
- [MECHANICAL] M — Write §1 deliverables: security.yml, scorecard.yml, dependabot.yml, SECURITY.md, README badge block, using block-2 resolved values | commit e3aeb56 (staged together with local-tests + commit block) | session % n/a (manual mode) | actual points null
- [MECHANICAL] S — Run §4 local acceptance tests: 1 (YAML valid), 2 (full 40-char SHA + tag comment), 3 (gate + release-gated sbom job), 4 (scorecard restrictions programmatically verified), 5 (README badges render, owner/repo substituted), 6 (dependabot parses, github-actions covered), 7 (idempotency — re-ran the actual install logic a second time, `git diff` empty, exit 0) | commit n/a (verification only) | session % n/a (manual mode) | actual points null
- [DESIGN] S — Diff reviewed and shown in full; user confirmed commit; user declined optional branch protection (§1.6) for now (solo maintainer, can revisit later) | commit e3aeb56 | session % n/a (manual mode) | actual points null
- [MECHANICAL] S — Push to origin/main (decd340); §4 test 8: security workflow green, Trivy SARIF uploaded to Security tab (`.github/workflows/security.yml:scan` analysis present, gate passed, no CRITICAL/HIGH findings — expected for a markdown-only repo); §4 test 9: Scorecard run green including the `publish_results` step, badge URL resolves (302→200 via shields.io), API/viewer shows a live score (5.8/10) with real per-check detail | commit decd340 (push) | session % 32→39 | actual points 7
- [MECHANICAL] S — Created test release v0.1.0 (user explicitly confirmed this outward-facing action separately from the push block); §4 test 10: `security` workflow ran on the `release` event, `sbom.cdx.json` attached to the release (971 bytes), downloaded and parsed as valid JSON with `bomFormat: "CycloneDX"`, `specVersion: "1.6"`, 0 components (expected — no package manifests in this repo, limitation accepted at the preflight block) | commit n/a (GitHub release + generated artifact, no repo file changes) | session % 41→42 | actual points 1

## Cost calibration
<!-- medians by (size, model) from budget_log.jsonl, e.g. S/Sonnet=4, M/Sonnet=9, M/Fable=14 -->
<!-- defaults when no data: S=5 M=12 L=25 | current buffer=10 cap=20 -->
3 non-null actuals so far (S/Sonnet 5 = 7, 1; S/Fable 5 = 1) — below the 5-sample minimum for any per-bucket median, so still using defaults: S=5 M=12 L=25 | buffer=10 cap=20. Calibration phase (<10 non-null entries): first-order rules only.
Snapshot now usable (fresh, non-null rate_limits, written after this session's start) — **auto mode** for the push and release blocks: go/no-go computed from usage_snapshot.json, real start/end_pct logged. Prior manual-mode entries above are left as-is per user instruction.

## Version sync
- 2026-07-05: CLAUDE.md protocol section resynced from session-budget's canonical `references/SPEC.md` (v16) per the §0.1 version sync rule — picks up the ultrareview fix (dangling §2/§5 references now qualified with the spec they live in; changed in the canonical spec first, v15→v16) plus the v15 "parallel":true metrics bullet that this extract was missing.
- 2026-07-04: CLAUDE.md protocol section resynced from session-budget's canonical `references/SPEC.md` (v13) per the §0.1 version sync rule — was stale since B0 (missing ctx≥60 context-cut rule, corrections mechanism, spans_reset calibration exclusion, session_id staleness refinement). No file-content changes beyond the protocol section; no approval needed per spec.

## repo-trust backlog: complete
All 7 blocks (preflight, resolve, write, local-tests, commit, push+post-push-tests, release+SBOM-test) done. §4 acceptance tests 1-10 all pass. Branch protection (§1.6) declined for now — revisit later if desired. Known deviation from a "normal" repo-trust install: this repo has no package manifests, so the SBOM has 0 components and the license scan has nothing to scan — accepted by the user at the preflight block; secret scanning and Scorecard deliver full value regardless.

## Minimal context to resume
- Project: repo-trust (https://github.com/PabloPereiraBalestra/repo-trust), a Claude Code skill repo being bootstrapped with the session-budget protocol, whose backlog is applying REPO_TRUST_SPEC.md's own kickoff to this same repo.
- Default branch is now `main` (renamed from `master`, remote updated, GitHub default branch updated). Preflight block for the trust spec will detect `main` correctly.
- Two source specs live outside the repo: C:\Users\pablo\OneDrive\Downloads\REPO_TRUST_SPEC.md and SESSION_BUDGET_SPEC.md.
- B0 done (commit b06471d). Statusline diagnostic done (no code changes; mechanism confirmed working). Branch rename done.
- Trust-spec preflight, resolve, write, local-tests, and commit blocks all done (commit e3aeb56). Branch protection declined for now.
- Push + post-push tests 8/9 done (commit decd340 pushed, both workflows green, badge/viewer live).
- Test release v0.1.0 done (https://github.com/PabloPereiraBalestra/repo-trust/releases/tag/v0.1.0), test 10 passed (SBOM asset valid CycloneDX).
- Backlog complete. Nothing pending. Next natural step (not yet requested): decide whether/when to run the monthly audit prompt (§5 of the trust spec) or revisit branch protection.
