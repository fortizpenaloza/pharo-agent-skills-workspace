---
name: ba-st-stargate
description: Use when building a Pharo/GS64 HTTP JSON REST API - declaring resources/controllers/routes, content negotiation with versioned media types, HATEOAS/hypermedia links, pagination, ETag-based conditional requests, or operational endpoints (health-check, metrics, app-info). Covers the Stargate framework on Teapot/Zinc.
---

# Stargate — RESTful API Framework

Stargate is a declarative REST framework built on Teapot/Zinc. One controller per resource; routes declared with `RouteSpecification`; serialization, pagination, caching, and HATEOAS wired through `RESTfulRequestHandlerBuilder`. Operational plugins (`health-check`, `metrics`, `application-info`, `application-control`, `application-configuration`, `loggers`) are first-class.

Repo: `/home/mtabacman/Development/Repos/ba-st-skills/Stargate/`. Read `docs/how-to/`, `docs/explanation/`, and the Pet-Store example code — they are the main reference.

## Installation

```smalltalk
Metacello new
    baseline: 'Stargate';
    repository: 'github://ba-st/Stargate:release-candidate';
    load: 'Development'.
```

Groups: `Core` (minimal), `Deployment` (+ all operational plugins), `Examples`, `Development`.

## Core concepts

| Class | Role |
|---|---|
| `HTTPBasedRESTfulAPI` | Entry point. Holds config + controllers, starts the Teapot server. |
| `SingleResourceRESTfulController` | Abstract base for a per-resource controller. Provides `typeIdConstraint`, `endpoint`. |
| `RouteSpecification` | `handling: <HTTP verb> at: <path-template> evaluating: <block>`. |
| `RESTfulRequestHandlerBuilder` | Fluent builder that configures encoding/decoding, pagination, ETags, HATEOAS, and caching. |
| `HttpRequestContext` | Per-request bag: pagination links, hypermedia controls, custom state. |
| `EntityTagHasher` | Builds content-based ETags. |

Media types are declared vendor+version:
```smalltalk
self jsonMediaType: 'pet' vendoredBy: 'stargate' version: '1.0.0'
"=> application/vnd.stargate.pet+json;version=1.0.0"
```

## Building an API — full worked example

```smalltalk
"1. Resource class"
Object subclass: #Pet ...

Pet class >> named: aName ofType: aPetType
    ^ self new initializeNamed: aName ofType: aPetType withStatus: 'available'

"2. Controller subclass"
SingleResourceRESTfulController subclass: #PetsRESTfulController
    instanceVariableNames: 'petsRepository requestHandler'
    classVariableNames: ''
    package: 'PetStore'

PetsRESTfulController >> typeIdConstraint  ^ IsInteger   "=> route: /pets/<id:IsInteger>"
PetsRESTfulController >> endpoint          ^ 'pets'

"3. Configure the request handler once, in initialize"
PetsRESTfulController >> initialize
    super initialize.
    petsRepository := PetsRepository new.
    requestHandler := RESTfulRequestHandlerBuilder new
        handling: 'pets'
        locatingResourcesWith:  [ :pet :ctx | petsRepository identifierOf: pet ]
        extractingIdentifierWith: [ :req | self identifierIn: req ];
        beHypermediaDriven;
        paginateCollectionsWithDefaultLimit: 5;
        decodeToNeoJSONObjectWhenAccepting: self petMediaType;
        whenResponding: self petMediaType encodeToJsonApplying: [ :pet :ctx :writer |
            writer for: Pet do: [ :m | m mapInstVars ] ];
        createEntityTagHashing: [ :hasher :pet :ctx |
            hasher include: (petsRepository identifierOf: pet) ];
        build

"4. Route declaration"
PetsRESTfulController >> declareGetPetsRoute
    ^ RouteSpecification
        handling: #GET
        at: self endpoint
        evaluating: [ :req :ctx |
            requestHandler
                from: req within: ctx
                getCollection: [ :pagination |
                    petsRepository between: pagination start and: pagination end ] ]

PetsRESTfulController >> declarePostPetRoute
    ^ RouteSpecification
        handling: #POST
        at: self endpoint
        evaluating: [ :req :ctx |
            requestHandler
                withRepresentationIn: req within: ctx
                createResourceWith: [ :json | Pet named: json name ofType: json type ]
                thenDo: [ :pet | petsRepository store: pet ] ]

"5. Start the server"
api := HTTPBasedRESTfulAPI
    configuredBy: { #port -> 8080. #serverUrl -> 'http://localhost:8080' asUrl }
    installing: { PetsRESTfulController new }.
api install; start.
```

## Content negotiation and versioning

Register one encoder/decoder pair per supported media type; the framework negotiates via `Accept`/`Content-Type` and sets the `Vary` response header automatically. Multiple versions coexist in the same controller:

```smalltalk
petV1 ^ self jsonMediaType: 'pet' vendoredBy: 'stargate' version: '1.0.0'
petV2 ^ self jsonMediaType: 'pet' vendoredBy: 'stargate' version: '2.0.0'

builder
    whenResponding: self petV1 encodeToJsonApplying: [ :pet :ctx :w | w for: Pet do: [ :m | m mapInstVars ] ];
    whenResponding: self petV2 encodeToJsonApplying: [ :pet :ctx :w | w for: Pet do: [ :m | m mapInstVars. m mapNewFieldsIn: pet ] ]
```

Client picks version by `Accept: application/vnd.stargate.pet+json;version=2.0.0`.

## HATEOAS / hypermedia controls

```smalltalk
builder beHypermediaDrivenBy: [ :linkBuilder :order :ctx :orderLocation |
    linkBuilder addLink: (orderLocation / 'complete') relatedTo: 'complete'.
    linkBuilder addLink: (orderLocation / 'comments') relatedTo: 'comments' ].

"In the encoder, map controls into the JSON payload"
writer for: Order do: [ :m |
    m mapInstVars.
    m mapAsHypermediaControls: [ :order | ctx hypermediaControlsFor: order ] ]

"Response body shape: { ..., \"links\": { \"self\": ..., \"complete\": ..., \"comments\": ... } }"
```

> `addLink:relatedTo:` (and `addAsSelfLink:`) send `asWebLink` to the argument — pass a **`ZnUrl`** (build sub-resource URLs with `/` as above, `orderLocation / 'complete'`), **not a bare `String`**. A String is parsed as an RFC Link-header value (`<url>; rel=…`) and fails with "Missing <".

## Pagination

```smalltalk
builder paginateCollectionsWithDefaultLimit: 5.

"Inside the handler:"
requestHandler from: req within: ctx getCollection: [ :pagination |
    ctx addPaginationLinkStartingAt: 1                     limitedTo: pagination limit relatedTo: 'first'.
    pagination start > 1 ifTrue: [
        ctx addPaginationLinkStartingAt: pagination start - pagination limit
            limitedTo: pagination limit relatedTo: 'prev' ].
    (petsRepository between: pagination start and: pagination end) ].
```

Client query: `GET /pets?start=1&limit=5`. Response body carries `_links` with `first`/`prev`/`next`/`last`.

## ETags & conditional requests

Two strategies:
```smalltalk
"A. Hash-based (content-derived)"
builder createEntityTagHashing: [ :hasher :pet :ctx |
    hasher
        include: (repo identifierOf: pet);
        include: (repo lastModificationOf: pet) ].

"B. Custom"
builder createEntityTagWith: [ :pet :mediaType :ctx :handler |
    'pet-' , pet name hash asString ].
```

The framework automatically:
- answers `304 Not Modified` when `If-None-Match` matches,
- answers `412 Precondition Failed` when `If-Match` does not match on writes implemented with `from:within:get:thenUpdateWith:` (typically PUT/PATCH), and `428 Precondition Required` when such a write omits it (`from:within:get:thenUpdateWith:` enforces this — no manual ETag handling in the controller).

## Caching (Cache-Control)

`directCachingWith:` configures `Cache-Control`; its condition block is evaluated with `(response, resource)`, so you can vary by media type or by the resource itself:

```smalltalk
builder directCachingWith: [ :caching |
    caching
        when: [ :response :resource | response contentType = self petMediaType ]
        apply: [ caching beAvailableFor: 1 hour; mustRevalidate ] ].
```

Directives (sent to `caching`):
- **`expireIn: aDuration`** sets the **`Expires`** header (absolute time) — it does **NOT** set `max-age`.
- **`beStaleAfter: aDuration`** sets **`max-age`**. **`beAvailableFor: aDuration`** = `bePublic` + `beStaleAfter:` + `expireIn:` (sets `public, max-age, Expires` together).
- `beImmutable`, `bePublic` / `bePrivate`, `mustRevalidate` / `requireRevalidation`, `doNotTransform`, `doNotCache`, `doNotStore`, `doNotExpire`.

So `public, max-age=3600` is `beAvailableFor: 1 hour` (or `bePublic; beStaleAfter: 1 hour`) — **`expireIn: 1 hour` alone yields no `max-age`.** On the response, `Cache-Control` is an **`Array`** of directive strings.

## Errors → HTTP status

`RESTfulRequestExceptionHandler` maps domain exceptions to HTTP errors automatically, inside the request handler's blocks:

| Exception | HTTP |
|---|---|
| `ObjectNotFound` | 404 Not Found |
| `ConflictingObjectFound` | 409 Conflict |
| `InstanceCreationFailed` | 422 Unprocessable Entity |
| `KeyNotFound` / `NeoJSONParseError` | 400 Bad Request |
| `TeaNoSuchParam` | 400 (missing query parameter) |

`from:within:get:thenUpdateWith:` / `withRepresentationIn:within:createResourceWith:thenDo:` wrap their work in `handleConflictsDuring:` (→409) and `handleDecodingFailedDuring:` (→400/422); `from:within:get:` maps `ObjectNotFound` (→404). **When a controller method is invoked directly** (no running server — e.g. a controller test) these surface as a **raised `HTTPClientError`** (`notFound`/`conflict`/`unprocessableEntity`/`badRequest`/`forbidden`); the running server turns the raise into the response. Extend the mapping with `addAsConflictError:` / `addAsNotFoundError:` / `addAsUnprocessableEntityError:` on the exception handler.

## Operations plugin

Configure operational endpoints under `#operations`:
```smalltalk
api := HTTPBasedRESTfulAPI
    configuredBy: { #operations ->
        (Dictionary new
            at: #authSchema    put: 'jwt';
            at: #authAlgorithm put: 'HS256';
            at: #authSecret    put: 'secret';
            at: 'health-check' put: { #enabled -> true } asDictionary;
            yourself) }.
```

Available plugins: `health-check`, `metrics` (Prometheus format), `application-info`, `application-control`, `application-configuration`, `loggers`.

Endpoints (require appropriate permissions):
- `GET    /operations/plugins`
- `GET    /operations/plugins/<endpoint>`
- `PATCH  /operations/plugins/<endpoint>` — enable/disable
- `POST   /operations/plugins/health-check` — run health check (PASS/WARN/FAIL)
- `GET    /operations/plugins/application-info`
- `GET    /operations/plugins/metrics`

## Gotchas & v7→v9 migration notes

- **Always set `#serverUrl`** in the config — it's the base for hypermedia links. Behind a proxy, use the external URL, not the bound port.
- **Use JWT for production operations**; basic auth is local-dev only.
- **`RESTfulRequestHandlerBuilder` replaced subclass hooks.** Legacy `ResourceRESTfulController`-style subclass-responsibility methods are gone; configure via the builder.
- **`ResourceRESTfulControllerSpecification` deprecated** — use builder configuration blocks.
- **Hypermedia methods now take `within:` context.** `mediaControlsFor:within:`, `entityTagOf:encodedAs:within:`.
- **`holdAsHypermediaControls:forSubresource:` → `holdAsHypermediaControls:for:`** — always scoped to an object.
- **Pagination: `addPaginationControl:` → `addPaginationControls:`** (builder-based).
- **`applicationBaselineName` → `projectName`** on `StargateApplication` (used to locate `BaselineOf*` for version detection).
- **Default `stackTraceDumper` is text now** — override to binary if you need Fuel context.
- **Use strings for header names as dictionary keys** — String/Symbol equality differs on GemStone.
- **ETag body = body after encoding**; hash dependencies must include everything visible to the client, or you'll serve stale 304s.
- **`beAvailableFor:`, `doNotExpire`, `requireRevalidation`** on the builder map to Cache-Control directives — prefer them over hand-built headers. (See the Caching section — `expireIn:` sets `Expires`, not `max-age`.)
- **`route urlTemplate` carries a leading slash** (`/pets`, `/pets/<id:IsInteger>`) — match that exact form when asserting route lists in tests.

## When to reach for Stargate

Any JSON HTTP API where you care about content negotiation, HATEOAS, ETag-based caching, versioning, or production observability. If you only need a three-route stub, Teapot alone is less ceremony — but the moment you want a second version or a health check, Stargate pays back the setup.

## Related skills

- `ba-st-hyperspace` — ETag/MediaType/LanguageTag primitives Stargate builds on.
- `ba-st-stargate-consul` — auto-register the running API with HashiCorp Consul.
- `ba-st-launchpad` — ship the API as a Launchpad application with Docker.
- `ba-st-bell` — wire structured logs into the server.
- `ba-st-superluminal` — call downstream APIs from inside a controller.
