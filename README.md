# repo-trust

<!-- repo-trust:start -->
[![security](https://github.com/PabloPereiraBalestra/repo-trust/actions/workflows/security.yml/badge.svg)](https://github.com/PabloPereiraBalestra/repo-trust/actions/workflows/security.yml)
[![OpenSSF Scorecard](https://api.scorecard.dev/projects/github.com/PabloPereiraBalestra/repo-trust/badge)](https://scorecard.dev/viewer/?uri=github.com/PabloPereiraBalestra/repo-trust)

Cada release incluye su SBOM (CycloneDX) como asset.
<!-- repo-trust:end -->

Skill de Claude Code para instalar señales de confianza públicas y verificables en un repositorio de GitHub:

- Escaneo de seguridad en CI (vulnerabilidades, secretos y licencias) con resultados visibles en la pestaña Security del repo.
- Un SBOM (CycloneDX) adjunto a cada release.
- Un badge de OpenSSF Scorecard (score 0-10 calculado por un tercero) en el README.

## Uso

Instalar en `~/.claude/skills/repo-trust` (personal, todos los repos) o `.claude/skills/repo-trust` (por proyecto). El detalle completo de la implementación está en [`references/SPEC.md`](references/SPEC.md); `SKILL.md` es solo el punto de entrada.

## Filosofía

Cero datos volátiles asumidos de memoria: versiones de actions, SHAs, nombres de inputs y formatos de URL se resuelven contra la fuente oficial en el momento de ejecución. Ver §0 del spec.
