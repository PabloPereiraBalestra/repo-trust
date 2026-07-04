# SESSION_STATE

Backlog source: REPO_TRUST_SPEC.md §5 kickoff, applied to this same repo (repo-trust) | confirmed by user on 2026-07-03

## Pending blocks
<!-- ordered by dependency: [TAG] size est_points — description | dep -->
- [MECHANICAL] S 6 — Preflight §0 of REPO_TRUST_SPEC.md on repo-trust: system/env/visibility/default-branch/ecosystem checks, no file writes | dep: B0
- [MECHANICAL] M 14 — Resolve volatile data §0.3 against official sources (action SHAs/tags, trivy-action inputs, scorecard publish_results restrictions, badge URL format, dependabot ecosystem key); show resolved table | dep: block 1
- [MECHANICAL] M 12 — Write §1 deliverables (security.yml, scorecard.yml, README badge block, dependabot.yml, SECURITY.md), idempotent | dep: block 2
- [MECHANICAL] S 6 — Run §4 local acceptance tests (non-push) + idempotency re-run | dep: block 3
- [DESIGN] S 5 — Show full diff, get user OK, commit; ask about optional branch protection (§1.6) | dep: block 4

## In progress
- [MECHANICAL] M 12 — B0: bootstrap session-budget deliverables for repo-trust (CLAUDE.md protocol section, SESSION_STATE.md, budget_log.jsonl, .claude/agents/implementer.md). Statusline script + settings.json wiring already installed globally (from prior initium install) — not touched.

## Completed
<!-- [TAG] size — description | commit <hash> | session % start→end | actual points -->

## Cost calibration
<!-- medians by (size, model) from budget_log.jsonl, e.g. S/Sonnet=4, M/Sonnet=9, M/Fable=14 -->
<!-- defaults when no data: S=5 M=12 L=25 | current buffer=10 cap=20 -->
No entries yet in this project's budget_log.jsonl — using defaults: S=5 M=12 L=25 | buffer=10 cap=20.
Session running in **manual mode**: ~/.claude/usage_snapshot.json does not currently exist (statusline script is installed and wired, but no live render has populated it in this session/environment). No automatic go/no-go gating — stop after each block for manual OK.

## Minimal context to resume
- Project: repo-trust (https://github.com/PabloPereiraBalestra/repo-trust), a Claude Code skill repo being bootstrapped with both the session-budget protocol and (as its own backlog) the repo-trust security-signals spec applied to itself.
- Two source specs live outside the repo: C:\Users\pablo\OneDrive\Downloads\REPO_TRUST_SPEC.md and SESSION_BUDGET_SPEC.md.
- B0 in progress: creating the 4 missing project-level deliverables; global statusline already installed (from initium project), do not recreate.
- After B0: 5 blocks remain to actually install the security workflows on repo-trust (see Pending blocks above).
- Manual mode: no usable snapshot, stop after every block for user go-ahead.
