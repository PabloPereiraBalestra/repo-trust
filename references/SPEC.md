# Repo Trust System — Implementation Spec (v2)

> v2 (2026-07-05, finalized on repo-trust itself): adds §5.0 single-invocation flow (install → operate → advise), §5.1 schematic report format, SLSA build-provenance attestation of the release SBOM (security marker v1→v2, §0.1 upgrade semantics) and the README verify-yourself block, §5.3 score history (`scorecard_history.jsonl`, citable trend), §5.2 prompts rewritten to the flow, and §1.7 CodeQL static analysis (target detection incl. the `actions` workflow-analysis language, default-setup conflict handling). All gating acceptance tests (§4, including 10/11/14) passed on this repo's push and v0.2.0 release. v1 behavior is unchanged where v2 doesn't say otherwise.
>
> v2.1 (2026-07-05, ahead of second-repo validation on `initium`): §0.2.7 gains source-file-based CodeQL target detection (a2) alongside manifest-mapping — a repo can have real source in a supported language with zero package manifests, discovered exactly this way on `initium`. New §0.2.8 defines per-deliverable behavior in degraded (private-repo) mode instead of treating degraded mode as a blanket skip: security.yml and codeql.yml install at full value regardless of visibility; scorecard.yml installs without `publish_results` if it can still produce a local result, else is skipped with reason; the README block becomes a private-repo notice (§1.3) instead of the badge template. Tests 15-16 added.
>
> v2.2 (2026-07-06, from installing on `initium` — private repo): security.yml marker v2→v3, adds `actions: read` to the `scan` job. Without it, `upload-sarif` fails on private repos with "Resource not accessible by integration" (confirmed against `github/codeql-action` issue #2117) — the actual on-repo failure that surfaced this, not a hypothetical. New §3.9 documents it; fixed unconditionally (not visibility-gated) since it's harmless on public repos.
>
> v2.3 (2026-07-06, same initium install, next failure in the same run): code scanning (Security tab) itself needs a GitHub Code Security/GHAS license on private repos — confirmed live ("Code scanning is not enabled for this repository"). Broader than the Scorecard-only gate assumed in v2.1: it also blocks Trivy's SARIF upload in security.yml and CodeQL's `analyze` step entirely. security.yml marker v3→v4 and codeql.yml marker v1→v2 add `continue-on-error: true` to the affected steps; §0.2.8 corrected (codeql.yml now conditional on license, not just target detection); new §3.10. Known residual gap noted in BACKLOG: `continue-on-error` can't distinguish "no license" from "CodeQL found something" on an unlicensed repo, since analyze does both in one call.
>
> v2.4 (2026-07-06, closing the v2.3 residual gap): codeql.yml marker v2→v3 adds an unconditional `actions/upload-artifact` step preserving `analyze`'s own `sarif-output` regardless of whether the Code Scanning upload succeeds — real findings survive as a downloadable artifact on unlicensed repos instead of being masked alongside the license error. §0.2.8 revised again: codeql.yml now installs whenever §0.2.7 finds a target, full stop — no longer gated on the license at all, since the artifact makes it worth having either way. No license pre-detection needed anywhere in this design.

Public, verifiable security signals for a GitHub repository: CI security scan (vulnerabilities + secrets + licenses) with results in the repo's Security tab, a CycloneDX SBOM attached to every release, and an OpenSSF Scorecard badge (third-party 0–10 score) in the README.

**Execution surface:** Claude Code, inside the target repo. This spec never runs from claude.ai.

**Portability goal:** works on the first attempt in any public GitHub repo, any ecosystem (or none), whether or not parts are already installed. Every assumption is checked in §0; every volatile fact (action versions, SHAs, input names) is resolved at implementation time against official sources, never asserted from memory. Failure modes degrade to defined fallbacks, never to improvisation.

**Trust model (why each piece exists):**
- A scan run locally is self-reported. Running it in GitHub Actions on a public repo makes logs, config and results auditable by anyone.
- The Scorecard score is computed by a third party's tool (OpenSSF / Linux Foundation) with public criteria and published to their public viewer. That is the citable number.
- A red badge is an honest signal. Never loosen severity thresholds to keep it green; fix or open an issue.

---

## 0. Preflight — run before ANY installation or audit

### 0.1 System state check

Verify each §1 deliverable:

| Deliverable | Check |
|---|---|
| `.github/workflows/security.yml` | exists AND contains marker `# repo-trust:security` (current version: v4) |
| `.github/workflows/scorecard.yml` | exists AND contains marker `# repo-trust:scorecard` (current version: v1) |
| `.github/workflows/codeql.yml` | exists AND contains marker `# repo-trust:codeql` (current version: v3) — only required when §0.2.7 finds at least one analyzable target; ➖ (not a missing deliverable) otherwise. No longer gated on a Code Security license (§0.2.8 revised) — the artifact-upload step (§3.10) preserves real value either way. |
| README badges block | markers `<!-- repo-trust:start -->` / `<!-- repo-trust:end -->` present |
| `.github/dependabot.yml` | exists (any content counts; merge, don't clobber) |
| `SECURITY.md` | exists at repo root |

If ANY item is missing, installation covers only the missing items. Re-running the full installation on an installed repo must be a no-op (§4 test 7).

**Marker version upgrade semantics.** The two workflow files are wholly owned by repo-trust (they exist only because this system wrote them); the README block is owned only between its markers. Upgrade rules:

- Workflow marker version **older** than this spec's template → rewrite the whole file from the current §1 template (report it as an upgrade in the install report, old → new version). Never patch it line-by-line.
- Workflow marker version **equal** → no-op (idempotency, §4 test 7).
- Marker version **newer** than this spec's template → STOP and report: the repo was installed by a newer spec; do not downgrade.
- README block: replace content between markers with the current §1.3 template (unchanged v1 behavior — the markers carry no version; the block content is always current-template).
- Files without any repo-trust marker: never touched, no exceptions (§3.6).

### 0.2 Environment check

1. **Git + remote**: repo has a GitHub remote. Extract `{owner}/{repo}` from it; never ask for what can be read.
2. **gh CLI**: `gh auth status` succeeds. If not: stop, report, no fallback (visibility and API checks depend on it).
3. **Visibility**: `gh repo view --json visibility`. If **private**: STOP and report. The badge and `publish_results` target a public viewer; private repos run in degraded mode (Security tab only, no badge, no LinkedIn number). Do not install the Scorecard workflow with `publish_results: true` on a private repo without explicit user approval of degraded mode.
4. **Default branch**: `gh repo view --json defaultBranchRef`.
5. **Ecosystems**: detect package manifests (`pyproject.toml`, `requirements*.txt`, `poetry.lock`, `package.json`, `Cargo.toml`, `go.mod`, etc.). If none (e.g. PBIP/TMDL repos): report that the SBOM will be thin and license scan near-empty; secret scanning and Scorecard still apply. Ask whether to proceed.
6. **Existing workflows**: list `.github/workflows/`. Never modify files without the `repo-trust` marker. If a pre-existing workflow already runs Trivy or Scorecard, report and ask before adding a parallel one.
7. **CodeQL target detection**: determine what CodeQL has to analyze. Two independent detection paths feed the same target set — a manifest alone or a manifest-less codebase can each trigger a target, and either without the other is common (confirmed 2026-07-05: a real repo can have source in a CodeQL-supported language with zero package manifests).
   - (a) **Manifest-mapped**: map §0.2.5's detected package manifests to CodeQL-supported languages using the current list resolved in §0.3 — do not assume the list from memory, it changes.
   - (a2) **Source-file-mapped**: independently of manifests, scan the repo tree (excluding standard dependency/build directories: `node_modules`, `vendor`, `dist`, `build`, `.venv`, etc.) for file extensions matching a CodeQL-supported language's typical source extensions (e.g. `.py`, `.js`/`.ts`, `.go`, `.java`/`.kt`, `.cs`, `.rb`, `.swift`, `.c`/`.cpp`/`.h`) — confirm the current extension-to-language mapping at §0.3 time, same as (a). A single matching file is enough to add that language as a target; this exists precisely for codebases with real source but no manifest (scripts-only repos, notebooks-adjacent repos, etc.) — do not require a manifest as a gate for source that plainly exists.
   - (b) Independently of (a)/(a2), if `.github/workflows/` is non-empty, `actions` is always an analyzable target (CodeQL analyzes GitHub Actions workflow files as their own language).
   - If the union of (a), (a2) and (b) is empty, report `➖ CodeQL — sin objetivo analizable` and skip §1.7 entirely (parallel to the license-scan-with-no-manifests case in §3.3); this is not drift.
   - (c) Check for a pre-existing GitHub-managed "default setup" for code scanning (`gh api repos/{owner}/{repo}/code-scanning/default-setup`). If enabled: report it, explain that GitHub does not run default setup and an advanced (workflow-file) setup on the same repo simultaneously (confirm the current exact interaction at §0.3 time — this is resolve-at-run-time, GitHub's behavior here has changed before), and ask before disabling default setup via `gh api -X PATCH .../code-scanning/default-setup -f state=not-configured` in favor of the versioned, marker-governed, SHA-pinned advanced workflow (consistent with every other deliverable in this spec). Skip silently only if the user declines — then report CodeQL as not installed by repo-trust, default setup left as-is.
8. **Degraded-mode deliverable scope** (private repos, continuing from §0.2.3): not every §1 deliverable behaves the same under degraded mode, and the gate is broader than Scorecard alone — RESOLVE at §0.3 time (both confirmed live 2026-07-06 against `initium`, a personal-account private repo with no GitHub Advanced Security / Code Security license):
   - Whether `ossf/scorecard-action`'s analysis functions at all against a private repo (it requires GHAS; without it, the action cannot run, published or not).
   - Whether **code scanning itself** (the Security-tab feature that receives SARIF, used by both Trivy's upload in security.yml and by CodeQL entirely) is enabled for the repo — on private repos this also requires a GitHub Code Security license (`gh api repos/{owner}/{repo}` → `security_and_analysis`; `null`/absent on a personal-account repo is a strong signal, but the authoritative check is attempting the upload and reading its error, since the field isn't reliably populated for non-org repos).
   
   Per-deliverable behavior:
   - `security.yml`: **always install**, regardless of visibility or license. The Trivy scan step and its `exit-code`/`severity` gate run and produce a real, honest pass/fail on every push/PR — that value is independent of the Security tab. The "Upload SARIF to Security tab" step gets `continue-on-error: true` (marker v3→v4, §3.10) so a missing Code Security license degrades *that step only* to a soft warning, never the job — the gate itself (Trivy's own exit code, evaluated *before* upload is attempted) still fails the job red on real findings, unaffected.
   - `codeql.yml`: **install whenever §0.2.7 finds a target — no longer gated on the license** (revised from the initial v2.1/v2.2 design, which skipped it entirely when unlicensed). CodeQL's `analyze` step both computes and uploads in one call, so a bare `continue-on-error` alone would silently swallow real findings alongside the license error on an unlicensed repo — but marker v3 (§3.10) adds an unconditional `actions/upload-artifact` step that preserves `analyze`'s own `sarif-output` as a downloadable artifact regardless of whether the Code Scanning upload succeeds. That gives every install real CodeQL value: Security-tab integration where licensed, a downloadable SARIF artifact where not — no license pre-detection required, since the artifact step doesn't depend on Code Scanning at all.
   - `scorecard.yml`: install **without** `publish_results: true` only if §0.3 confirms GHAS makes the analysis runnable at all; **skip entirely** with a stated reason otherwise (this is initium's case — personal account, no GHAS, the action cannot run published or not). Never set `publish_results: true` on a private repo (§0.2.3) — there is no public viewer to publish to.
   - README badges block (§1.3): **skip** — no public badge or public API to link to. Install a short private-repo notice instead, between the same markers, stating what runs (Security tab if licensed, Actions-run-log gate always, SBOM per release if applicable) and that no public signal exists while the repo stays private (ties to §3.1's degraded-mode definition and the §5.1 report's private-repo rendering rule).
   - `dependabot.yml`, `SECURITY.md`: unaffected by visibility, install as usual.
   - `scorecard_history.jsonl` (§5.3): not created if scorecard.yml is skipped (no score to log); if scorecard.yml runs without publishing, log the local score anyway — it is still a real number, just not a citable public one, and the LinkedIn trust block (§5.2) never had access to it in degraded mode regardless (§3.1: no LinkedIn number from a private repo).

### 0.3 Volatile data resolution — MANDATORY before writing any file

Nothing marked `<RESOLVE>` in §1 may be written from memory. Resolve each against its official source at implementation time and show the resolved table to the user before writing files:

| Item | Source |
|---|---|
| `ossf/scorecard-action` latest release tag + commit SHA | github.com/ossf/scorecard-action (releases) |
| `aquasecurity/trivy-action` latest release tag + commit SHA | github.com/aquasecurity/trivy-action (releases) |
| `actions/checkout`, `actions/upload-artifact`, `github/codeql-action/upload-sarif`, `softprops/action-gh-release` current tags + SHAs | each repo's releases |
| trivy-action input name for scanner selection (expected `scanners`) and confirmation that `severity`, `exit-code`, `ignore-unfixed`, `format: cyclonedx` are current inputs | trivy-action README |
| scorecard-action `publish_results` workflow restrictions (approved steps list, permissions rules) | scorecard-action README |
| GitHub Actions workflow badge URL format | GitHub Actions docs |
| dependabot.yml `package-ecosystem` key for each detected ecosystem | GitHub Dependabot docs |
| `actions/attest-build-provenance` latest release tag + commit SHA, input names (`subject-path`), required permissions (`id-token: write`, `attestations: write`) | github.com/actions/attest-build-provenance (releases + README) |
| `gh attestation verify` exact syntax (flags for owner/repo) and public-repo verifiability conditions | GitHub artifact-attestations docs |
| `github/codeql-action/init`, `github/codeql-action/analyze` current tags + SHAs | github/codeql-action releases |
| Current CodeQL-supported language identifiers (for mapping §0.2.7 manifests) and whether `actions` (GitHub Actions workflow analysis) is GA and its exact `languages:` value | GitHub CodeQL docs (supported languages page) |
| Current interaction between default setup and advanced (workflow-based) setup for code scanning on the same repo | GitHub code-scanning / default-setup docs |
| Per detected compiled language (java-kotlin, go, c-cpp, csharp, swift): whether `build-mode: none` is supported or `autobuild`/`manual` is required | GitHub CodeQL docs (per-language build-mode) |
| Current source-extension-to-CodeQL-language mapping (for §0.2.7a2 detection) | GitHub CodeQL docs (supported languages page) |
| Whether Scorecard analysis / `scorecard-action` produces any usable result against a private repo (with or without `publish_results`) | OpenSSF Scorecard docs / scorecard-action README |

All third-party actions are pinned by **full commit SHA** with the version tag as a trailing comment. This is both supply-chain hygiene and rewarded by Scorecard's dependency-pinning check (confirm exact check name in scorecard docs/checks.md when resolving).

---

## 1. Deliverables

All idempotent: check before create, merge before write, never clobber user content.

### 1.1 `.github/workflows/security.yml`

```yaml
# repo-trust:security v4
name: security
on:
  push:
    branches: [<DEFAULT_BRANCH>]
  pull_request:
  release:
    types: [published]

permissions:
  contents: read

jobs:
  scan:
    if: github.event_name != 'release'
    runs-on: ubuntu-latest
    permissions:
      contents: read
      security-events: write
      actions: read
    steps:
      - uses: actions/checkout@<RESOLVE:sha> # <RESOLVE:tag>
      - name: Trivy scan (vuln + secret + license)
        uses: aquasecurity/trivy-action@<RESOLVE:sha> # <RESOLVE:tag>
        with:
          scan-type: 'fs'
          scan-ref: '.'
          format: 'sarif'
          output: 'trivy-results.sarif'
          severity: 'CRITICAL,HIGH'
          ignore-unfixed: true
          exit-code: '1'
          <RESOLVE:scanners-input>: 'vuln,secret,license'
      - name: Upload SARIF to Security tab
        if: always()
        continue-on-error: true
        uses: github/codeql-action/upload-sarif@<RESOLVE:sha> # <RESOLVE:tag>
        with:
          sarif_file: 'trivy-results.sarif'

  sbom:
    if: github.event_name == 'release'
    runs-on: ubuntu-latest
    permissions:
      contents: write
      id-token: write
      attestations: write
    steps:
      - uses: actions/checkout@<RESOLVE:sha> # <RESOLVE:tag>
      - name: Generate CycloneDX SBOM
        uses: aquasecurity/trivy-action@<RESOLVE:sha> # <RESOLVE:tag>
        with:
          scan-type: 'fs'
          scan-ref: '.'
          format: 'cyclonedx'
          output: 'sbom.cdx.json'
      - name: Attest SBOM build provenance
        uses: actions/attest-build-provenance@<RESOLVE:sha> # <RESOLVE:tag>
        with:
          subject-path: 'sbom.cdx.json'
      - name: Attach SBOM to release
        uses: softprops/action-gh-release@<RESOLVE:sha> # <RESOLVE:tag>
        with:
          files: sbom.cdx.json
```

Design decisions (fixed, do not renegotiate per repo): gate fails on CRITICAL/HIGH with `ignore-unfixed: true`; SARIF uploads even when the gate fails (`if: always()`); SBOM is generated per release, not per push; the SBOM is attested with GitHub-native SLSA build provenance (`attest-build-provenance`) **before** it is attached, so every published SBOM asset is verifiable with `gh attestation verify` — GitHub-native attestations over Sigstore/cosign because they need no key management, are free on public repos, and the verify command is a one-liner a stranger can run. Attestation of provenance only covers assets this workflow produces (the SBOM); user-uploaded release assets are out of scope. The `scan` job's `actions: read` permission (added v2→v3, §3.9) is unconditional — harmless on public repos, required on private ones — rather than a visibility-conditional template, matching this spec's preference for one code path over two.

### 1.2 `.github/workflows/scorecard.yml`

Scorecard MUST live in its own workflow file. When `publish_results: true`, the producing workflow has hard restrictions (resolve the current list in §0.3; historically: no top-level env vars, `id-token: write` only on the Scorecard job, Ubuntu-hosted runner, only approved actions as steps). Violating them makes the publish step fail.

```yaml
# repo-trust:scorecard v1
name: scorecard
on:
  branch_protection_rule:
  schedule:
    - cron: '30 3 * * 1'
  push:
    branches: [<DEFAULT_BRANCH>]

permissions: read-all

jobs:
  analysis:
    name: Scorecard analysis
    runs-on: ubuntu-latest
    permissions:
      security-events: write
      id-token: write
    steps:
      - uses: actions/checkout@<RESOLVE:sha> # <RESOLVE:tag>
        with:
          persist-credentials: false
      - name: Run analysis
        uses: ossf/scorecard-action@<RESOLVE:sha> # <RESOLVE:tag>
        with:
          results_file: results.sarif
          results_format: sarif
          publish_results: true
      - name: Upload artifact
        uses: actions/upload-artifact@<RESOLVE:sha> # <RESOLVE:tag>
        with:
          name: scorecard-results
          path: results.sarif
          retention-days: 5
      - name: Upload to code-scanning
        uses: github/codeql-action/upload-sarif@<RESOLVE:sha> # <RESOLVE:tag>
        with:
          sarif_file: results.sarif
```

### 1.3 README badges block

Insert between markers; if markers exist, replace the content between them (upgrade path), never append a second copy. Place directly under the H1 title.

````markdown
<!-- repo-trust:start -->
[![security](<RESOLVE:actions-badge-url>)](https://github.com/{owner}/{repo}/actions/workflows/security.yml)
[![OpenSSF Scorecard](https://api.scorecard.dev/projects/github.com/{owner}/{repo}/badge)](https://scorecard.dev/viewer/?uri=github.com/{owner}/{repo})

Cada release incluye su SBOM (CycloneDX) como asset, con provenance atestada (SLSA).

<details><summary>Verificalo vos mismo</summary>

```sh
# Score de Scorecard (tercero independiente, criterios públicos)
curl -s https://api.scorecard.dev/projects/github.com/{owner}/{repo} | jq .score

# SBOM del último release
gh release download --repo {owner}/{repo} --pattern 'sbom.cdx.json'

# Provenance del SBOM (quién lo construyó, en qué workflow, desde qué commit)
gh attestation verify sbom.cdx.json --repo {owner}/{repo}
```

</details>
<!-- repo-trust:end -->
````

The verify-yourself block exists because a checkable claim is categorically more credible than an asserted one — it is part of the block between markers, so it upgrades with the block (§0.1). Exact `gh` syntax is `<RESOLVE>`-class: confirm against current docs at install time (§0.3). On repos without attestations installed yet (marker < v2), omit the provenance line and command.

**Private-repo variant** (§0.2.8): replace the whole block above with this notice instead — no public badges, no public API, no verify-yourself commands (there is nothing public to check):

````markdown
<!-- repo-trust:start -->
**Seguridad (repo privado):** escaneo de vulnerabilidades/secretos/licencias corriendo en CI (pestaña Security), sin score ni badge público mientras el repo sea privado.
<!-- repo-trust:end -->
````

Same marker pair, same upgrade rule (§0.1) — the *content* is visibility-conditional, decided fresh at every install/upgrade (a repo going public later gets the full block on the next run, not manually).

### 1.4 `.github/dependabot.yml`

Create if absent; if present, merge missing `updates` entries only. Always include `github-actions`; add one block per ecosystem detected in §0.2.5 with the key confirmed in §0.3.

```yaml
version: 2
updates:
  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "weekly"
  # + one block per detected ecosystem
```

### 1.5 `SECURITY.md`

Only if absent. Minimal vulnerability disclosure policy in Spanish (repo audience) or English (if repo docs are English): where to report, expected response time. Feeds Scorecard's security-policy check.

### 1.6 Branch protection (optional, ask first)

Offer once per repo: enable branch protection on the default branch via `gh api` (require PR before merge; status checks once `security` has run at least once). Raises the corresponding Scorecard checks. Solo maintainer can self-merge; that is fine. Skip silently if declined.

### 1.7 `.github/workflows/codeql.yml`

Only written when §0.2.7 finds at least one analyzable target (a mapped language, or `actions` because `.github/workflows/` is non-empty) — not gated on a Code Security license (§0.2.8): the artifact-upload step below preserves real value even without one. One `matrix.language` entry per detected target — this repo, having no package manifests, gets exactly `actions`.

```yaml
# repo-trust:codeql v3
name: codeql
on:
  push:
    branches: [<DEFAULT_BRANCH>]
  pull_request:
  schedule:
    - cron: '0 4 * * 3'

permissions:
  contents: read

jobs:
  analyze:
    name: CodeQL analysis (${{ matrix.language }})
    runs-on: ubuntu-latest
    permissions:
      contents: read
      security-events: write
      actions: read
    strategy:
      fail-fast: false
      matrix:
        language: [<RESOLVE:detected-targets>]
    steps:
      - uses: actions/checkout@<RESOLVE:sha> # <RESOLVE:tag>
      - name: Initialize CodeQL
        uses: github/codeql-action/init@<RESOLVE:sha> # <RESOLVE:tag>
        with:
          languages: ${{ matrix.language }}
          build-mode: <RESOLVE:build-mode-per-language>
      - name: Perform CodeQL analysis
        id: analyze
        continue-on-error: true
        uses: github/codeql-action/analyze@<RESOLVE:sha> # <RESOLVE:tag>
        with:
          category: '/language:${{ matrix.language }}'
      - name: Upload SARIF as artifact
        if: always()
        uses: actions/upload-artifact@<RESOLVE:sha> # <RESOLVE:tag>
        with:
          name: codeql-sarif-${{ matrix.language }}
          path: ${{ steps.analyze.outputs.sarif-output }}
          retention-days: 5
```

Design decisions (fixed, do not renegotiate per repo): weekly schedule (matches Scorecard's cadence, catches newly-disclosed patterns in unchanged code); one matrix entry per detected target rather than one workflow per language, so results stay in a single file under the marker; `build-mode: none` for `actions` and other interpreted/config languages, `autobuild` as the default for compiled languages unless §0.3 resolution says a given language needs `manual` (if `autobuild` later fails for a specific repo, that is a per-repo finding for phase C to surface, not a reason to change this template); `continue-on-error: true` on the analyze step (§3.10) so a missing Code Security license on a private repo degrades gracefully rather than permanently reddening a workflow that can never pass; the unconditional artifact-upload step (`if: always()`, marker v2→v3, §3.10) closes the resulting gap — `analyze`'s own `sarif-output` (the absolute path to its generated SARIF, produced regardless of whether the Code Scanning upload itself succeeds) is preserved as a downloadable artifact on every run, licensed or not, so a repo without a Code Security license still gets real, inspectable CodeQL findings instead of losing them silently to a soft-failed upload — no license pre-detection needed, since the artifact step never depends on Code Scanning succeeding. Findings land in the same code-scanning alerts feed as Trivy's SARIF upload (§1.1) — phase B (§5.0) reads both from one API call, distinguished by `tool.name`; the schematic report (§5.1) keeps them under the single `Escaneo` header, since both are scan results, not a new category.

**Conflict with GitHub's default setup**: this workflow is an "advanced setup". If §0.2.7(c) found default setup enabled and the user approved disabling it, do that via `gh api` *before* this file is written and pushed — writing an advanced-setup workflow while default setup is still on can leave code scanning in a broken or ambiguous state (exact current behavior is `<RESOLVE>` per §0.3).

---

## 2. Fact status

Verified against official sources on 2026-07-03 (re-verify anything older than ~3 months at §0.3 time):

- Scorecard badge URL format `https://api.scorecard.dev/projects/github.com/{owner}/{repo}/badge` and viewer link, badge auto-updates per run, requires `publish_results: true` + `id-token: write` (scorecard / scorecard-action READMEs).
- `publish_results: true` imposes workflow restrictions enforced by the OpenSSF API (scorecard-action README).
- Scorecard scores 0–10 across checks covering branch protection, code review, dependency management, CI, vulnerability disclosure (scorecard docs; per-check details in docs/checks.md).
- trivy `fs` scan supports SARIF output, `severity`, `exit-code`, `ignore-unfixed`; license scanning is opt-in via scanner selection; CycloneDX output supported (trivy docs, trivy-action README).

Everything else in §0.3 is resolve-at-run-time by design.

---

## 3. Known constraints

1. **Private repo** = no badge, no public score, no LinkedIn number. Degraded mode is Security-tab-only and must be explicitly approved. Per-deliverable behavior in degraded mode is defined in §0.2.8 — not every deliverable is simply "skipped".
2. **Solo maintainer ceiling**: code-review (and similar collaboration) checks score low structurally. Expected; document, don't chase. The score's upward trend is the content, not a perfect 10.
3. **Repos without package manifests**: SBOM thin, license scan near-empty. Secret scanning and Scorecard still deliver value. State this in the install report.
4. **First Scorecard publish lag**: results and badge can take minutes after the first successful run; a 404 on the badge before the first publish is not a failure.
5. **Red badge on gate failure is by design.** The remediation is fixing the finding or opening a tracking issue, never lowering `severity` or removing `exit-code`.
6. **Never edit workflows lacking the repo-trust marker.**
7. **CodeQL with zero analyzable targets** (no mapped language, no `.github/workflows/`): §1.7 is skipped and reported ➖, same honesty rule as license scanning with no manifests (§3.3). Its absence is not drift.
8. **CodeQL vs. GitHub's default setup**: the two cannot coexist cleanly on the same repo (§0.2.7c). repo-trust always proposes the advanced (workflow-file) setup, for the same reason every other deliverable here is a versioned file rather than a repo-settings toggle: it is auditable, SHA-pinned, and governed by the marker/upgrade rules in §0.1 — not something a stranger reading the repo has to trust blindly.
9. **`upload-sarif` on private repos needs `actions: read`**: without it the `scan` job's SARIF upload fails with "Resource not accessible by integration" trying to read workflow-run metadata, even with `security-events: write` already granted — a private-repo-only failure mode (confirmed live against `github/codeql-action` issue #2117; the default `GITHUB_TOKEN` is more restricted on private repos, same root cause as §0.3's Scorecard/GHAS finding). Discovered installing on `initium` (2026-07-05); fixed unconditionally in the `security.yml` template (marker v2→v3) rather than gated on visibility.
10. **Code scanning (Security tab) requires a GitHub Code Security license on private repos** — same GHAS-family gate as Scorecard, but broader: it blocks Trivy's SARIF *upload* (not the scan/gate itself) in security.yml, and blocks CodeQL's `analyze` step's upload half entirely (confirmed live: "Code scanning is not enabled for this repository", discovered on `initium` immediately after fixing constraint 9). Fixed in two layers: `continue-on-error: true` on the affected upload/analyze steps (security.yml marker v4, codeql.yml v2) so a missing license degrades to a soft warning instead of a permanently red run — Trivy's actual vulnerability gate (its `exit-code`, evaluated before the upload step) is unaffected; and (codeql.yml marker v2→v3) an unconditional `actions/upload-artifact` step preserving `analyze`'s `sarif-output` regardless of upload outcome, so real CodeQL findings survive as a downloadable artifact on unlicensed repos instead of being silently masked alongside the license error — closing the gap noted when v2 shipped, no license pre-detection needed.

---

## 4. Acceptance tests

Run all that don't require a push locally; report the rest as pending-first-push:

1. All installed workflow files parse as valid YAML (`python3 -c "import yaml,sys; yaml.safe_load(open(sys.argv[1]))" <file>`).
2. Every third-party action reference is a full 40-char SHA with a tag comment.
3. `security.yml` contains the gate (`severity: 'CRITICAL,HIGH'`, `exit-code: '1'`), `actions: read` on the `scan` job (§3.9), and the release-gated `sbom` job with the attest step (`subject-path: 'sbom.cdx.json'`) and `id-token: write` + `attestations: write` permissions on that job only.
4. `scorecard.yml` satisfies every restriction resolved in §0.3.
5. README renders both badge lines and the verify-yourself block between markers; `{owner}/{repo}` correctly substituted everywhere (badges, curl, gh commands).
6. `dependabot.yml` parses and covers github-actions + all detected ecosystems.
7. **Idempotency**: re-run the full installation → zero diffs.
8. *(post-push)* `security` workflow completes; findings visible in Security tab.
9. *(post-push)* Scorecard run completes with publish step green; `curl -sI` on the badge URL returns 200 and the viewer shows a score.
10. *(post-release)* Test release carries `sbom.cdx.json`; the asset parses as JSON with `"bomFormat": "CycloneDX"`.
11. *(post-release)* The downloaded `sbom.cdx.json` verifies: `gh attestation verify sbom.cdx.json --repo {owner}/{repo}` exits 0 and names this repo's `security` workflow as the builder.
12. *(post-audit)* `scorecard_history.jsonl` consistent with the live score per §5.3.
13. `codeql.yml` (when §0.2.7 found a target — regardless of license, §0.2.8): marker present, `matrix.language` covers every detected target with no extras, `build-mode` set per §0.3 resolution for each, permissions match the template exactly (no broader scope), `continue-on-error: true` on the analyze step, unconditional artifact-upload step present and referencing `steps.analyze.outputs.sarif-output` (§3.10).
14. *(post-push)* CodeQL run completes for every matrix language; results appear in code-scanning alerts (`tool.name` = "CodeQL"), distinguishable from Trivy's entries in the same feed.
15. On a manifest-less repo with real source in a CodeQL-supported language: §0.2.7(a2) detects that language as a target (test 13 still applies to the resulting file) — proves source-based detection isn't gated on a package manifest.
16. On a private repo: README block (§1.3) is replaced by the private-repo notice, not the badge template (no `{owner}/{repo}` badge URLs present); `scorecard.yml` is either absent-with-reason or present without `publish_results: true` per §0.2.8 — never `publish_results: true` on a private repo.
17. On a private repo without a Code Security license: `codeql.yml` still installs (v3+) — its `analyze` step's Security-tab upload soft-fails, but a `codeql-sarif-<language>` artifact is present on every run with real content; `security.yml`'s Trivy gate (`exit-code`) still fails the job on real CRITICAL/HIGH findings even though its own SARIF upload step is soft-failing.

---

## 5. Report format & operating prompts

### 5.0 Single-invocation flow: install → operate → advise

Every invocation runs three phases in order. The v1 split — installation as the skill, audit as a separate bolted-on prompt — is superseded: an "audit" is simply phases B+C of the same flow, and a kickoff is A+B+C.

**Phase A — Install what's missing.** §0 preflight in full (including §0.3 volatile-data resolution whenever anything will be written), then the §1 deliverables that §0.1 found missing. On a fully-installed repo this phase is a verified no-op (§4 test 7). Output: the install report (§5.1) when anything was written; otherwise a single line `➖ instalación completa — nada que escribir`.

**Phase B — Operate (live reads; the only write is the §5.3 history append).** Read live, never from memory:

1. `security` workflow: conclusion of the latest run on the default branch (`gh run list --workflow=security.yml`), and open code-scanning alert counts by severity (`gh api /repos/{owner}/{repo}/code-scanning/alerts`) — this single call also carries CodeQL's findings when §1.7 is installed (filter/group by `tool.name` to separate Trivy from CodeQL in the report); if the alerts API is unavailable, degrade to the workflow log and say so. If `codeql.yml` is installed, also read its own latest run conclusion (`gh run list --workflow=codeql.yml`) per matrix language.
2. Scorecard: score and per-check breakdown from the public API (`https://api.scorecard.dev/projects/github.com/{owner}/{repo}`); badge liveness (HTTP 200 on the badge URL). On a successful score read, append to `scorecard_history.jsonl` per §5.3 (the flow's only phase-B write, and it is append-only).
3. Latest release: exists, carries `sbom.cdx.json`, asset parses as CycloneDX; staleness (commits on the default branch newer than the release).
4. Once attestations are installed: provenance attestation present and verifiable on the release assets.

A read that should work and doesn't renders ❌; a check that cannot apply renders ➖ (§5.1) — never silently skipped. Output: the audit report (§5.1).

**Phase C — Advise.** Map phase-B findings to next actions with the table below. Cap at the **3 highest-impact recommendations** per run, ordered by expected effect on the public signal — ten recommendations is noise. Phase C proposes; it never writes files or changes repo settings without explicit user approval in the same conversation.

| Phase-B finding | Recommendation (one per finding) |
|---|---|
| Scorecard SAST check low, `codeql.yml` not installed, §0.2.7 finds a target | offer §1.7 CodeQL install |
| Scorecard Branch-Protection / Code-Review low, >1 active maintainer | offer §1.6 branch protection |
| Scorecard Branch-Protection / Code-Review low, solo maintainer | state the §3.2 structural ceiling; recommend nothing (don't chase) |
| Scorecard Pinned-Dependencies below max | list the unpinned refs, offer SHA-pinning |
| Open CRITICAL/HIGH findings | show count + top finding; offer a fix or a tracking issue (§6.3 48h rule) |
| `security` workflow red | diagnose the failing step, offer the fix; never loosen the gate (§3.5) |
| Release stale (newer commits, old release) | recommend cutting a release so SBOM/attestation stay representative |
| No release yet | recommend a first release — it unlocks the SBOM (and attestation) signal |
| Badge 404 / API empty right after first Scorecard run | report §3.4 first-publish lag as ⚠️; no action |
| Score dropped vs. recorded history | show the delta and which check fell; offer a targeted fix |
| A §0.1 deliverable missing on an installed repo | that is drift: phase A reinstalls it, report ❌→✅ |

Honesty rule for recommendations: never propose theater (self-approved review flows, empty PRs, gaming a check without changing the underlying posture). Recommend what moves the real posture; the score follows it.

### 5.1 Schematic report format

Every user-facing output of this system — install report, audit report, LinkedIn trust block — uses one schematic format instead of prose: glyph-led lines grouped under short headers, scannable in a glance. Explanatory prose, when needed, goes after the report block, never inside it.

**Glyph vocabulary (fixed):**

| Glyph | Meaning | Use when |
|---|---|---|
| ✅ | pass | check verified live and healthy |
| ⚠️ | degraded / warning | works with a stated limitation: thin SBOM (§3.3), first-publish lag (§3.4), structurally low checks (§3.2), pending post-push tests |
| ❌ | fail | gate red, badge dead, deliverable missing, or a live read that should work failed |
| ➖ | not applicable | check cannot apply to this repo and that is acceptable (e.g. license scan with no package manifests) |

Rules:

- Every line: glyph first, one short clause, then the value/link if any.
- Never soften ❌ into ⚠️. Degraded means "working with a stated limitation"; failing means failing. Same honesty rule as the badge (trust model above): the remediation is fixing or opening an issue, never re-labeling.
- ⚠️ and ➖ lines carry their reason in the same line, one clause ("⚠️ SBOM delgado — repo sin manifests de paquetes").
- Private repo (§3.1): Scorecard/badge lines render ❌ with "repo privado — modo degradado, sin score público"; scan lines stay live via the Security tab. This IS the degraded-mode report; no separate format.
- Labels in Spanish (repo audience); glyphs carry the state, so the format survives translation.

**Group headers (fixed order; omit a group only if all its lines would be ➖):**

1. `Escaneo` — security workflow state, Trivy CRITICAL/HIGH counts, secrets
2. `Scorecard` — score, badge liveness, weakest checks
3. `SBOM · Release` — latest release, SBOM asset presence/validity
4. `Política` — SECURITY.md, dependabot, branch protection, license posture

**The three report types:**

1. **Install report** (end of kickoff): one line per §1 deliverable — created / already present (➖ with "ya instalado") / skipped with reason — then each pending post-push test as a ⚠️ line.
2. **Audit report**: one line per live check under the four headers, closing with a single drift line (✅ `sin deriva respecto del spec` or ❌ + what drifted).
3. **LinkedIn trust block**: at most 4 glyph-led lines in Spanish, built exclusively from live reads: score with public-viewer link, latest release with SBOM link, what is scanned, and (once attestations exist) a provenance/verify line. Feed constraints: LinkedIn strips markdown — glyphs, plain text and bare URLs only. A datum that cannot be read live is omitted and the omission stated, never reconstructed from memory.

**Example (audit report):**

```
Escaneo
✅ workflow security verde (main)
✅ 0 hallazgos CRITICAL/HIGH
Scorecard
✅ score 5.8 — badge vivo → https://scorecard.dev/viewer/?uri=github.com/{owner}/{repo}
⚠️ checks débiles: code-review, branch-protection — mantenedor solo (§3.2)
SBOM · Release
✅ v0.1.0 con sbom.cdx.json válido
Política
✅ SECURITY.md presente, dependabot activo
➖ licencias — sin manifests de paquetes
✅ sin deriva respecto del spec
```

### 5.2 Operating prompts

The prompts map onto the §5.0 phases: kickoff = A+B+C, audit = B+C, and the LinkedIn trust block is a standalone on-demand output.

Kickoff (once per repo):

```
Arrancamos con el protocolo "Repo trust" (REPO_TRUST_SPEC.md), flujo completo §5.0 (fases A+B+C). Todas las salidas en formato §5.1.
Fase A — instalar:
0. Preflight §0 completo: estado, entorno, visibilidad, ecosistemas. Si el repo es privado, pará y decímelo antes de seguir.
1. Resolvé los datos volátiles (§0.3) contra las fuentes oficiales y mostrame la tabla resuelta.
2. Escribí los deliverables §1 que falten, idempotente, sin tocar workflows ajenos.
3. Corré los tests §4 que no requieren push y listame los pendientes.
4. Mostrame el diff completo y esperá mi OK antes de commitear.
Fase B — operar: lecturas en vivo según §5.0 (workflow security y alertas, Scorecard con historial §5.3, release y SBOM, attestation si está instalada).
Fase C — aconsejar: mostrame como máximo 3 recomendaciones de la tabla §5.0.
```

Audit (per repo, mensual o cuando algo se vea raro):

```
Auditoría repo-trust: corré las fases B+C de §5.0.
Fase B — lecturas en vivo, nunca de memoria: workflow security (última corrida en el branch default y alertas de code scanning por severidad), Scorecard vía API (score, checks más débiles, badge vivo; con lectura exitosa del score, apendeá a scorecard_history.jsonl según §5.3), última release con sbom.cdx.json válido y su staleness, y attestation de provenance una vez instalada.
Emití el reporte de auditoría en formato §5.1: líneas con glifo bajo los cuatro headers (Escaneo, Scorecard, SBOM · Release, Política), delta de score según §5.3 cuando haya historial, y cerrá con la línea de deriva (✅ sin deriva, o ❌ y qué derivó).
Fase C — decime como máximo 3 recomendaciones de la tabla §5.0, ordenadas por impacto en la señal pública.
```

LinkedIn trust block (a demanda, para posts):

```
Bloque de confianza para LinkedIn: leé todo EN VIVO, nada sale de memoria. Armá como máximo 4 líneas en español encabezadas por glifo, SIN markdown (LinkedIn lo elimina: glifos, texto plano y URLs peladas nomás):
1. Score actual de Scorecard con la URL del viewer público; si el historial §5.3 muestra mejora, agregá la tendencia (solo cuando es positiva).
2. Última release con la URL del asset sbom.cdx.json.
3. Qué se escanea: vulnerabilidades, secretos, licencias.
4. Línea de provenance con el one-liner de gh attestation verify, una vez que las attestations estén instaladas.
Si algún dato no se puede leer en vivo, decime la omisión y omitilo; nunca lo reconstruyas.
```

### 5.3 Score history

The Scorecard score's upward trend is the content (§3.2); this section makes the trend citable instead of anecdotal.

**Storage**: `scorecard_history.jsonl` at the target repo's root. Append-only JSONL, committed to the repo — public and auditable: anyone can check a claimed trend against the git history, and backdating would show in the commit dates. One line per recorded observation:

```json
{"ts":"<ISO local>","score":<n.n>,"weakest":[{"check":"<Scorecard check name>","score":<n>}, ...]}
```

`weakest` holds the up-to-3 lowest-scoring checks at that observation (ties broken alphabetically) — enough to later explain *why* the score moved without storing the full breakdown.

**Append rule** (phase B, after a successful live score read): append when the file has no entries, OR the score differs from the last entry, OR the last entry is ≥28 days old. Otherwise skip — repeated same-score audits within a month add noise, not signal. Never rewrite or delete existing lines; a wrong line is corrected by the next appended observation.

**Rendering**:

- *Audit report* (Scorecard group): with ≥2 entries and a change, the score line carries the delta and since-when — `✅ score 6.4 — antes 5.8 (2026-07-05)`. A **drop** renders ⚠️ with the fallen check named (the §5.0 advice table row "score dropped" then owns the recommendation).
- *LinkedIn trust block*: include the trend line only when the delta is **positive** (`Score OpenSSF: 5.8 → 6.4 desde jul 2026`); with one entry or no improvement, show only the current score. Omitting a negative trend from a post is selection, not fabrication — the current score is always shown live, and the full history stays public in the repo. The audit never omits it.

**Not an install deliverable**: the file is created lazily by the first phase-B run that reads a score; it is absent from §0.1 by design (its absence on a fresh install is not drift).

**§4 test 12** *(post-audit)*: after a phase-B run with a successful score read, `scorecard_history.jsonl` exists, every line parses as JSON matching the schema, and the last line's score equals the live score just read (or the skip condition held).

---

## 6. Success criteria

Per repo, evaluated on any audit:

1. Scorecard badge live, auto-updating, resolving to a public score.
2. 100% of releases since installation carry `sbom.cdx.json`.
3. `security` green on the default branch, or red with an open issue referencing the finding. Never red-and-silent for more than 48h.
4. Every LinkedIn trust block built exclusively from live data.
5. **First-run criterion** (gate for skill packaging): on a fresh public repo, kickoff completes with all §4 local tests green and zero improvisation outside this spec.

---

## 7. Scope

The skill installs (§0–§1, now including CodeQL per §1.7), operates and advises (§5.0) — nothing else. Do not implement beyond what this spec defines: no extra hooks or monitors beyond §1; those remain future extensions. Phase C may *recommend* an out-of-scope extension only when phase-B evidence supports it (that is precisely the "evidence of need" this section has always demanded); implementing it still requires the extension entering the spec first. When done: emit the §5.1 report(s), list files created/modified, test results, and any deviation with its reason.

*(CodeQL was excluded from v1 for lack of evidence of need; brought into v2 scope 2026-07-05 once GitHub's `actions`-language target showed a workflow-only repo is not a null target after all, and the user asked for it explicitly — see §1.7, §3.7–3.8.)*

---

## 8. Packaging as a skill (after 2 repos validated)

This is a **Claude Code** skill (the work happens inside repos), installable at `~/.claude/skills/` (personal, all repos):

```
repo-trust/
├── SKILL.md
└── references/
    └── SPEC.md        ← this document, unchanged
```

Draft SKILL.md frontmatter:

```yaml
---
name: repo-trust
description: This skill should be used when the user wants public, verifiable security signals on a GitHub repo - CI scanning with Trivy (vulnerabilities, secrets, licenses), a CycloneDX SBOM attached to every release, and an OpenSSF Scorecard badge. Trigger on "repo trust", "badges de seguridad", "scorecard", "SBOM del repo", "preparar el repo para publicar", "que el repo dé confianza", or requests to make a repository's security posture visible to downloaders.
---
Run the preflight in references/SPEC.md §0, then follow the spec. Resolve every <RESOLVE> item against official sources before writing files; never from memory.
```

Body stays minimal; the spec is the source of truth (progressive disclosure).
