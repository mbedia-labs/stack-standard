# stack-standard

Estándar común para todos mis repos (en cualquier organización): ramas, ciclo de despliegue,
Renovate, Node y pnpm. Este repo es la única fuente de verdad; los repos solo lo referencian.

## Ramas

- `main` = producción. `stg` = integración y staging.
- Todo desarrollo sale de `stg` en una rama propia (`feat/*`, `fix/*`…) y vuelve a `stg` con un PR que pasa CI.
- Nadie hace push directo a `main`. Lo actualiza solo la promoción semanal (o un hotfix, que después se fusiona en `stg`).
- Un push a `stg` despliega staging. Un push o una promoción a `main` despliega producción.

## Ciclo semanal

| Cuándo | Qué |
|---|---|
| Lunes antes de las 5:00 (Europe/Madrid) | Renovate abre un PR agrupado con minor, patch, pin y digest contra `stg`. Si el CI pasa, se fusiona solo. Los mayores van en PRs separados, siempre con revisión manual. |
| Jueves 06:00 UTC | `promote-stg-to-main` hace fast-forward de `main` al commit más reciente de `stg` con todo en verde y lanza el deploy de producción. Si `main` ha divergido (hotfix), falla y avisa. |

La promoción lleva a producción **todo** lo que está en verde en `stg`, no solo las dependencias.

Las alertas de seguridad no esperan: Renovate las abre en cualquier momento, sin `minimumReleaseAge`, con automerge.

## Uso en un repo

`renovate.json`:

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": ["github>mbedia-labs/stack-standard"]
}
```

Solo se puede añadir:

- `packageRules` propias del repo, cada una con un `description` que explique el motivo.
- `"baseBranchPatterns": ["main"]` si el repo no tiene `stg` (no despliega).

Nada de horarios, límites ni `rangeStrategy` propios: si algo debe cambiar, se cambia aquí para todos.

`.github/workflows/promote-stg-to-main.yml`:

```yaml
name: Promote stg → main
on:
  workflow_dispatch:
  schedule:
    - cron: "0 6 * * 4"
permissions: {}
concurrency:
  group: promote-stg-to-main
  cancel-in-progress: false
jobs:
  promote:
    permissions:
      contents: write
      checks: read
      actions: write
    uses: mbedia-labs/stack-standard/.github/workflows/promote-stg-to-main.yml@<sha> # vX.Y.Z
    with:
      deploy_workflow: deploy.yml                 # varios: "deploy.yml deploy-web.yml"
      runner: ${{ vars.CI_RUNNER || 'ubuntu-latest' }}
      # deploy_fields: "environment=production"   # inputs extra del deploy
      # pass_sha: true                            # añade sha=<commit promocionado>
```

Fijar siempre por SHA (Renovate lo actualiza con un PR), nunca `@main`. El workflow de deploy debe
aceptar `workflow_dispatch` en `main`.

## Runtime y gestor de paquetes

- Node 24 LTS: `.nvmrc` = `24`, `engines.node` = `>=24`, `node-version-file: .nvmrc` en CI, `FROM node:24-*` en Docker.
- pnpm es el único gestor: `packageManager: pnpm@<última>`. Nada de npm, npx, yarn ni bun. La config de pnpm va en `pnpm-workspace.yaml`.
- `minimumReleaseAge`: 24 h en pnpm y 3 días en Renovate. No se salta nunca.

## Cambios en este repo

Cada cambio va con un tag `vX.Y.Z`. Los repos fijan el workflow por SHA y Renovate les propone el
nuevo. El preset (`default.json`) se lee de `main`, así que cualquier cambio en él afecta a todos
los repos en la siguiente ejecución de Renovate.
