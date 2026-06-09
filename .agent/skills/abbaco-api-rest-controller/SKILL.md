---
name: abbaco-api-rest-controller
description: Use when designing or code-reviewing the `<Thing>-API-Model` package of an Abbaco API — `SingleResourceRESTfulController` subclass with reflective `declare<Verb><Thing>Route` methods auto-collected by `ResourceRESTfulController>>routes`, a single `RESTfulRequestHandlerBuilder` that wires location/identifier extraction, hypermedia (`beHypermediaDriven` or `beHypermediaDrivenBy:`), pagination (`paginateCollectionsWithDefaultLimit:`), NeoJSON encoding/decoding (`decodeToNeoJSONObjectWhenAccepting:`, `whenResponding:encodeToJsonApplying:`), entity tags (`createEntityTagHashing:`), caching (`directCachingWith:`), vendor-versioned media types (`application/vnd.mercap.<resource>+json;version=X.Y.Z`), and a `StargateApplication` subclass for bootstrap (note `projectName` replaced `applicationBaselineName` in v9). JWT bearer auth is the default; an optional Auth0 section covers role assignment and the `Auth0UserManagementAPIClient` pattern.
---

# Abbaco API — REST Controller Layer

The `<Thing>-API-Model` package contains exactly two production classes:

1. **`<Thing>RESTfulController`** — subclass of Stargate's `SingleResourceRESTfulController`. One instance per resource type, wires routes to the management system through Kepler.
2. **`<Thing>APIApplication`** — subclass of `StargateApplication`. Declares the CLI command name, configuration parameters (Stargate, Sagan/PostgreSQL, optionally Auth0), the `controllersToInstall`, and bootstraps the Kepler `CompositeSystem`.

## 1. The controller

```smalltalk
Class {
    #name : 'PortfolioRESTfulController',
    #superclass : 'SingleResourceRESTfulController',
    #instVars : [ 'rootSystem', 'requestHandler', 'authenticationFilter' ],
    #category : 'Portfolio-API-Model',
    #package : 'Portfolio-API-Model'
}

{ #category : 'instance creation' }
PortfolioRESTfulController class >> workingWith: aRootSystem authenticatedBy: anAuthenticationFilter [

    ^ self basicNew initializeWorkingWith: aRootSystem authenticatedBy: anAuthenticationFilter
]

{ #category : 'initialization' }
PortfolioRESTfulController >> initializeWorkingWith: aRootSystem authenticatedBy: anAuthenticationFilter [

    rootSystem := aRootSystem.
    authenticationFilter := anAuthenticationFilter.
    self initializeRequestHandler
]

{ #category : 'private' }
PortfolioRESTfulController >> typeIdConstraint [ ^ IsUUID ]

{ #category : 'private' }
PortfolioRESTfulController >> system [ ^ rootSystem >> #PortfolioManagementSystem ]

{ #category : 'private' }
PortfolioRESTfulController >> requestHandler [ ^ requestHandler ]

{ #category : 'accessing - media types' }
PortfolioRESTfulController >> portfolioVersion1dot0dot0MediaType [

    ^ self jsonMediaType: 'portfolio' vendoredBy: 'mercap' version: '1.0.0'
]
```

### Naming and structure rules

- Class name: `<Thing>RESTfulController`. Instance variables in this exact order: `rootSystem`, `requestHandler`, `authenticationFilter`.
- Factory: `workingWith:authenticatedBy:`. The auth filter is injected so tests can substitute a no-op filter.
- **Do not define `endpoint` or `identifierTemplate`** — they come from `SingleResourceRESTfulController` and are derived from the `handling:` string passed to `initializeRequestHandler` and from `typeIdConstraint`.
- **`typeIdConstraint`** must be defined in every subclass, returning `IsUUID`. The superclass uses it to build the `<identifier:IsUUID>` constraint in `identifierTemplate`. The URL segment itself comes from the `handling:` string passed to the request-handler builder (plural, kebab-case): `'portfolios'`, `'paid-subscriptions'`, `'user-profiles'`.
- Media type accessor: `<thing>Version<major>dot<minor>dot<patch>MediaType` produced via `self jsonMediaType: '<resource>' vendoredBy: 'mercap' version: 'X.Y.Z'` (defined on `ResourceRESTfulController`, lines 37–41). The result is `application/vnd.mercap.<resource>+json;version=X.Y.Z`. The `<resource>` slug is the singular kebab-case form.
- Access the management system via `rootSystem >> #<Thing>ManagementSystem` — always through Kepler, never by direct class reference.
- Define one class-side `new`-style factory only (`workingWith:authenticatedBy:`). The pet-store example uses `class >> new` to return an in-memory variant; abbaco code should not, because the controller always needs the Kepler root system and an explicit auth filter.

## 2. Reflective route declarations

`ResourceRESTfulController>>routes` auto-collects every method whose selector starts with `declare` and ends with `Route`. So you declare routes by writing methods with that naming convention; nothing else registers them.

```smalltalk
{ #category : 'routes' }
PortfolioRESTfulController >> declareGetPortfoliosRoute [

    ^ RouteSpecification
        handling: #GET
        at: self endpoint
        evaluating: [ :httpRequest :requestContext |
            self getPortfoliosBasedOn: httpRequest within: requestContext ]
]

{ #category : 'routes' }
PortfolioRESTfulController >> declareGetPortfolioRoute [

    ^ RouteSpecification
        handling: #GET
        at: self identifierTemplate
        evaluating: [ :httpRequest :requestContext |
            self getPortfolioBasedOn: httpRequest within: requestContext ]
]

{ #category : 'routes' }
PortfolioRESTfulController >> declareCreatePortfolioRoute [

    ^ ( RouteSpecification
        handling: #POST
        at: self endpoint
        evaluating: [ :httpRequest :requestContext |
            self createPortfolioBasedOn: httpRequest within: requestContext ] )
        authenticatedBy: authenticationFilter
]

{ #category : 'routes' }
PortfolioRESTfulController >> declareUpdatePortfolioRoute [

    ^ ( RouteSpecification
        handling: #PATCH
        at: self identifierTemplate
        evaluating: [ :httpRequest :requestContext |
            self updatePortfolioBasedOn: httpRequest within: requestContext ] )
        authenticatedBy: authenticationFilter
]

{ #category : 'routes' }
PortfolioRESTfulController >> declareDeletePortfolioRoute [

    ^ ( RouteSpecification
        handling: #DELETE
        at: self identifierTemplate
        evaluating: [ :httpRequest :requestContext |
            self deletePortfolioBasedOn: httpRequest within: requestContext ] )
        authenticatedBy: authenticationFilter
]
```

Standard CRUD route table:

| Selector | Verb | Path | Auth |
|---|---|---|---|
| `declareGet<Things>Route` | GET | `<endpoint>` | no |
| `declareGet<Thing>Route` | GET | `<endpoint>/<identifier:IsUUID>` | no |
| `declareCreate<Thing>Route` | POST | `<endpoint>` | yes |
| `declareUpdate<Thing>Route` | PATCH | `<endpoint>/<identifier:IsUUID>` | yes |
| `declareDelete<Thing>Route` | DELETE | `<endpoint>/<identifier:IsUUID>` | yes |

**Reads are unauthenticated; writes require the `authenticationFilter`**. To add an authenticated route, wrap the `RouteSpecification` and send `authenticatedBy: authenticationFilter` (see the `Create` / `Update` / `Delete` examples above).

> The generated `route urlTemplate` carries a **leading slash** — `/portfolios`, `/portfolios/<identifier:IsUUID>`. Assert exactly that form in `testRoutes` (see `abbaco-api-testing` §3).

### Action endpoints (activate / deactivate / cancel / …)

When a resource has an explicit lifecycle transition that does not fit CRUD, declare an action route:

```smalltalk
{ #category : 'routes' }
PaidSubscriptionRESTfulController >> declareCancelPaidSubscriptionRoute [

    ^ ( RouteSpecification
        handling: #POST
        at: ( '<1s>/cancel' expandMacrosWith: self identifierTemplate )
        evaluating: [ :httpRequest :requestContext |
            self cancelPaidSubscriptionBasedOn: httpRequest within: requestContext ] )
        authenticatedBy: authenticationFilter
]
```

Action endpoints are POST. They return `204 No Content` on success (use `thenDo:` in the API method, see Section 5b). When the resource has reciprocal transitions (activate ↔ deactivate, cancel ↔ uncancel), make each one **idempotent**: the second POST to the same URL must not raise — use the `if:isActive: [ ] else: [ ]` (or status-aware equivalent) guard.

### Sub-resources

Nested resources use `expandMacrosWith:`:

```smalltalk
^ RouteSpecification
    handling: #GET
    at: ( '<1s>/positions' expandMacrosWith: self identifierTemplate )
    evaluating: [ :req :ctx | self getPositionsBasedOn: req within: ctx ]
```

For sub-resources with their own identifiers, see `RESTfulRequestHandlerBuilder >> handling:extractingIdentifierWith:locatingParentResourceWith:`.

## 3. `RESTfulRequestHandlerBuilder` — the single source of wiring

Every controller has exactly one `requestHandler` instance, built once in `initializeRequestHandler` and reused by every API method.

```smalltalk
{ #category : 'initialization' }
PortfolioRESTfulController >> initializeRequestHandler [

    requestHandler := RESTfulRequestHandlerBuilder new
        handling: 'portfolios'
            locatingResourcesWith: [ :portfolio :requestContext |
                self system identifierOf: portfolio ]
            extractingIdentifierWith: [ :httpRequest | self identifierIn: httpRequest ];
        beHypermediaDriven;
        paginateCollectionsWithDefaultLimit: 25;
        decodeToNeoJSONObjectWhenAccepting: self portfolioVersion1dot0dot0MediaType;
        whenResponding: self portfolioVersion1dot0dot0MediaType
            encodeToJsonApplying: [ :resource :requestContext :writer |
                self configurePortfolioEncodingOn: writer within: requestContext ];
        createEntityTagHashing: [ :hasher :portfolio :requestContext |
            hasher
                include: portfolio identifier;
                include: portfolio name;
                include: portfolio description;
                include: portfolio image ];
        directCachingWith: [ :caching |
            caching
                when: [ :response | response contentType = self portfolioVersion1dot0dot0MediaType ]
                apply: [ caching
                    beAvailableFor: 1 hour;   "→ public, max-age=3600 (see caching note below)"
                    mustRevalidate ] ];
        build
]
```

The builder API (full surface in `RESTfulRequestHandlerBuilder` — inspect via the Pharo MCP):

| Selector | Purpose |
|---|---|
| `handling: anEndpoint locatingResourcesWith: <block> extractingIdentifierWith: <block>` | Sets the URL tail (`'portfolios'`), how to compute a stored resource's location identifier, and how to pull the identifier out of the URL. |
| `beHypermediaDriven` | Adds `self`-link to every encoded resource. |
| `beHypermediaDrivenBy: aBlock` | Same plus a per-resource block that adds action links (`activate`, `deactivate`, …). The block receives `( :builder :resource :requestContext :resourceLocation )`. **`builder addLink: aUrl relatedTo: 'rel'` needs a `ZnUrl`/URL, not a bare `String`** (a String is parsed as an RFC Link-header value → "Missing <"); append sub-resource segments on the ZnUrl: `resourceLocation / 'metrics'`. |
| `paginateCollectionsWithDefaultLimit: anInteger` | Enables pagination on collection endpoints; clients page with `?start=` and `?limit=` (read by the policy via `httpRequest at: #start` / `#limit`). Read any other query parameter with `httpRequest at: 'name' ifAbsent: [ … ]`. |
| `decodeToNeoJSONObjectWhenAccepting: aMediaType` | Default decoder: parse the body as a `NeoJSONObject` and pass it to the API method. Use this when no domain-side `createInstanceUsing:` mapping is needed. |
| `whenAccepting: aMediaType decodeFromJsonApplying: <block>` | Custom decoder. The block receives the JSON string and a `NeoJSONReader`; configure the reader inside (typical pattern: see Section 4). |
| `whenResponding: aMediaType encodeToJsonApplying: <block>` | Encoder block called with `( :resource :requestContext :writer )`; configure the `writer` (a `NeoJSONWriter`). |
| `whenResponding: aMediaType encodeToJsonApplying: <block> as: aSymbol` | Same but the writer's mapping key is `aSymbol`. Useful when multiple types share the encoder block. |
| `createEntityTagHashing: <block>` | Define which fields contribute to the ETag. The block receives `( :hasher :resource :requestContext )` and uses `hasher include: <value>` per field. The media type is folded in automatically. |
| `directCachingWith: <block>` | Configure `Cache-Control`. `caching when: <condBlock> apply: <directiveBlock>` varies by response (the condition block is culled with `(response, resource)`, so you can branch on the resource). **`expireIn:` sets the `Expires` header, NOT `max-age`** — `max-age` comes from `beStaleAfter:` / `beAvailableFor:` (the latter = `bePublic` + `beStaleAfter:` + `expireIn:`). Other directives: `beImmutable`, `bePublic`, `bePrivate`, `mustRevalidate`, `doNotTransform`, `doNotCache`, `doNotStore`. |
| `build` | Realize the handler. Always last. |

The builder enforces ordering via `AssertionChecker` — for example, `beHypermediaDrivenBy:` requires that the resource locator is configured first and that no encoding rules are set yet (see `RESTfulRequestHandlerBuilder>>beHypermediaDrivenBy:`). Follow the order shown above.

## 4. NeoJSON decoding and encoding

The encoder block receives a `NeoJSONWriter` and configures it for the resource's class:

```smalltalk
{ #category : 'private - encoding' }
PortfolioRESTfulController >> configurePortfolioEncodingOn: writer within: requestContext [

    writer
        for: IdentifiedPortfolio
        do: [ :mapping |
            mapping
                mapAccessor: #identifier;
                mapAccessor: #name;
                mapAccessor: #description;
                mapAccessor: #image;
                mapAccessor: #owner;
                mapAsHypermediaControls: [ :portfolio | requestContext hypermediaControlsFor: portfolio ] ]
]
```

`mapAsHypermediaControls:` injects HAL-style links produced by the `beHypermediaDriven[By:]` builder block. Without it, the response body lacks `links.self` etc.

When you want every instance variable mapped without listing them, use `mapInstVars` (the pet-store example does this — see `PetsRESTfulController>>configurePetEncodingOn:within:`). For abbaco APIs we recommend explicit `mapAccessor:` calls so the wire shape doesn't drift silently when the value object adds a private field.

### Decoder: `createInstanceUsing:` + `mapCreationSending:withArguments:`

When the JSON shape calls a class-side factory directly, declare the decoding via `createInstanceUsing:`:

```smalltalk
{ #category : 'private - decoding' }
PortfolioRESTfulController >> configurePortfolioDecodingOn: reader [

    ^ reader
        for: Portfolio createInstanceUsing: [ :mapping |
            mapping
                mapProperty: #name;
                mapProperty: #description;
                mapProperty: #image;
                mapProperty: #owner;
                mapCreationSending: #named:describedAs:withImage:ownedBy:
                    withArguments: { #name. #description. #image. #owner } ];
        nextAs: Portfolio
```

Wire it on the builder:

```smalltalk
whenAccepting: self portfolioVersion1dot0dot0MediaType
    decodeFromJsonApplying: [ :json :reader |
        self configurePortfolioDecodingOn: reader ];
```

This routes decoded payloads through the value object's `named:…` factory, so its `AssertionChecker` preconditions fire on invalid input. Stargate translates `InstanceCreationFailed` into HTTP 400 / 422 automatically.

For simpler endpoints (especially PATCH where the client sends a partial document), use `decodeToNeoJSONObjectWhenAccepting:` and let the API method pull individual fields with `decoded at: #name ifAbsent: [ existing name ]`. See the pet-store update method (`PetsRESTfulController>>updatePetBasedOn:within:`) for that pattern.

> **`NeoJSONObject >> at:` returns `nil` for a missing key** (keys are symbols) — it does *not* raise. So `decodeToNeoJSONObjectWhenAccepting:` works for **POST too** (no DTO class needed; read `decoded at: #name`). For a **required** POST field, force a clean 400 with `decoded at: #name ifAbsent: [ KeyNotFound signalFor: #name ]` (KeyNotFound → `badRequest`). For PATCH, default to the existing value: `decoded at: #name ifAbsent: [ original name ]`. Letting a missing field flow through as `nil` into a factory yields a `MessageNotUnderstood` (500), not a 400.

## 5. API operation methods

One method per declared route, in category `API`. They all delegate to the `requestHandler`:

```smalltalk
{ #category : 'API' }
PortfolioRESTfulController >> getPortfoliosBasedOn: httpRequest within: requestContext [

    ^ requestHandler
        from: httpRequest
        within: requestContext
        getCollection: [ :pagination |
            | all start end |

            all := self system portfolios.
            start := pagination start min: all size.
            end := pagination end min: all size.
            all isEmpty ifTrue: [ #() ] ifFalse: [ all copyFrom: start to: end ] ]
]

{ #category : 'API' }
PortfolioRESTfulController >> getPortfolioBasedOn: httpRequest within: requestContext [

    ^ requestHandler
        from: httpRequest
        within: requestContext
        get: [ :id | self system portfolioIdentifiedBy: id ]
]

{ #category : 'API' }
PortfolioRESTfulController >> createPortfolioBasedOn: httpRequest within: requestContext [

    self
        assertRequestIsAuthorizedTo: self requiredPermissionForWriting
        within: requestContext.

    ^ requestHandler
        withRepresentationIn: httpRequest
        within: requestContext
        createResourceWith: [ :decoded |
            Portfolio
                named: decoded name
                describedAs: decoded description
                withImage: decoded image
                ownedBy: decoded owner ]
        thenDo: [ :portfolio | self system startManagingPortfolio: portfolio ]
]

{ #category : 'API' }
PortfolioRESTfulController >> updatePortfolioBasedOn: httpRequest within: requestContext [

    self
        assertRequestIsAuthorizedTo: self requiredPermissionForWriting
        within: requestContext.

    ^ requestHandler
        from: httpRequest
        within: requestContext
        get: [ :id | self system portfolioIdentifiedBy: id ]
        thenUpdateWith: [ :original :decoded |
            self system
                updatePortfolio: original
                with: ( Portfolio
                    named: ( decoded at: #name ifAbsent: [ original name ] )
                    describedAs: ( decoded at: #description ifAbsent: [ original description ] )
                    withImage: ( decoded at: #image ifAbsent: [ original image ] )
                    ownedBy: original owner ) ]
]

{ #category : 'API' }
PortfolioRESTfulController >> deletePortfolioBasedOn: httpRequest within: requestContext [

    self
        assertRequestIsAuthorizedTo: self requiredPermissionForWriting
        within: requestContext.

    ^ requestHandler
        from: httpRequest
        within: requestContext
        get: [ :id | self system portfolioIdentifiedBy: id ]
        thenDo: [ :portfolio | self system stopManagingPortfolio: portfolio ]
]
```

### House rules for API methods

- **Write operations start with `self assertRequestIsAuthorizedTo:within:`** (defined on `ResourceRESTfulController`, raises `HTTPForbidden` when permission is missing). Reads do not. The permission symbol comes from `self requiredPermissionForWriting`, a single accessor per controller (e.g. `^ #createOrUpdate:portfolio`).
- **All delegation goes through `requestHandler`** — no direct JSON manipulation, no direct `ZnResponse` construction.
- **Return value follows the handler method**:
  - `from:within:get:` → 200 OK with body.
  - `from:within:getCollection:` → 200 OK with paginated body.
  - `withRepresentationIn:within:createResourceWith:thenDo:` → 201 Created with body and `Location` header.
  - `from:within:get:thenUpdateWith:` → 200 OK with updated body.
  - `from:within:get:thenDo:` → 204 No Content. **Use this for action endpoints and DELETE.**
- **Domain exceptions become RAISED `HTTPClientError`s** (not response objects) at the controller-method layer: `ObjectNotFound→notFound(404)`, `ConflictingObjectFound→conflict(409)`, `InstanceCreationFailed→unprocessableEntity(422)`, `KeyNotFound`/`NeoJSONParseError→badRequest(400)`. The HTTP server converts the raise to a response; a **direct controller call (and controller tests) sees the raise**, so don't `on:do:` them, and assert error paths with `should: […] raise: HTTPClientError <kind>` (only success paths return a `ZnResponse`). The validation/conflict hooks fire *inside* the handler blocks: `InstanceCreationFailed` raised in `createResourceWith:` / `thenUpdateWith:` → 422; `ConflictingObjectFound` raised when storing → 409.
- **PATCH auto-enforces `If-Match`.** `from:within:get:thenUpdateWith:` reads `If-Match` and asserts it before applying the update: missing → `428 preconditionRequired`, stale → `412 preconditionFailed`. No manual ETag plumbing in the controller. (`from:within:get:thenDo:`, used by DELETE, does **not** enforce `If-Match`.)
- **`createResourceWith: aBlock thenDo: bBlock` runs `bBlock value: (aBlock value: decoded)`** — `thenDo:` receives only the creation block's *result*, not `decoded`. To thread extra decoded data (e.g. an owned collection) into `thenDo:`, return an `Association`/pair from `createResourceWith:` and unpack it there; validate that data **inside** `createResourceWith:` so its `InstanceCreationFailed` maps to 422.

### Action endpoints are idempotent and return 204

Match pepper's contract: action POSTs (`/cancel`, `/activate`, `/deactivate`) return `204 No Content`, must be idempotent (the second POST is a no-op), and check current state before acting:

```smalltalk
{ #category : 'API' }
PaidSubscriptionRESTfulController >> cancelPaidSubscriptionBasedOn: httpRequest within: requestContext [

    self
        assertRequestIsAuthorizedTo: self requiredPermissionForWriting
        within: requestContext.

    ^ requestHandler
        from: httpRequest
        within: requestContext
        get: [ :id | self system paidSubscriptionIdentifiedBy: id ]
        thenDo: [ :paidSubscription |
            paidSubscription isCancelled ifFalse: [
                self system cancelPaidSubscriptionIdentifiedBy: paidSubscription identifier expiringOn: …
            ] ]
]
```

The idempotency guard (`isCancelled ifFalse:`) belongs on the **controller's delegation boundary**, not in the management system — the management system can refuse a redundant transition (raising `ObjectNotFound` or similar), and the controller suppresses that for HTTP idempotency.

## 6. `StargateApplication` subclass

```smalltalk
Class {
    #name : 'PortfolioAPIApplication',
    #superclass : 'StargateApplication',
    #instVars : [ 'rootSystem' ],
    #category : 'Portfolio-API-Model',
    #package : 'Portfolio-API-Model'
}

{ #category : 'accessing' }
PortfolioAPIApplication class >> commandName [ ^ 'portfolio-api' ]

{ #category : 'accessing' }
PortfolioAPIApplication class >> description [
    ^ 'I provide a RESTful API over HTTP to manage Abbaco-owned portfolios'
]

{ #category : 'private' }
PortfolioAPIApplication class >> projectName [ ^ 'Portfolio-API' ]

{ #category : 'accessing' }
PortfolioAPIApplication class >> initialize [

    <ignoreForCoverage>
    self initializeVersion
]

{ #category : 'accessing' }
PortfolioAPIApplication class >> configurationParameters [

    ^ super configurationParameters ,
        self saganConfigurationParameters ,
        { self authenticationSecretParameter }
]

{ #category : 'private - configuration' }
PortfolioAPIApplication class >> saganConfigurationParameters [

    ^ {
        ( MandatoryConfigurationParameter
            named: 'PG Hostname'
            describedBy: 'PostgreSQL host'
            inside: #( 'Sagan' ) ).
        ( MandatoryConfigurationParameter
            named: 'PG Port'
            describedBy: 'PostgreSQL port'
            inside: #( 'Sagan' )
            convertingWith: #asNumber ).
        ( MandatoryConfigurationParameter
            named: 'PG Username'
            describedBy: 'PostgreSQL username'
            inside: #( 'Sagan' ) ).
        ( MandatoryConfigurationParameter
            named: 'PG Password'
            describedBy: 'PostgreSQL password'
            inside: #( 'Sagan' ) ) asSensitive.
        ( MandatoryConfigurationParameter
            named: 'PG Database Name'
            describedBy: 'PostgreSQL database name'
            inside: #( 'Sagan' ) ).
        ( OptionalConfigurationParameter
            named: 'Create Empty Database'
            describedBy: 'When true, recreate the schema on startup. Development / fresh deploys only.'
            inside: #( 'Sagan' )
            defaultingTo: false
            convertingWith: #asBoolean ) }
]

{ #category : 'private - configuration' }
PortfolioAPIApplication class >> authenticationSecretParameter [

    ^ ( MandatoryConfigurationParameter
        named: 'Authentication Secret'
        describedBy: 'JSON Web Token client secret (HS256)'
        inside: #( 'Authentication' ) ) asSensitive
]

{ #category : 'private - activation/deactivation' }
PortfolioAPIApplication >> basicStartWithin: context [

    self installRootSystem.
    super basicStartWithin: context
]

{ #category : 'private - activation/deactivation' }
PortfolioAPIApplication >> installRootSystem [

    rootSystem := CompositeSystem new.

    RDBMSRepositoryProviderModule
        toInstallOn: rootSystem
        connectingWith: self saganLogin
        configuredBy: [ :options |
            options at: #maxIdleSessionsCount put: 10.
            options at: #minIdleSessionsCount put: 5.
            options at: #maxActiveSessionsCount put: 12 ].

    PortfolioManagementModule toInstallOn: rootSystem.

    rootSystem startUp.
    self saganShouldCreateEmptyDatabase ifTrue: [
        ( rootSystem >> #RepositoryProviderSystem ) prepareForInitialPersistence ]
]

{ #category : 'private - configuration' }
PortfolioAPIApplication >> saganLogin [

    ^ Login new
        database: PostgreSQLPlatform new;
        username: self configuration sagan pgUsername;
        password: self configuration sagan pgPassword;
        host: self configuration sagan pgHostname;
        port: self configuration sagan pgPort;
        databaseName: self configuration sagan pgDatabaseName;
        setSSL;
        yourself
]

{ #category : 'private - configuration' }
PortfolioAPIApplication >> saganShouldCreateEmptyDatabase [

    ^ self configuration sagan createEmptyDatabase
]

{ #category : 'private - accessing' }
PortfolioAPIApplication >> authenticationFilter [

    ^ JWTBearerAuthenticationFilter
        with: self configuration authentication authenticationSecret
        forAlgorithmNamed: 'HS256'
]

{ #category : 'private - accessing' }
PortfolioAPIApplication >> controllersToInstall [

    ^ { PortfolioRESTfulController
        workingWith: rootSystem
        authenticatedBy: self authenticationFilter }
]

{ #category : 'private - activation/deactivation' }
PortfolioAPIApplication >> basicStop [

    rootSystem ifNotNil: [
        rootSystem shutDown.
        rootSystem := nil ].
    super basicStop
]
```

### House rules for the application class

- **`projectName`** is the Pharo baseline name minus `BaselineOf` (e.g. `'Portfolio-API'` → `BaselineOfPortfolioAPI`). It replaces the v8 `applicationBaselineName` selector (see Stargate's migration guide on the project's GitHub). Stargate uses it to resolve the version on startup.
- **`commandName`** is the Launchpad CLI command. It also drives container image names, log paths, and Postman `BASE_URL` defaults.
- **`configurationParameters`** always extends `super configurationParameters` (which contributes the standard Stargate parameters: `Public URL`, `Port`, `Operations Secret`, `Log HTTP Requests`, `Concurrent Connections Threshold`). Append:
  1. **Sagan/PostgreSQL parameters** — hostname, port, username, password (`asSensitive`), database name, optional `Create Empty Database` flag.
  2. **Authentication secret** — `asSensitive`.
  3. (Optional) **Auth0 parameters** — see Section 7.
- **`installRootSystem`** runs before `super basicStartWithin:` so the controllers can be wired with a live `rootSystem`. Build a `CompositeSystem`, install the persistence module first (everything else depends on it), then every `<Thing>ManagementModule`. Call `rootSystem startUp`. Conditionally call `prepareForInitialPersistence`.
- **Authentication** is JWT HS256 by default — `JWTBearerAuthenticationFilter with: secret forAlgorithmNamed: 'HS256'`. Reads are unauthenticated; writes go through the filter via `RouteSpecification authenticatedBy:` (Section 2).
- **`controllersToInstall`** returns the list of controller instances. Stargate wires them into the Teapot server during `installAndStartAPI`.
- The operational plugins (`/health`, `/metrics`, `/application-info`, `/application-configuration`, `/application-control`, `/loggers`) come for free from `StargateApplication`; do not re-register them.

## 7. Optional — Auth0 integration

Some abbaco APIs (notably the subscription API) use Auth0 instead of (or in addition to) a static JWT secret. The pattern uses a small client class plus a "role assigner" collaborator.

When you need Auth0:

1. **Add Auth0 configuration parameters** to `configurationParameters`:
   ```smalltalk
   ( MandatoryConfigurationParameter named: 'Tenant' describedBy: 'Auth0 tenant domain' inside: #( 'Auth0' ) ).
   ( MandatoryConfigurationParameter named: 'Client ID' describedBy: '…' inside: #( 'Auth0' ) ).
   ( MandatoryConfigurationParameter named: 'Client Secret' describedBy: '…' inside: #( 'Auth0' ) ) asSensitive.
   ( MandatoryConfigurationParameter named: 'Audience' describedBy: '…' inside: #( 'Auth0' ) ).
   ```

2. **Build an `Auth0UserManagementAPIClient`** in the application:
   ```smalltalk
   PortfolioAPIApplication >> auth0UserManagementAPIClient [
       | tenant application |
       tenant := Auth0Tenant on: self configuration auth0 tenant.
       application := Auth0Application
           identifiedBy: self configuration auth0 clientID
           sharing: self configuration auth0 clientSecret
           withAccessTo: self configuration auth0 audience.
       ^ Auth0UserManagementAPIClient accessingTo: tenant through: application
   ]
   ```

3. **Build a `<Thing>RoleAssigner`** that the controller calls after state changes:
   ```smalltalk
   self system startManagingPortfolio: portfolio.
   self portfolioRoleAssigner assignRoleFor: portfolio owner.
   ```

4. **Pass the role assigner to the controller** through the factory:
   ```smalltalk
   PortfolioRESTfulController
       workingWith: rootSystem
       authenticatedBy: self authenticationFilter
       assigningRolesOn: self portfolioRoleAssigner.
   ```

The Auth0 client classes live in the `Auth0-Core-Model` package:

- `Auth0Tenant` — a tenant domain wrapper.
- `Auth0Application` — credentials for a Machine-to-Machine Auth0 application.
- `AccessTokenProvider` — fetches and caches access tokens.
- `Auth0UserManagementAPIClient` — wraps the Management API for role assignment, user lookup, etc.
- `ClientCredentialFlow` — the M2M token grant.

Auth0 JWT validation can still flow through `JWTBearerAuthenticationFilter` if Auth0 issues HS256 tokens (less common — RS256 is the Auth0 default). For RS256, configure the filter with the JWKS URL or a fetched public key. Either way, treat the token as a bearer; the `Authorization: Bearer <jwt>` flow is the same.

**Do not adopt Auth0 by default for new APIs.** Most abbaco services authenticate with a shared HS256 secret. Auth0 is an option for services that need user-management features (assigning roles based on subscription state, looking up email/profile, etc.).

## 8. Migration notes when porting from older code

Older abbaco services use the `PersistentAPISkeleton` wrapper. When porting an old controller to the latest Stargate:

- Replace `PersistentAPIApplication` with `StargateApplication`.
- Replace `applicationBaselineName` with `projectName` (Stargate v9 — see migration guide).
- Replace `SinglePostgreSQLDatabaseProviderModuleFactory` with a hand-rolled `RDBMSRepositoryProviderModule` (see `abbaco-api-persistence`) plus an explicit `Login` builder.
- Replace any `repository configureMappingsIn: aConfig` with `aConfig new cull: repository`.
- Routes already declared as `RouteSpecification` work as-is, so long as their selectors match `declare<…>Route`.
- Inline JSON encoding (`writer for: <ClassSymbol> customDo: [ :mapping | mapping encoder: [ :resource | … ] ]`) still works in NeoJSON, but prefer `for: <Class> do: [ :mapping | mapping mapAccessor: …; mapAsHypermediaControls: … ]` for new code.
- JSON-RPC handlers from `Stargate-JSON-RPC` are still available for resources that genuinely need RPC; do not adopt them by default.

## 9. Common mistakes

- **Defining `endpoint` or `identifierTemplate` in the subclass** — both are derived from `handling:` and `typeIdConstraint`. Defining them produces duplicate construction and silently masks the UUID constraint. Delete both methods; only define `typeIdConstraint`.
- **Omitting `typeIdConstraint`** — `SingleResourceRESTfulController#identifierTemplate` calls `subclassResponsibility`. Without `typeIdConstraint [ ^ IsUUID ]`, the controller fails to load.
- **Declaring routes outside `declare*Route` methods** — they are not auto-collected by `routes` and the URL is not registered.
- **Wrong selector spelling** — `declarePOSTPortfolioRoute` vs `declareCreatePortfolioRoute`. The framework's reflection only filters by prefix/suffix, but team conventions matter: stick to `declare<HumanVerb><Thing>Route` (`Get`, `GetSingle`, `Create`, `Update`, `Delete`, `Activate`, `Deactivate`, `Cancel`, …).
- **Forgetting `authenticatedBy: authenticationFilter` on a write route** — writes execute without auth checks; controllers will still fail at `assertRequestIsAuthorizedTo:within:` on permission absence, but no `WWW-Authenticate` challenge is sent and clients can't recover.
- **Bypassing `self system` / `rootSystem >> #…`** and referring to the management system class directly — couples the controller to the impl class and breaks the Kepler interface proxy.
- **Returning the resource from an action endpoint's `thenDo:` block** — the return value is ignored; the response is `204 No Content`. Don't pretend otherwise in tests.
- **Encoding `nil` without a guard** — `mapAccessor: #image` on a portfolio with `image` = nil writes a `null` field. If clients expect either a string or an absent field, use `mapProperty: #image getter: [ :p | p image ifNil: [ nil ] ]` or omit the field with a custom encoder block.
- **Building JSON manually with `NeoJSONWriter on: stream`** — bypasses the request handler so content negotiation, hypermedia, and ETag wiring all break. Always go through `whenResponding:encodeToJsonApplying:`.
- **Mis-setting `as: <symbol>` on `whenResponding:encodeToJsonApplying:as:`** — must match the key used in `writer for: <symbol> do:`.
- **Putting the auth secret in a non-`asSensitive` parameter** — the value ends up in `/operations/configuration` and Bell logs. Always `asSensitive` for secrets, JWTs, Auth0 client secrets, and PostgreSQL passwords.
- **Skipping `projectName`** — `StargateApplication class >> projectName` is `subclassResponsibility`; without it the version banner crashes during startup.
- **Forgetting to call `rootSystem startUp` in `installRootSystem`** — modules are registered but never installed; `rootSystem >> #PortfolioManagementSystem` returns nothing.
- **Calling `prepareForInitialPersistence` unconditionally** — drops production tables. Always gate it behind a config flag (`Create Empty Database` / `RDBMS_CREATE_EMPTY_DATABASE`).
- **Reusing pepper's `assertContentLanguageIn:isEnglishBecause:` / Buoy localization helpers** — abbaco APIs are not localized today. Skip the language enforcement; client-supplied `Content-Language` is ignored.
