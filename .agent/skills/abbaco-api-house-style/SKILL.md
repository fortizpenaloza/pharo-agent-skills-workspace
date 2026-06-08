---
name: abbaco-api-house-style
description: Cross-cutting invariants and anti-patterns that hold the Abbaco API family together — the seven rules every Abbaco API must satisfy and the deliberate exclusions ("no localization, no JSON-RPC, no PersistentAPISkeleton wrapper, no Jenkins"). Loaded alongside the focused sub-skills (`abbaco-api-domain-model`, `abbaco-api-persistence`, `abbaco-api-rest-controller`, `abbaco-api-testing`) — see the workspace CLAUDE.md for routing.
---

# Abbaco — API House Style

## What these APIs are

Every Abbaco API is the same RESTful micro-API pattern applied to one bounded context in the Abbaco product (subscriptions, user profiles, owned portfolios, positions, …). They are HTTP/JSON services consumed by the Abbaco frontend and by other backend services.

Architecturally they are close cousins of the Pepper APIs, with one decisive difference: **persistence is to a PostgreSQL relational database via Sagan-RDBMS**, not to GemStone. The implication is a dedicated mapping layer that pepper does not need.

```
HTTP JSON API  (latest Stargate — StargateApplication + SingleResourceRESTfulController)
     │
     ▼
Kepler CompositeSystem  (subsystem registration, interface lookup)
     │
     ▼
ManagementSystem  (domain operations: start managing / synchronize / activate / …)
     │
     ▼
Sagan repositories  (RDBMSRepository in prod, InMemoryRepository in tests)
     │
     ▼
PostgreSQL  (schema managed by Sagan's prepareForInitialPersistence; schema evolution by external SQL scripts)
```

The library versions that anchor this stack are the latest releases at `libraries-code/Stargate/` and `libraries-code/Sagan/`. The reference projects under `code-reference/` (`abbaco-subscription-api`, `abbaco-user-profile-api`, `positions-api`) target an older `PersistentAPISkeleton` wrapper; use them only as **domain shape** references — class names, instance variables, status enums, Auth0 wiring patterns — not as API conventions. New code follows the conventions in this skill family.

## The seven invariants

1. **Package split** is always:
   - `BaselineOf<Name>API`
   - `<Thing>-Model` + `<Thing>-Model-Tests`
   - `<Thing>-API-Model` + `<Thing>-API-Model-Tests`

   No GemStone test-extension package; Postgres integration coverage lives inside `<Thing>-API-Model-Tests` or a sibling, not in a separate package.

2. **Dependencies are pinned to the latest** Stargate, Sagan, Kepler, Launchpad. They are declared in `BaselineOf<Name>API`. Do **not** depend on `PersistentAPISkeleton` for new APIs — talk to Stargate and Sagan directly.

3. **Domain model** is: a plain value object (`Portfolio`, `Trial`, `BondGroup`, …), an `Identified<Thing>` wrapper that adds a UUID **plus** a `sequentialNumber` slot (Sagan-RDBMS auto-increment PK), a `<Thing>ManagementSystem` (Kepler `SubsystemImplementation`), and a `<Thing>ManagementModule` (`SystemModule`). See `abbaco-api-domain-model`.

4. **Persistence** is a `<Thing>RDBMSMappingConfiguration` (subclass of `RDBMSMappingConfiguration`) that declares tables, class models, and descriptors. It is applied to a repository created via `RepositoryProviderSystem >> createRepositoryFor:storingObjectsOfType:checkingConflictsAccordingTo:`. The provider behind `#mainDB` is `RDBMSRepositoryProvider using: <Login>` in production and `InMemoryRepositoryProvider new` in tests. See `abbaco-api-persistence`.

5. **API** is a `<Thing>RESTfulController` extending Stargate's `SingleResourceRESTfulController`. Routes are declared with reflective `declare<Verb><Thing>Route` methods (Stargate auto-collects every method whose selector starts with `declare` and ends with `Route` — see `ResourceRESTfulController>>routes`). All wiring goes through one `RESTfulRequestHandlerBuilder` instance. See `abbaco-api-rest-controller`.

6. **Media types** are vendor-versioned: `application/vnd.mercap.<resource>+json;version=1.0.0`. Declared as an accessor method on the controller (e.g. `portfolioVersion1dot0dot0MediaType`).

7. **Testing is layered**: pure unit tests (`<Thing>Test` extends `TestCase`), domain user-story tests (`SystemBasedUserStoryTest` with `InMemoryRepositoryProvider`), full-stack controller tests (`SingleResourceRESTfulControllerTest`), API user-story tests with real HTTP and `ZnClient`, **PostgreSQL integration tests** that swap in `RDBMSRepositoryProvider`, and Postman/Newman tests in `api-tests/` driven by docker-compose. CI runs on **GitHub Actions**, not Jenkins. See `abbaco-api-testing`.

## What is intentionally **not** part of this house style

- **Localization.** Abbaco APIs are not localized today. Do not introduce `localized` / `localizedWithAll:` calls or `locales/{en,es,pt}/*.json` files in new code. Strings stay as-is. (If a project later needs i18n, that's a deliberate, separate decision — and the pepper skills already document the pattern.)
- **JSON-RPC.** The legacy abbaco-subscription-api ships a JSON-RPC layer alongside REST. New APIs should be REST-first. Only add JSON-RPC if a specific consumer requires it.
- **PersistentAPISkeleton wrapper.** Older abbaco APIs subclass `PersistentAPIApplication` from `mercap/Persistent-API-Skeleton`. New APIs subclass `StargateApplication` directly and wire Sagan into the Kepler `CompositeSystem` themselves. The migration to `projectName` (Stargate v9 — see `libraries-code/Stargate/docs/MigrationGuide.md`) applies.
- **Jenkins.** CI is GitHub Actions; the existing `Jenkinsfile` files in `code-reference/` are being phased out.

Routing to the focused sub-skills lives in the workspace `CLAUDE.md`. This skill carries the cross-cutting "all of these hold together" contract; the sub-skills carry the depth — they're also where canonical-source pointers live when they're needed.
