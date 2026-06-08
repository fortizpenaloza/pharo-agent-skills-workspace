---
name: ba-st-launchpad
description: Use when building a Pharo/GS64 command-line application hosted inside an image - declaring configuration parameters (mandatory/optional/sensitive), reading them from CLI args / env vars / settings files, writing stack traces, or shipping via the ba-st Docker base images.
---

# Launchpad — CLI Harness for Pharo / GS64 Applications

Launchpad turns a Smalltalk class into a runnable CLI: it handles argument parsing, environment variables, settings files, inline help, exit codes, and stack-trace dumping. It pairs with official Docker base images (`ghcr.io/ba-st/launchpad:*` and `ghcr.io/ba-st/launchpad-gs64-*`) for production deployment.

Repo: `/home/mtabacman/Development/Repos/ba-st-skills/Launchpad/`. Tutorial under `docs/tutorial/`; Docker files under `docker/`; entrypoint scripts under `scripts/`.

## Installation

```smalltalk
Metacello new
    baseline: 'Launchpad';
    repository: 'github://ba-st/Launchpad:release-candidate';
    load: 'Development'.
```

Pin to a version (e.g. `v5.0.0`) for production.

## Core concepts

- **`LaunchpadApplication`** — abstract superclass. Subclass, declare metadata on the class side, implement `basicStartWithin:` on the instance side.
- **Configuration parameters** — three flavors declared on the class side:
  - `MandatoryConfigurationParameter` — exits with code 1 if missing.
  - `OptionalConfigurationParameter` — has a default; logs a warning on fallback unless suppressed.
  - A parameter wrapped with `asSensitive` masks its value in logs (`**********`).
- **Configuration providers** — Chain of Responsibility; resolution order: **CLI args > env vars > settings files**.
- **`LaunchpadHelpPrinter`** — auto-generates NAME / SYNOPSIS / PARAMETERS / ENVIRONMENT sections from metadata.
- **Stack-trace dumpers** — `StackTraceTextDumper` (human-readable), `StackTraceBinarySerializer` (Fuel), `NullStackTraceDumper` (discouraged).

## Defining an application

```smalltalk
LaunchpadApplication subclass: #MyGreeter
    instanceVariableNames: ''
    classVariableNames: ''
    package: 'MyApp'

"Class-side metadata"
MyGreeter class >> commandName        ^ 'greet'
MyGreeter class >> description        ^ 'Greet someone'
MyGreeter class >> version            ^ '1.0.0'
MyGreeter class >> configurationParameters
    ^ {
        MandatoryConfigurationParameter
            named: 'Name'
            describedBy: 'Person to greet'.
        OptionalConfigurationParameter
            named: 'Greeting'
            describedBy: 'Greeting message'
            defaultingTo: 'Hello' }

"Instance-side lifecycle"
MyGreeter >> stackTraceDumper
    ^ StackTraceTextDumper new

MyGreeter >> basicStartWithin: context
    context outputStreamDo: [ :out |
        out
            nextPutAll: self configuration greeting; space;
            nextPutAll: self configuration name; cr ].
    self exitSuccess
```

Access values via `self configuration <camelCasedName>` — Launchpad auto-converts `'Greeting'` → `#greeting`.

## Invoking the CLI

```bash
pharo Pharo.image launchpad start greet --name=Alice
pharo Pharo.image launchpad start greet --name=Alice --greeting=Hi
pharo Pharo.image launchpad list            # list all apps
pharo Pharo.image launchpad list -v         # verbose (version + description)
pharo Pharo.image launchpad explain greet   # help for one app
pharo Pharo.image launchpad start --help
```

Flags understood by `start`: `--debug-mode`, `--dry-run`, `--settings-file=<path>` (repeatable).

## Parameter patterns

```smalltalk
"1. Mandatory"
MandatoryConfigurationParameter
    named: 'API Key'
    describedBy: 'Authentication token'

"2. Optional with default"
OptionalConfigurationParameter
    named: 'Port'
    describedBy: 'Listening port'
    defaultingTo: '8080'
"Suppress the default-used warning:"
  ... doNotWarnWhenUsingDefault

"3. Sensitive (masked in logs)"
(MandatoryConfigurationParameter
    named: 'Password'
    describedBy: 'Admin password') asSensitive

"4. Custom converter"
MandatoryConfigurationParameter
    named: 'Count'
    describedBy: 'Number of items'
    convertingWith: #asNumber      "or #asUppercase / any cull:-capable receiver"

"5. Nested inside sections (section names become prefixes)"
MandatoryConfigurationParameter
    named: 'Port'
    describedBy: 'HTTP port'
    inside: #( 'Communications' 'HTTP' )
"=> CLI: --communications.http.port=8081"
"=> ENV: COMMUNICATIONS__HTTP__PORT=8081"
"=> JSON: { \"communications\": { \"http\": { \"port\": 8081 } } }"
```

## Configuration providers

Resolution order — first hit wins:
1. **CLI args** — `--parameter-name=value` (non-alphanumeric in the name becomes `-`).
2. **Environment variables** — `PARAMETER_NAME=value` (uppercased; non-alphanumerics become `_`).
3. **Settings files** — `--settings-file=path`, multiple allowed (order = priority within the provider). JSON or INI:
   ```json
   { "communications": { "http": { "port": 8081 } } }
   ```
   ```ini
   [Communications.HTTP]
   port = 8081
   ```

## Stack trace dumpers

```smalltalk
MyApp >> stackTraceDumper ^ StackTraceTextDumper new          "text on stderr — default"
MyApp >> stackTraceDumper ^ StackTraceBinarySerializer new    "Fuel-serialized context for later introspection in Pharo"
MyApp >> stackTraceDumper ^ NullStackTraceDumper new          "prints a warning — discouraged in production"
```

## Docker integration

Pharo base: `ghcr.io/ba-st/launchpad:v5`. Includes scripts `launchpad`, `launchpad-start`, `launchpad-explain`, `launchpad-list`, `launchpad-healthcheck`.
Loader: `ghcr.io/ba-st/pharo-loader:v12.0.1` — used in multi-stage builds.
GS64 base: `ghcr.io/ba-st/launchpad-gs64-3.7.1:v5` (TCP command server + graceful SIGTERM).

```dockerfile
FROM ghcr.io/ba-st/pharo-loader:v12.0.1 AS loader
RUN pharo metacello install github://owner/repo:branch BaselineOfProject

FROM ghcr.io/ba-st/launchpad:v5
COPY --from=loader /opt/pharo/Pharo.image  ./
COPY --from=loader /opt/pharo/Pharo*.sources ./
CMD ["launchpad-start", "greet", "--name=Alice"]
```

Environment variables read by the runtime:
- `LAUNCHPAD__COMMAND_SERVER_PORT` — TCP port of the command server (default 22222).
- `LAUNCHPAD__LOG_FORMAT` — `json` flips Bell to structured output (in conjunction with the `ba-st-bell` skill).

## Gotchas

- **All class-side abstract methods are required.** Missing `commandName`/`version`/`configurationParameters`/`description`/`stackTraceDumper` raises at startup.
- **Parameter access is auto-lowercased-camel.** `'Public URL'` → `self configuration publicUrl`. Spaces become internal humps.
- **Sensitive wrappers are one-way.** Don't unwrap in logging code; let Bell's logger see the masked form. Use the raw value only inside the app's logic.
- **Provider order matters.** CLI wins over env over files. Pick parameter names that won't clash with env variables you don't control.
- **Converters can blow up.** If `convertingWith: #asNumber` fails, the app exits with code 1 — validate upstream or handle with a more forgiving converter.
- **Exit codes.** `self exitSuccess` → 0; `self exitFailure` → 1; unresolved mandatory → 1 automatically.
- **Avoid `NullStackTraceDumper` in production** — silencing crashes makes diagnosis impossible in containers.
- **Configuration is cached on first access.** Reload with `CurrentApplicationConfiguration value reload` if you rotate settings files at runtime (rare).

## When to reach for Launchpad

Any time you want a long-lived Pharo/GS64 process with well-structured configuration and an operable CLI: HTTP services (typically alongside Stargate), background workers, one-shot jobs. For a throwaway script or a test-only entry point, plain `CommandLineHandler` or a doit is enough.
