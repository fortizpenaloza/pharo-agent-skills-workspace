---
name: ba-st-bell
description: Use when emitting structured logs (trace/debug/info/warning/error, plain-text or JSON to stdout/stderr) from Pharo or GS64 Smalltalk code, or wiring a log subscriber for an application. Covers Bell's LogRecord + StandardStreamLogger API.
---

# Bell — Structured Logging for Pharo / GS64

Bell is a thin, opinionated layer over [Beacon](https://github.com/pharo-project/pharo-beacon) that gives you:
- `LogRecord` — a signal carrying message + level + structured data.
- Ready-made loggers that write to stdout/stderr in plain text or JSON.

v1 is logs only — metrics (counters/gauges/histograms) are planned but not shipped yet.

Repo: `/home/mtabacman/Development/Repos/ba-st-skills/Bell/`.

## Installation

```smalltalk
Metacello new
    baseline: 'Bell';
    repository: 'github://ba-st/Bell:release-candidate';
    load: 'Development'.
```

Groups: `Deployment`, `Tests`, `Development`. As a dependency:

```smalltalk
spec
    baseline: 'Bell'
        with: [ spec repository: 'github://ba-st/Bell:v{XX}' ];
    project: 'Bell-Deployment'
        copyFrom: 'Bell' with: [ spec loads: 'Deployment' ].
```

## Core concepts

- **`LogRecord`** — Beacon signal with `messageText`, `logLevel` (TRACE/DEBUG/INFO/WARNING/ERROR), auto-populated `timestamp` and `processId`, and an `OrderedDictionary` of structured data.
- **`StandardStreamLogger`** — abstract plain-text logger. Concrete: `StandardOutputLogger`, `StandardErrorLogger`. Writes `[LEVEL] YYYY-MM-DDTHH:mm:ss msg` lines.
- **`StandardStreamStructuredLogger`** — abstract JSON logger. Concrete: `StandardOutputStructuredLogger`, `StandardErrorStructuredLogger`. Each record serialized as a single JSON object line (includes `level`, `timestamp`, `process`, `message`, structured data keys).

## Emission patterns

The emission API lives on the class side of `LogRecord`.

```smalltalk
"1. Simple level-scoped emission"
LogRecord emitTraceInfo:       'Entering handler'.
LogRecord emitDebuggingInfo:   'PORT: 5322'.
LogRecord emitInfo:            'Receiving commands over TCP/22222'.
LogRecord emitWarning:         'Port not found, defaulting to 5322'.
LogRecord emitError:           'Missing required parameter'.

"2. Attach structured data (NOT a Dictionary — a block that fills one)"
LogRecord
    emitStructuredDebuggingInfo: 'Configuration'
    with: [ :data |
        data at: #port     put: 5322.
        data at: #hostname put: 'localhost' ].

"3. Track an operation with a during: block — emits [DONE] or [FAILED] automatically"
LogRecord emitInfo: 'Obtaining configuration' during: [
    LogRecord emitWarning: 'Port not found, defaulting to 5322'.
    LogRecord emitInfo:    'Hostname: localhost' ].
```

## Attaching a logger (required — otherwise nothing appears)

Bell inherits Beacon's model: signals emitted *without* an active subscriber vanish.

```smalltalk
"Plain text to stderr, only for the duration of the block"
StandardStreamLogger onStandardError runFor: LogRecord during: [
    LogRecord emitInfo: 'Starting app'.
    self doWork ].

"JSON to stdout (typical for containerized services)"
StandardStreamStructuredLogger onStandardOutput runFor: LogRecord during: [
    LogRecord
        emitStructuredInfo: 'Listening'
        with: [ :d | d at: #port put: 8080 ] ].

"Print a record directly (useful for testing custom sinks)"
record := LogRecord withMessage: 'Test' andStructuredDataBy: [ :d | ].
StandardStreamLogger onStandardError nextPut: record.
```

## Wiring inside a Launchpad application

The canonical production pattern: during `basicStartWithin:`, select the logger based on a `LAUNCHPAD__LOG_FORMAT` parameter (`json` vs. `text`), then `runFor: LogRecord during: [ ... application main loop ... ]`.

```smalltalk
| loggerClass |
loggerClass := self configuration logFormat = 'json'
    ifTrue:  [ StandardStreamStructuredLogger ]
    ifFalse: [ StandardStreamLogger ].
loggerClass onStandardOutput runFor: LogRecord during: [
    self serve ]
```

## Gotchas

- **No output without a logger.** `LogRecord emitInfo: ...` with no active subscriber silently does nothing — this surprises people repeatedly. Attach a `StandardStreamLogger` (or equivalent Beacon `SignalLogger`) via `runFor:during:` early in app startup.
- **Structured data is a *block*, not a dict.** Pass `[ :data | data at: #k put: v ]` — the framework builds the `OrderedDictionary` and cull-invokes your block.
- **No global configuration.** There's no `log4j.properties`, no implicit env-var pickup. Configuration is explicit code. Do it once, at startup.
- **Timestamp / processId are automatic.** Don't add them to structured data yourself.
- **`during:` blocks auto-append status.** A `LogRecord emitInfo: 'X' during: [ ... ]` writes `[INFO] X ... [DONE]` on success, `[ERROR] X ... [FAILED]` on exception. This is load-bearing — don't wrap exceptions out from under it.
- **JSON loggers emit one JSON object per line.** They are NDJSON-compatible; don't try to "pretty print" them.
- **The Beacon signal class is `LogRecord`.** Subscribe `runFor: LogRecord` exactly — subclasses of your own would need their own subscriptions.

## When to reach for Bell

Any Smalltalk service that needs operator-visible logs. In particular: a containerized service running behind Launchpad wants `StandardStreamStructuredLogger onStandardOutput` so that stdout lines are valid JSON consumable by Docker/Kubernetes/Fluentd. For a library (not a service), don't attach a logger — just emit `LogRecord` signals and let the hosting application decide.
