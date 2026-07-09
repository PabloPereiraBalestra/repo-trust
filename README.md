# repo-trust

<!-- repo-trust:start -->
[![security](https://github.com/PabloPereiraBalestra/repo-trust/actions/workflows/security.yml/badge.svg)](https://github.com/PabloPereiraBalestra/repo-trust/actions/workflows/security.yml)
[![OpenSSF Scorecard](https://api.scorecard.dev/projects/github.com/PabloPereiraBalestra/repo-trust/badge)](https://scorecard.dev/viewer/?uri=github.com/PabloPereiraBalestra/repo-trust)

Every release includes its SBOM (CycloneDX) as an asset, with attested provenance (SLSA).

<details><summary>Verify it yourself</summary>

```sh
# Scorecard score (independent third party, public criteria)
curl -s https://api.scorecard.dev/projects/github.com/PabloPereiraBalestra/repo-trust | jq .score

# SBOM from the latest release
gh release download --repo PabloPereiraBalestra/repo-trust --pattern 'sbom.cdx.json'

# SBOM provenance (who built it, in which workflow, from which commit)
gh attestation verify sbom.cdx.json --repo PabloPereiraBalestra/repo-trust
```

</details>
<!-- repo-trust:end -->

A Claude Code skill for installing public, verifiable trust signals on a GitHub repository:

- Security scanning in CI (vulnerabilities, secrets and licenses) with results visible in the repo's Security tab.
- An SBOM (CycloneDX) attached to every release.
- An OpenSSF Scorecard badge (score 0-10 computed by a third party) in the README.

## Usage

Install at `~/.claude/skills/repo-trust` (personal, all repos) or `.claude/skills/repo-trust` (per project). The full implementation detail is in [`references/SPEC.md`](references/SPEC.md); `SKILL.md` is just the entry point.

## Philosophy

Zero volatile data assumed from memory: action versions, SHAs, input names and URL formats are resolved against the official source at execution time. See §0 of the spec.
