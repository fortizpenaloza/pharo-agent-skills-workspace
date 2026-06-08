---
name: ba-st-hyperspace
description: Use when writing Smalltalk HTTP code (client or server) that needs RFC-level helpers - URL building/encoding, Media Type negotiation with versions/quality, ETag generation and conditional requests, Link/WebLink headers, Accept-Language/Content-Language, CURIE expansion, Cache-Control directives. Building block layer between Zinc and application code.
---

# Hyperspace — HTTP/Internet Building Blocks on Zinc

Hyperspace provides standards-compliant helpers on top of [Zinc HTTP Components](https://github.com/svenvc/zinc): ETags (RFC 7232), Link headers (RFC 5988), BCP 47 language tags, vendor media types with quality/version, CURIEs, Cache-Control parsing, and HTTP error hierarchies. It is used internally by Stargate (server) and Superluminal (client) — you reach for it directly when building custom HTTP logic.

Repo: `/home/mtabacman/Development/Repos/ba-st-skills/Hyperspace/`.

## Installation

```smalltalk
Metacello new
    baseline: 'Hyperspace';
    repository: 'github://ba-st/Hyperspace:release-candidate';
    load: 'Development'.
```

Groups: `Deployment`, `Tests`, `Development`. Hyperspace also installs Buoy as a transitive dependency.

## What it provides

| Concern | Class(es) | RFC |
|---|---|---|
| ETag validators | `EntityTag` | 7232 |
| Link header values | `WebLink` | 5988 |
| Media types with quality/version | `ZnMimeType` extensions | 7231 |
| Language tags | `LanguageTag` (from Buoy), Zn extensions | BCP 47 |
| CURIEs | `CURIE`, `SafeCURIE` | W3C |
| Cache-Control directives | `CachingDirectivesParser`, Zn extensions | 7234 |
| HTTP error hierarchy | `HTTPClientError`, `HTTPServerError` | — |

## URL handling (Zinc extensions)

```smalltalk
"Encoded query parameters"
'http://api.example.com/search' asUrl
    queryAt: 'q' put: 'hello world'.
"=> http://api.example.com/search?q=hello%20world"

"URL-encode a URL inside another URL"
'https://example.com' asUrl
    queryAt: 'redirect' putUrl: 'https://other.com/path?x=1'.

"Pagination convenience"
'https://api.example.com/items' asUrl start: 0 limit: 50.
"=> https://api.example.com/items?start=0&limit=50"

"Replace host/scheme (handy for reverse-proxy rewriting)"
'http://api.example.com:1111/resource' asUrl
    asHostedAt: 'https://alternative.org' asUrl.
"=> https://alternative.org/resource"
```

## Media types — versions, quality, wildcard matching

```smalltalk
"Vendor media type with version parameter"
mt := 'application/vnd.stargate.pet+json' asMediaType version: '1.0.0'.
"=> application/vnd.stargate.pet+json;version=1.0.0"

"Read a quality factor"
'text/html;q=0.8' asMediaType quality.  "=> 0.8"

"Wildcard / subtype matching for negotiation"
mt := 'application/vnd.stargate.pet+json;version=1.0.0' asMediaType.
mt accepts: 'application/json' asMediaType.  "=> true  (+json subtype kinship)"
mt accepts: 'text/*' asMediaType.            "=> false (different main type)"
```

## Language tags & Accept-Language / Content-Language

```smalltalk
"Client-side: set Accept-Language with quality preferences"
request := ZnRequest get: 'https://api.example.com' asUrl.
request setAcceptLanguage: 'fr-CH, fr;q=0.9, en;q=0.8, de;q=0.7, *;q=0.5'.

"Server-side: advertise the language used in the response"
response addContentLanguage: 'en-US'.
response contentLanguageTags.  "=> OrderedCollection of LanguageTag"

"Link header with hreflang hints"
link := 'https://example.com/docs' asUrl asWebLink
    addLanguageHint: 'en';
    addLanguageHint: 'fr';
    relationType: 'alternate'.
"=> <https://example.com/docs>;hreflang=en;hreflang=fr;rel=alternate"
```

## ETags & conditional requests (RFC 7232)

```smalltalk
"Parse and create — both strong and weak variants"
tag     := EntityTag fromString: '"abc123"'.
weakTag := EntityTag fromString: 'W/"abc123"'.
strong  := EntityTag with: 'current-version'.

"Client: set If-Match to prevent mid-air collisions on PATCH/PUT/DELETE"
request setIfMatchTo: strong.
"=> If-Match: \"current-version\""

"Client: set If-None-Match for cache revalidation on GET"
request setIfNoneMatchTo: weakTag.
"=> If-None-Match: W/\"abc123\""

"Server: stamp the response"
response setEntityTag: (EntityTag with: 'new-version').

"Consume the ETag from a response"
response
    withEntityTagDo: [ :t | self cache: body under: t ]
    ifAbsent:       [ self cacheNothing ].
```

## Cache-Control directives (RFC 7234)

```smalltalk
response
    addCachingDirective: 'max-age=3600';
    addCachingDirective: 'must-revalidate'.
response cachingDirectives.
"=> #('max-age=3600' 'must-revalidate')"
```

The parser correctly treats commas inside quoted strings as literal characters, not directive separators.

## CURIE / SafeCURIE

```smalltalk
"Compact URI with prefix"
CURIE prefixedBy: 'users' referencing: 'octocat/tokens' asUrl.
"=> users:octocat/tokens"
```

Rejects absolute URLs on the reference side — only relative references may be compacted.

## HTTP error hierarchy

Use specific subclasses so callers can match on code class:
```smalltalk
[ self doRequest ]
    on: HTTPClientError notFound  do: [ :e | self show404 ]
    on: HTTPClientError unauthorized do: [ :e | self login ].

"Raise a 500 from your own handler"
HTTPServerError internalServerError signalMessage: 'DB unreachable'.
```

## Gotchas

- **ETag parsing is strict.** `W/` prefix is case-sensitive; values must be ASCII quoted strings. Bad input raises `InstanceCreationFailed`.
- **CURIE rejects absolute URLs** on the reference side. Resolve/strip them first.
- **Media type wildcard matching ignores version parameters.** `accepts:` matches main/subtype plus compatible parameters; version mismatch is NOT a miss in wildcard matching (by design — this mirrors RFC 7231 quality-factor negotiation).
- **`hreflang` hints are advisory.** Adding them to a `WebLink` does not set `Content-Language` on the actual response.
- **Zinc URL extensions mutate `self`.** `url queryAt: k put: v` modifies and returns `self` — chain freely, but don't expect immutability.
- **Cache-Control parser understands quoted commas.** String inside `"…"` is treated literally.
- **Pick the right error class.** Use `HTTPClientError notFound` (404) vs. `HTTPServerError internalServerError` (500) — generic `Error` will mask protocol semantics.

## When to use directly

You're usually *inside* Stargate (server) or Superluminal (client), and those wrap Hyperspace for you. Reach for Hyperspace directly when:
- building a Zinc-based HTTP component that isn't a full REST API;
- you need to validate/parse a non-trivial HTTP header (ETag, Link, Cache-Control) by yourself;
- you're building a negotiation layer across multiple media type versions.
