---
name: abbaco-api-testing
description: Use when writing or code-reviewing tests for an Abbaco API — `TestCase` unit tests over domain value objects; `SystemBasedUserStoryTest` over the `ManagementSystem` with `InMemoryRepositoryProvider`; `SingleResourceRESTfulControllerTest` (full-stack HTTP, no network) wiring the controller against a Kepler `CompositeSystem` in `setUp`; an API user-story test (real HTTP via `ZnClient`) for authorization and content negotiation; a PostgreSQL integration test that swaps in `RDBMSRepositoryProvider` and exercises the persist-query round-trip; the Postman/Newman `api-tests/` suite driven by docker-compose with a postgres sidecar; and the GitHub Actions CI workflow (`.github/workflows/unit-tests.yml`) that runs unit + Postgres-integration in one job and gates a Docker build/publish job.
---

# Abbaco API — Testing Layers

An Abbaco API has **six test layers**, each in its own package or directory. Keep them all green; they catch different classes of regression.

| Layer | Package / path | Superclass | What it covers |
|---|---|---|---|
| Domain unit | `<Thing>-Model-Tests` | `TestCase` | Factories, preconditions, value-object behaviour. |
| Domain user-story | `<Thing>-Model-Tests` | `SystemBasedUserStoryTest` | Management-system scenarios with real Kepler wiring + `InMemoryRepositoryProvider`. |
| HTTP full-stack | `<Thing>-API-Model-Tests` | `SingleResourceRESTfulControllerTest` (Stargate-SUnit) | Controller + request handler + routing + encoding/decoding, without a network. The test owns a `CompositeSystem` wired with `InMemoryRepositoryProvider`. |
| API user-story | `<Thing>-API-Model-Tests` | `SystemBasedUserStoryTest` (with real HTTP) | Real HTTP server + `ZnClient`; authorization, content negotiation, error codes. |
| **PostgreSQL integration** | `<Thing>-API-Model-Tests` (or sibling) | extends domain user-story, swaps `InMemoryRepositoryProvider` for `RDBMSRepositoryProvider` | Mapping correctness on real Postgres — every aspect persists and round-trips. |
| Newman / Postman | `api-tests/` | docker-compose | End-to-end against the real container stack (API + Postgres). |

CI (GitHub Actions) runs the first five inside one image-load step (Smalltalk CI) plus the Newman suite as a separate job (currently not wired — see Section 6). Pepper's `<Thing>-API-Model-Tests-GS64-Extensions` package does **not** apply to abbaco — there is no GemStone parallel test suite.

## 1. Domain unit tests (`<Thing>Test`)

```smalltalk
Class {
    #name : 'PortfolioTest',
    #superclass : 'TestCase',
    #category : 'Portfolio-Model-Tests',
    #package : 'Portfolio-Model-Tests'
}

{ #category : 'tests' }
PortfolioTest >> testCreation [

    | portfolio |

    portfolio := Portfolio
        named: 'Long-term holdings'
        describedAs: 'Buy and hold equities'
        withImage: 'https://example.test/portfolio.png'
        ownedBy: 'user-123'.

    self
        assert: portfolio name equals: 'Long-term holdings';
        assert: portfolio description equals: 'Buy and hold equities';
        assert: portfolio image equals: 'https://example.test/portfolio.png';
        assert: portfolio owner equals: 'user-123'
]

{ #category : 'tests' }
PortfolioTest >> testCreationWithEmptyNameNotAllowed [

    self
        should: [
            Portfolio
                named: ''
                describedAs: 'desc'
                withImage: 'https://x.test'
                ownedBy: 'u' ]
        raise: InstanceCreationFailed
        withMessageText: 'The portfolio name must be a non-empty string of at most 40 characters'
]

{ #category : 'tests' }
PortfolioTest >> testCreationWithLongDescriptionNotAllowed [

    self
        should: [
            Portfolio
                named: 'OK'
                describedAs: ( String new: 251 withAll: $a )
                withImage: 'https://x.test'
                ownedBy: 'u' ]
        raise: InstanceCreationFailed
        withMessageText: 'The portfolio description cannot be longer than 250 characters'
]

{ #category : 'tests' }
PortfolioTest >> testSynchronize [

    | original updated |

    original := Portfolio named: 'A' describedAs: 'a' withImage: 'http://x' ownedBy: 'u'.
    updated := Portfolio named: 'B' describedAs: 'b' withImage: 'http://y' ownedBy: 'u'.

    original synchronizeWith: updated.

    self
        assert: original name equals: 'B';
        assert: original description equals: 'b';
        assert: original image equals: 'http://y';
        assert: original owner equals: 'u'  "owner is identity, never changes"
]
```

### Conventions

- **One test per behavior.** Selectors prefixed `test`, English. Category `tests`.
- **Assert both happy and failure path** for every factory. Use `should:raise:withMessageText:` to lock the exact message string. There is no localization layer to round-trip.
- **No mocks**, no stubs. Construct domain objects directly.
- **Cover `synchronizeWith:`** for every value object — it's how Sagan's `update:executing:` mutates state, and missing fields here are silent data loss.

## 2. Domain user-story tests (`<Thing>ManagementSystemUserStoryTest`)

```smalltalk
Class {
    #name : 'PortfolioManagementSystemUserStoryTest',
    #superclass : 'SystemBasedUserStoryTest',
    #category : 'Portfolio-Model-Tests',
    #package : 'Portfolio-Model-Tests'
}

{ #category : 'private - running' }
PortfolioManagementSystemUserStoryTest >> setUpRequirements [

    | repositorySystem |

    repositorySystem := RepositoryProviderSystem new.
    repositorySystem register: InMemoryRepositoryProvider new as: #mainDB.
    self
        registerSubsystem: repositorySystem;
        requireInstallationOf: PortfolioManagementModule
]

{ #category : 'tests - start managing' }
PortfolioManagementSystemUserStoryTest >> testStartManagingPortfolio [

    | identified all |

    identified := self systemUnderTest startManagingPortfolio: ( Portfolio
        named: 'Long-term'
        describedAs: 'Buy and hold'
        withImage: 'http://x'
        ownedBy: 'user-1' ).

    all := self systemUnderTest portfolios.
    self
        assert: all size equals: 1;
        assert: all anyOne identifier equals: identified identifier
]

{ #category : 'tests - querying' }
PortfolioManagementSystemUserStoryTest >> testPortfolioIdentifiedByRaisesWhenNotFound [

    | uuid |

    uuid := UUID new.
    self
        should: [ self systemUnderTest portfolioIdentifiedBy: uuid ]
        raise: ObjectNotFound
        withMessageText: ( 'There''s no portfolio identified by <1s>' expandMacrosWith: uuid asString )
]

{ #category : 'tests - updates' }
PortfolioManagementSystemUserStoryTest >> testUpdatePortfolioSynchronizesAttributes [

    | original updatedShape all stored |

    original := self systemUnderTest startManagingPortfolio: ( Portfolio
        named: 'A' describedAs: 'a' withImage: 'http://x' ownedBy: 'user-1' ).
    updatedShape := Portfolio named: 'B' describedAs: 'b' withImage: 'http://y' ownedBy: 'user-1'.

    self systemUnderTest updatePortfolio: original with: updatedShape.

    all := self systemUnderTest portfolios.
    self assert: all size equals: 1.
    stored := all anyOne.
    self
        assert: stored name equals: 'B';
        assert: stored description equals: 'b';
        assert: stored image equals: 'http://y'
]
```

### Conventions

- **Superclass `SystemBasedUserStoryTest`** (Kepler-SUnit). It installs the requested modules, calls `startUp`, and exposes `systemUnderTest` (the first interface in `systemInterfacesToInstall`).
- **`setUpRequirements`** registers a `RepositoryProviderSystem` with `InMemoryRepositoryProvider` under `#mainDB`, then `requireInstallationOf: <Thing>ManagementModule`. This mirrors production wiring with zero database cost.
- **Test one scenario per management-system selector.** Categories: `tests - <verb>` (`tests - start managing`, `tests - querying`, `tests - updates`, `tests - lifecycle`).
- **Assert raises** for the not-found / conflict / invalid-state paths with `should:raise:withMessageText:`. The message is plain English.
- **Skip localization helpers** — abbaco APIs do not localize. No `inSpanishDo:` / `inPortugueseDo:` blocks.

## 3. HTTP full-stack tests (`<Thing>RESTfulControllerTest`)

### What Stargate-SUnit provides vs what your test class adds

`SingleResourceRESTfulControllerTest` (from the `Stargate-SUnit-Model` package) provides:

| Inherited selector | Returns / does |
|---|---|
| `requestToPOST: contents as: aMediaType` | `TeaRequest` for POST. |
| `requestToGET: aUrl accepting: aMediaRange` | `TeaRequest` for GET. |
| `requestToGETResourceIdentifiedBy: anIdentifier accepting: aMediaRange` | GET `<endpoint>/<id>`. |
| `requestToGETResourceIdentifiedBy: anIdentifier accepting: aMediaRange conditionalTo: anETag` | GET with `If-None-Match`. |
| `requestToPATCHResourceIdentifiedBy: anIdentifier with: contents accepting: aMediaRange conditionalTo: anETag` | PATCH `<endpoint>/<id>`. |
| `requestToDELETEResourceIdentifiedBy: anIdentifier` | DELETE `<endpoint>/<id>`. |
| `requestToGETSubresource: aSubresourceUrl identifiedBy: anIdentifier accepting: aMediaRange` | GET on a sub-resource path. |
| `parametersWith: anIdentifier` | A path-parameters dictionary keyed by `resourceController identifierKey`. |
| `newHttpRequestContext` | An empty `HttpRequestContext`. |
| `withJsonFromContentsIn: aResponse do: aBlock` | Parses the response body as `NeoJSONObject` and yields it. |
| `withJsonFromItemsIn: aResponse do: aBlock` | Yields `(NeoJSONObject fromString: contents) items`. |
| `assertCachingDirectivesFor: aResponse with: aString` | Caching-directive equality. |
| `assertExpiresHeaderFor: aResponse with: aDuration` | Expires-header tolerance check. |
| `baseUrl` (subclass responsibility) | Test base URL, e.g. `'http://portfolios.test' asUrl`. |
| `setUpResourceController` (subclass responsibility) | Where `resourceController := <YourController> workingWith: rootSystem authenticatedBy: …`. |

**Helpers your test class must define itself** in `private - support`: action-endpoint request builders (POST to `/cancel` etc.), the test fixtures (`postLongTermPortfolio`, `startManagingLongTermPortfolio`, `getPortfolios`, `getPortfolioIdentifiedBy:`), `newWriteHttpRequestContext` (returns an `HttpRequestContext` with the write permission), and any domain-specific assertion helpers. **Read JSON `links` via `at: 'links'` then `at: '<rel>'`** — there are no `selfLocation` / `entityTag` / `links activate` accessor shortcuts in the latest Stargate-SUnit. Read the `ETag` from `response headers at: 'ETag'`.


### Template

```smalltalk
Class {
    #name : 'PortfolioRESTfulControllerTest',
    #superclass : 'SingleResourceRESTfulControllerTest',
    #instVars : [ 'rootSystem' ],
    #category : 'Portfolio-API-Model-Tests',
    #package : 'Portfolio-API-Model-Tests'
}

{ #category : 'private - support' }
PortfolioRESTfulControllerTest >> baseUrl [

    ^ 'http://portfolios.test' asUrl
]

{ #category : 'running' }
PortfolioRESTfulControllerTest >> setUp [

    | repositorySystem |

    "Register the subsystems DIRECTLY — the `<Thing>ManagementModule`'s `toInstallOn:` only stores
     the root system (it registers on `install`), so `<Module> toInstallOn: rootSystem` alone leaves
     `rootSystem >> #<Thing>ManagementSystem` unresolved ('System implementing … not found'). This
     direct form mirrors the domain user-story test's setUpRequirements and always works."
    rootSystem := CompositeSystem new.
    repositorySystem := RepositoryProviderSystem new.
    repositorySystem register: InMemoryRepositoryProvider new as: #mainDB.
    rootSystem
        register: repositorySystem;
        register: PortfolioManagementSystem new.
    rootSystem startUp.

    super setUp  "calls setUpResourceController"
]

{ #category : 'running' }
PortfolioRESTfulControllerTest >> tearDown [

    rootSystem ifNotNil: [
        rootSystem shutDown.
        rootSystem := nil ].
    super tearDown
]

{ #category : 'running' }
PortfolioRESTfulControllerTest >> setUpResourceController [

    resourceController := PortfolioRESTfulController
        workingWith: rootSystem
        authenticatedBy: self authenticationFilter
]

{ #category : 'private - support' }
PortfolioRESTfulControllerTest >> systemUnderTest [

    ^ rootSystem >> #PortfolioManagementSystem
]

{ #category : 'tests - routes' }
PortfolioRESTfulControllerTest >> testRoutes [

    | routeSummaries |

    routeSummaries := ( resourceController routes
        collect: [ :route | route httpMethod , ' ' , route urlTemplate ] ) asSortedCollection asArray.
    self
        assert: routeSummaries
        equals: #(
            'DELETE /portfolios/<identifier:IsUUID>'
            'GET /portfolios'
            'GET /portfolios/<identifier:IsUUID>'
            'PATCH /portfolios/<identifier:IsUUID>'
            'POST /portfolios' )   "urlTemplate carries a LEADING SLASH"
]

{ #category : 'tests - creation' }
PortfolioRESTfulControllerTest >> testCreatePortfolio [

    | response |

    response := self postLongTermPortfolio.

    self
        assert: response isCreated;
        assert: response contentType asMediaType
            equals: resourceController portfolioVersion1dot0dot0MediaType;
        assert: response hasEntity;
        withJsonFromContentsIn: response do: [ :json |
            self
                assert: json name equals: 'Long-term';
                assert: json description equals: 'Buy and hold';
                assert: json owner equals: 'user-1';
                assert: ( json at: 'links' ) self
                    equals: response location asString ]
]

{ #category : 'tests - querying' }
PortfolioRESTfulControllerTest >> testGetPortfoliosWhenNoneAreManaged [

    | response |

    response := self getPortfolios.

    self
        assert: response isSuccess;
        withJsonFromContentsIn: response do: [ :json |
            self assert: json items isEmpty ]
]

{ #category : 'tests - querying' }
PortfolioRESTfulControllerTest >> testGetPortfolioNotFound [

    "The handler maps ObjectNotFound to a RAISED HTTPClientError at this layer (the HTTP
     server would turn it into a 404 response). So assert the raise, not `response isNotFound`."
    self
        should: [
            resourceController
                getPortfolioBasedOn: ( self
                    requestToGETResourceIdentifiedBy: UUID new asString
                    accepting: resourceController portfolioVersion1dot0dot0MediaType )
                within: self newHttpRequestContext ]
        raise: HTTPClientError notFound
]

{ #category : 'tests - querying' }
PortfolioRESTfulControllerTest >> testGetPortfolioWithMatchingETagReturns304 [

    | identified initial etag request response |

    identified := self startManagingLongTermPortfolio.
    initial := self getPortfolioIdentifiedBy: identified identifier.
    etag := initial headers at: 'ETag'.

    request := self
        requestToGETResourceIdentifiedBy: identified identifier
        accepting: resourceController portfolioVersion1dot0dot0MediaType
        conditionalTo: etag.

    response := resourceController getPortfolioBasedOn: request within: self newHttpRequestContext.
    self assert: response isNotModified
]

{ #category : 'tests - querying' }
PortfolioRESTfulControllerTest >> testGetPortfoliosCacheControl [

    | response |

    response := self getPortfolios.

    self
        assert: response isSuccess;
        assert: ( ( response headers at: 'Cache-Control' )
            anySatisfy: [ :directive | directive includesSubstring: '3600' ] )
]

{ #category : 'tests - creation' }
PortfolioRESTfulControllerTest >> testCreatePortfolioWithEmptyNameRaisesUnprocessableEntity [

    self
        should: [
            resourceController
                createPortfolioBasedOn: ( self
                    requestToPOST: '{"name":"","description":"x","image":"http://x","owner":"u"}'
                    as: resourceController portfolioVersion1dot0dot0MediaType )
                within: self newWriteHttpRequestContext ]
        raise: HTTPClientError unprocessableEntity
]

{ #category : 'tests - updates' }
PortfolioRESTfulControllerTest >> testUpdatePortfolioWithSameNameHasNoEffect [

    | original etag response |

    original := self startManagingLongTermPortfolio.
    etag := ( self getPortfolioIdentifiedBy: original identifier ) headers at: 'ETag'.

    response := resourceController
        updatePortfolioBasedOn: ( self
            requestToPATCHResourceIdentifiedBy: original identifier
            with: '{"name":"Long-term"}'
            accepting: resourceController portfolioVersion1dot0dot0MediaType
            conditionalTo: etag )
        within: self newWriteHttpRequestContext.

    self
        assert: response isSuccess;
        withJsonFromContentsIn: response do: [ :json |
            self assert: json name equals: 'Long-term' ]
]

{ #category : 'tests - deletion' }
PortfolioRESTfulControllerTest >> testDeletePortfolio [

    | original response |

    original := self startManagingLongTermPortfolio.
    response := resourceController
        deletePortfolioBasedOn: ( self requestToDELETEResourceIdentifiedBy: original identifier )
        within: self newWriteHttpRequestContext.

    self
        assert: response isNoContent;
        assert: self systemUnderTest portfolios isEmpty
]

{ #category : 'private - support' }
PortfolioRESTfulControllerTest >> postLongTermPortfolio [

    ^ resourceController
        createPortfolioBasedOn: ( self
            requestToPOST: '{"name":"Long-term","description":"Buy and hold","image":"http://x","owner":"user-1"}'
            as: resourceController portfolioVersion1dot0dot0MediaType )
        within: self newWriteHttpRequestContext
]

{ #category : 'private - support' }
PortfolioRESTfulControllerTest >> startManagingLongTermPortfolio [

    ^ self systemUnderTest startManagingPortfolio: ( Portfolio
        named: 'Long-term' describedAs: 'Buy and hold' withImage: 'http://x' ownedBy: 'user-1' )
]

{ #category : 'private - support' }
PortfolioRESTfulControllerTest >> newWriteHttpRequestContext [

    ^ super newHttpRequestContext
        permissions: { resourceController requiredPermissionForWriting };
        yourself
]

{ #category : 'private - support' }
PortfolioRESTfulControllerTest >> getPortfolios [

    ^ resourceController
        getPortfoliosBasedOn: ( self
            requestToGET: self baseUrl asString , '/portfolios'
            accepting: resourceController portfolioVersion1dot0dot0MediaType )
        within: self newHttpRequestContext
]

{ #category : 'private - support' }
PortfolioRESTfulControllerTest >> getPortfolioIdentifiedBy: anIdentifier [

    ^ resourceController
        getPortfolioBasedOn: ( self
            requestToGETResourceIdentifiedBy: anIdentifier
            accepting: resourceController portfolioVersion1dot0dot0MediaType )
        within: self newHttpRequestContext
]
```

### Conventions

- **Superclass `SingleResourceRESTfulControllerTest`** (Stargate-SUnit). Provides `requestToPOST:as:`, `requestToGETResourceIdentifiedBy:accepting:`, `requestToGETResourceIdentifiedBy:accepting:conditionalTo:`, `requestToDELETEResourceIdentifiedBy:`, `newHttpRequestContext`, `parametersWith:`, `withJsonFromContentsIn:do:`, `withJsonFromItemsIn:do:`, etc.
- **`setUp`** builds a Kepler `CompositeSystem` and **registers the subsystems directly** — a `RepositoryProviderSystem` holding `InMemoryRepositoryProvider new as: #mainDB`, plus `<Thing>ManagementSystem new` (and any dependency systems) — calls `startUp`, then `super setUp` (which triggers `setUpResourceController`). Do **not** rely on `<Thing>ManagementModule toInstallOn: rootSystem` alone: that `toInstallOn:` only *stores* the root system (registration happens on `install`), so the interface never resolves ("System implementing … not found"). If you prefer modules, call `(… toInstallOn: rootSystem) install` and register `InstalledModuleRegistrationSystem` first — the direct form skips that ceremony.
- **`setUpResourceController`** instantiates the controller against the test's `rootSystem`, with `self authenticationFilter` — provide a no-op filter (or Stargate's built-in `JWTBearerAuthenticationFilter` configured with a test secret) so auth flows can be exercised explicitly.
- **`tearDown`** calls `rootSystem shutDown` to release subsystems and resets the slot. Always wrap in `ifNotNil:`.
- **`newWriteHttpRequestContext`** must be defined locally to add `requiredPermissionForWriting` for tests that exercise authenticated writes successfully.
- **Drive the controller's API methods directly** — no real HTTP server in this layer.
- **Assert success dimensions** on the returned response: `isSuccess`/`isCreated`/`isNoContent`/`isNotModified`, `contentType asMediaType`, `hasEntity`, `location`, body shape via `withJsonFromContentsIn:do:` / `withJsonFromItemsIn:do:`. **Error paths RAISE** an `HTTPClientError` (`notFound` / `conflict` / `unprocessableEntity` / `badRequest` / `forbidden`) at this layer rather than returning a response — assert them with `should: […] raise: HTTPClientError <kind>` (see `abbaco-api-rest-controller` §5).
- **Cache-Control gotcha**: `( response headers at: 'Cache-Control' )` returns an **`Array`** (Zinc multi-value header), not a string. Use `anySatisfy: [ :directive | directive includesSubstring: '<expected-seconds>' ]`. The `max-age` seconds appear **only if the controller used `beAvailableFor:` / `beStaleAfter:`** — `expireIn:` alone sets `Expires`, not `max-age`, so a test asserting `'3600'` against an `expireIn:`-based controller fails (see `abbaco-api-rest-controller`).
- **ETag gotcha**: use the inherited `requestToGETResourceIdentifiedBy:accepting:conditionalTo:` helper. There is no `setIfNoneMatch:` on `ZnRequest`.
- **Category naming**: `tests - routes`, `tests - creation`, `tests - querying`, `tests - updates`, `tests - deletion`, `tests - lifecycle` (for action endpoints), `tests - content negotiation`.

### What every controller test must cover

- `testRoutes` — assert sorted route specs.
- Empty collection: `testGet<Things>WhenNoneAreManaged`.
- Single resource lookup happy + not-found.
- Create round-trip with content type, location, body shape.
- Update happy path + idempotent update with same value.
- Delete returns 204 and the resource is gone.
- Conflict on duplicate (when conflict checking is configured) → 409.
- Missing required field → 400 (`testCreate<Thing>MissingNameRaisesBadRequest`).
- Wrong JSON type → 422 (`testCreate<Thing>WithUnexpectedNameTypeRaisesUnprocessableEntity`).
- Empty / over-long string → 422 with the exact precondition message.
- ETag conditional GET returns 304.
- Cache-Control includes the configured `max-age`.
- Hypermedia `links.self` present on every encoded resource.
- For action endpoints: round-trip test (POST → 204 → POST again → 204; observable state changed) — see Section 3a.

### 3a. Action-endpoint round-trip (when applicable)

Action endpoints (`/cancel`, `/activate`, `/deactivate`) need a custom request-builder helper because Stargate-SUnit's `SingleResourceRESTfulControllerTest` does not ship a `requestToPOSTAction:identifiedBy:` selector. Define one in your test class's `private - support` category:

```smalltalk
{ #category : 'private - support' }
PaidSubscriptionRESTfulControllerTest >> requestToPOSTAction: actionUrl identifiedBy: anIdentifier [

    ^ TeaRequest
        fromZnRequest: ( ZnRequest post: actionUrl )
        pathParams: ( self parametersWith: anIdentifier )
]
```

The same shape covers a **sub-resource GET with query parameters** (e.g. `…/<id>/metrics?asOf=…`) — the inherited `requestToGETSubresource:identifiedBy:accepting:` takes no query string, so build the request directly and let the controller read the query with `httpRequest at: 'asOf' ifAbsent: […]`:

```smalltalk
{ #category : 'private - support' }
ThingMetricsRESTfulControllerTest >> requestToGETMetricsIdentifiedBy: anIdentifier asOf: anIsoDateOrNil [

    | url |
    url := self baseUrl asString , '/things/' , anIdentifier , '/metrics'.
    anIsoDateOrNil ifNotNil: [ :asOf | url := url , '?asOf=' , asOf ].
    ^ TeaRequest fromZnRequest: ( ZnRequest get: url asUrl ) pathParams: ( self parametersWith: anIdentifier )
]
```

Then write the round-trip test:

```smalltalk
{ #category : 'tests - lifecycle' }
PaidSubscriptionRESTfulControllerTest >> testCancelPaidSubscriptionIsIdempotent [

    | original cancelUrl first second |

    original := self startConfirmedSubscription.

    "Discover the cancel URL from the encoded resource's hypermedia links."
    self
        withJsonFromContentsIn: ( self getPaidSubscriptionIdentifiedBy: original identifier )
        do: [ :json | cancelUrl := ( ( json at: 'links' ) at: 'cancel' ) asUrl ].

    "First cancel"
    first := resourceController
        cancelPaidSubscriptionBasedOn:
            ( self requestToPOSTAction: cancelUrl identifiedBy: original identifier )
        within: self newWriteHttpRequestContext.
    self assert: first isNoContent.

    "Second cancel — idempotent"
    second := resourceController
        cancelPaidSubscriptionBasedOn:
            ( self requestToPOSTAction: cancelUrl identifiedBy: original identifier )
        within: self newWriteHttpRequestContext.
    self assert: second isNoContent.

    self
        withJsonFromContentsIn: ( self getPaidSubscriptionIdentifiedBy: original identifier )
        do: [ :json | self assert: json status equals: 'Cancelled' ]
]
```

The action URL **must** come from the previous GET response's `links.<action>` (HATEOAS), not from a synthesized `'/cancel'` string concatenation. The point is to exercise the encoder's hypermedia output as the test client.

## 4. API user-story tests (`<Thing>RESTfulAPIUserStoryTest`)

This layer starts a real `HTTPBasedRESTfulAPI` on a free port, sends real HTTP via `ZnClient`, and asserts numeric response codes — the only place to test JWT authorization end-to-end.

```smalltalk
Class {
    #name : 'PortfolioRESTfulAPIUserStoryTest',
    #superclass : 'SystemBasedUserStoryTest',
    #instVars : [ 'api', 'configuration', 'port' ],
    #category : 'Portfolio-API-Model-Tests',
    #package : 'Portfolio-API-Model-Tests'
}

{ #category : 'private - running' }
PortfolioRESTfulAPIUserStoryTest >> setUpRequirements [

    | repositorySystem |

    repositorySystem := RepositoryProviderSystem new.
    repositorySystem register: InMemoryRepositoryProvider new as: #mainDB.
    self
        registerSubsystem: repositorySystem;
        requireInstallationOf: PortfolioManagementModule
]

{ #category : 'running' }
PortfolioRESTfulAPIUserStoryTest >> setUp [

    super setUp.
    port := self freeListeningTCPPort.
    configuration := {
        ( #port -> port ).
        ( #serverUrl -> self serverUrl ).
        ( #debugMode -> true ).
        ( #operations -> {
            ( #authSchema -> 'jwt' ).
            ( #authAlgorithm -> 'HS256' ).
            ( #authSecret -> self secret ) } asDictionary ) }.
    api := HTTPBasedRESTfulAPI
        configuredBy: configuration
        installing: { ( PortfolioRESTfulController
            workingWith: rootSystem
            authenticatedBy: self authenticationFilter ) }.
    api install; start
]

{ #category : 'running' }
PortfolioRESTfulAPIUserStoryTest >> tearDown [

    super tearDown.
    api ifNotNil: [ api stop ].
    api := nil
]

{ #category : 'tests - authorization' }
PortfolioRESTfulAPIUserStoryTest >> testCreatePortfolioUnauthorizedDueToMissingCredentials [

    | response client |

    client := self newHttpClient.
    self configureWithMissingAuthentication: client.
    client
        url: self portfoliosURL;
        contents: '{"name":"X","description":"","image":"","owner":"u"}';
        contentType: PortfolioRESTfulController new portfolioVersion1dot0dot0MediaType.

    response := client post; response.
    self assertIsUnauthorized: response
]
```

### What this layer tests

For every authenticated route (POST, PATCH, DELETE, action endpoints):

| Scenario | Assertion |
|---|---|
| Missing `Authorization` header | 401, `WWW-Authenticate: Bearer` |
| Invalid signature | 401 |
| Insufficient permissions | 403 |

Plus content negotiation:

| Scenario | Assertion |
|---|---|
| Unsupported `Accept` header | 406 Not Acceptable |
| Unsupported `Content-Type` (POST/PATCH) | 415 Unsupported Media Type |

Helper methods (same shape as pepper):

```smalltalk
{ #category : 'private - support' }
PortfolioRESTfulAPIUserStoryTest >> newHttpClient [

    ^ ZnClient new
        beOneShot;
        timeout: 1;
        setBearerAuthentication: ( self tokenWithKey: self secret );
        ifFail: [ self fail ];
        yourself
]

{ #category : 'private - asserting' }
PortfolioRESTfulAPIUserStoryTest >> assertIsUnauthorized: aResponse [

    self
        assert: aResponse isAuthenticationRequired;
        assert: ( aResponse headers at: 'WWW-Authenticate' ) equals: 'Bearer'
]
```

## 5. PostgreSQL integration tests

The tests above all run against `InMemoryRepositoryProvider`. To validate the Sagan-RDBMS mapping, write a parallel test that swaps in `RDBMSRepositoryProvider` and exercises the persist-query round-trip.

```smalltalk
Class {
    #name : 'RDBMSPortfolioManagementSystemTest',
    #superclass : 'PortfolioManagementSystemUserStoryTest',
    #category : 'Portfolio-API-Model-Tests',
    #package : 'Portfolio-API-Model-Tests'
}

{ #category : 'private - running' }
RDBMSPortfolioManagementSystemTest >> repositoryProvider [

    ^ RDBMSRepositoryProvider using: ( Login new
        database: PostgreSQLPlatform new;
        username: 'postgres';
        password: 'secret';
        host: ( OSEnvironment current at: 'PG_HOSTNAME' ifAbsent: [ 'localhost' ] );
        port: 5432;
        databaseName: 'test';
        setSSL;
        yourself )
]

{ #category : 'private - running' }
RDBMSPortfolioManagementSystemTest >> setUpRequirements [

    | repositorySystem |

    repositorySystem := RepositoryProviderSystem new.
    repositorySystem register: self repositoryProvider as: #mainDB.
    self
        registerSubsystem: repositorySystem;
        requireInstallationOf: PortfolioManagementModule
]

{ #category : 'running' }
RDBMSPortfolioManagementSystemTest >> setUp [

    super setUp.
    ( rootSystem >> #RepositoryProviderSystem ) prepareForInitialPersistence
]

{ #category : 'running' }
RDBMSPortfolioManagementSystemTest >> tearDown [

    rootSystem ifNotNil: [
        ( rootSystem >> #RepositoryProviderSystem ) destroyRepositories ].
    super tearDown
]

{ #category : 'tests - mapping' }
RDBMSPortfolioManagementSystemTest >> testPersistAndQueryRoundTrip [

    | identified found |

    identified := self systemUnderTest startManagingPortfolio: ( Portfolio
        named: 'Long-term'
        describedAs: 'Buy and hold'
        withImage: 'http://x'
        ownedBy: 'user-1' ).

    found := self systemUnderTest portfolioIdentifiedBy: identified identifier.

    self
        assert: found name equals: 'Long-term';
        assert: found description equals: 'Buy and hold';
        assert: found image equals: 'http://x';
        assert: found owner equals: 'user-1'
]
```

### Rules

- **Subclass the in-memory user-story test** and override `setUpRequirements` (or `repositoryProvider` if the parent uses it). The shared test methods immediately exercise the Postgres mapping.
- **Add `<RDBMSSubclass> class >> shouldInheritSelectors [ ^ true ]` as soon as the subclass defines *any* of its own test methods.** SUnit inherits a parent's tests only when the subclass has **no** test selectors of its own (or the parent is abstract). Add one mapping-coverage test (`testPersistAndQueryRoundTrip…`) without this override and the subclass **silently stops running the entire inherited suite** — it reports just your new test as green while the real coverage vanishes. With `shouldInheritSelectors ^ true` it runs inherited **plus** own. (Same trap applies to any RDBMS *controller*-test subclass that adds its own tests.)
- **`setUp` calls `prepareForInitialPersistence`** to recreate the schema. **`tearDown` calls `destroyRepositories`** to drop it.
- **Add at least one `testPersistAndQueryRoundTrip` per aggregate.** Construct an instance with every mapped field populated, store it, fetch it back by identifier, assert every field. This is the test that catches descriptor-vs-table drift.
- **Add `testFilterByX` for every indexed column** — exercises `findAllMatching:` with `:criteria | criteria satisfying: …` on real Postgres (Glorp can fail on criteria builders even when in-memory passes).
- **For status-driven entities, add a transition test**: store with status A, update to status B, requery, assert the conversion in both directions.
- **The connection parameters come from environment variables** (`PG_HOSTNAME` etc.) so the same test runs locally (against a manually-started Postgres) and in CI (against a sidecar container — Section 6).

## 6. CI integration — GitHub Actions

CI is **GitHub Actions**. Legacy Jenkinsfiles are being phased out and should not be carried forward.

### `.github/workflows/unit-tests.yml`

```yaml
name: Unit Tests

on:
  push:
    branches:
      - release-candidate
    tags:
      - '**'
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
      - name: Checkout
        uses: actions/checkout@v6

      - name: Start PostgreSQL
        run: |
          docker rm --force Portfolio-API-postgresql || true
          docker run --name Portfolio-API-postgresql -d \
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
          until docker exec Portfolio-API-postgresql pg_isready -U postgres -d test; do
            sleep 1
          done

      - name: Set up Smalltalk CI
        uses: hpi-swa/setup-smalltalkCI@v1
        with:
          smalltalk-image: Pharo64-11

      - name: Load image and run tests
        run: smalltalkci -s Pharo64-11 .smalltalkci/unit-tests.ston
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        timeout-minutes: 15

      - name: Stop PostgreSQL
        if: always()
        run: |
          docker kill Portfolio-API-postgresql || true
          docker rm --force Portfolio-API-postgresql || true

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
      - name: Checkout
        uses: actions/checkout@v6

      - name: Extract metadata
        id: docker_metadata
        uses: docker/metadata-action@v6
        with:
          images: ${{ vars.REGISTRY }}/abbaco/portfolio-api
          tags: |
            type=ref,event=branch
            type=ref,event=pr
            type=semver,pattern=v{{version}}
            type=semver,pattern=v{{major}}.{{minor}}
            type=semver,pattern=v{{major}}

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v4

      - name: Login to Docker registry
        uses: docker/login-action@v4
        with:
          registry: ${{ vars.REGISTRY }}
          username: ${{ vars.REGISTRY_USERNAME }}
          password: ${{ secrets.REGISTRY_TOKEN }}

      - name: Build and push Docker image
        uses: docker/build-push-action@v7
        with:
          context: .
          file: ./docker/Dockerfile
          push: true
          tags: ${{ steps.docker_metadata.outputs.tags }}
          labels: ${{ steps.docker_metadata.outputs.labels }}
          provenance: false
          sbom: false
```

### Notes

- The Postgres container is started with `docker run` (not `services:`) so the test job has full control over its lifecycle and can run integration tests that toggle `prepareForInitialPersistence`/`destroyRepositories` between tests. SSL is on with self-signed certs because Sagan's `setSSL` requires it.
- **`.smalltalkci/unit-tests.ston`** is the Smalltalk CI configuration. It loads the baseline and runs the test groups for `<Thing>-Model-Tests` and `<Thing>-API-Model-Tests` in one image load. The integration tests (`RDBMSPortfolioManagementSystemTest`) connect to `localhost:5432` because the host port is mapped.
- **`build-and-publish`** is gated on `release-candidate` pushes, tag pushes, and explicit dispatch with `push_image: true`. It uses the Docker registry variables (`REGISTRY`, `REGISTRY_USERNAME`, `REGISTRY_TOKEN`) — same convention as the abbaco-subscription-api workflow.
- **Newman/Postman is not currently in CI.** The `api-tests/run-tests.sh` script is run locally for now. If you decide to add a Newman job, model it on the unit-tests workflow but launch the full `docker-compose.yml` from `api-tests/`.

### Per-API additional workflows

The subscription-api also has cron-driven workflows (`cancel-overdue-trials.yml`, `cancel-overdue-paid-subscriptions.yml`, etc.) that invoke long-running maintenance commands. Add similar workflows when an API has its own scheduled maintenance jobs; otherwise omit them.

## 7. Newman / Postman tests (`api-tests/`)

End-to-end HTTP tests, Postman collection executed by Newman in docker-compose:

```
api-tests/
├── docker-compose.yml      # api + postgres + (optionally) other apis the SUT depends on
├── enviroment.json         # Postman environment variables (note: spelled "enviroment" in existing abbaco code)
├── tests.json              # Postman collection
└── run-tests.sh            # orchestration: bring up postgres, build and start API, run newman, tear down
```

### `docker-compose.yml`

Use the abbaco subscription-api shape:

```yaml
services:
  db:
    image: postgres:14
    command: -c ssl=on -c ssl_cert_file=/etc/ssl/certs/ssl-cert-snakeoil.pem -c ssl_key_file=/etc/ssl/private/ssl-cert-snakeoil.key
    environment:
      POSTGRES_PASSWORD: secret
      POSTGRES_USER: postgres
      POSTGRES_DB: test

  portfolio-api:
    build: { context: ../ }
    depends_on: [ db ]
    environment:
      STARGATE__PUBLIC_URL: http://portfolio-api:8080
      STARGATE__PORT: 8080
      STARGATE__OPERATIONS_SECRET: API-tests
      AUTHENTICATION__AUTHENTICATION_SECRET: api-tests-secret
      SAGAN__PG_HOSTNAME: db
      SAGAN__PG_PORT: 5432
      SAGAN__PG_USERNAME: postgres
      SAGAN__PG_PASSWORD: secret
      SAGAN__PG_DATABASE_NAME: test
      SAGAN__CREATE_EMPTY_DATABASE: 'true'
    volumes:
      - ./logs/:/opt/pharo/logs/
```

### `run-tests.sh`

```bash
#!/usr/bin/env bash
set -eux

COMPOSE_FILE=docker-compose.yml
export COMPOSE_FILE
export COMPOSE_PROJECT_NAME=$(echo "${1:-api-tests}" | tr '[:upper:]' '[:lower:]' | tr --delete '.')

docker compose up -d db
sleep 10

docker compose up -d --build portfolio-api
sleep 5

docker run --rm \
    --volume "$(pwd)":/etc/newman \
    --network "${COMPOSE_PROJECT_NAME}_default" \
    postman/newman:6-alpine \
    run tests.json \
    --environment enviroment.json \
    --color off \
    --disable-unicode \
    --reporters cli,junit \
    --reporter-junit-export api-test-result.xml \
    || docker compose logs portfolio-api

docker compose down || docker compose kill
```

### Postman test conventions

One folder per operation (`Querying`, `Creation`, `Updating`, `Deletion`, `Authorization`). Each request asserts at minimum:

```javascript
pm.test('Successful request', () => pm.expect(pm.response).to.be.success);
pm.test('Content-Type matches', () => {
    pm.expect(pm.response.headers.get('Content-Type'))
        .to.equal('application/vnd.mercap.portfolio+json;version=1.0.0');
});
pm.test('Self link present', () => {
    const data = pm.response.json();
    pm.expect(data.links.self).to.have.string(pm.variables.get('BASE_URL') + '/portfolios');
});
```

Use Postman collection variables (`{{BASE_URL}}`, `{{AUTHENTICATION_BEARER}}`) and chain requests with `pm.collectionVariables.set('portfolioId', …)` so subsequent requests can reference created resources. For action endpoints, **pull the action URL from the previous response's `links.<action>` rather than concatenating `/cancel` to a hardcoded base** — same rule as the controller-test layer.


## 8. Common mistakes

- **Adding `WithoutJWT` tests to the controller test class** — auth failures belong in the API user-story test (Section 4), where they appear as 401/403 HTTP responses. The controller test sees `HTTPForbidden` from `assertRequestIsAuthorizedTo:within:`, not HTTP 401.
- **Asserting on `( response headers at: 'Cache-Control' ) equals: 'max-age=…'`** — the header value is an `Array` of directives. Use `anySatisfy: [ :d | d includesSubstring: '<seconds>' ]`.
- **Calling `request setIfNoneMatch: etag`** — that selector does not exist on `ZnRequest`. Use the inherited `requestToGETResourceIdentifiedBy:accepting:conditionalTo:` helper.
- **Splitting an action round-trip into two tests** — loses HATEOAS discovery and idempotency verification. Use one test that POSTs the same action URL twice and asserts `204` both times.
- **Asserting `response isSuccess` on a 204 action endpoint** — works (`isSuccess` covers 2xx), but `isNoContent` is more precise and catches the regression where someone changes `thenDo:` back to `get:`.
- **Synthesizing the action URL** (`( self urlForResourceIdentifiedBy: id ) / 'cancel'`) instead of following `links.cancel` from the server response — the encoder's hypermedia output is never exercised.
- **Using `RDBMSRepositoryProvider` in the controller-test layer** — order-dependent test failures because state leaks across tests. Always use `InMemoryRepositoryProvider` here. The Postgres mapping has its own dedicated layer (Section 5).
- **Skipping the round-trip integration test** — every aggregate needs at least one `testPersistAndQueryRoundTrip` in the RDBMS layer; otherwise descriptor drift (a renamed instVar, a wrong column name, a broken conversion) ships silently.
- **Hard-coding `localhost` in the integration test connection** — works locally, fails in CI when Postgres is on a sidecar with a different hostname. Read from `OSEnvironment current at: 'PG_HOSTNAME' ifAbsent: [ 'localhost' ]`.
- **Forgetting `tearDown` to call `destroyRepositories`** — leaves data in the test database between runs; the next test's `prepareForInitialPersistence` recreates the schema but not before pollution causes failures.
- **Adding new routes without updating `tests.json`** — Pharo CI passes, integration tests silently ignore the new endpoint, and a broken route ships.
- **Adding new mapped fields without updating `testPersistAndQueryRoundTrip`** — the new field's persist/load is never exercised.
- **Carrying over pepper's localized assertions** (`inPortugueseDo:`, `Accept-Language: pt` round-trips) — abbaco APIs are not localized; these tests fail because there's no translator wired.
- **Carrying over a Jenkinsfile** — CI is GitHub Actions. Delete legacy Jenkinsfiles when copying scaffolding from older projects.
