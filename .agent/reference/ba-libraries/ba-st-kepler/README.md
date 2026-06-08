---
name: ba-st-kepler
description: Use when organizing a Smalltalk application as a set of loosely-coupled subsystems with start/stop lifecycles, declared interface dependencies, or a pub-sub event bus. Covers Kepler's SubsystemImplementation, CompositeSystem, registerInterfaces, and dependency resolution.
---

# Kepler — Subsystem Composition with Lifecycles

Kepler lets you break an application into small subsystems, each of which:
- declares the **interfaces it implements** (by symbolic key),
- declares the **interfaces it depends on**,
- has a **start/stop lifecycle** managed by a parent composite.

Dependencies are resolved on startup through a global `SystemInterfaces` registry; access is by `self >> #InterfaceKey`.

Repo: `/home/mtabacman/Development/Repos/ba-st-skills/Kepler/`. Reference tests: `source/Kepler-System-Tests/` (`SampleCustomerSystem`, `SampleProjectSystem`, `CompositeSystemTest`).

## Installation

```smalltalk
Metacello new
    baseline: 'Kepler';
    repository: 'github://ba-st/Kepler:release-candidate';
    load: 'Development'.
```

Groups: `Core`, `Deployment`, `Extended` (adds notifications/events), `Development`. Kepler ships `locales/` (English + Spanish) for error messages.

## Core types

| Class | Role |
|---|---|
| `SystemImplementation` | Abstract root — provides `startUp`/`shutDown` templates calling `startUpWhenStopped`/`shutDownWhenStarted`. |
| `SubsystemImplementation` | A system embedded in a composite; declares `dependencies` and `implementedInterfaces`. |
| `CompositeSystem` | Container coordinating startup/shutdown order and resolving `#InterfaceKey`s. |
| `SystemInterfaces` | Global namespace binding interface keys → named method-selector contracts. |

## Define a subsystem

```smalltalk
"1. Subclass SubsystemImplementation"
SubsystemImplementation subclass: #SampleCustomerSystem
    instanceVariableNames: 'customers'
    classVariableNames: ''
    package: 'Sample'

"2. Declare interfaces (class-side) — called in postLoad by BaselineOfKepler"
SampleCustomerSystem class >> registerInterfaces
    self
        registerInterfaceAt: #CustomerManagementSystem
        named: 'Customer Management'
        declaring: #( #addCustomer: ).
    self
        registerInterfaceAt: #CustomerQueryingSystem
        named: 'Customer Querying'
        declaring: #( #customers )

"3. Declare what interfaces *this instance* implements"
SampleCustomerSystem >> implementedInterfaces
    ^ #( #CustomerManagementSystem #CustomerQueryingSystem )

"4. Declare dependencies on OTHER subsystems' interfaces"
SampleCustomerSystem >> dependencies
    ^ #()  "none for this example"

"5. Implement the declared selectors"
SampleCustomerSystem >> initialize
    super initialize.
    customers := OrderedCollection new

SampleCustomerSystem >> addCustomer: aName
    customers add: aName

SampleCustomerSystem >> customers
    ^ customers copy
```

## Depend on other subsystems

```smalltalk
SampleProjectSystem >> dependencies
    ^ #( #CustomerQueryingSystem )

SampleProjectSystem >> addProjectNamed: aName for: aCustomerName
    | known |
    known := (self >> #CustomerQueryingSystem) customers.
    (known includes: aCustomerName) ifFalse: [
        AssertionChecker
            enforce: [ false ]
            because: 'Unknown customer: ' , aCustomerName ].
    projectsByCustomer
        at: aCustomerName
        ifAbsentPut: [ OrderedCollection new ];
        at: aCustomerName put: ((projectsByCustomer at: aCustomerName) add: aName; yourself)
```

The `>>` operator returns the bound implementation that was resolved during `startUp`. Calling it before `startUp`, or for an unregistered interface, raises `SystemControlError`.

## Composing and running

```smalltalk
| app |
app := CompositeSystem new.
app
    register: SampleCustomerSystem new;
    register: SampleProjectSystem new;
    startUp.

(app >> #CustomerManagementSystem) addCustomer: 'Tesla'.
(app >> #ProjectManagementSystem) addProjectNamed: 'Landing Page' for: 'Tesla'.

app shutDown.
```

Startup order:
1. `CompositeSystem startUp` → iterates subsystems calling `startUpWhenStopped`.
2. Each subsystem resolves its declared `dependencies` against siblings registered in the composite.
3. Missing dependency → startup fails loudly.

Shutdown reverses the order, then each subsystem `resetDependencies`.

## Extensions

### Notifications (`Kepler-Notifications`)
Pub-sub event bus as a subsystem:
```smalltalk
notifier := app >> #EventNotificationSystem.
notifier subscribe: self to: CustomerAdded toBeNotifiedSending: #onCustomerAdded:.
notifier notifySubscribersTo: (CustomerAdded about: 'Tesla').
```

### SUnit support (`Kepler-SUnit-Model`)
Base class `SystemBasedUserStoryTest` manages root-system lifecycle:
```smalltalk
MyUserStoryTest >> setUpRequirements
    rootSystem register: SampleCustomerSystem new.
    rootSystem register: SampleProjectSystem new.
"rootSystem is started and stopped by the framework per test."
```

### Modules (`SystemModule`, `SystemInstallation`)
For multi-package installations, modules reflectively discover subsystems via `register*SystemForInstallationIn:` hooks; `InstalledModuleRegistrationSystem` tracks reinstall idempotency.

## Gotchas

- **Missing `registerInterfaces`.** No registration → `SystemInterfaces` namespace has no binding → `>>` lookup fails. Ensure the method exists and is picked up during baseline postLoad.
- **Accessing dependencies before `startUp`.** Always via `self >> #Key`, never during `initialize`. `initialize` runs before registration with a composite.
- **Re-registering started subsystems is rejected.** Register then start; don't mix.
- **Unresolved dependency = startup fails.** A subsystem declaring `#CustomerQueryingSystem` with no sibling implementing it raises on `startUp`. Register every required subsystem first.
- **Cyclic deps.** A → B → A deadlocks resolution. Design tree-shaped graphs; break cycles with events (`Kepler-Notifications`) or by splitting the offending subsystem.
- **Orphaned subsystems.** `SubsystemImplementation` outside a `CompositeSystem` can't resolve `>>`. Always embed in a composite, even a singleton one.
- **Interface keys are symbols, names are strings.** `#CustomerManagementSystem` is the lookup key; `'Customer Management'` is the human-facing name used in diagnostics.
- **Don't cache `self >> #X` across lifecycles.** Binding is only valid between `startUp` and `shutDown`.

## When to reach for Kepler

Reach for it when your application has 3+ subsystems with initialization order and/or mutual dependencies: persistence, HTTP API, background workers, caches, event bus. For a single-file script or a single-purpose library, Kepler is overkill — plain classes and constructor injection are fine.
