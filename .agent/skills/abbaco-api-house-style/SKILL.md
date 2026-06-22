---
name: abbaco-api-house-style
description: Cross-cutting invariants and anti-patterns that hold the Abbaco API family together — the seven rules every Abbaco API must satisfy, the application/installation/baseline wiring on top of the Mercap Persistent-API-Skeleton, and the deliberate exclusions ("no localization, no JSON-RPC, no Jenkins"). Loaded alongside the focused sub-skills (`abbaco-api-domain-model`, `abbaco-api-persistence`, `abbaco-api-rest`, `abbaco-api-integration-tests`) — see the workspace CLAUDE.md for routing.
---

# Abbaco — API House Style

## What these APIs are

Every Abbaco API is the same RESTful micro-API pattern applied to one bounded context in the Abbaco product (subscriptions, user profiles, owned portfolios, positions, yield curves, …). They are HTTP/JSON services consumed by the Abbaco frontend and by other backend services.

Architecturally they are close cousins of the Pepper APIs, with one decisive difference: **persistence is to a PostgreSQL relational database via Sagan-RDBMS**, not to GemStone. The implication is a dedicated mapping layer that pepper does not need.

```
HTTP JSON API  (Stargate — PersistentAPIApplication + SingleResourceRESTfulController)
     │
     ▼
Kepler CompositeSystem  (built by a SystemInstallation; subsystem registration, interface lookup)
     │
     ▼
<Thing>System  (domain operations: start managing / synchronize / activate / …)
     │
     ▼
Sagan repositories  (RDBMSRepository in prod, InMemoryRepository in tests)
     │
     ▼
PostgreSQL  (schema created by a CreateEmptyRDBMSApplication bootstrap; schema evolution by external SQL scripts)
```

**This stack is built on the Mercap Persistent-API-Skeleton** (`github://mercap/Persistent-API-Skeleton`, a Mercap fork of the ba-st skeleton). The skeleton supplies the application base, the schema bootstrap, the Postgres provider, and the RDBMS mapping base; it transitively pins Stargate, Sagan, Kepler, and Launchpad. The reference projects under `code-reference/` (`abbaco-subscription-api`, `abbaco-user-profile-api`, `positions-api`) follow the **same** stack — read them as working examples of these conventions, not just as domain-shape references.

> **History note.** An earlier draft of this skill family told new APIs to *drop* the skeleton and subclass `StargateApplication` directly, hand-rolling the Sagan wiring. That was aspirational and never adopted — the working services (and every new one) build on the skeleton. The conventions below describe the skeleton-based stack that actually ships.

## The seven invariants

1. **Package split** is always:
   - `BaselineOf<Name>API`
   - `<Thing>-Model` + `<Thing>-Model-Tests`
   - `<Thing>-API-Model` + `<Thing>-API-Model-Tests`

   The runnable application classes (the API app, the empty-RDBMS bootstrap, the `SystemInstallation`) live either in `<Thing>-API-Model` or in a dedicated `<Thing>-API-System` package (the subscription-api style). No GemStone test-extension package; Postgres integration coverage lives inside `<Thing>-API-Model-Tests`, not a separate package.

2. **Dependencies are declared in `BaselineOf<Name>API`, which depends on the Mercap Persistent-API-Skeleton** — `github://mercap/Persistent-API-Skeleton:vN` (pin a concrete tag, e.g. `v9.0.0`). The skeleton provides:
   - `PersistentAPIApplication` — the HTTP API application base.
   - `CreateEmptyRDBMSApplication` — the schema-bootstrap application base.
   - `SinglePostgreSQLDatabaseProviderModuleFactory` — the Postgres repository-provider module.
   - `RDBMSMappingConfiguration` — the mapping base class.
   - `SystemInstallation` (via Kepler) — the composite-system builder seam.

   …and transitively pins Stargate, Sagan, Kepler, Launchpad, Buoy. The baseline loads the skeleton's `PostgreSQL Persistence` group (production code) and `Dependent-SUnit-Extensions` group (Stargate-SUnit, Kepler-SUnit, Launchpad-SUnit for tests). See **Application, installation, and baseline wiring** below and `abbaco-api-integration-tests`.

3. **Domain model** is: a plain value object (`Portfolio`, `Trial`, `BondGroup`, …), an `Identified<Thing>` wrapper that adds a UUID **plus** a `sequentialNumber` slot (Sagan-RDBMS auto-increment PK), a `<Thing>System` (Kepler `SubsystemImplementation`), and a `<Thing>SystemModule` (`SystemModule`). Its unit and user-story tests live in `<Thing>-Model-Tests`. See `abbaco-api-domain-model`.

   > **Naming:** the abbaco skill examples use `<Thing>ManagementSystem` / `<Thing>ManagementModule`; the working services use `<Domain>System` / `<Domain>SystemModule` + a `#<Domain>SystemInterface` Kepler interface symbol. Both are fine — pick one per service and be consistent. The Kepler **module-registration selector** rule is the same either way and is easy to get wrong: see `abbaco-api-domain-model` §4.

4. **Persistence** is a `<Thing>RDBMSMappingConfiguration` (subclass of `RDBMSMappingConfiguration`, from the skeleton) that declares tables, class models, and descriptors. It is applied to a repository created via `RepositoryProviderSystem >> createRepositoryFor:storingObjectsOfType:checkingConflictsAccordingTo:`. The provider behind `#mainDB` is supplied by `SinglePostgreSQLDatabaseProviderModuleFactory` in production and `InMemoryRepositoryProvider new` in tests. Its PostgreSQL integration tests live in `<Thing>-API-Model-Tests`. See `abbaco-api-persistence`.

5. **API** is a `<Thing>RESTfulController` extending Stargate's `SingleResourceRESTfulController`. Routes are declared with reflective `declare<Verb><Thing>Route` methods (Stargate auto-collects every method whose selector starts with `declare` and ends with `Route`). All wiring goes through one `RESTfulRequestHandlerBuilder`. The **application** that hosts the controller is a `PersistentAPIApplication` subclass (see below). Controller and API tests live in `<Thing>-API-Model-Tests`. See `abbaco-api-rest`.

6. **Media types** are vendor-versioned: `application/vnd.mercap.<resource>+json;version=1.0.0`. Declared as an accessor method on the controller (e.g. `portfolioVersion1dot0dot0MediaType`).

7. **Testing is layered**, and each layer lives **with the code it tests**:

   | Layer | Superclass | Home skill |
   |---|---|---|
   | Domain unit (`<Thing>Test`) | `TestCase` | `abbaco-api-domain-model` |
   | Domain user-story (`<Thing>SystemUserStoryTest`) | `SystemBasedUserStoryTest` + `InMemoryRepositoryProvider` | `abbaco-api-domain-model` |
   | Full-stack controller (`<Thing>RESTfulControllerTest`) | `SingleResourceRESTfulControllerTest` | `abbaco-api-rest` |
   | API user-story (real HTTP + `ZnClient`) | `HTTPBasedRESTfulAPITest` / `SystemBasedUserStoryTest` | `abbaco-api-rest` |
   | PostgreSQL integration (swap in the RDBMS provider) | extends the domain user-story / boots the real install path | `abbaco-api-persistence` |
   | Newman/Postman + CI | docker-compose / GitHub Actions | `abbaco-api-integration-tests` |

   CI runs on **GitHub Actions**, not Jenkins. Shared SUnit conventions (`assert:equals:`, domain-specific fixture names, `assertCollection:hasSameElements:`) live in `smalltalk-conventions`.

## Application, installation, and baseline wiring

This is the small amount of glue that turns the model + controller into a runnable, loadable service. It is deliberately thin — the skeleton does the heavy lifting.

### `<Thing>APIApplication` — subclass of `PersistentAPIApplication`

`PersistentAPIApplication` (skeleton) already owns the `rootSystem` ivar and the start/stop lifecycle. Its `basicStartWithin:` calls `installAndStartRootSystem`, whose body is:

```smalltalk
rootSystem := self installation install: self class version.
rootSystem startUp
```

So **do not override `installRootSystem` or hand-build a `CompositeSystem`** — the framework drives it. The subclass only supplies metadata, config, the installation, and the controllers:

```smalltalk
Class { #name : '<Thing>APIApplication', #superclass : 'PersistentAPIApplication', … }

<Thing>APIApplication class >> commandName        ^ '<thing>-api'
<Thing>APIApplication class >> projectName        ^ #<Name>          "resolves the version; was applicationBaselineName pre-v9"
<Thing>APIApplication class >> description         ^ 'I provide a RESTful API over HTTP for …'
<Thing>APIApplication class >> saganConfigurationParameters
    ^ SaganParameterDefinitionProvider saganConfigurationParametersForPostgreSQL
<Thing>APIApplication class >> configurationParameters
    ^ super configurationParameters , self <anyAppSpecificParameters>
<Thing>APIApplication class >> initialize  <ignoreForCoverage>  self initializeVersion

<Thing>APIApplication >> installation        ^ <Thing>SystemInstallation installedBy: self
<Thing>APIApplication >> controllersToInstall ^ { <Thing>RESTfulController workingWith: rootSystem . … }
```

`PersistentAPIApplication` reads config via `self configuration sagan` (the inherited `saganConfiguration`) and `self stargateConfiguration`. Secrets (`PG Password`, JWT secret) are `asSensitive`.

### `<Thing>SystemInstallation` — subclass of `SystemInstallation`

The single source of truth for "how to build this service's composite system", shared by **both** the API app and the empty-RDBMS app:

```smalltalk
<Thing>SystemInstallation class >> installedBy: anApplication  ^ self new initializeInstalledBy: anApplication
<Thing>SystemInstallation >> name                ^ '<Name> API'
<Thing>SystemInstallation >> modulesToInstall
    ^ Array
        with: ( SinglePostgreSQLDatabaseProviderModuleFactory configuredBy: application saganConfiguration )
        with: <Thing>SystemModule
        with: <OtherSystemModule>          "Kepler resolves install order from #dependencies"
<Thing>SystemInstallation >> beAwareOfShutDownOf: aCompositeSystem  "nothing to release today"
```

- **Use `configuredBy:`, not the deprecated `configuredBy:withPoolingOptions:`.** Pooling limits (`Min/Max Idle Sessions Count`, `Max Active Sessions Count`) are Sagan configuration parameters, not a code-level options block.
- The skeleton's `SystemInstallation install:` builds the `CompositeSystem`, registers the module-registration system, and installs each module's `toInstallOn:` result. `SinglePostgreSQLDatabaseProviderModuleFactory`'s `toInstallOn:` is itself a `SystemModule` that registers `RepositoryProviderSystem` under `#mainDB`.

### `<Thing>EmptyRDBMSApplication` — subclass of `CreateEmptyRDBMSApplication`

Deploy-time schema bootstrap. Runs as a **separate container/CLI command before the API container**, creates the schema, exits. This **replaces the old `Create Empty Database` runtime flag** — the deploy pipeline running this container *is* "create the empty database on first deploy."

```smalltalk
Class { #name : '<Thing>EmptyRDBMSApplication', #superclass : 'CreateEmptyRDBMSApplication', … }

<Thing>EmptyRDBMSApplication class >> commandName  ^ '<thing>-empty-rdbms'      "override the generic default"
<Thing>EmptyRDBMSApplication class >> projectName  ^ #<Name>
<Thing>EmptyRDBMSApplication class >> saganConfigurationParameters
    ^ self saganConfigurationParametersForPostgreSQL
<Thing>EmptyRDBMSApplication class >> initialize  <ignoreForCoverage>  self initializeVersion
<Thing>EmptyRDBMSApplication >> installation       ^ <Thing>SystemInstallation installedBy: self
```

The base class drives the rest: `basicStartWithin:` installs via the shared installation, `startUp`, `( rootSystem >> #RepositoryProviderSystem ) prepareForInitialPersistence`, shut down, `exitSuccess`. Its config surface is Sagan-only — no Stargate, no auth.

### `BaselineOf<Name>API` — subclass of `BaselineOf`

```smalltalk
BaselineOf<Name>API >> projectClass  ^ MetacelloCypressBaselineProject

BaselineOf<Name>API >> baseline: spec
    <baseline>
    spec for: #pharo do: [ self setUpDependencies: spec; setUpPackages: spec. … groups … ]

BaselineOf<Name>API >> setUpDependencies: spec
    spec
        baseline: 'PersistentAPISkeleton' with: [ spec repository: 'github://mercap/Persistent-API-Skeleton:v9.0.0' ];
        project: 'Persistent-API-Skeleton-Deployment' copyFrom: 'PersistentAPISkeleton' with: [ spec loads: 'PostgreSQL Persistence' ];
        project: 'Persistent-API-Skeleton-SUnit' copyFrom: 'PersistentAPISkeleton' with: [ spec loads: 'Dependent-SUnit-Extensions' ]

BaselineOf<Name>API >> setUpPackages: spec
    spec
        package: '<Thing>-Model'           with: [ spec requires: #( 'Persistent-API-Skeleton-Deployment' ) ];
        package: '<Thing>-API-Model'       with: [ spec requires: #( '<Thing>-Model' ) ];
        package: '<Thing>-API-System'      with: [ spec requires: #( '<Thing>-API-Model' ) ];   "if used"
        package: '<Thing>-Model-Tests'     with: [ spec requires: #( '<Thing>-Model' 'Persistent-API-Skeleton-SUnit' ) ];
        package: '<Thing>-API-Model-Tests' with: [ spec requires: #( '<Thing>-API-System' 'Persistent-API-Skeleton-SUnit' ) ]
```

- The skeleton version (`v9.0.0`) is pinned **only here** — bumping it is a one-line change.
- `Deployment` group = the runnable applications; `Tests`/`CI` = the test packages. The Docker image loads the `Deployment` group and dispatches on `commandName`.
- **Verify a baseline in the dev image with `Metacello new baseline: '<Name>API'; …; record: 'default'`** and inspect the resolved spec — a full `load` fails in the dev image because the project's own packages aren't on a Monticello repository there (they load from the project's git repo in CI). See `abbaco-api-integration-tests`.

## What is intentionally **not** part of this house style

- **Localization.** Abbaco APIs are not localized today. Do not introduce `localized` / `localizedWithAll:` calls or `locales/{en,es,pt}/*.json` files in new code. Strings stay as-is. (If a project later needs i18n, that's a deliberate, separate decision — the pepper skills already document the pattern.)
- **JSON-RPC.** The legacy abbaco-subscription-api ships a JSON-RPC layer alongside REST. New APIs are REST-first. Only add JSON-RPC if a specific consumer requires it.
- **Hand-rolled `RDBMSRepositoryProviderModule`.** Use the skeleton's `SinglePostgreSQLDatabaseProviderModuleFactory configuredBy:` directly in `modulesToInstall` — the factory *is* the module.
- **`Create Empty Database` runtime flag.** Schema creation is the `<Thing>EmptyRDBMSApplication` container's job, run before the API container — not a boolean checked at API startup.
- **Jenkins.** CI is GitHub Actions; the existing `Jenkinsfile` files in `code-reference/` are being phased out.

Routing to the focused sub-skills lives in the workspace `CLAUDE.md`. This skill carries the cross-cutting "all of these hold together" contract; the sub-skills carry the depth — they're also where canonical-source pointers live when needed.
