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
    uses: soft-grobdi/devops-pipelines/.github/workflows/global-docker-build.yml@main
    with:
      image-name: pedidos-grobdi
    permissions:
      contents: read
      packages: write

  scan:
    needs: build
    uses: soft-grobdi/devops-pipelines/.github/workflows/global-security-scan.yml@main
    with:
      image-ref: ${{ needs.build.outputs.image-ref }}
    permissions:
      contents: read
      packages: read

  smoketest:
    needs: build
    uses: soft-grobdi/devops-pipelines/.github/workflows/global-container-smoketest.yml@main
    with:
      health-path: /up
      port: 8080
```

```yaml
# .github/workflows/cd-development.yml
name: CD · development

on:
  push:
    branches: [develop]

jobs:
  migrate:
    uses: soft-grobdi/devops-pipelines/.github/workflows/global-migrate.yml@main
    with:
      railway-service: pedidos-grobdi
      environment: development
    secrets: inherit

  deploy:
    needs: migrate
    uses: soft-grobdi/devops-pipelines/.github/workflows/global-deploy-railway.yml@main
    with:
      railway-service: pedidos-grobdi
      environment: development
    secrets: inherit
```

```yaml
# .github/workflows/cd-production.yml
name: CD · production

on:
  push:
    branches: [master]

jobs:
  migrate:
    uses: soft-grobdi/devops-pipelines/.github/workflows/global-migrate.yml@main
    with:
      railway-service: pedidos-grobdi
      environment: production
    secrets: inherit

  deploy:
    needs: migrate
    uses: soft-grobdi/devops-pipelines/.github/workflows/global-deploy-railway.yml@main
    with:
      railway-service: pedidos-grobdi
      environment: production
    secrets: inherit
```

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
    uses: soft-grobdi/devops-pipelines/.github/workflows/global-rollback-migrate.yml@main
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

Los ejemplos usan `@main`. Cuando haya más de un repo consumidor conviene
empezar a taguear releases (`@v1`, `@v1.1.0`) para no romper un pipeline en
producción por un cambio en `devops-pipelines` que todavía no se probó en
el repo que lo consume.
