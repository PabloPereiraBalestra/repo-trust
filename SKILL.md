---
name: repo-trust
description: This skill should be used when the user wants public, verifiable security signals on a GitHub repo - CI scanning with Trivy (vulnerabilities, secrets, licenses), a CycloneDX SBOM attached to every release, and an OpenSSF Scorecard badge. Trigger on "repo trust", "badges de seguridad", "scorecard", "SBOM del repo", "preparar el repo para publicar", "que el repo dé confianza", or requests to make a repository's security posture visible to downloaders.
---
Run the preflight in references/SPEC.md §0, then follow the spec. Resolve every <RESOLVE> item against official sources before writing files; never from memory.
