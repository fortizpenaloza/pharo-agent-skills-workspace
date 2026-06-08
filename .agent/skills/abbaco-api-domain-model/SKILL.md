---
name: abbaco-api-domain-model
description: Use when designing or code-reviewing the `<Thing>-Model` package of an Abbaco API — value objects with class-side factories, `Identified<Thing>` wrappers (UUID + sequentialNumber for Sagan-RDBMS auto-increment), Kepler `SubsystemImplementation` (the ManagementSystem), `SystemModule` (the ManagementModule), repository wiring against `RepositoryProviderSystem`, AssertionChecker preconditions, and `synchronizeWith:` for in-place updates. The actual mapping declaration lives in the persistence skill — load `abbaco-api-persistence` next.
---

# Abbaco API — Domain Model Layer

The `<Thing>-Model` package holds four kinds of classes, always in this order of dependency:

1. **Value object** (`Portfolio`, `Trial`, `BondGroup`…) — minimal-behaviour holder for the domain data.
2. **`Identified<Thing>`** — wraps the value object and adds a UUID **and** a `sequentialNumber` (Sagan-RDBMS auto-increment primary key).
3. **`<Thing>ManagementSystem`** — Kepler `SubsystemImplementation`; exposes the domain operations and owns the Sagan repositories.
4. **`<Thing>ManagementModule`** — Kepler `SystemModule`; registers the implementation into a `CompositeSystem`.

The mapping itself (table layout, attribute-to-column wiring, conversions) lives in `<Thing>RDBMSMappingConfiguration` — covered in `abbaco-api-persistence`. This skill ends where the mapping configuration begins.

## 1. Value object — full template

```smalltalk
Class {
    #name : 'Portfolio',
    #superclass : 'Object',
    #instVars : [ 'name', 'description', 'image', 'owner' ],
    #category : 'Portfolio-Model',
    #package : 'Portfolio-Model'
}

{ #category : 'private' }
Portfolio class >> assertNameIsValid: aName [

    AssertionChecker
        enforce: [ aName notEmpty and: [ aName size <= 40 ] ]
        because: 'The portfolio name must be a non-empty string of at most 40 characters'
        raising: InstanceCreationFailed
]

{ #category : 'private' }
Portfolio class >> assertDescriptionIsValid: aDescription [

    AssertionChecker
        enforce: [ aDescription size <= 250 ]
        because: 'The portfolio description cannot be longer than 250 characters'
        raising: InstanceCreationFailed
]

{ #category : 'instance creation' }
Portfolio class >> named: aName describedAs: aDescription withImage: anImage ownedBy: anOwner [

    self
        assertNameIsValid: aName;
        assertDescriptionIsValid: aDescription.
    ^ self new
        initializeNamed: aName
        describedAs: aDescription
        withImage: anImage
        ownedBy: anOwner
]

{ #category : 'initialization' }
Portfolio >> initializeNamed: aName describedAs: aDescription withImage: anImage ownedBy: anOwner [

    name := aName.
    description := aDescription.
    image := anImage.
    owner := anOwner
]

{ #category : 'accessing' }
Portfolio >> description [ ^ description ]

{ #category : 'accessing' }
Portfolio >> image [ ^ image ]

{ #category : 'accessing' }
Portfolio >> name [ ^ name ]

{ #category : 'accessing' }
Portfolio >> owner [ ^ owner ]

{ #category : 'printing' }
Portfolio >> printOn: aStream [

    aStream
        nextPutAll: 'Portfolio(';
        nextPutAll: name;
        nextPutAll: ' owned by ';
        print: owner;
        nextPut: $)
]

{ #category : 'updating' }
Portfolio >> synchronizeWith: aPortfolio [

    name := aPortfolio name.
    description := aPortfolio description.
    image := aPortfolio image
]
```

### Idioms that are non-negotiable

- **Factory method(s) on the class side**: `named:describedAs:withImage:ownedBy:`, `for:withEmail:to:identifiedIn:by:` (subscription style), etc. **Never expose `new` directly.** Validate arguments first, then `self new initialize…`.
- **One `initialize<Shape>:` per factory** — the initializer mirrors the factory's selector.
- **Preconditions via `AssertionChecker enforce:because:raising:`** — raise `InstanceCreationFailed` at construction time. The error messages are plain English (no `localized`).
- **`synchronizeWith:` is for shape-replacement updates only.** Sagan's `update:executing:` calls it via `original synchronizeWith: updated`. Update only the mutable aspects — never the identity (`owner` here, since portfolios don't change owners). `synchronizeWith:` does **no** re-validation: it expects `updated` to be a fresh, factory-validated value object whose preconditions have already fired, and just copies the mutable fields across.

  Reserve **domain-named mutators** (`expire`, `confirm`, `cancelExpiringOn:`, `closeAt:`, …) for state-machine transitions whose precondition is about the receiver's current state, not the argument's shape. Those mutators *do* validate (with `AssertionChecker`) — they're not the same concern as a generic shape replacement. The two patterns coexist on the same aggregate without conflict.
- **`printOn:`** is for logging/debugging; don't put business logic there.
- **No instance variable for `sequentialNumber` or `identifier` on the value object.** Identity belongs to the wrapper (next section). The value object is the unit of "what the user sees and edits"; the wrapper is the unit of "what the database stores."

### When the value object owns a child collection

If the entity owns a homogeneous collection (e.g. a portfolio owns a list of `SecurityPosition`s), the collection lives directly on the value object:

```smalltalk
#instVars : [ 'name', 'description', 'image', 'owner', 'positions' ]
```

The factory accepts the collection. `synchronizeWith:` decides whether to replace the collection wholesale or to merge — usually wholesale, with the controller responsible for ordering. The persistence skill shows how to map the collection with `OneToManyTypedAttributeMappingDefinition`.

### When the entity has a status / lifecycle

If the entity has a state machine (subscription `Pending`/`Confirmed`/`Cancelled`/`Expired`, or activate/deactivate), model the status as a polymorphic class hierarchy with `is<State>` testing methods:

```smalltalk
Pending class >> new [ ^ super new ]
Pending >> isPending [ ^ true ]
Pending >> isConfirmed [ ^ false ]
Confirmed class >> new ...
```

State transitions are methods on the value object (`confirm`, `cancelExpiringOn:`, `expire`) that mutate the status slot. The mapping uses `PluggableMappingConversionDefinition` to translate the status object to/from a string column — see `abbaco-api-persistence`.

### When the entity has time-versioned history

When the aggregate's *shape* changes over time and historical reads matter (an audit-log style aggregate, a portfolio's composition timeline, a policy whose terms change on specific dates), model history with **append-only versioning** — never with `effectiveFrom` / `effectiveTo` pairs.

The pattern:

- Each historical row carries a single `effectiveFrom` (a `Date`, not a `DateAndTime` — calendar-day precision is almost always enough). **No `effectiveTo` column.**
- The "version effective at a given date" is computed by the management system: `max(effectiveFrom) WHERE effectiveFrom <= aDate` for that parent. Whichever row is the most recent ≤ the cutoff wins; the one after it implicitly supersedes it.
- The `<Thing>System` exposes a date-keyed lookup via a block-based API: `with<HistoricalThing>Of: parent on: aDate do: aFoundBlock ifNone: aNoneBlock`. A bare `compositionOf: parent on: aDate` form that raises `ObjectNotFound` is built on top for callers that need the raising flavor. Never an `ifNone: [ nil ]` returning nil — see [smalltalk-conventions](../smalltalk-conventions/SKILL.md#-syntax-style--best-practices) on the no-`isNil` rule.
- **No clock on the system.** All write selectors take an explicit `effectiveFrom: aDate`. The HTTP controller (or operator script) decides what "today" means and passes it in. The system has nothing to ask `Date today` of.
- Stop-managing (delete) purges the parent row and leaves history intact. Date-keyed lookups continue to resolve because the historical rows hold the original parent identifier.

```smalltalk
"Write — controller supplies the date"
PortfolioCompositionsSystem >> updatePortfolio: aPortfolio
                                with: newPortfolio
                                positions: somePositions
                                effectiveFrom: aDate

"Read — public, raises if the parent never had a version effective at aDate"
PortfolioCompositionsSystem >> compositionOf: aPortfolio effectiveAt: aDate

"Read — private, block-based, the workhorse"
PortfolioCompositionsSystem >> withCompositionOf: aPortfolio
                                effectiveAt: aDate
                                do: aFoundBlock
                                ifNone: aNoneBlock
```

**Retroactive fix-up is a first-class API surface.** A teammate discovers yesterday's update was missed → call `updatePortfolio:...effectiveFrom: yesterday`. The system inserts a row with `effectiveFrom = yesterday`; `compositionOf: portfolio effectiveAt: yesterday` immediately resolves to it because the new row's `effectiveFrom` is now the latest ≤ yesterday. No previous row needs to be edited.

**No-op short-circuit.** If the proposed members are identical (set-equality) to whatever's effective at `aDate` already, the update skips writing a new row — duplicating an effective row would only bloat history. Use the block-based `with<HistoricalThing>do:ifNone:` to check; the `ifNone:` branch (no row at all at that date) is also a "write a new row" path.

The persistence layer maps `effectiveFrom` as `DATE NOT NULL` and indexes `(parent_sequential_number, effective_from DESC)` — that index serves the "latest ≤ aDate" query directly. See `abbaco-api-persistence`.

## 2. `Identified<Thing>` wrapper — full template

The wrapper carries **two** identity slots: `uuid` (the public, externally-visible identifier — the one URLs are built from) and `sequentialNumber` (Sagan-RDBMS's auto-increment integer PK, never exposed).

```smalltalk
Class {
    #name : 'IdentifiedPortfolio',
    #superclass : 'Object',
    #instVars : [ 'sequentialNumber', 'uuid', 'portfolio' ],
    #category : 'Portfolio-Model',
    #package : 'Portfolio-Model'
}

{ #category : 'instance creation' }
IdentifiedPortfolio class >> identifying: aPortfolio with: aUUID [

    ^ self new initializeIdentifying: aPortfolio with: aUUID
]

{ #category : 'initialization' }
IdentifiedPortfolio >> initializeIdentifying: aPortfolio with: aUUID [

    portfolio := aPortfolio.
    uuid := aUUID asString
]

{ #category : 'accessing' }
IdentifiedPortfolio >> identifier [ ^ uuid ]

{ #category : 'accessing' }
IdentifiedPortfolio >> sequentialNumber [ ^ sequentialNumber ]

{ #category : 'accessing' }
IdentifiedPortfolio >> name [ ^ portfolio name ]

{ #category : 'accessing' }
IdentifiedPortfolio >> description [ ^ portfolio description ]

{ #category : 'accessing' }
IdentifiedPortfolio >> image [ ^ portfolio image ]

{ #category : 'accessing' }
IdentifiedPortfolio >> owner [ ^ portfolio owner ]

{ #category : 'printing' }
IdentifiedPortfolio >> printOn: aStream [

    portfolio printOn: aStream
]

{ #category : 'updating' }
IdentifiedPortfolio >> synchronizeWith: anIdentifiedPortfolio [

    portfolio synchronizeWith: anIdentifiedPortfolio asPortfolio
]

{ #category : 'converting' }
IdentifiedPortfolio >> asPortfolio [

    ^ portfolio
]
```

### Why the wrapper carries `sequentialNumber`

Sagan's `SequentialNumberMappingDefinition` and `SequentialNumberFieldDefinition` together produce an auto-increment integer column (`SEQUENTIAL_NUMBER`) and read its value back into a slot called `sequentialNumber`. The slot is **never written by application code** — Sagan assigns it when `store:` runs and uses it as the primary key for joins, `update:`, and `purge:`. Application code never compares or displays it. The `uuid` is the only identity exposed to clients.

### `asPortfolio` and the synchronize boundary

`synchronizeWith:` on the wrapper unwraps the right-hand side via `asPortfolio` and delegates to the value object. This keeps the value object oblivious to wrapping. Sagan calls `original synchronizeWith: anUpdated` from `update:executing:` (see `RDBMSRepository.class.st` line 192-198), passing whatever shape the application handed to `update:`. Always pass a wrapper — see the management system's `update<Thing>:with:` below.

### Identifier comparison

Compare wrappers by `identifier`, never by Smalltalk `=`. Repositories use the conflict-checking strategy and the descriptor's primary-key mapping for identity; user code uses `withOneWhere: #identifier is: anIdentifier asString do: …`.

## 3. `<Thing>ManagementSystem` — Kepler `SubsystemImplementation`

```smalltalk
Class {
    #name : 'PortfolioManagementSystem',
    #superclass : 'SubsystemImplementation',
    #instVars : [ 'portfolios', 'idGenerator' ],
    #category : 'Portfolio-Model',
    #package : 'Portfolio-Model'
}

{ #category : 'instance creation' }
PortfolioManagementSystem class >> generatingIdentifiersWith: aUuidGenerator [

    ^ super new initializeGeneratingIdentifiersWith: aUuidGenerator
]

{ #category : 'instance creation' }
PortfolioManagementSystem class >> new [

    ^ self generatingIdentifiersWith: [ UUID new ]
]

{ #category : 'registering' }
PortfolioManagementSystem class >> registerInterfaces [

    <ignoreForCoverage>
    self
        registerInterfaceAt: #PortfolioManagementSystem
        named: 'Portfolio Management'
        declaring: #(
            #portfolios
            #portfolioIdentifiedBy:
            #identifierOf:
            #startManagingPortfolio:
            #updatePortfolio:with:
            #stopManagingPortfolio:
        )
]

{ #category : 'installing' }
PortfolioManagementSystem >> dependencies [

    ^ #( RepositoryProviderSystem )
]

{ #category : 'installing' }
PortfolioManagementSystem >> implementedInterfaces [

    ^ #( #PortfolioManagementSystem )
]

{ #category : 'initialization' }
PortfolioManagementSystem >> initializeGeneratingIdentifiersWith: aUuidGenerator [

    idGenerator := aUuidGenerator
]

{ #category : 'private - lifecycle' }
PortfolioManagementSystem >> startUpWhenStopped [

    super startUpWhenStopped.
    self initializePortfolios
]

{ #category : 'private - lifecycle' }
PortfolioManagementSystem >> initializePortfolios [

    portfolios := self >> #RepositoryProviderSystem
        createRepositoryFor: #mainDB
        storingObjectsOfType: IdentifiedPortfolio
        checkingConflictsAccordingTo:
            ( CriteriaBasedConflictCheckingStrategy
                forSingleAspectMatching: [ :portfolio | portfolio name -> portfolio owner ] ).

    PortfolioRDBMSMappingConfiguration new cull: portfolios
]

{ #category : 'accessing' }
PortfolioManagementSystem >> name [ ^ 'Portfolio Management' ]

{ #category : 'private' }
PortfolioManagementSystem >> nextIdentifier [ ^ idGenerator value ]

{ #category : 'management' }
PortfolioManagementSystem >> startManagingPortfolio: aPortfolio [

    | identified |

    identified := IdentifiedPortfolio identifying: aPortfolio with: self nextIdentifier.
    portfolios store: identified.
    ^ identified
]

{ #category : 'management' }
PortfolioManagementSystem >> stopManagingPortfolio: anIdentifiedPortfolio [

    portfolios purge: anIdentifiedPortfolio
]

{ #category : 'updating' }
PortfolioManagementSystem >> updatePortfolio: original with: updated [

    ^ portfolios update: original executing: [ :stored | stored synchronizeWith: updated ]
]

{ #category : 'querying' }
PortfolioManagementSystem >> portfolios [

    ^ portfolios findAll
]

{ #category : 'querying' }
PortfolioManagementSystem >> portfolioIdentifiedBy: anIdentifier [

    ^ portfolios
        withOneWhere: #identifier is: anIdentifier asString
        do: [ :portfolio | portfolio ]
        else: [
            ObjectNotFound signal:
                ( 'There''s no portfolio identified by <1s>' expandMacrosWith: anIdentifier asString ) ]
]

{ #category : 'querying' }
PortfolioManagementSystem >> identifierOf: anIdentifiedPortfolio [

    ^ anIdentifiedPortfolio identifier
]
```

### House rules for management systems

- **Superclass is always `SubsystemImplementation`** (Kepler).
- **Class-side `new` routes through a parametric factory** (`generatingIdentifiersWith:`). The default generator is `[ UUID new ]`; tests inject deterministic ones.
- **`registerInterfaces`** declares the Kepler interface symbol (`#<Thing>ManagementSystem`) and **enumerates every public selector** the controller will call. Missing selectors raise `DoesNotUnderstand` from inside the Kepler interface proxy at runtime. Use the `<ignoreForCoverage>` pragma. Pair this with `implementedInterfaces` returning the same symbol set.
- **`dependencies` returns `#( RepositoryProviderSystem )`** — Sagan-Kepler's interface for obtaining repository providers. Reach it via `self >> #RepositoryProviderSystem` (Kepler's lookup operator). Add other dependencies (`TimeSystemInterface`, `IdentifierGeneratorInterface`, …) as needed.
- **`startUpWhenStopped` is the Kepler lifecycle hook that creates repositories.** Do **not** create them in `initialize`; the subsystem must reinitialize cleanly across Kepler start/stop cycles.
- **Repository creation goes through `RepositoryProviderSystem`**, not directly through `RDBMSRepositoryProvider`:
  ```smalltalk
  self >> #RepositoryProviderSystem
      createRepositoryFor: #mainDB
      storingObjectsOfType: IdentifiedPortfolio
      checkingConflictsAccordingTo: <strategy>
  ```
  This selector is provided by `Sagan-Kepler/RepositoryProviderSystem`. The `#mainDB` symbol matches the name under which the application registered the provider (see `abbaco-api-persistence`).
- **Apply the mapping immediately after creating the repository**: `PortfolioRDBMSMappingConfiguration new cull: portfolios`. The mapping configuration is `cull:`-applicable — it accepts a repository and configures it. (See `RDBMSMappingConfiguration.class.st`.)
- **Conflict checking** uses `CriteriaBasedConflictCheckingStrategy forSingleAspectMatching:` with a block (or symbol) that extracts the unique aspect. For composite uniqueness, return an `Association` or array. Use `DoNotCheckForConflictsStrategy new` when there is no business uniqueness constraint (e.g. trials are unique by user, but historical paid subscriptions allow many per user with different statuses).
- **Lookup**: `withOneWhere: #identifier is: anIdentifier asString do: … else: …`. Always provide an `else:` that signals `ObjectNotFound` with a human-readable message. The `asString` is required because UUIDs are stored as strings.
- **Updates** go through `update:executing:` with an explicit `synchronizeWith:` block. This gives Sagan a transactional context and ensures the conflict-checking strategy reruns. Do not write through `findAllMatching:` results directly.
- **Method categories**: `instance creation`, `registering`, `installing`, `initialization`, `private`, `private - lifecycle`, `management`, `querying`, `updating`, `accessing`.

### Active/inactive resources

If the resource has activate/deactivate semantics, follow pepper's two-repository pattern: keep an active repository (with conflict checking) and an inactive one (with `DoNotCheckForConflictsStrategy new`). `activate:` / `deactivate:` move the wrapper between them inside `transact:`. Most abbaco resources don't need this — the subscription-api uses status-on-the-value-object instead, and the position-api has no activation at all. Pick the model the domain actually has; don't add activation just to mirror pepper.

### Status-driven entities (alternative to active/inactive)

When the lifecycle is richer than active/inactive, model status on the value object (see Section 1) and use `withOneMatching:` queries that combine `user = aUser` with `statusType = Confirmed`. The repository stays single; the status is part of the query, not the storage location.

### `registerInterfaces` in interactive Pharo development

**Kepler does not call `registerInterfaces` automatically** outside the baseline's `postLoadDoIt:`. When you implement a new management system class interactively, call it manually before running any user-story test that references the new interface:

```smalltalk
PortfolioManagementSystem registerInterfaces.
```

Without this, `systemUnderTest >> #PortfolioManagementSystem` raises a lookup error even though the implementation class exists.

## 4. `<Thing>ManagementModule` — Kepler `SystemModule`

```smalltalk
Class {
    #name : 'PortfolioManagementModule',
    #superclass : 'SystemModule',
    #instVars : [ 'rootSystem' ],
    #category : 'Portfolio-Model',
    #package : 'Portfolio-Model'
}

{ #category : 'instance creation' }
PortfolioManagementModule class >> toInstallOn: aCompositeSystem [

    ^ self new initializeToInstallOn: aCompositeSystem
]

{ #category : 'initialization' }
PortfolioManagementModule >> initializeToInstallOn: aCompositeSystem [

    rootSystem := aCompositeSystem
]

{ #category : 'private' }
PortfolioManagementModule >> name [ ^ 'Portfolio Management' ]

{ #category : 'private' }
PortfolioManagementModule >> rootSystem [ ^ rootSystem ]

{ #category : 'private' }
PortfolioManagementModule >> systemInterfacesToInstall [

    ^ #( PortfolioManagementSystem )
]

{ #category : 'private' }
PortfolioManagementModule >> registerPortfolioManagementSystemForInstallationIn: systems [

    ^ self register: [ PortfolioManagementSystem new ] in: systems
]
```

One `register<Interface>ForInstallationIn:` method per interface symbol in `systemInterfacesToInstall`. Kepler discovers them by selector convention, so the spelling must match exactly.

If the management system needs construction parameters (e.g. `SubscriptionsSystem startingTrialValidFor: aTimePeriod`), pass them inside the registration block:

```smalltalk
register: [ SubscriptionsSystem startingTrialValidFor: ( TimeUnits day with: 7 ) ] in: systems
```

## 5. Common mistakes

- **Putting `sequentialNumber` or `uuid` on the value object** — identity belongs to the `Identified<Thing>` wrapper. Sagan needs `sequentialNumber` for the auto-increment PK; the value object should be reusable across stored and unstored contexts.
- **Exposing `new` directly** instead of a `named:…ownedBy:` factory — preconditions never fire and invalid state slips into Sagan.
- **Storing the raw value object in the repository** instead of `Identified<Thing>` — UUID lookup breaks, `withOneWhere: #identifier is:` returns nothing.
- **Forgetting `registerInterfaces`** or omitting selectors that the controller calls — controller tests pass directly against the management system, but real Kepler-routed calls (through the interface proxy) raise `DoesNotUnderstand`.
- **Forgetting `implementedInterfaces`** — Kepler can't resolve `self >> #<Interface>` from inside the system itself.
- **Initializing repositories in `initialize`** instead of `startUpWhenStopped` — subsystems do not reinitialize cleanly when Kepler restarts.
- **Hard-coding `UUID new`** in `startManagingPortfolio:` — tests can't inject deterministic identifiers. Always go through `self nextIdentifier`.
- **Calling `RDBMSRepositoryProvider` directly inside the management system** — the system should not know whether persistence is in-memory or PostgreSQL. Always go through `self >> #RepositoryProviderSystem`.
- **Applying the mapping configuration once at module-install time instead of in `startUpWhenStopped`** — a Kepler restart re-creates the repository, and the new repository has no descriptor. Apply the mapping immediately after `createRepositoryFor:…`.
- **Using `update: original with: updated` instead of `update:executing:`** — the older selector exists in some abbaco code (subscription-api uses `update:with:` via `synchronizeWith:`), but the latest Sagan API is `update:executing:` taking a monadic block. Stick to `update:executing: [ :stored | stored synchronizeWith: updated ]`.
- **Reusing pepper's `localized` calls** — abbaco APIs are not localized today. Plain English error messages are correct; do not import Buoy's i18n extension.
- **Adding activation/deactivation reflexively** — only add it if the domain genuinely has an active/inactive distinction. For lifecycle (Pending/Confirmed/Cancelled/Expired), prefer status on the value object.
