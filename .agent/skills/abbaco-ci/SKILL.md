---
name: abbaco-ci
description: Setting up or debugging GitHub Actions CI for an Abbaco Pharo API — the three workflows (unit-tests, loading-groups, markdown-lint), the smalltalkCI `.ston` load specs, the Tonel repository layout the load depends on (`.project` + `source/.properties` + the `export_package` gotcha), Pharo-version alignment with the dev image, and the fresh-image verification discipline that catches load-time bugs the warm dev image hides. Load when wiring CI for a new service, or when CI fails to load/test code that passes in the dev image.
---

# Abbaco API — CI (GitHub Actions + smalltalkCI)

This is the battle-tested CI setup for an Abbaco service, plus the gotchas that make CI fail while the dev image stays green. It is the authoritative reference for the `.github/workflows/` + `.smalltalkci/` layer; the rest of the out-of-image layer (Newman, docker-compose, Dockerfile, baseline structure) lives in `abbaco-api-integration-tests`, and the baseline package itself is specified in `abbaco-api-house-style`.

> **The one rule that prevents most of this pain: never trust the warm dev image.** The MCP/development image has classes installed incrementally and globals initialized by hand, so it passes tests that a **freshly loaded** image fails. CI, Docker, and a clean `Metacello load` all start from a fresh image. **Always verify against a fresh image** — either let CI tell you, or build one locally (§5) — before believing the code loads and the suites pass. The §6 failure playbook is entirely "dev passed, fresh image didn't."

## 1. The three workflows

Copy these from a working service (e.g. `abbaco-subscription-api`) and substitute names. All three trigger on `pull_request`, on push to `release-candidate`, and on `workflow_dispatch`.

### `unit-tests.yml`

One job loads the baseline via smalltalkCI against a PostgreSQL sidecar and runs every in-image group; a second job builds and publishes the Docker image, gated on `release-candidate` / tags / explicit dispatch.

```yaml
name: Unit Tests
on:
  push:
    branches: [ release-candidate ]
    tags: [ '**' ]
  pull_request:
  workflow_dispatch:
    inputs:
      push_image: { description: Build and push the image if tests pass, required: true, default: false, type: boolean }
permissions:
  contents: read
env:
  POSTGRES_PASSWORD: secret
  POSTGRES_USER: postgres
  POSTGRES_DB: test
jobs:
  unit-tests:
    runs-on: ubuntu-latest
    name: Unit tests on Pharo64-13          # ← match the dev image's Pharo version (§3)
    steps:
      - uses: actions/checkout@v7           # ← keep current; was v6/v4 in older templates
      - name: Start PostgreSQL
        run: |
          docker rm --force <Name>-API-postgresql || true
          docker run --name <Name>-API-postgresql -d -p 127.0.0.1:5432:5432 \
            -e POSTGRES_PASSWORD=${{ env.POSTGRES_PASSWORD }} -e POSTGRES_USER=${{ env.POSTGRES_USER }} \
            -e POSTGRES_DB=${{ env.POSTGRES_DB }} postgres:14 \
            -c ssl=on -c ssl_cert_file=/etc/ssl/certs/ssl-cert-snakeoil.pem \
            -c ssl_key_file=/etc/ssl/private/ssl-cert-snakeoil.key
      - name: Wait for PostgreSQL
        run: until docker exec <Name>-API-postgresql pg_isready -U postgres -d test; do sleep 1; done
      - uses: hpi-swa/setup-smalltalkCI@v1
        with: { smalltalk-image: Pharo64-13 }
      - name: Load image and run tests
        run: smalltalkci -s Pharo64-13 .smalltalkci/unit-tests.ston
        env: { GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }} }
        timeout-minutes: 15
      - name: Stop PostgreSQL
        if: always()
        run: docker kill <Name>-API-postgresql || true; docker rm --force <Name>-API-postgresql || true
  build-and-publish:
    needs: unit-tests
    if: github.ref == 'refs/heads/release-candidate' || github.ref_type == 'tag' || (github.event_name == 'workflow_dispatch' && inputs.push_image)
    runs-on: ubuntu-latest
    permissions: { contents: read, security-events: write }
    name: Build and publish Docker image
    steps:
      - uses: actions/checkout@v7
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
        with: { registry: ${{ vars.REGISTRY }}, username: ${{ vars.REGISTRY_USERNAME }}, password: ${{ secrets.REGISTRY_TOKEN }} }
      - uses: docker/build-push-action@v7
        with: { context: ., file: ./docker/Dockerfile, push: true, tags: ${{ steps.docker_metadata.outputs.tags }}, labels: ${{ steps.docker_metadata.outputs.labels }}, provenance: false, sbom: false }
```

- Postgres is `docker run` (not `services:`) so the integration tests can toggle `prepareForInitialPersistence`/`destroyRepositories` between tests; SSL is on with snakeoil certs because Sagan's `setSSL` requires it. Non-SSL connections still work (the sidecar allows both), so the tests connect without `setSSL`.
- **`build-and-publish` consumes `./docker/Dockerfile`** (the Dockerfile/§4.8 stage — see `abbaco-api-integration-tests`). It is skipped on PRs, so a PR is green without the Dockerfile; it first runs (and fails) on the next `release-candidate` push if the Dockerfile is not yet there. Land the Dockerfile before/with the first push to `release-candidate`, or expect that one red job until then.

### `loading-groups.yml`

Matrix-loads the `Deployment` and `Development` baseline groups so load-order / dependency breakage is caught independently of the tests.

```yaml
name: Baseline Groups
on: { push: { branches: [ release-candidate ] }, pull_request: , workflow_dispatch: }
permissions: { contents: read }
jobs:
  group-loading:
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        smalltalk: [ Pharo64-13 ]
        load-spec: [ deployment, development ]
    name: Baseline Groups / ${{ matrix.smalltalk }} + ${{ matrix.load-spec }}
    steps:
      - uses: actions/checkout@v7
      - uses: hpi-swa/setup-smalltalkCI@v1
        with: { smalltalk-image: ${{ matrix.smalltalk }} }
      - name: Load group in image
        run: smalltalkci -s ${{ matrix.smalltalk }} .smalltalkci/loading.${{ matrix.load-spec }}.ston
        env: { GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }} }
        timeout-minutes: 15
```

### `markdown-lint.yml`

```yaml
name: Markdown Lint
on: { push: { branches: [ release-candidate ] }, pull_request: , workflow_dispatch: }
jobs:
  remark-lint:
    name: runner / markdownlint
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
      - uses: reviewdog/action-markdownlint@v0.1
        with: { github_token: ${{ secrets.GITHUB_TOKEN }}, fail_on_error: true, reporter: github-pr-review }
```

No markdownlint config file is needed (default rules); keep READMEs lint-clean. `action-markdownlint` flags only the lines a PR changes, so a new doc must satisfy the default rules in full. When content genuinely can't satisfy a rule (e.g. a wide table row, which can't be wrapped to the line-length limit), scope a `<!-- markdownlint-disable <RULE> -->` / `<!-- markdownlint-enable <RULE> -->` pair around just that block rather than loosening the rule repo-wide.

## 2. The smalltalkCI load specs (`.smalltalkci/`)

```
.smalltalkci/
├── unit-tests.ston          # baseline group 'CI', runs the test packages
├── loading.deployment.ston  # baseline group 'Deployment', no tests
└── loading.development.ston # baseline group 'Development', no tests
```

```smalltalk
"unit-tests.ston"
SmalltalkCISpec {
  #loading : [ SCIMetacelloLoadSpec {
      #baseline : '<Name>API',          "the BaselineOf<Name>API name, no prefix"
      #directory : '../source',          "smalltalkCI opens source/ directly — see §4"
      #load : [ 'CI' ],
      #platforms : [ #pharo ],           "version-agnostic; the workflow picks the image (§3)"
      #failOn : [ #Warning ] } ],
  #testing : { #packages : [ '<Name>*' ] } }
```

`loading.deployment.ston` / `loading.development.ston` are identical but `#load : [ 'Deployment' ]` / `[ 'Development' ]` and `#testing : { #failOnZeroTests : false }` (no test packages). **`#directory : '../source'`** is relative to `.smalltalkci/`; it points at the `source/` Tonel directory, which is why §4 matters.

## 3. Pharo version must match the development image

smalltalkCI loads the project into the Pharo image named by `smalltalk-image:` / `smalltalkci -s`. **If that version differs from the image the code was developed and verified on, you get behavioral differences** (Glorp/Sagan/Iceberg internals differ across Pharo majors) that look like project bugs. The `.ston` is deliberately `#platforms : [ #pharo ]` (version-agnostic) — the *workflow* is the single place the version lives. Check `SystemVersion current` in the dev image and pin the same `Pharo64-NN` in `unit-tests.yml` and `loading-groups.yml`. (This project: **Pharo64-13**.) `Pharo64-13`/`-12`/`-11` are all valid `setup-smalltalkCI` image ids.

## 4. The Tonel layout CI loads from (the `export_package` gotcha)

smalltalkCI's `#directory : '../source'` opens `source/` as a Tonel repository. For that to work the on-disk layout must be exactly:

```
<repo>/
├── .project          # { 'srcDirectory' : 'source', 'tags' : [ 'Mercap' ] }   (STON string keys)
└── source/
    ├── .properties    # { #format : 'tonel', #version : '3.0' }   ← the format marker
    ├── BaselineOf<Name>API/   ← the baseline lives UNDER source/, not at the repo root
    └── <Name>-*/              ← every package
```

- **`export_package` (the Pharo MCP tool) does NOT write `source/.properties`.** It exports the Tonel package directories only; Iceberg normally writes the repository-level `.properties` marker, but a tool-driven export skips it. **Without `source/.properties`, the load fails with `Could not resolve: BaselineOf<Name>API ... filetree:///…/source`** — the reader can't tell the directory is Tonel and falls back to FileTree. Add the marker by hand after exporting.
- **The baseline package lives under `source/`** (same dir as everything else), because Metacello resolves `…/source` and only finds packages inside the configured source dir. A baseline at the repo root is not found.
- Verifying a baseline by `Metacello new baseline: '<Name>API'; …; record: 'default'` in the dev image resolves the spec **without loading** — it does not exercise this layout. A real load needs CI or a local fresh build (§5).

## 5. Build a fresh image locally (`local-image/`)

A local fresh-load build is the fastest way to reproduce CI without waiting on Actions, and to hand someone a runnable image. The pattern: download the **dev-matched** Pharo, store a GitHub token credential (the baseline pulls the private skeleton), load the baseline from the local `source/`, save.

```
local-image/
├── build-image.sh   # curl get.pharo.org/64/130+vm | bash; ./pharo … st load.st; rename image
├── load.st          # IceCredentialStore current storeCredential: (IceTokenCredentials new
│                    #   username: 'x-access-token'; token: (OSEnvironment current at: 'GITHUB_TOKEN'); yourself)
│                    #   forHostname: 'github.com'.
│                    # Metacello new baseline: 'YieldCurvesAPI';
│                    #   repository: 'tonel://', (OSEnvironment current at: 'PROJECT_SOURCE');
│                    #   onConflict: [:e | e useIncoming ]; onWarning: [:w | w resume ]; load.
│                    # Smalltalk snapshot: true andQuit: true   "on success"
└── build/           # git-ignored: downloaded VM + saved image
```

Run: `GITHUB_TOKEN="$(gh auth token)" ./build-image.sh`, then `cd build && ./pharo-ui <Name>API.image` (GUI) or `./pharo <Name>API.image test '<Name>-Model-Tests'` (headless, against a local Postgres). Gotchas learned: headless `NonInteractiveTranscript` does **not** understand `showln:` — use `show:`/`cr`; deps resolve over SSH (`git@github.com:…`), so SSH access to the private skeleton works even without the token, but keep the token path as the portable fallback. Git-ignore `build/`.

## 6. Failure playbook — "passes in dev, fails in a fresh image"

Work top-down; each is a real failure mode from a fresh load that the warm dev image hid.

1. **`Could not resolve: BaselineOf<Name>API … filetree://…/source`** → missing `source/.properties` (§4). Add the Tonel marker.
2. **`I got an error while cloning … authentication error`** during load, but tests still run → usually benign Iceberg clone retries; Metacello resolves via tarball/SSH. Not the failure unless loading actually stops.
3. **All RDBMS tests error in `tearDown` with `relation "<table>" does not exist`** (one specific table) → that system's **interface was never registered on load**, so it never started and never created its table. The fix is a package manifest (`Manifest<Package> class >> initialize` calling `registerInterfaces`) — see `abbaco-api-domain-model` ("`registerInterfaces` must run on package load"). This is the canonical dev-passes/fresh-fails bug: the dev image had the interface registered by hand.
4. **`DescriptorSystem has been copied and cannot longer be configured`** → an artifact of calling `prepareForInitialPersistence` twice on one provider in a probe; not a product bug.
5. **Behavior differs from dev with no code reason** → Pharo version mismatch (§3).

When a test fails only in CI, reproduce with a **fresh local image** (§5) and run the suite against a clean PostgreSQL — do not debug in the warm dev image, which will keep passing. Confirm the suite is green on the fresh image *before* pushing.

## 7. Cross-references

- `abbaco-api-integration-tests` — the rest of out-of-image: baseline structure, Newman/Postman `api-tests/`, docker-compose deploy chain, Dockerfile, `migrations/`.
- `abbaco-api-house-style` — the `BaselineOf<Name>API` package (groups, skeleton pin, `projectClass`).
- `abbaco-api-domain-model` — system interface registration via the package manifest (failure mode #3).
- `abbaco-api-persistence` — the schema lifecycle (`prepareForInitialPersistence` / `destroyRepositories`) the integration tests drive.
