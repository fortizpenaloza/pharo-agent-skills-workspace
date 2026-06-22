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

The workhorse body must use the **RDBMS-portable criteria-builder form** — it runs against both the
in-memory tests and Postgres (see `abbaco-api-persistence` §7). Not `and:` (→ `mustBeBoolean`), not a
derived-method comparison like `each parent identifier = …` (→ "no mapping"), not a 2-arg comparator
sort:

```smalltalk
PortfolioCompositionsSystem >> withCompositionOf: aPortfolio effectiveAt: aDate do: aFoundBlock ifNone: aNoneBlock
    ^ compositions
        withOneMatching: [ :each :criteria |
            criteria
                satisfying: ( criteria does: each portfolio equal: aPortfolio )
                and: [ each effectiveFrom <= aDate ] ]
        sortedBy: [ :each | each effectiveFrom ] descending
        do: aFoundBlock
        else: aNoneBlock
```

When the stored historical object is an `Identified<Thing>` wrapper embedding a value object (the usual
case), the parent reference and `effectiveFrom` live on the embedded object, so navigate through the
embed attribute (`each composition portfolio`, `each composition effectiveFrom`) — and make sure the
wrapper exposes that accessor so the same path resolves in-memory (see §2 and `abbaco-api-persistence` §4).

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

{ #category : 'accessing' }
IdentifiedPortfolio >> portfolio [
    "Same value object as asPortfolio, exposed under the slot/embed-attribute name so the
     `each portfolio <field>` query-navigation path resolves identically in-memory and against RDBMS."

    ^ portfolio
]
```

### Why the wrapper carries `sequentialNumber`

Sagan's `SequentialNumberMappingDefinition` and `SequentialNumberFieldDefinition` together produce an auto-increment integer column (`SEQUENTIAL_NUMBER`) and read its value back into a slot called `sequentialNumber`. The slot is **never written by application code** — Sagan assigns it when `store:` runs and uses it as the primary key for joins, `update:`, and `purge:`. Application code never compares or displays it. The `uuid` is the only identity exposed to clients.

### `asPortfolio` and the synchronize boundary

`synchronizeWith:` on the wrapper unwraps the right-hand side via `asPortfolio` and delegates to the value object. This keeps the value object oblivious to wrapping. Sagan calls `original synchronizeWith: anUpdated` from `update:executing:` (see `RDBMSRepository.class.st` line 192-198), passing whatever shape the application handed to `update:`. Always pass a wrapper — see the management system's `update<Thing>:with:` below.

**Expose the value object under the embed-attribute name too.** Persistence maps the wrapper by *embedding* the value object in the `portfolio` slot, and RDBMS queries reach the value object's fields by navigating that attribute — `each portfolio owner` works, `each owner` raises "no mapping". So the wrapper also provides a plain `portfolio` accessor (shown above) alongside `asPortfolio`; the in-memory criteria evaluate the identical `each portfolio …` path on the real wrapper. See `abbaco-api-persistence` §4.

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
            "Compound uniqueness (name per owner). The `accordingTo:` criteria-builder form
             works in-memory AND against RDBMS; the `forSingleAspectMatching: [ :p | a -> b ]`
             Association form does NOT — it fails against Postgres. See abbaco-api-persistence §8."
            ( CriteriaBasedConflictCheckingStrategy
                accordingTo: [ :each :criteria :aPortfolio |
                    criteria
                        satisfying: ( each owner = aPortfolio owner )
                        and: [ each name = aPortfolio name ] ]
                explainingConflictWith: [ :aPortfolio |
                    'There is already a portfolio named "' , aPortfolio name , '" owned by ' , aPortfolio owner ] ).

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
- **Conflict checking**: a *single* unique attribute uses `forSingleAspectMatching: #name` (unary selector). **Compound uniqueness must use `accordingTo:explainingConflictWith:`** with the criteria builder — the `forSingleAspectMatching: [ :p | a -> b ]` Association form works in-memory but fails against RDBMS (`GlorpDatabaseReadError: Invalid data type`); see `abbaco-api-persistence` §8. Use `DoNotCheckForConflictsStrategy new` when there is no business uniqueness constraint (e.g. trials are unique by user, but historical paid subscriptions allow many per user with different statuses).
- **Lookup**: `withOneWhere: #identifier is: anIdentifier asString do: … else: …`. Always provide an `else:` that signals `ObjectNotFound` with a human-readable message. The `asString` is required because UUIDs are stored as strings.
- **Updates** go through `update:executing:` with an explicit `synchronizeWith:` block. This gives Sagan a transactional context and ensures the conflict-checking strategy reruns. Do not write through `findAllMatching:` results directly.
- **Repository query blocks must be RDBMS-portable.** Write every `findAllMatching:` / `withOneMatching:…` filter in the 2-arg criteria-builder form (`[ :each :criteria | criteria satisfying: … and: [ … ] ]`) and every sort as a property sort (`#field descending` or `[ :each | each path ] descending`) — never `and:`/`or:`, derived-method navigation (`each x identifier`), or 2-arg comparator sorts, which pass in-memory and fail against Postgres. See `abbaco-api-persistence` §7 (Query criteria).
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

One registration method per interface symbol in `systemInterfacesToInstall`. **Kepler discovers them reflectively, and the matching rule is exact and easy to get subtly wrong:**

> **The selector must begin with `register` and end with `SystemForInstallationIn:`.** `SystemModule >> withSystemsToInstallDo:` collects methods with `KeywordMessageSendingCollector sendingAllMessagesBeginningWith: 'register' andEndingWith: 'SystemForInstallationIn:'`. The text *between* those anchors is free — it's there for readability — but the `SystemForInstallationIn:` ending is mandatory. The method returns the **unstarted** system instance.

Name the method after the **system class** (which ends in `System`), never after the interface symbol:

- System `PortfolioManagementSystem` → `registerPortfolioManagementSystemForInstallationIn:` ✅ — ends in `SystemForInstallationIn:`. Here the interface symbol *is* the system name, so deriving from either works **by luck**.
- When the Kepler interface symbol is `#<Thing>SystemInterface` (ends in `Interface`, not `System`), still name the selector after the system class: `register<Thing>SystemForInstallationIn:`. Deriving it from the interface symbol gives `register<Thing>SystemInterfaceForInstallationIn:`, which ends in `InterfaceForInstallationIn:` — the collector silently skips it, the module registers **nothing**, and boot later fails with `SystemControlError: System implementing "<name>" not found`.

> **This failure is invisible to the unit/user-story/controller test suites.** `SystemBasedUserStoryTest` and `SingleResourceRESTfulControllerTest` register subsystems **directly** (`registerSubsystem:` / `register:` on a `CompositeSystem`), never through module reflection — so a mis-spelled registration selector ships green. The reflective path runs only when something installs through a `SystemInstallation` / application boot. Keep at least one test that exercises that real install path (see `abbaco-api-persistence` §10.2).

If the management system needs construction parameters (e.g. `SubscriptionsSystem startingTrialValidFor: aTimePeriod`), pass them inside the registration block:

```smalltalk
register: [ SubscriptionsSystem startingTrialValidFor: ( TimeUnits day with: 7 ) ] in: systems
```

## 5. Testing the domain model (`<Thing>-Model-Tests`)

Two layers live with the model: pure unit tests over the value objects, and user-story tests over the system with an in-memory repository. (The controller, HTTP, and PostgreSQL-integration layers live in `abbaco-api-rest` and `abbaco-api-persistence`; the out-of-image Newman/CI layer in `abbaco-api-integration-tests`. The shared SUnit conventions — `assert:equals:`, domain-specific fixture names, `assertCollection:hasSameElements:` — are in `smalltalk-conventions`.)

### 5.1 Domain unit tests (`<Thing>Test` extends `TestCase`)

Cover **domain rules only** — factories, preconditions, value-object behaviour, `synchronizeWith:`, value `=`/`hash`. Do **not** write type-rejection tests (the model layer asserts business rules, not types).

```smalltalk
Class { #name : 'PortfolioTest', #superclass : 'TestCase',
        #category : 'Portfolio-Model-Tests', #package : 'Portfolio-Model-Tests' }

{ #category : 'tests' }
PortfolioTest >> testCreation [

    | portfolio |
    portfolio := Portfolio named: 'Long-term holdings' describedAs: 'Buy and hold equities'
        withImage: 'https://example.test/portfolio.png' ownedBy: 'user-123'.
    self
        assert: portfolio name equals: 'Long-term holdings';
        assert: portfolio description equals: 'Buy and hold equities';
        assert: portfolio owner equals: 'user-123'
]

{ #category : 'tests' }
PortfolioTest >> testCreationWithEmptyNameNotAllowed [

    self
        should: [ Portfolio named: '' describedAs: 'desc' withImage: 'https://x.test' ownedBy: 'u' ]
        raise: InstanceCreationFailed
        withMessageText: 'The portfolio name must be a non-empty string of at most 40 characters'
]

{ #category : 'tests' }
PortfolioTest >> testSynchronize [

    | original updated |
    original := Portfolio named: 'A' describedAs: 'a' withImage: 'http://x' ownedBy: 'u'.
    updated := Portfolio named: 'B' describedAs: 'b' withImage: 'http://y' ownedBy: 'u'.
    original synchronizeWith: updated.
    self
        assert: original name equals: 'B';
        assert: original description equals: 'b';
        assert: original owner equals: 'u'   "owner is identity, never changes"
]
```

Conventions: one test per behaviour, category `tests`; assert **both** happy and failure path for every factory with `should:raise:withMessageText:` (locks the exact English message — there is no localization layer); **no mocks/stubs** (construct domain objects directly); **cover `synchronizeWith:`** for every value object (it's how `update:executing:` mutates state — a missing field is silent data loss); and for any value object that gets persisted, test value `=`/`hash` (a DB read-back is a *new* instance — see `abbaco-api-persistence`).

### 5.2 Domain user-story tests (`<Thing>SystemUserStoryTest` extends `SystemBasedUserStoryTest`)

`SystemBasedUserStoryTest` (Kepler-SUnit) installs the requested subsystems, calls `startUp`, and exposes `systemUnderTest` (the first interface in `systemInterfacesToInstall`). Wire the repository provider in `setUpRequirements` — `InMemoryRepositoryProvider` under `#mainDB`. **Register subsystems directly** (or `requireInstallationOf:`); this layer never boots through a `SystemInstallation`.

```smalltalk
Class { #name : 'PortfolioSystemUserStoryTest', #superclass : 'SystemBasedUserStoryTest',
        #category : 'Portfolio-Model-Tests', #package : 'Portfolio-Model-Tests' }

{ #category : 'private - running' }
PortfolioSystemUserStoryTest >> setUpRequirements [

    | repositorySystem |
    repositorySystem := RepositoryProviderSystem new.
    repositorySystem register: InMemoryRepositoryProvider new as: #mainDB.
    self registerSubsystem: repositorySystem; registerSubsystem: PortfolioSystem new
]

{ #category : 'tests - start managing' }
PortfolioSystemUserStoryTest >> testStartManagingPortfolio [

    | identified |
    identified := self systemUnderTest startManagingPortfolio:
        ( Portfolio named: 'Long-term' describedAs: 'Buy and hold' withImage: 'http://x' ownedBy: 'user-1' ).
    self
        assert: self systemUnderTest portfolios size equals: 1;
        assert: self systemUnderTest portfolios anyOne identifier equals: identified identifier
]

{ #category : 'tests - querying' }
PortfolioSystemUserStoryTest >> testPortfolioIdentifiedByRaisesWhenNotFound [

    | uuid |
    uuid := UUID new.
    self
        should: [ self systemUnderTest portfolioIdentifiedBy: uuid ]
        raise: ObjectNotFound
        withMessageText: ( 'There''s no portfolio identified by <1s>' expandMacrosWith: uuid asString )
]
```

Conventions: one scenario per system selector, categories `tests - <verb>` (`tests - start managing`, `tests - querying`, `tests - updates`, `tests - lifecycle`); assert the not-found / conflict / invalid-state paths with `should:raise:withMessageText:`; no localization helpers. For a time-versioned aggregate (§1), test the retroactive-fix-up and no-op-short-circuit cases here.

> **One thing these tests cannot catch.** Both layers wire the `CompositeSystem`/subsystems **by hand**, so they never exercise Kepler's module-registration reflection (§4) or the production install path. A mis-spelled `register…SystemForInstallationIn:` selector ships green here and only fails at real boot. Keep one PostgreSQL integration test that boots through the real `SystemInstallation install:` (see `abbaco-api-persistence`).

## 6. Common mistakes

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
