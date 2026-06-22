---
name: abbaco-api-integration-tests
description: The out-of-image testing and delivery layer of an Abbaco API — the `BaselineOf<Name>API` Metacello load that CI and Docker drive, the GitHub Actions workflow (`.github/workflows/unit-tests.yml`) that runs the in-image suites against a PostgreSQL sidecar and gates a Docker build/publish job, the Newman/Postman `api-tests/` suite driven by docker-compose, and the docker-compose deploy chain that runs the empty-RDBMS bootstrap container before the API container. Use when wiring CI, the Postman suite, the Dockerfile, or the baseline. The in-image Smalltalk test layers live with their code — see `abbaco-api-domain-model`, `abbaco-api-persistence`, `abbaco-api-rest`.
---

# Abbaco API — Integration & Delivery (out of the image)

The five in-image test layers (unit, domain user-story, controller, API user-story, PostgreSQL integration) live **with the code they test** — see `abbaco-api-domain-model`, `abbaco-api-persistence`, and `abbaco-api-rest`. This skill is everything **outside** the image: how the project loads (the baseline), how CI runs it, how the real container stack is exercised (Newman), and how it deploys (docker-compose + Dockerfile).

## 1. The baseline is the load entry point

`BaselineOf<Name>API` (a `BaselineOf` subclass) is what CI, Docker, and a fresh image all run to materialise the project. Its structure — pinning the **Mercap Persistent-API-Skeleton** (`github://mercap/Persistent-API-Skeleton:vN`) and declaring the `YieldCurves-*`-style packages + groups — is specified in `abbaco-api-house-style` (Application, installation, and baseline wiring). Two operational facts matter here:

- **Groups:** the `Deployment` group is the runnable applications (the API app + the empty-RDBMS bootstrap); `Tests` / `CI` are the test packages. Docker loads `Deployment` and dispatches on `commandName`; CI loads the test groups.
- **Verifying a baseline in the dev image:** `Metacello new baseline: '<Name>API'; onConflict: [ :e | e useLoaded ]; onUpgrade: [ :e | e useLoaded ]; record: 'default'` resolves the spec (dependency repos, group membership, the package DAG) **without** loading — inspect the resolved `MetacelloVersionSpec` to confirm the skeleton pins `vN` and the packages carry the right `requires:`. A full `load` in the dev image fails with `NotFound: <Thing>-Model` because the project's own packages live only in memory there (no Tonel repository); that resolves in CI, where they load from the project's git repo. The dependency half (skeleton + SUnit extensions) resolves offline against what's already in the image.

## 2. CI — GitHub Actions (`.github/workflows/unit-tests.yml`)

CI is **GitHub Actions**, not Jenkins. One job loads the baseline via Smalltalk CI against a PostgreSQL sidecar and runs every in-image group (the PostgreSQL integration tests connect to `localhost:5432` via the mapped host port). A second job builds and publishes the Docker image, gated on `release-candidate`, tags, or explicit dispatch.

```yaml
name: Unit Tests

on:
  push:
    branches: [ release-candidate ]
    tags: [ '**' ]
  pull_request:
  workflow_dispatch:
    inputs:
      push_image:
        description: Whether to build and push the Docker image if tests pass
        required: true
        default: false
        type: boolean

permissions:
  contents: read

env:
  POSTGRES_PASSWORD: secret
  POSTGRES_USER: postgres
  POSTGRES_DB: test

jobs:
  unit-tests:
    runs-on: ubuntu-latest
    name: Unit tests on Pharo64-11
    steps:
      - uses: actions/checkout@v6
      - name: Start PostgreSQL
        run: |
          docker rm --force <Name>-API-postgresql || true
          docker run --name <Name>-API-postgresql -d \
            -p 127.0.0.1:5432:5432 \
            -e POSTGRES_PASSWORD=${{ env.POSTGRES_PASSWORD }} \
            -e POSTGRES_USER=${{ env.POSTGRES_USER }} \
            -e POSTGRES_DB=${{ env.POSTGRES_DB }} \
            postgres:14 \
            -c ssl=on \
            -c ssl_cert_file=/etc/ssl/certs/ssl-cert-snakeoil.pem \
            -c ssl_key_file=/etc/ssl/private/ssl-cert-snakeoil.key
      - name: Wait for PostgreSQL
        run: |
          until docker exec <Name>-API-postgresql pg_isready -U postgres -d test; do sleep 1; done
      - uses: hpi-swa/setup-smalltalkCI@v1
        with:
          smalltalk-image: Pharo64-11
      - name: Load image and run tests
        run: smalltalkci -s Pharo64-11 .smalltalkci/unit-tests.ston
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        timeout-minutes: 15
      - name: Stop PostgreSQL
        if: always()
        run: docker kill <Name>-API-postgresql || true; docker rm --force <Name>-API-postgresql || true

  build-and-publish:
    needs: unit-tests
    if: |
      github.ref == 'refs/heads/release-candidate' ||
      github.ref_type == 'tag' ||
      (github.event_name == 'workflow_dispatch' && inputs.push_image)
    runs-on: ubuntu-latest
    permissions:
      contents: read
      security-events: write
    name: Build and publish Docker image
    steps:
      - uses: actions/checkout@v6
      - id: docker_metadata
        uses: docker/metadata-action@v6
        with:
          images: ${{ vars.REGISTRY }}/abbaco/<thing>-api
          tags: |
            type=ref,event=branch
            type=ref,event=pr
            type=semver,pattern=v{{version}}
            type=semver,pattern=v{{major}}.{{minor}}
            type=semver,pattern=v{{major}}
      - uses: docker/setup-buildx-action@v4
      - uses: docker/login-action@v4
        with:
          registry: ${{ vars.REGISTRY }}
          username: ${{ vars.REGISTRY_USERNAME }}
          password: ${{ secrets.REGISTRY_TOKEN }}
      - uses: docker/build-push-action@v7
        with:
          context: .
          file: ./docker/Dockerfile
          push: true
          tags: ${{ steps.docker_metadata.outputs.tags }}
          labels: ${{ steps.docker_metadata.outputs.labels }}
          provenance: false
          sbom: false
```

Notes:
- Postgres is started with `docker run` (not `services:`) so the integration tests can toggle `prepareForInitialPersistence` / `destroyRepositories` between tests. SSL is on with self-signed certs because Sagan's `setSSL` requires it.
- **`.smalltalkci/unit-tests.ston`** loads `BaselineOf<Name>API` and runs the `<Thing>-Model-Tests` + `<Thing>-API-Model-Tests` groups in one image load. Because the test `setUp` recreates the schema, no separate bootstrap step is needed in CI.
- `build-and-publish` reuses the registry variables (`REGISTRY`, `REGISTRY_USERNAME`, `REGISTRY_TOKEN`).
- Add cron-driven workflows only when the API has its own scheduled maintenance (subscription-api has `cancel-overdue-*.yml`); otherwise omit them.

## 3. Newman / Postman (`api-tests/`)

End-to-end HTTP tests against the **real container stack**, run by Newman in docker-compose:

```
api-tests/
├── docker-compose.yml      # db + empty-rdbms + api (+ any upstream the SUT depends on)
├── enviroment.json         # Postman environment (note: spelled "enviroment" in existing abbaco code)
├── tests.json              # Postman collection
└── run-tests.sh            # bring up the stack, run newman, tear down
```

One folder per operation (`Querying`, `Creation`, `Updating`, `Deletion`, and `Authorization` once auth lands). Each request asserts at least success, the exact `Content-Type` (vendor media type + version), and `links.self`. Chain requests with `pm.collectionVariables.set(...)`; for action endpoints, pull the action URL from the previous response's `links.<action>` — never concatenate `/cancel` to a base.

```bash
#!/usr/bin/env bash
set -eux
export COMPOSE_FILE=docker-compose.yml
export COMPOSE_PROJECT_NAME=$(echo "${1:-api-tests}" | tr '[:upper:]' '[:lower:]' | tr --delete '.')

docker compose up -d db
# wait for the empty-rdbms bootstrap to create the schema and exit 0
docker compose up --build --exit-code-from <thing>-empty-rdbms <thing>-empty-rdbms
# (optional) load fixtures the out-of-scope capture/seed jobs would produce
docker compose up -d --build <thing>-api
sleep 5

docker run --rm \
    --volume "$(pwd)":/etc/newman \
    --network "${COMPOSE_PROJECT_NAME}_default" \
    postman/newman:6-alpine \
    run tests.json --environment enviroment.json \
    --color off --disable-unicode \
    --reporters cli,junit --reporter-junit-export api-test-result.xml \
    || docker compose logs <thing>-api

docker compose down || docker compose kill
```

## 4. The docker-compose deploy chain (empty-RDBMS → API)

The compose file mirrors the real deploy ordering. Schema creation is a **separate container that runs to completion before the API starts** — there is no `CREATE_EMPTY_DATABASE` flag on the API service (see `abbaco-api-house-style`). The API container and the empty-RDBMS container are the **same image** dispatching on `commandName`.

```yaml
services:
  db:
    image: postgres:14
    command: -c ssl=on -c ssl_cert_file=/etc/ssl/certs/ssl-cert-snakeoil.pem -c ssl_key_file=/etc/ssl/private/ssl-cert-snakeoil.key
    environment:
      POSTGRES_PASSWORD: secret
      POSTGRES_USER: postgres
      POSTGRES_DB: test
    healthcheck:                       # used by depends_on conditions below
      test: [ "CMD", "pg_isready", "-U", "postgres", "-d", "test" ]
      interval: 2s
      timeout: 5s
      retries: 15

  <thing>-empty-rdbms:
    build: { context: ../ }
    command: <thing>-empty-rdbms
    depends_on:
      db: { condition: service_healthy }
    environment:                       # Sagan vars only — no Stargate, no auth, no market config
      SAGAN__PG_HOSTNAME: db
      SAGAN__PG_PORT: 5432
      SAGAN__PG_USERNAME: postgres
      SAGAN__PG_PASSWORD: secret
      SAGAN__PG_DATABASE_NAME: test

  <thing>-api:
    build: { context: ../ }
    command: <thing>-api
    depends_on:
      <thing>-empty-rdbms: { condition: service_completed_successfully }
    environment:
      STARGATE__PUBLIC_URL: http://<thing>-api:8080
      STARGATE__PORT: 8080
      STARGATE__OPERATIONS_SECRET: API-tests
      AUTHENTICATION__AUTHENTICATION_SECRET: api-tests-secret   # once auth lands
      SAGAN__PG_HOSTNAME: db
      SAGAN__PG_PORT: 5432
      SAGAN__PG_USERNAME: postgres
      SAGAN__PG_PASSWORD: secret
      SAGAN__PG_DATABASE_NAME: test
    volumes:
      - ./logs/:/opt/pharo/logs/
```

- The empty-RDBMS service must **exit zero** before the API starts — `condition: service_completed_successfully`.
- Env-var names follow the abbaco convention: `STARGATE__PUBLIC_URL`, `SAGAN__PG_HOSTNAME`, `AUTHENTICATION__AUTHENTICATION_SECRET`, etc.
- Pre-loaded fixtures (substituting for out-of-scope capture/seed jobs) load via a SQL file mounted into a `db-init` sidecar or a `docker exec psql` step in `run-tests.sh` **after** the empty-RDBMS container exits.

### Dockerfile & `migrations/`

Standard abbaco Pharo Dockerfile (copy from `code-reference/abbaco-subscription-api/docker/`). The image's entry point dispatches between applications on the CLI argument — `<thing>-api` vs `<thing>-empty-rdbms` — matching each application's `commandName`. `migrations/` holds per-change `step*.sh` SQL scripts for schema *changes* against an existing production DB (see `abbaco-api-persistence` §9); it is empty until the first post-bootstrap schema change, because `<Thing>EmptyRDBMSApplication` is the initial bootstrap.

## 5. Common mistakes

- **Carrying over a Jenkinsfile** — CI is GitHub Actions. Delete legacy `Jenkinsfile`s when copying scaffolding from older projects.
- **Putting a `CREATE_EMPTY_DATABASE` env var on the API service** — that flag is gone. Schema creation is the empty-RDBMS container, ordered before the API via `service_completed_successfully`.
- **Adding new routes without updating `tests.json`** — Pharo CI passes, the Newman suite silently ignores the new endpoint, and a broken route ships.
- **Synthesizing an action URL** (`base / 'cancel'`) in a Postman request instead of following `links.<action>` from the prior response — the encoder's hypermedia output is never exercised.
- **Hard-coding `localhost` in the in-image integration test connection** — read `PG_HOSTNAME` from the environment (see `abbaco-api-persistence` §10) so the same test runs locally and against the CI sidecar.
- **Verifying a baseline with a full `load` in the dev image and reporting the `NotFound` as a defect** — use `record:` + resolved-spec inspection there; a real load needs the project's git repo (CI) or the Tonel sources on disk.
- **Spelling the Postman environment file `environment.json`** — existing abbaco code uses `enviroment.json` (sic); match it so `run-tests.sh --environment` resolves.
