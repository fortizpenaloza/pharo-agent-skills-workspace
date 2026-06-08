---
name: ba-st-stargate-consul
description: Use when you have a Stargate API that must auto-register itself with HashiCorp Consul on startup (and deregister on shutdown) - service metadata, HTTP/Docker health checks, and Consul agent configuration. Covers ConsulServiceDiscoveryPlugin and ConsulAwareStargateApplication.
---

# Stargate-Consul — Consul Service Discovery for Stargate APIs

This is a Stargate *operational plugin* that registers the running service with a Consul agent at startup and deregisters at shutdown. It's opinionated and Stargate-specific; for direct Consul client usage (KV store, discovery queries), talk to Consul's HTTP API separately.

Repo: `/home/mtabacman/Development/Repos/ba-st-skills/Stargate-Consul/`. Integration-test harness in `api-tests/` and `compose-test-*.sh` scripts are the canonical deployment reference.

## Installation

```smalltalk
Metacello new
    baseline: 'StargateConsul';
    repository: 'github://ba-st/Stargate-Consul:release-candidate';
    load: 'Development'.
```

Pulls Stargate, Launchpad, Superluminal as dependencies. Requires Pharo 8+ (Pharo 7 dropped per MigrationGuide).

## Core classes

- **`ConsulServiceDiscoveryPlugin`** — the operational plugin. Registers/deregisters services on `startOn:` / `stop`. Retries with exponential backoff (2 attempts by default). Endpoint: `'consul-service-discovery'`, disabled by default.
- **`ConsulServiceDefinitionBuilder`** — fluent builder for the JSON service definition (serialized as `NeoJSONObject`). Validates port > 0, non-empty name/id.
- **`ConsulAwareStargateApplication`** — abstract `StargateApplication` subclass. Override `serviceDefinitions`. Auto-configures the Consul agent location, default health checks, and Docker hostname detection.
- **`ConsulAgentHTTPBasedCheck`** — periodic HTTP health-check definition.
- **`ConsulAgentDockerBasedCheck`** — docker-exec health-check definition.

## Wiring a Stargate API into Consul — minimal example

```smalltalk
StargateApplication subclass: #MyConsulAPI ...
    "Actually subclass ConsulAwareStargateApplication:"
ConsulAwareStargateApplication subclass: #MyConsulAPI ...

MyConsulAPI class >> stargateConfigurationParameters
    ^ super stargateConfigurationParameters , {
        MandatoryConfigurationParameter
            named: 'Consul Agent Location'
            describedBy: 'HTTP URL of the consul agent'.
        OptionalConfigurationParameter
            named: 'Scheme'
            describedBy: 'Health-check scheme'
            defaultingTo: 'http' }

MyConsulAPI >> serviceDefinitions
    ^ { self
            buildServiceDefinitionNamed: 'my-api'
            configuredBy: [ :builder |
                builder
                    addTag: 'v1';
                    addTag: 'production';
                    metadataAt: 'version' put: '1.0.0' ] }

MyConsulAPI >> controllersToInstall
    ^ { PetsRESTfulController new. OrdersRESTfulController new }
```

On start, the plugin `PUT`s `/v1/agent/service/register`; on stop, it calls `/v1/agent/service/deregister/<id>`. If the agent is unreachable, deregistration swallows the error (graceful shutdown).

## Configuration parameters

Read by `ConsulAwareStargateApplication`:

| Parameter | Kind | Meaning |
|---|---|---|
| `Consul Agent Location` | mandatory URL | empty/absent disables the plugin |
| `Scheme` | optional, default `http` | transport for HTTP health checks |

Sample env for Docker:
```bash
STARGATE__CONSUL_AGENT_LOCATION=http://consul-agent:8500
STARGATE__SCHEME=http
```

## Health checks

**HTTP-based (periodic probe):**
```smalltalk
ConsulAgentHTTPBasedCheck
    named: 'health-check'
    executing: #POST
    against: 'http://api:8080/operations/health-check' asUrl
    withHeaders: {
        (#accept        -> 'application/json').
        (#authorization -> 'Bearer TOKEN') }
    every: 10 seconds
    timeoutAfter: 1 minute
    deregisteringAfterStatusCriticalFor: 5 minutes.
```
Writes Go duration strings (`"10s"`, `"1m"`, `"5m"`) as `Interval`, `Timeout`, `deregister_critical_service_after` in the Consul JSON. 2xx = passing; any other status = critical.

**Docker exec-based:**
```smalltalk
ConsulAgentDockerBasedCheck
    named: 'docker-check'
    executing: '/bin/sh'
    withArguments: { '-c'. 'curl -f localhost:8080/health' }
    inContainer: 'container-id'
    every: 10 seconds.
```
Requires the Consul agent to reach the Docker socket; stdout is limited to 4 KB.

## Direct plugin usage (bypassing `ConsulAwareStargateApplication`)

```smalltalk
plugin := ConsulServiceDiscoveryPlugin
    reportingLifecycleOf: definition
    toAgentOn: 'http://localhost:8500' asUrl.

plugin registerToConsul:   definition.
plugin deregisterFromConsul: definition.
```

Or through the operations configuration dictionary:
```smalltalk
Dictionary new
    at: #operations put: (Dictionary new
        at: 'consul-service-discovery' put: {
            #enabled             -> true.
            #consulAgentLocation -> 'http://localhost:8500' asUrl.
            #definitions         -> { definition }.
            #retry               -> [ :retry |
                retry backoffExponentiallyWithTimeSlot: 100 milliSeconds ].
        } asDictionary;
        yourself);
    yourself.
```

## Metadata

```smalltalk
builder
    metadataAt: 'version'     put: '1.0.0';
    metadataAt: 'team'        put: 'api-team';
    metadataAt: 'environment' put: 'production'.
```
`ConsulAwareStargateApplication` auto-fills some fields from `BasicApplicationInformationProvider`.

## Running locally

Use the shipped compose files (`api-tests/pharo/docker-compose.yml`, `compose-test-pharo.sh`):
```yaml
services:
  api:
    environment:
      STARGATE__CONSUL_AGENT_LOCATION: http://consul-agent:8500
  consul-agent:
    image: hashicorp/consul:1.15
    ports: [ "8500:8500" ]
```
Smoke test:
```bash
curl http://localhost:8080/echo/hello
curl http://localhost:8500/v1/agent/services
curl -s http://localhost:8500/v1/health/checks/my-api | jq '.[0].Status'
```

## Gotchas

- **Plugin is disabled by default** — either configure `Consul Agent Location` (via `ConsulAwareStargateApplication`) or `#enabled -> true` in the operations dict. No registration without explicit enable.
- **Deregistration is fire-and-forget.** A dead Consul agent won't block shutdown; you may see orphaned services in Consul if the agent is down during a hard kill.
- **HTTP health check interprets anything non-2xx as critical.** Make sure `operations/health-check` returns 200 even when reporting WARN; see Stargate's health-check plugin response shape.
- **Retry is hardcoded to 2 tries** (with exponential backoff) unless you provide `#retry` in the options dict.
- **Health-check JWT.** The integration tests sign a token with HS256 and pass it via the `authorization` header; mirror the scheme in production — don't leave the health-check endpoint unauthenticated.
- **Docker hostname auto-detection.** `ConsulAwareStargateApplication` uses `HOSTNAME` env var for the registered address. Ensure it's set in your container (Docker usually does; some orchestrators don't).
- **Service ID defaults to Name** if you don't set it explicitly — beware collisions when running multiple replicas against one agent.
- **Consul port must be reachable from the API container.** In compose, use the service name (`consul-agent:8500`), not `localhost`.
- **`FakeConsulAgentAPI`** (test fixture) fails the first request on purpose to exercise the retry logic — don't be surprised by that in tests.

## Migration notes (v2 → v3)

- Pharo 7 support dropped.
- Dependency versions bumped: Stargate v7 → v11, Launchpad v4 → v7, Superluminal v3 → v7.

## When to reach for this skill

Only when you actually run Consul. For pure Kubernetes (Service + EndpointSlices), Consul is overkill; let the orchestrator do discovery. Use Stargate-Consul when you have mixed infra, Nomad, or a Consul-centric service mesh (Connect).
