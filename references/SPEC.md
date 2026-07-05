# Repo Trust System — Implementation Spec (v2-draft)

> v2-draft (2026-07-05): adds §5.1 schematic report format. Pending v2 work: operate & advise flow, release attestations, score history. v1 behavior is unchanged where this draft doesn't say otherwise.

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
| `.github/workflows/security.yml` | exists AND contains marker `# repo-trust:security v1` |
| `.github/workflows/scorecard.yml` | exists AND contains marker `# repo-trust:scorecard v1` |
| README badges block | markers `<!-- repo-trust:start -->` / `<!-- repo-trust:end -->` present |
| `.github/dependabot.yml` | exists (any content counts; merge, don't clobber) |
| `SECURITY.md` | exists at repo root |

If ANY item is missing, installation covers only the missing items. Re-running the full installation on an installed repo must be a no-op (§4 test 7).

### 0.2 Environment check

1. **Git + remote**: repo has a GitHub remote. Extract `{owner}/{repo}` from it; never ask for what can be read.
2. **gh CLI**: `gh auth status` succeeds. If not: stop, report, no fallback (visibility and API checks depend on it).
3. **Visibility**: `gh repo view --json visibility`. If **private**: STOP and report. The badge and `publish_results` target a public viewer; private repos run in degraded mode (Security tab only, no badge, no LinkedIn number). Do not install the Scorecard workflow with `publish_results: true` on a private repo without explicit user approval of degraded mode.
4. **Default branch**: `gh repo view --json defaultBranchRef`.
5. **Ecosystems**: detect package manifests (`pyproject.toml`, `requirements*.txt`, `poetry.lock`, `package.json`, `Cargo.toml`, `go.mod`, etc.). If none (e.g. PBIP/TMDL repos): report that the SBOM will be thin and license scan near-empty; secret scanning and Scorecard still apply. Ask whether to proceed.
6. **Existing workflows**: list `.github/workflows/`. Never modify files without the `repo-trust` marker. If a pre-existing workflow already runs Trivy or Scorecard, report and ask before adding a parallel one.

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

All third-party actions are pinned by **full commit SHA** with the version tag as a trailing comment. This is both supply-chain hygiene and rewarded by Scorecard's dependency-pinning check (confirm exact check name in scorecard docs/checks.md when resolving).

---

## 1. Deliverables

All idempotent: check before create, merge before write, never clobber user content.

### 1.1 `.github/workflows/security.yml`

```yaml
# repo-trust:security v1
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
        uses: github/codeql-action/upload-sarif@<RESOLVE:sha> # <RESOLVE:tag>
        with:
          sarif_file: 'trivy-results.sarif'

  sbom:
    if: github.event_name == 'release'
    runs-on: ubuntu-latest
    permissions:
      contents: write
    steps:
      - uses: actions/checkout@<RESOLVE:sha> # <RESOLVE:tag>
      - name: Generate CycloneDX SBOM
        uses: aquasecurity/trivy-action@<RESOLVE:sha> # <RESOLVE:tag>
        with:
          scan-type: 'fs'
          scan-ref: '.'
          format: 'cyclonedx'
          output: 'sbom.cdx.json'
      - name: Attach SBOM to release
        uses: softprops/action-gh-release@<RESOLVE:sha> # <RESOLVE:tag>
        with:
          files: sbom.cdx.json
```

Design decisions (fixed, do not renegotiate per repo): gate fails on CRITICAL/HIGH with `ignore-unfixed: true`; SARIF uploads even when the gate fails (`if: always()`); SBOM is generated per release, not per push.

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

```markdown
<!-- repo-trust:start -->
[![security](<RESOLVE:actions-badge-url>)](https://github.com/{owner}/{repo}/actions/workflows/security.yml)
[![OpenSSF Scorecard](https://api.scorecard.dev/projects/github.com/{owner}/{repo}/badge)](https://scorecard.dev/viewer/?uri=github.com/{owner}/{repo})

Cada release incluye su SBOM (CycloneDX) como asset.
<!-- repo-trust:end -->
```

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

1. **Private repo** = no badge, no public score, no LinkedIn number. Degraded mode is Security-tab-only and must be explicitly approved.
2. **Solo maintainer ceiling**: code-review (and similar collaboration) checks score low structurally. Expected; document, don't chase. The score's upward trend is the content, not a perfect 10.
3. **Repos without package manifests**: SBOM thin, license scan near-empty. Secret scanning and Scorecard still deliver value. State this in the install report.
4. **First Scorecard publish lag**: results and badge can take minutes after the first successful run; a 404 on the badge before the first publish is not a failure.
5. **Red badge on gate failure is by design.** The remediation is fixing the finding or opening a tracking issue, never lowering `severity` or removing `exit-code`.
6. **Never edit workflows lacking the repo-trust marker.**

---

## 4. Acceptance tests

Run all that don't require a push locally; report the rest as pending-first-push:

1. Both workflow files parse as valid YAML (`python3 -c "import yaml,sys; yaml.safe_load(open(sys.argv[1]))" <file>`).
2. Every third-party action reference is a full 40-char SHA with a tag comment.
3. `security.yml` contains the gate (`severity: 'CRITICAL,HIGH'`, `exit-code: '1'`) and the release-gated `sbom` job.
4. `scorecard.yml` satisfies every restriction resolved in §0.3.
5. README renders both badge lines between markers; `{owner}/{repo}` correctly substituted.
6. `dependabot.yml` parses and covers github-actions + all detected ecosystems.
7. **Idempotency**: re-run the full installation → zero diffs.
8. *(post-push)* `security` workflow completes; findings visible in Security tab.
9. *(post-push)* Scorecard run completes with publish step green; `curl -sI` on the badge URL returns 200 and the viewer shows a score.
10. *(post-release)* Test release carries `sbom.cdx.json`; the asset parses as JSON with `"bomFormat": "CycloneDX"`.

---

## 5. Report format & operating prompts

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

*(Prose-format prompts below predate §5.1; their outputs must already be emitted in the §5.1 format. The prompt texts themselves are rewritten in a later v2 change.)*

Kickoff (once per repo):

```
Arrancamos con el protocolo "Repo trust" (REPO_TRUST_SPEC.md).
0. Preflight §0 completo: estado, entorno, visibilidad, ecosistemas. Si el repo es privado, pará y decímelo antes de seguir.
1. Resolvé los datos volátiles (§0.3) contra las fuentes oficiales y mostrame la tabla resuelta.
2. Escribí los deliverables §1 que falten, idempotente, sin tocar workflows ajenos.
3. Corré los tests §4 que no requieren push y listame los pendientes.
4. Mostrame el diff completo y esperá mi OK antes de commitear.
```

Audit (per repo, mensual o cuando algo se vea raro):

```
Auditoría repo-trust: corré el preflight §0.1, verificá que los tests §4 post-push siguen verdes (badge responde, último release tiene SBOM, workflow security verde o con issue abierto), y decime en 3 líneas: estado, score actual de Scorecard, y cualquier deriva respecto del spec.
```

LinkedIn trust block (a demanda, para posts):

```
Bloque de confianza para LinkedIn: leé EN VIVO (1) el score actual de Scorecard del repo vía su API o badge, (2) la última release y su asset sbom.cdx.json, (3) el estado del workflow security. Armá 3 líneas en español: score con link al viewer público, release con link al SBOM, y qué se escanea (vulnerabilidades, secretos, licencias). Si algún dato no se puede leer en vivo, decilo y omitilo. Nada sale de memoria.
```

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

Implement only what this spec defines. No CodeQL, no signing/attestations, no extra hooks or monitors: those are explicit future extensions, proposed separately with evidence of need. When done: list files created/modified, test results, and any deviation with its reason.

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
