# Pharo Agent Skills Workspace

This is the team's durable home for the Smalltalk conventions, process guidance, and Abbaco API house style we want every Claude session — and every teammate — to start from. The skills, CLAUDE.md and library references are shared; per-session memory is **not**.

## Purpose of this workspace

- `.agent/skills/` — the playbook. Mandatory conventions, the Smalltalk creation process, the TDD discipline, and the five-part Abbaco API family (`abbaco-api-house-style` + `abbaco-api-domain-model` + `abbaco-api-persistence` + `abbaco-api-rest` + `abbaco-api-integration-tests`). Each in-image test layer lives with the code it tests (in the model/persistence/REST skills); `abbaco-api-integration-tests` owns the out-of-image layer (baseline load, GitHub Actions CI, Newman, docker-compose).
- `.agent/reference/ba-libraries/` — library API surface for the ba-st open-source dependencies the Abbaco APIs consume. Not skills; just documentation a session reads on demand.
- Project-level artefacts (specifications, implementation plans, the Pharo image) live at the workspace root or in dedicated subdirectories per project.

This workspace is portable across Pharo projects. **Keep it project-agnostic.** When a session captures something worth keeping for the whole team, promote it into a SKILL.md or into this file — never let the project-specific examples leak into the shared playbook.

## Working with Pharo via MCP

The pharo-smalltalk MCP server is the editor of record for Smalltalk source. Use its tools — `eval`, `get_method_source`, `search_implementors`, `search_references`, `run_class_test`, `run_package_test` — to read, change, and verify code in the running image.

- **Never use shell-level edits** (`sed`, manual file rewrites) to mutate Pharo source. The image is the source of truth; the filesystem `.st` files are a side effect of `export_package`.
- **Prefer image queries over filesystem searches** when looking at framework code (e.g., `search_implementors` over `grep`).
- The `create-smalltalk-code` skill documents the Fluid class-definition syntax and the `compile:withInternalLineEndings classified:` pattern for installing methods. Read it before touching the image.

## Creating Smalltalk code — entry points

Every code-creating session follows the same opening sequence:

1. **Load `smalltalk-conventions` first.** It owns naming, immutability, encapsulation, method size, the `withAll:` rule, the no-`isNil`/`notNil` rule, and the SUnit assertion conventions. Everything else builds on top.
2. **Load `test-driven-development`.** Strict Red-Green-Refactor; write the failing test first, watch it fail, then write the minimal code.
3. **Load `create-smalltalk-code`** when ready to install classes/methods via MCP — Fluid syntax + the `compile:` pattern.
4. **Then load the task-specific skill** (the Abbaco API family below, or whatever the project needs).

## Skill routing

### Foundations (load for any Smalltalk work)

| Skill | When to load |
|---|---|
| `smalltalk-conventions` | Always, before writing any code. |
| `create-smalltalk-code` | When installing classes/methods via the Pharo MCP. |
| `test-driven-development` | Always — TDD discipline, with concrete examples. |

### Abbaco API family (load when building an Abbaco REST API)

| Skill | When to load |
|---|---|
| `abbaco-api-house-style` | First — the seven cross-cutting invariants, the deliberate anti-patterns, and the application/installation/baseline wiring on the Mercap Persistent-API-Skeleton. |
| `abbaco-api-domain-model` | Designing the value object, `Identified<Thing>` wrapper, system, module — plus the domain unit tests and user-story tests. Includes the time-versioned-history pattern and the Kepler module-registration selector rule. |
| `abbaco-api-persistence` | Designing the Sagan-RDBMS mapping configuration, repository setup, schema lifecycle, SQL migrations — plus the PostgreSQL integration tests (and the `update:executing:` RDBMS copy gotcha). |
| `abbaco-api-rest` | Designing the REST layer — Stargate controller, routes, NeoJSON encoding, ETags, JWT, hypermedia — plus the full-stack controller tests and real-HTTP API user-story tests. |
| `abbaco-api-integration-tests` | The out-of-image layer: the `BaselineOf<Name>API` load, GitHub Actions CI, Newman/Postman `api-tests/`, the docker-compose deploy chain, and the Dockerfile. |

Skills are read on demand — load only what the current task needs. Cross-references from one skill to another are deliberate; follow them when prompted.

## Library references

`.agent/reference/ba-libraries/` contains one folder per ba-st open-source library. Each has a `README.md` describing the public API surface; they are not skills and do not run on load. Consult them like Stack Overflow — when a session needs to look up a class signature or a usage pattern.

| Library | What it provides |
|---|---|
| `ba-st-overview` | Master routing table across the family below. |
| `ba-st-aconcagua` | Quantities with units (Measure, currency, distance, duration arithmetic). |
| `ba-st-bell` | Structured logging — `LogRecord`, `StandardStreamLogger`, trace/debug/info/warning/error. |
| `ba-st-buoy` | Assertions (`AssertionChecker`), `Optional`/`Binding`, collection/SUnit extensions, equality helpers. |
| `ba-st-chalten` | Immutable time model — `Year`/`Month`/`Day`/`TimeOfDay`/`DateTime`, multi-calendar, timezone-aware. |
| `ba-st-hyperspace` | RFC HTTP helpers — URLs, media types, ETags, Cache-Control, Link headers, CURIEs. |
| `ba-st-kepler` | Subsystem composition — `SubsystemImplementation`, `CompositeSystem`, interface registration, lifecycle, SUnit support. |
| `ba-st-launchpad` | CLI harness — `LaunchpadApplication`, configuration parameters, config providers, Docker integration. |
| `ba-st-stargate` | REST API framework — `HTTPBasedRESTfulAPI`, `SingleResourceRESTfulController`, request handler builder, content negotiation, HATEOAS, ETags. |
| `ba-st-stargate-consul` | Stargate + Consul service-discovery integration. |
| `ba-st-superluminal` | HTTP client toolkit — `HttpRequest` builder, `RESTfulAPIClient`, Teachable testing. |

## Memory discipline

`.claude/memory/` is **intentionally absent** from this shared workspace. Memory captures session-local context that should not travel between developers.

- Each developer's session memory lives at `~/.claude/projects/<workspace-slug>/memory/` — the harness manages it.
- If a session captures a convention worth keeping for the team, **promote it into a SKILL.md (or into this CLAUDE.md) and discard the memory file**. The skills are the durable home; memory is the scratchpad.
- A new session writing the first memory in `~/.claude/projects/<slug>/memory/` is normal. A `.claude/memory/` reappearing at the workspace root is a leak — clean it up.

## Project-specific artefacts

Any specification, implementation plan, or running Pharo image lives at the workspace root or in dedicated per-project subdirectories. Those are project-specific and travel with the project, not with the skills. **Do not treat them as conventions** — they are domain documents for whichever API is currently being built here.
