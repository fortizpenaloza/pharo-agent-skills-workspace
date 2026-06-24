---
name: abbaco-api-integration-tests
description: Real-container end-to-end (Newman/Postman) testing for an Abbaco API and the containerized delivery it runs on — the `api-tests/` Newman suite driven by docker-compose, the docker-compose deploy chain that runs the empty-RDBMS bootstrap container before the API container, the Dockerfile (commandName dispatch), and `migrations/`. Use when wiring the Postman suite, docker-compose, or the Dockerfile. The automated GitHub Actions CI that loads and runs the in-image suites is a separate skill — see `abbaco-ci`. The in-image Smalltalk test layers live with their code — see `abbaco-api-domain-model`, `abbaco-api-persistence`, `abbaco-api-rest`.
---

# Abbaco API — End-to-End Testing & Delivery (out of the image)

The out-of-image work splits into two skills; **this is not the CI skill**:

- **How every push is loaded and tested (GitHub Actions, smalltalkCI, the Tonel load layout, Pharo-version match, the fresh-image failure playbook) → `abbaco-ci`.**
- **This skill — how the *real container stack* is exercised end-to-end (Newman/Postman) and how the service is packaged and deployed (docker-compose, Dockerfile, `migrations/`).**

The five in-image Smalltalk test layers (unit, domain user-story, controller, API user-story, PostgreSQL integration) live **with the code they test** — see `abbaco-api-domain-model`, `abbaco-api-persistence`, `abbaco-api-rest`.

## 1. The baseline & groups (what the container loads)

`BaselineOf<Name>API` is the Metacello entry point; its structure (skeleton pin, packages, groups) is specified in `abbaco-api-house-style`, and how CI loads it from disk (the Tonel `.project` + `source/.properties` layout) is in `abbaco-ci`. What matters for **delivery** is the groups:

- **`Deployment`** — the runnable applications (the API app + the empty-RDBMS bootstrap). The Docker image loads `Deployment` and dispatches on `commandName` (§3).
- **`Tests` / `CI`** — the test packages; loaded by CI (`abbaco-ci`), not by the deployed image.

## 2. Newman / Postman (`api-tests/`)

End-to-end HTTP tests against the **real container stack**, run by Newman in docker-compose:

```
api-tests/
├── docker-compose.yml      # db + empty-rdbms + api (+ any upstream the SUT depends on)
├── enviroment.json         # Postman environment (note: spelled "enviroment" in existing abbaco code)
├── tests.json              # Postman collection
└── run-tests.sh            # bring up the stack, run newman, tear down
```

One folder per operation (`Querying`, `Creation`, `Updating`, `Deletion`, and `Authorization` once auth lands). Each request asserts at least success, the exact `Content-Type` (vendor media type + version), and `links.self`. Chain requests with `pm.collectionVariables.set(...)`; for action endpoints, pull the action URL from the previous response's `links.<action>` — never concatenate `/cancel` to a base.

**Dynamic-variable trap.** A dynamic var like `{{$guid}}` is re-resolved *every* time Postman interpolates it. If a prerequest stores a value that still contains `{{$guid}}` into a collection variable and a later request interpolates that variable (e.g. into a filter URL), the second interpolation produces a **different** GUID — the stored value and the sent value silently disagree (a filter quietly returns nothing). Resolve it once in the prerequest (`const uid = pm.variables.replaceIn('{{$guid}}')`) and build both the stored value and any later reuse from `uid`.

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

## 3. The docker-compose deploy chain (empty-RDBMS → API)

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

> The Docker image (built/published by the `abbaco-ci` `unit-tests` workflow's `build-and-publish` job) loads the baseline `Deployment` group. So the Docker build is a **fresh load** — it hits the same load-time pitfalls CI does (see `abbaco-ci`: missing `source/.properties`, unregistered system interfaces). If CI's test job is green on a fresh image, the Docker build loads too.

## 4. Common mistakes

- **Putting a `CREATE_EMPTY_DATABASE` env var on the API service** — that flag is gone. Schema creation is the empty-RDBMS container, ordered before the API via `service_completed_successfully`.
- **Adding new routes without updating `tests.json`** — the in-image suites pass, the Newman suite silently ignores the new endpoint, and a broken route ships.
- **Synthesizing an action URL** (`base / 'cancel'`) in a Postman request instead of following `links.<action>` from the prior response — the encoder's hypermedia output is never exercised.
- **Spelling the Postman environment file `environment.json`** — existing abbaco code uses `enviroment.json` (sic); match it so `run-tests.sh --environment` resolves.
- **The empty-RDBMS container not exiting** — if its app doesn't terminate after creating the schema, `service_completed_successfully` never fires and the API never starts. The bootstrap app must run-and-exit.
