---
name: ba-st-superluminal
description: Use when writing Smalltalk code that calls external HTTP APIs - GET/POST/PUT/PATCH/DELETE with headers, bearer or basic auth, JSON/form/multipart bodies, query strings, response handling, caching with ETags, or building a typed client class. Covers HttpRequest and RESTfulAPIClient.
---

# Superluminal — HTTP Client Toolkit

Superluminal provides two layers:
1. **`HttpRequest`** — a fluent request builder (URL, headers, query string, body).
2. **`RESTfulAPIClient`** — a higher-level client with connection pooling, in-memory caching, automatic ETag handling (`If-None-Match`/`If-Match`), and typed error raising.

Repo: `/home/mtabacman/Development/Repos/ba-st-skills/Superluminal/`. Integration scripts in `api-tests/`; reference under `docs/reference/`.

## Installation

```smalltalk
Metacello new
    baseline: 'Superluminal';
    repository: 'github://ba-st/Superluminal:release-candidate';
    load: 'API Client'.
```

Groups: `Core` (minimal builder), `API Client` (full client), `Tests`, `Development`. `Superluminal-Deprecated-V2` is available for migration from v1.

## Building requests

```smalltalk
"1. GET with a bearer token"
HttpRequest
    get: 'https://api.example.com/users'
    configuredUsing: [ :req | req headers setBearerTokenTo: 'token123' ].

"2. POST with a JSON body"
HttpRequest
    post: 'https://api.example.com/users'
    configuredUsing: [ :req |
        req body json: (Dictionary new at: 'name' put: 'Alice'; yourself) ].

"3. GET with query parameters"
HttpRequest
    get: 'https://api.example.com/search'
    configuredUsing: [ :req |
        req queryString: [ :qs |
            qs fieldNamed: 'q'     pairedTo: 'pharo'.
            qs fieldNamed: 'limit' pairedTo: 10 ] ].

"4. Form-encoded POST"
HttpRequest
    post: 'https://api.example.com/login'
    configuredUsing: [ :req |
        req body formUrlEncoded: [ :form |
            form fieldNamed: 'username' pairedTo: 'user1'.
            form fieldNamed: 'password' pairedTo: 'secret' ] ].

"5. Multipart file upload"
HttpRequest
    post: 'https://api.example.com/files'
    configuredUsing: [ :req |
        req body multiPart: [ :mp |
            mp fieldNamed: 'file'
               attaching: (FileReference fromString: '/path/file.txt') contents ] ].
```

Other verbs: `put:`, `patch:`, `delete:`, `query:` — same shape.

## Authentication helpers

```smalltalk
req headers setBearerTokenTo: 'eyJhbGc...'.            "Authorization: Bearer ..."
req headers setUsername: 'user' andPassword: 'pass'.   "HTTP Basic"
req headers setAuthorizationTo: 'Digest username="user"'.   "any scheme, raw"
req headers set: 'X-API-Key' to: 'apikey123'.          "custom header"
```

## Sending and handling responses

```smalltalk
apiClient := RESTfulAPIClient cachingOnLocalMemory.

"2xx → success block"
apiClient
    get: 'https://httpbin.org/get'
    withSuccessfulResponseDo: [ :body |
        data := NeoJSONObject fromString: body.
        self show: data ].

"Accept header shortcut"
apiClient
    get: 'https://api.example.com/users'
    accepting: ZnMimeType applicationJson asMediaType
    withSuccessfulResponseDo: [ :users | ... ].

"4xx / 5xx → HTTPClientError / HTTPServerError (from Hyperspace)"
[ apiClient
    deleteAt: 'https://api.example.com/item/1'
    configuredBy: [ :req | ]
    withSuccessfulResponseDo: [ :_ | ] ]
on: HTTPClientError notFound
do: [ :err | "handle 404" ].
```

Errors carry `code` and a `message` extracted from the response JSON's `message` field (or a default).

## ETag caching (automatic)

`RESTfulAPIClient cachingOnLocalMemory` transparently stores responses keyed by `(URL, body, headers)` along with their `ETag` and `Cache-Control`. On the next matching GET/QUERY it sets `If-None-Match`; on 304 it returns the cached body without re-downloading. On PUT/PATCH/DELETE it automatically sends `If-Match` using the last-seen ETag (mid-air collision protection).

- Pool: up to 5 connections per authority+port, min 1 idle.
- Call `apiClient finalize` to close pools on cleanup.
- `no-store` caching directive opts out (response is fetched but not persisted).
- Writes to a location clear its cached entries.

## Custom typed clients — recommended pattern

Compose a client class around `RESTfulAPIClient`. Keep URLs and media types centralized:

```smalltalk
MyGitHubClient >> initializeOn: anApiClient
    apiClient := anApiClient

MyGitHubClient >> repositoriesFor: username
    | url |
    url := 'https://api.github.com/users/{1}/repos' expandMacrosWith: username.
    ^ apiClient
        get: url
        accepting: ZnMimeType applicationJson asMediaType
        withSuccessfulResponseDo: [ :body |
            (NeoJSONObject fromString: body) collect: [ :repo | repo name ] ]

"Usage"
client := MyGitHubClient new initializeOn: RESTfulAPIClient cachingOnLocalMemory.
repos  := client repositoriesFor: 'pharo-project'.
```

Don't create a fresh `RESTfulAPIClient` per call — it's the thing holding pools and caches.

## Testing with Teachable

`Superluminal-SUnit-Model` ships `APIClientTest` which uses Teachable (a stub library) to fake Zinc responses without a network.

```smalltalk
self
    configureHttpClientToRespondWith: ((self jsonOkResponseWith: #(1 2 3))
        addCachingDirective: 'Max-Age=60';
        setEntityTag: '"etag123"';
        yourself).

"Verify that the client set If-Match correctly on the follow-up"
self httpClient
    whenSend: #setIfMatchTo:
    evaluate: [ :etag | self assert: etag equals: '"etag123"' asEntityTag ].
```

## No built-in retries / pagination / hypermedia

Retries and hypermedia (HAL/HATEOAS) traversal are not in core. Wrap calls externally if you need retries, or imitate the `#retry` pattern used in service discovery:

```smalltalk
options at: #retry put: [ :retry |
    retry upTo: 3 timesEvery: 200 milliSeconds;
          on: Error evaluating: [ :attempt :err | ... log ... ] ].
```

## Gotchas

- **V1 → V2 migration.** Load `Superluminal-Deprecated-V2` to keep old API calls compiling during the migration; replace them and unload.
- **ETags and `no-store`.** A response with `Cache-Control: no-store` is NOT cached, so follow-up writes won't auto-attach `If-Match`. If you rely on mid-air collision protection, ensure the server emits an ETag without `no-store`.
- **Cache key includes body** (for QUERY/POST variants). Different bodies = different cache entries.
- **Cache invalidates on writes.** A PUT/POST/PATCH/DELETE to a location clears its GET cache — subsequent GETs will hit the network.
- **Exception hierarchy.** All 4xx subclass `HTTPClientError`; 5xx subclass `HTTPServerError`. Match specific subclasses (e.g., `HTTPClientError unauthorized`) to keep handlers focused.
- **Connection pool leaks.** Always `finalize` a `RESTfulAPIClient` you built transiently, or let GC close it — the pool keeps sockets open.
- **Don't wrap everything in an API client class.** Single-call scripts are clearer with `HttpRequest` directly; reserve `RESTfulAPIClient` + a typed class for repeated use.

## When to reach for Superluminal

Any time a Smalltalk service needs to call another HTTP service — downstream APIs, service discovery, third-party vendors. Inside a Stargate controller, instantiate one `RESTfulAPIClient` per external dependency and keep it for the application lifetime.
