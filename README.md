# devops-pipelines

Workflows reutilizables de GitHub Actions para los repos de `soft-grobdi`.
No asumen ningún lenguaje o framework — el repo que llama aporta el
Dockerfile, el `docker-compose.yml` y los pasos de lint/tests propios de su
stack; estos workflows aportan build, escaneo, migración y despliegue.

Convención: todo archivo reutilizable lleva el prefijo `global-`.

## Workflows disponibles

| Workflow | Qué hace | Inputs principales |
|---|---|---|
| [`global-docker-build.yml`](.github/workflows/global-docker-build.yml) | Construye una imagen y la sube a GHCR, etiquetada con el SHA del commit | `image-name`, `dockerfile-path`, `context`, `build-args` |
| [`global-security-scan.yml`](.github/workflows/global-security-scan.yml) | gitleaks sobre el código + Trivy sobre la imagen construida | `image-ref`, `severity` |
| [`global-container-smoketest.yml`](.github/workflows/global-container-smoketest.yml) | Levanta el `docker compose` del repo y espera un 2xx en el endpoint de salud | `compose-file`, `health-path`, `port`, `timeout-seconds` |
| [`global-migrate.yml`](.github/workflows/global-migrate.yml) | Corre el comando de migración contra la base real de un ambiente, sin desplegar código | `railway-service`, `environment`, `migrate-command` |
| [`global-deploy-railway.yml`](.github/workflows/global-deploy-railway.yml) | Despliega con la Railway CLI al ambiente indicado | `railway-service`, `environment` |
| [`global-rollback-migrate.yml`](.github/workflows/global-rollback-migrate.yml) | Revierte N lotes de migración. Solo se dispara a mano (`workflow_dispatch`) desde el repo que llama, nunca automáticamente | `railway-service`, `environment`, `steps`, `rollback-command` |

Cada archivo trae en su cabecera un ejemplo mínimo de cómo invocarlo.

## Por qué esto cierra un tipo de bug que Docker local no ve

Estos workflows corren en `ubuntu-latest` con un `actions/checkout` real —
un clon de git sobre Linux, sensible a mayúsculas, igual que el runner de
Railway. Un `docker compose up --build` corrido en Windows lee el disco
local (case-insensitive) y nunca puede ver una ruta que en el índice de git
está registrada con una mayúscula distinta a la del namespace/import que la
referencia. Corriendo aquí, ese tipo de error se ve como fallo de build en
el primer PR.

## Ambientes y secretos, del lado del repo que llama

Cada repo que consume estos workflows define, en **Settings → Environments**:

- `development` — secreto `RAILWAY_TOKEN` con alcance del ambiente de
  desarrollo en Railway. *Required reviewers*: cualquier persona del equipo
  **excepto** quien generó el cambio (usar la protección de
  "prevenir auto-revisión" si el plan de GitHub de la organización la trae;
  si no, dos equipos que nunca compartan autor y aprobador del mismo PR).
- `production` — secreto `RAILWAY_TOKEN` con alcance de producción.
  *Required reviewers*: solo el equipo de release.

El nombre del `environment:` en cada job de estos workflows (`inputs.environment`)
debe coincidir exactamente con el nombre del GitHub Environment del repo que
llama, porque de ahí sale tanto la protección como el secreto.

## Ejemplo completo de consumo (repo Laravel + Railway)

```yaml
# .github/workflows/ci.yml
name: CI

on:
  pull_request:
  push:
    branches: [develop, master]

jobs:
  laravel-checks:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: shivammathur/setup-php@v2
        with:
          php-version: "8.2"
      - run: composer install --prefer-dist --no-progress
      - run: vendor/bin/pint --test
      - run: vendor/bin/phpstan analyse   # Larastan
      - run: php artisan test

  migrations-check:
    runs-on: ubuntu-latest
    services:
      mariadb:
        image: mariadb:10.11
        env:
          MARIADB_ROOT_PASSWORD: root
          MARIADB_DATABASE: testing
        ports: ["3306:3306"]
        options: >-
          --health-cmd="healthcheck.sh --connect --innodb_initialized"
          --health-interval=10s --health-timeout=5s --health-retries=10
    steps:
      - uses: actions/checkout@v4
      - uses: shivammathur/setup-php@v2
        with:
          php-version: "8.2"
      - run: composer install --prefer-dist --no-progress
      - run: php artisan migrate --force
      - run: php artisan migrate:rollback --force

  build:
    uses: soft-grobdi/devops-pipelines/.github/workflows/global-docker-build.yml@v1
    with:
      image-name: pedidos-grobdi
    permissions:
      contents: read
      packages: write

  scan:
    needs: build
    uses: soft-grobdi/devops-pipelines/.github/workflows/global-security-scan.yml@v1
    with:
      image-ref: ${{ needs.build.outputs.image-ref }}
    permissions:
      contents: read
      packages: read

  smoketest:
    needs: build
    uses: soft-grobdi/devops-pipelines/.github/workflows/global-container-smoketest.yml@v1
    with:
      health-path: /up
      port: 8080
```

```yaml
# .github/workflows/cd.yml
name: CD

# Un solo workflow para los dos ambientes: development y production son el
# mismo flujo (migrar → desplegar), solo cambia a qué ambiente apuntan. El
# ambiente se resuelve a partir de la rama que disparó el push.
#
# Rama de git -> ambiente de Railway:
#   develop -> development
#   master  -> production

on:
  push:
    branches: [develop, master]

jobs:
  resolve-environment:
    runs-on: ubuntu-latest
    outputs:
      environment: ${{ steps.map.outputs.environment }}
    steps:
      - id: map
        run: |
          if [ "${{ github.ref_name }}" = "master" ]; then
            echo "environment=production" >> "$GITHUB_OUTPUT"
          else
            echo "environment=development" >> "$GITHUB_OUTPUT"
          fi

  migrate:
    needs: resolve-environment
    uses: soft-grobdi/devops-pipelines/.github/workflows/global-migrate.yml@v1
    with:
      railway-service: pedidos-grobdi
      environment: ${{ needs.resolve-environment.outputs.environment }}
    secrets: inherit

  deploy:
    needs: [resolve-environment, migrate]
    uses: soft-grobdi/devops-pipelines/.github/workflows/global-deploy-railway.yml@v1
    with:
      railway-service: pedidos-grobdi
      environment: ${{ needs.resolve-environment.outputs.environment }}
    secrets: inherit
```

Si `development` y `production` alguna vez necesitan pasos distintos (no solo
distinto ambiente), ese es el momento de separarlos en dos archivos — hasta
ahora no hizo falta.

```yaml
# .github/workflows/rollback-migrate.yml
name: Rollback migrations (manual)

on:
  workflow_dispatch:
    inputs:
      environment:
        type: choice
        options: [development, production]
      steps:
        type: number
        default: 1

jobs:
  rollback:
    uses: soft-grobdi/devops-pipelines/.github/workflows/global-rollback-migrate.yml@v1
    with:
      railway-service: pedidos-grobdi
      environment: ${{ inputs.environment }}
      steps: ${{ inputs.steps }}
    secrets: inherit
```

## Pendiente de verificar al dar de alta el primer repo consumidor

- Sintaxis exacta de la Railway CLI (`railway run`, `railway up`) contra la
  versión vigente — `global-migrate.yml` y `global-deploy-railway.yml` lo
  marcan inline.
- Que la protección "Prevent self-review" de GitHub Environments esté
  disponible en el plan de la organización; si no, ver la alternativa en la
  sección de ambientes de arriba.
- Si conviene que `global-deploy-railway.yml` despliegue la imagen exacta
  que ya construyó y escaneó `global-docker-build.yml`/`global-security-scan.yml`,
  en vez de que `railway up` reconstruya desde el checkout.

## Versionado

Los ejemplos fijan **`@v1`**, no `@master`. La diferencia importa: si los
repos consumidores apuntaran a `@master`, cualquier push a `master` en este
repo afectaría de inmediato al siguiente run de CI/CD de todos ellos —
`pedidos-grobdi` incluido, con producción real de por medio. Fijando un
tag, un cambio aquí no le llega a nadie hasta que ese alguien decide,
a propósito, mover su pin a un tag nuevo.

**Tags flotantes de major** (`v1`, `v2`, ...): se mueven a propósito con
cada release menor/parche dentro de esa major, así los consumidores que
fijan `@v1` reciben mejoras y fixes sin tener que actualizar su pin cada
vez — el mismo patrón que usan `actions/checkout@v4` o cualquier Action
oficial de GitHub. Un cambio incompatible (que rompe los `inputs` de un
workflow existente, por ejemplo) se publica como `v2`, nunca moviendo `v1`.

```bash
# cortar/mover el tag flotante v1 al commit actual, después de mergear a master
git tag -f v1
git push origin v1 --force

# tag de versión exacta, no se mueve nunca (opcional, para tener historial)
git tag v1.0.0
git push origin v1.0.0
```

### Probar un cambio en `devops-pipelines` sin afectar a nadie más

1. Trabajar el cambio en una rama de este repo (`develop`, o una rama de
   feature — no importa el nombre, no es un ambiente desplegable).
2. Desde **un solo** repo consumidor, apuntar temporalmente ese workflow a
   la rama de prueba: `uses: soft-grobdi/devops-pipelines/.github/workflows/global-docker-build.yml@develop`
   (o al SHA exacto del commit, para no depender de que la rama no vuelva a moverse).
3. Correr ese PR/pipeline de prueba y confirmar que funciona end-to-end.
4. Recién ahí: mergear a `master` en `devops-pipelines`, mover el tag `v1`
   (o cortar `v2` si el cambio rompe compatibilidad), y en el repo
   consumidor devolver la referencia a `@v1` — nunca dejarla apuntando a
   una rama.

Ningún otro repo se entera del cambio hasta el paso 4. `master` y las ramas
de trabajo de este repo pueden moverse libremente sin que eso implique un
despliegue en ningún lado — acá no hay ambientes propios, solo definiciones
de workflow.
