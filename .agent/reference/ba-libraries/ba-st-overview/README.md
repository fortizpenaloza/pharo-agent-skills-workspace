---
name: ba-st-overview
description: Use when writing Smalltalk code (Pharo or GS64) and considering which of the ba-st open-source libraries to pull in - lists Aconcagua, Bell, Buoy, Chalten, Hyperspace, Kepler, Launchpad, Stargate, Stargate-Consul, Superluminal with their responsibilities and when to reach for them.
---

# ba-st Framework Suite Overview

The [ba-st](https://github.com/ba-st) GitHub organization publishes a set of composable Smalltalk libraries targeting Pharo (10–13) and GemStone/S 64 (3.7.x). They are deliberately small, deeply tested, and stack cleanly. This skill is the routing table. When you are picking tools for a Smalltalk project, use it to find the right per-framework skill.

## Local repository location

All 10 projects are checked out under `/home/mtabacman/Development/Repos/ba-st-skills/`. Each repository follows the same layout:
- `source/` — the Smalltalk packages (Tonel format)
- `rowan/` — Rowan spec for GS64 loading
- `docs/` — `how-to/`, `reference/`, `explanation/`, `tutorial/` (a la [Diátaxis](https://diataxis.fr/))
- `BaselineOf<Name>` in `source/` — Metacello baseline with load groups

## Project map

| Project | Purpose | Use when you need… | Skill |
|--------|---------|-----|-----|
| Aconcagua | Measures as first-class objects (amount + unit) with arithmetic | currency, physical quantities, strongly-typed numbers | `ba-st-aconcagua` |
| Bell | Beacon-based structured logging (+ planned metrics) | emitting log records to stdout/stderr, plain or JSON | `ba-st-bell` |
| Buoy | Pharo/GS64 extensions: assertions, optionals, collections, SUnit extras | runtime preconditions, Optional/Binding, richer SUnit | `ba-st-buoy` |
| Chalten | Immutable value-object time model (Year/Month/Day/TimeOfDay/DateTime, multi-calendar) | any non-trivial date math, settlements, timezones | `ba-st-chalten` |
| Hyperspace | HTTP building blocks on top of Zinc (URLs, MediaTypes, ETags, Links, LanguageTags) | constructing URLs, ETags, Accept/Content-Language, RFC helpers | `ba-st-hyperspace` |
| Kepler | Subsystem composition (start/stop lifecycle, interface registry) | wiring loosely-coupled subsystems in an app | `ba-st-kepler` |
| Launchpad | CLI harness: applications, config parameters, providers (CLI/env/file), stack traces, Docker images | building a command-line app hosted inside an image | `ba-st-launchpad` |
| Stargate | RESTful API framework on Teapot/Zinc (HATEOAS, content negotiation, versioning, ETags, pagination, operations plugins) | standing up a HTTP JSON API | `ba-st-stargate` |
| Stargate-Consul | Stargate operational plugin integrating HashiCorp Consul service discovery | auto register/deregister Stargate services into Consul | `ba-st-stargate-consul` |
| Superluminal | HTTP client toolkit (HttpRequest fluent builder, RESTfulAPIClient with caching/ETags) | calling external HTTP APIs from Smalltalk | `ba-st-superluminal` |

## Typical stacks

### Full production JSON service
`Buoy` → assertions/optionals. `Chalten`+`Aconcagua` → domain values. `Kepler` → subsystem composition. `Bell` → logs. `Stargate` → HTTP API. `Launchpad` → CLI entry + Docker image. `Stargate-Consul` → service registration. `Superluminal` → calling downstream APIs.

### Library/domain model only
`Buoy` + `Aconcagua` + `Chalten` usually suffice. Add `Bell` if you want logging.

## Metacello loading pattern

Every project ships a `BaselineOf<Name>` with at least these groups: `Core`, `Deployment`, `Tests`, `Development`. The uniform load idiom is:

```smalltalk
Metacello new
    baseline: '<Name>';
    repository: 'github://ba-st/<Name>:release-candidate';
    load: 'Development'.
```

Replace `release-candidate` with a tag (`v9.0.0`, etc.) for pinned installs. For declaring as a dependency in your own baseline, copy the `Deployment` group and import the baseline name.

## Authoring conventions across all ba-st projects

- **Tonel** source format. Classes live under `source/<Project>-<Category>/`.
- **Tests** always in `source/<Project>-<Category>-Tests/`. They are excellent reference material for *how to use* the library — read them before guessing at the API.
- **Immutable value objects** are the default style (`=`/`hash` implemented, no setters on domain objects).
- **`AssertionChecker enforce:because:`** (from Buoy) is the standard precondition mechanism; avoid raw `self error:` in ba-st-style code.
- **HATEOAS / hypermedia / ETags** are first-class across Stargate/Superluminal/Hyperspace — don't reinvent them.
- **License:** MIT code, CC-BY-SA 4.0 docs.

## When NOT to use these libraries

- Don't introduce Aconcagua just to multiply two numbers; use it when the *units* actually matter.
- Don't introduce Chalten for "what's today's date" one-liners; use it when you need cross-calendar, timezone-aware, or arithmetic-rich time logic.
- Don't introduce Stargate-Consul unless you actually run Consul.
- Kepler is overkill for single-service scripts; reach for it when you have 3+ subsystems with lifecycles.

## How to dig deeper

When you need details on any single framework, load its dedicated skill (`ba-st-<name>`). Each contains the Metacello snippet, core classes, 5–10 working code examples, and a gotchas section distilled from the project's `docs/` and test suite.
