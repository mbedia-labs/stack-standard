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
| Lunes antes de las 5:00 (Europe/Madrid) | Renovate abre un PR agrupado con minor, patch, pin y digest contra `stg`. Si el CI pasa, se fusiona solo. Igual con el runtime de Node y las imágenes Docker (un PR por imagen). Los mayores van en PRs separados, siempre con revisión manual. |
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
- `"baseBranchPatterns": ["main"]` si el repo no tiene `stg`: no despliega, o es un servicio de
  infraestructura sin entorno de staging (ver [Imágenes Docker](#imágenes-docker)).

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

Fijar siempre por SHA (Renovate lo actualiza con un PR), nunca `@main`. Cada workflow de deploy debe
aceptar `workflow_dispatch` en `main`.

Un push hecho con `GITHUB_TOKEN` no dispara workflows, así que la promoción los lanza ella misma. Para
no redesplegar lo que no ha cambiado, aplica el filtro `on.push.paths` (o `paths-ignore`) de cada
workflow a los ficheros cambiados desde el `main` anterior, con las mismas reglas de glob que GitHub. Un
deploy sin filtro se lanza siempre. Por eso hay que listar en `deploy_workflow` todos los deploys de
producción, también los que solo cambian de vez en cuando (por ejemplo una imagen de terceros).

## Runtime y gestor de paquetes

- Node 24 LTS: `.nvmrc` = `24`, `engines.node` = `>=24`, `node-version-file: .nvmrc` en CI, `FROM node:24-*` en Docker.
- pnpm es el único gestor: `packageManager: pnpm@<última>`. Nada de npm, npx, yarn ni bun. La config de pnpm va en `pnpm-workspace.yaml`.
- `minimumReleaseAge`: 24 h en pnpm y 3 días en Renovate. No se salta nunca.

## Imágenes Docker

Toda imagen que se despliega sale de un fichero que Renovate lee, con versión exacta y digest:

| Caso | Dónde vive la versión |
|---|---|
| Código propio | `FROM` del Dockerfile de la app. |
| Imagen de terceros en Cloud Run | `infra/<servicio>/Dockerfile` con solo `FROM <imagen>:<versión>@sha256:…`. El CI la construye, la sube a Artifact Registry y despliega con `gcloud run deploy --image …@<digest>`. Sin YAML de servicio ni `services replace`. |
| Host con docker compose | `image: <imagen>:<versión>@sha256:…` literal en el compose. Sin variables ni `.env` para la imagen. |

- Tag con versión exacta (`3.5.0`, `v2.9.1`). Nunca `latest` ni tags de major (`3`): cambian sin
  aviso y Renovate no tiene versión con la que comparar.
- El `FROM` apunta al registro de origen (Docker Hub, ghcr.io), no a un mirror: Renovate consulta
  las versiones ahí.
- El digest lo añade y mantiene Renovate (`docker:pinDigests`).

Servicios de infraestructura sin staging (solo `main`): el CI del PR arranca la imagen nueva y
comprueba que responde (smoke test). Es la puerta del automerge, porque fusionar despliega a
producción.

## Cambios en este repo

Cada cambio va con un tag `vX.Y.Z`. Los repos fijan el workflow por SHA y Renovate les propone el
nuevo. El preset (`default.json`) se lee de `main`, así que cualquier cambio en él afecta a todos
los repos en la siguiente ejecución de Renovate.
