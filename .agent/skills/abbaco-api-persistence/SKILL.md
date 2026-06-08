---
name: abbaco-api-persistence
description: Use when designing or code-reviewing the Sagan-RDBMS layer of an Abbaco API — `<Thing>RDBMSMappingConfiguration` declaring `ClassModelDefinition`, `RealTableDefinition`, and `ConcreteDescriptorDefinition`; mapping definitions (`DirectMappingDefinition`, `OneToOne…`, `OneToMany…`, `EmbeddedValueOneToOne…`, `AdHocMappingDefinition`, `PluggableMappingConversionDefinition`); `RDBMSRepositoryProvider` setup over a PostgreSQL `Login`; the `RepositoryProviderSystem` registration step (`#mainDB`); the schema lifecycle (`prepareForInitialPersistence`, `destroyRepositories`); and the convention for SQL migration scripts that live alongside but outside Sagan.
---

# Abbaco API — Sagan-RDBMS Persistence Layer

This is the part of the abbaco stack that pepper does not have. Pepper persists to GemStone, where Smalltalk objects are stored directly. Abbaco persists to PostgreSQL via Sagan-RDBMS, which means **every persisted aggregate needs an explicit mapping**: a class model (what attributes Sagan should track), a table definition (the SQL schema), and a descriptor (how the two relate).

The mapping configuration is a separate class — `<Thing>RDBMSMappingConfiguration` — applied to a repository at startup. The management system creates the repository through `RepositoryProviderSystem` (covered in `abbaco-api-domain-model`); this skill covers everything from the mapping class downward.

## 1. The mapping configuration class

Subclass `RDBMSMappingConfiguration` and implement `cull: aRepository`. The `cull:` selector is what makes the configuration a callable: when the management system writes `MyConfig new cull: portfolios`, Sagan invokes `cull:` with the repository as argument, and the configuration uses `repository configureWith:` to register everything.

The house style is to split `cull:` into three private methods — one per Sagan concept — for readability:

```smalltalk
Class {
    #name : 'PortfolioRDBMSMappingConfiguration',
    #superclass : 'RDBMSMappingConfiguration',
    #category : 'Portfolio-Model',
    #package : 'Portfolio-Model'
}

{ #category : 'applying' }
PortfolioRDBMSMappingConfiguration >> cull: aRepository [

    aRepository configureWith: [ :repository |
        self
            declareTablesIn: repository;
            declareClassModelsIn: repository;
            declareDescriptorsIn: repository ]
]

{ #category : 'private - declaring' }
PortfolioRDBMSMappingConfiguration >> declareTablesIn: aRepository [

    aRepository
        beAwareOfTableDefinedBy: ( RealTableDefinition
            named: self portfolioTableName
            fieldsDefinedBy: {
                SequentialNumberFieldDefinition new.
                ( CharacterFieldDefinition named: 'uuid' sized: 36 ).
                ( CharacterFieldDefinition named: 'name' sized: 40 ).
                ( CharacterFieldDefinition nullableNamed: 'description' sized: 250 ).
                ( CharacterFieldDefinition nullableNamed: 'image' sized: 1024 ).
                ( CharacterFieldDefinition named: 'owner' sized: 128 ) }
            indexesDefinedBy: {
                ( IndexDefinition forFieldNamed: 'uuid' ).
                ( IndexDefinition forFieldNamed: 'owner' ) } )
]

{ #category : 'private - declaring' }
PortfolioRDBMSMappingConfiguration >> declareClassModelsIn: aRepository [

    aRepository beAwareOfClassModelDefinedBy: ( ClassModelDefinition
        for: IdentifiedPortfolio
        attributesDefinedBy: {
            ( BasicAttributeDefinition named: RDBMSConstants sequentialNumberAttribute ).
            ( BasicAttributeDefinition named: #uuid ).
            ( BasicAttributeDefinition named: #name ).
            ( BasicAttributeDefinition named: #description ).
            ( BasicAttributeDefinition named: #image ).
            ( BasicAttributeDefinition named: #owner ) } )
]

{ #category : 'private - declaring' }
PortfolioRDBMSMappingConfiguration >> declareDescriptorsIn: aRepository [

    aRepository beAwareOfDescriptorDefinedBy: ( ConcreteDescriptorDefinition
        for: IdentifiedPortfolio
        onTableNamed: self portfolioTableName
        mappingsDefinedBy: {
            ( SequentialNumberMappingDefinition onTableNamed: self portfolioTableName ).
            ( DirectMappingDefinition
                fromAttributeNamed: #uuid
                toFieldNamed: 'uuid'
                onTableNamed: self portfolioTableName ).
            ( DirectMappingDefinition
                fromAttributeNamed: #name
                toFieldNamed: 'name'
                onTableNamed: self portfolioTableName ).
            ( DirectMappingDefinition
                fromAttributeNamed: #description
                toFieldNamed: 'description'
                onTableNamed: self portfolioTableName ).
            ( DirectMappingDefinition
                fromAttributeNamed: #image
                toFieldNamed: 'image'
                onTableNamed: self portfolioTableName ).
            ( DirectMappingDefinition
                fromAttributeNamed: #owner
                toFieldNamed: 'owner'
                onTableNamed: self portfolioTableName ) } )
]

{ #category : 'private - table names' }
PortfolioRDBMSMappingConfiguration >> portfolioTableName [

    ^ 'PORTFOLIO'
]
```

### Rules that always apply

- **The class being mapped is the `Identified<Thing>` wrapper, not the value object.** Sagan stores wrappers — that's where `sequentialNumber` and `uuid` live. The value object is wrapped inside.
- **Use a single source of truth for every table name and field name.** Define `portfolioTableName`, `<field>FieldName` accessors and call them everywhere. The same `'PORTFOLIO'` string must appear in `declareTablesIn:`, `declareDescriptorsIn:`, and every `onTableNamed:` argument; if it drifts, Sagan fails to find the descriptor at query time.
- **`SequentialNumberMappingDefinition` is mandatory** for any table with a `SequentialNumberFieldDefinition` — together they are how Sagan handles the auto-increment primary key. The attribute must be `RDBMSConstants sequentialNumberAttribute` (which evaluates to `#sequentialNumber`).
- **Indexes go in the table definition**, not the descriptor. Index any field used in `withOneWhere:is:` or `findAllMatching:` filter blocks.
- **Nullable vs not** is declared on the field: `CharacterFieldDefinition named: 'name' sized: 40` is NOT NULL; `CharacterFieldDefinition nullableNamed: 'description' sized: 250` is NULL-able. Match this to the value-object preconditions: required attributes are NOT NULL, optional ones are nullable.
- **String size is mandatory** for `CharacterFieldDefinition`. Pick sizes from the value-object preconditions (e.g. portfolio name is `<= 40` chars → `sized: 40`). Use `1024` for URL-shaped fields, `36` for UUIDs, `64`–`128` for opaque identifiers, `250` for free-form short text.
- **One `RDBMSMappingConfiguration` per aggregate root.** When an API has multiple aggregate roots, give each its own configuration class and apply each to the appropriate repository.

## 2. Mapping cookbook — domain shape → mapping definition

| Domain shape | Mapping definition | Notes |
|---|---|---|
| Plain attribute (string, integer, decimal, date, boolean) | `DirectMappingDefinition fromAttributeNamed: → toFieldNamed: → onTableNamed:` | Default. Add `conversionDefinedBy:` only if the in-memory and on-disk representations differ. |
| Auto-increment primary key | `SequentialNumberMappingDefinition onTableNamed:` paired with `SequentialNumberFieldDefinition new` | Required on every persisted aggregate. |
| Boolean stored as boolean | `DirectMappingDefinition` + `BooleanFieldDefinition named:` | No conversion needed — Glorp's PostgreSQL platform handles it. |
| Enum / status object | `DirectMappingDefinition` with `PluggableMappingConversionDefinition` (or `AdHocMappingDefinition` if multi-field) | See `SubscriptionRDBMSMappingConfiguration >> statusMappingDefinition` for a multi-field variant. |
| Date that needs cross-format translation (Chalten ↔ SQL date) | `DirectMappingDefinition` + `PluggableMappingConversionDefinition` | See section 3 below. |
| Optional attribute with null sentinel | `AdHocMappingDefinition forNullableAttributeNamed: … consideringNullAs: <sentinel>` | The sentinel is a singleton domain object representing "absent". |
| Embedded value object (one-to-one in same table) | `EmbeddedValueOneToOneMappingDefinition` or `…WithTranslationDefinition` | Translation form is needed when the embedded object's field names differ from the columns in the parent table. |
| Aggregate-owned reference (separate table, foreign key) | `OneToOneMappingDefinition` | Use when the referenced object has its own identity. |
| Collection of basic types | `OneToManyBasicAttributeMappingDefinition` | Collection of strings, numbers, dates. |
| Collection of typed objects | `OneToManyTypedAttributeMappingDefinition` | Collection of full domain objects. Each owned object needs its own table. |
| Many-to-many | `ManyToManyAttributeMappingDefinition` | Requires a join table. |
| Cross-table reference by sequential number alone | `ReadOnlyForeignSequentialNumberAttributeDefinition` | When the referenced object lives in another aggregate and you only want to record its key. |

The full set of mapping definitions lives in the `Sagan-RDBMS` package; each is a single class with self-explanatory selectors. When in doubt, inspect the class via the Pharo MCP (`get_class_source`) — they're small.

## 3. Type conversions: `PluggableMappingConversionDefinition`

Use this when the in-memory representation and the database representation differ but the mapping is still one column ↔ one attribute:

```smalltalk
{ #category : 'private - conversions' }
PortfolioRDBMSMappingConfiguration >> createdAtConversionDefinition [

    ^ PluggableMappingConversionDefinition
        named: 'createdAtConverter'
        convertingFromDatabaseToSmalltalkUsing: [ :timestamp | timestamp asGregorianDateAndTime ]
        fromSmalltalkToDatabaseUsing: [ :dateAndTime | dateAndTime asSmalltalkDateTime ]
```

Wire it on a `DirectMappingDefinition`:

```smalltalk
( DirectMappingDefinition
    fromAttributeNamed: #createdAt
    toFieldNamed: 'created_at'
    onTableNamed: self portfolioTableName
    conversionDefinedBy: self createdAtConversionDefinition )
```

**Rules**:

- The conversion `name:` is a free-form label used for diagnostics; pick something descriptive.
- The two blocks must be inverses: `from-database-to-smalltalk` then `from-smalltalk-to-database` should round-trip, modulo the canonical form chosen for storage.
- **Handle nil in the conversion blocks** when the column is nullable. A converter that crashes on nil produces `MessageNotUnderstood` deep inside Glorp, with no useful stack. Reference: `SubscriptionRDBMSMappingConfiguration >> expirationDateAllowingNilConversionDefinition` in the abbaco subscription code.
- For more complex multi-field translations (status + expiration date stored as two columns but mapped as one polymorphic status object), use `AdHocMappingDefinition forAttributeNamed:sending:to:toMapAssociations:`. Reference: `SubscriptionRDBMSMappingConfiguration >> statusMappingDefinition`.

## 4. Embedded value objects (single-table)

Use `EmbeddedValueOneToOneMappingDefinition` (or `…WithTranslationDefinition`) when an attribute is a small immutable value object whose state should fold into the parent's columns.

Example pattern from `CelestialBodyMappingConfiguration` (paraphrased for an abbaco context — say a `Portfolio` carries an embedded `Money`):

```smalltalk
( EmbeddedValueOneToOneMappingWithTranslationDefinition
    forAttributeNamed: #netAssetValue
    translatingFieldsUsingAll: { 
        ( TableFieldTranslationDefinition
            translatingFieldNamed: 'amount'
            onTableNamed: self portfolioTableName
            toFieldNamed: MoneyMappingConfiguration amountFieldName
            onTableNamed: MoneyMappingConfiguration moneyTableName ).
        ( TableFieldTranslationDefinition
            translatingFieldNamed: 'currency'
            onTableNamed: self portfolioTableName
            toFieldNamed: MoneyMappingConfiguration currencyFieldName
            onTableNamed: MoneyMappingConfiguration moneyTableName ) } )
```

The embedded type still needs its own `RDBMSMappingConfiguration` (here `MoneyMappingConfiguration`) declaring class-model and table — but the table is virtual (the columns live in the parent's table), and the descriptor is plain `DirectMappingDefinition`s. Apply both configurations to the parent repository:

```smalltalk
PortfolioRDBMSMappingConfiguration new cull: portfolios.
MoneyMappingConfiguration new cull: portfolios.
```

## 5. Owned collections

For an aggregate that owns a typed collection (e.g. a portfolio owns positions), declare:

- A child table (`POSITION` with a `portfolio_sequential_number` foreign-key column).
- A `ClassModelDefinition` for the child with a `BasicAttributeDefinition` for the foreign key.
- A `ConcreteDescriptorDefinition` for the child mapping that foreign key.
- On the parent's descriptor, a `OneToManyTypedAttributeMappingDefinition forAttributeNamed: #positions …` linking the parent's primary key to the child's foreign key.

The parent's `synchronizeWith:` is responsible for replacing/merging the collection; Sagan persists the change inside the surrounding `transact:`.


## 6. The `RDBMSRepositoryProvider` and the `RepositoryProviderSystem`

The provider knows the database connection. The provider system (Kepler subsystem) maps a logical name (`#mainDB`) to a provider instance, so the management system never sees connection details.

### In production (StargateApplication wiring)

The application class:
1. Reads PostgreSQL connection parameters from `MandatoryConfigurationParameter`s.
2. Builds a Glorp `Login`.
3. Constructs `RDBMSRepositoryProvider using: login` (single session) or `RDBMSRepositoryProvider usingSessionPoolWith: login configuredBy: …` (pooled — preferred for production).
4. Creates a `CompositeSystem` and registers a custom module that installs the `RepositoryProviderSystem` with the RDBMS provider under `#mainDB`, plus every `<Thing>ManagementModule`.
5. Calls `( rootSystem >> #RepositoryProviderSystem ) prepareForInitialPersistence` once at startup if `RDBMS_CREATE_EMPTY_DATABASE` is true (development / fresh deploys only).

```smalltalk
Login new
    database: PostgreSQLPlatform new;
    username: self configuration sagan pgUsername;
    password: self configuration sagan pgPassword;
    host: self configuration sagan pgHostname;
    port: self configuration sagan pgPort;
    databaseName: self configuration sagan pgDatabaseName;
    setSSL;
    yourself
```

For pooled connections:

```smalltalk
RDBMSRepositoryProvider usingSessionPoolWith: login configuredBy: [ :options |
    options at: #maxIdleSessionsCount put: 10.
    options at: #minIdleSessionsCount put: 5.
    options at: #maxActiveSessionsCount put: 12 ]
```

(The connection-retry options `maximumConnectionAttempts` and `timeSlotBetweenConnectionRetriesInMs` apply to both pooled and single-session providers.)

### Custom RDBMS provider module

Sagan-Kepler ships only `InMemoryRepositoryProviderModule`. For production, write a small per-API module:

```smalltalk
Class {
    #name : 'RDBMSRepositoryProviderModule',
    #superclass : 'SystemModule',
    #instVars : [ 'rootSystem', 'login', 'sessionPoolConfiguration' ],
    #category : 'Portfolio-API-Model',
    #package : 'Portfolio-API-Model'
}

{ #category : 'instance creation' }
RDBMSRepositoryProviderModule class >> toInstallOn: aCompositeSystem connectingWith: aLogin configuredBy: aPoolConfigurationBlock [

    ^ self new initializeToInstallOn: aCompositeSystem connectingWith: aLogin configuredBy: aPoolConfigurationBlock
]

{ #category : 'initialization' }
RDBMSRepositoryProviderModule >> initializeToInstallOn: aCompositeSystem connectingWith: aLogin configuredBy: aPoolConfigurationBlock [

    rootSystem := aCompositeSystem.
    login := aLogin.
    sessionPoolConfiguration := aPoolConfigurationBlock
]

{ #category : 'private' }
RDBMSRepositoryProviderModule >> rootSystem [ ^ rootSystem ]

{ #category : 'private' }
RDBMSRepositoryProviderModule >> name [ ^ 'RDBMS Repository Provider' ]

{ #category : 'private' }
RDBMSRepositoryProviderModule >> systemInterfacesToInstall [

    ^ #( #RepositoryProviderSystem )
]

{ #category : 'private' }
RDBMSRepositoryProviderModule >> registerRepositoryProviderSystemForInstallationIn: systems [

    ^ self
        register: [
            RepositoryProviderSystem new
                register: ( RDBMSRepositoryProvider
                    usingSessionPoolWith: login
                    configuredBy: sessionPoolConfiguration )
                as: #mainDB;
                yourself ]
        in: systems
]
```

In tests, swap this for `InMemoryRepositoryProviderModule` (which Sagan-Kepler ships ready-to-use). The management system code is identical — it always asks `RepositoryProviderSystem` for `#mainDB`.

### Schema lifecycle

| Operation | Selector | Effect |
|---|---|---|
| Create / recreate the schema | `( rootSystem >> #RepositoryProviderSystem ) prepareForInitialPersistence` | Drops and recreates every defined table. **Destructive — production data loss.** Run only on first deploy or development resets. |
| Free connections at shutdown | `( rootSystem >> #RepositoryProviderSystem ) prepareForShutDown` (called by Kepler) | Closes pooled sessions cleanly. |
| Drop everything (tests) | `( rootSystem >> #RepositoryProviderSystem ) destroyRepositories` | Drops all tables. Use only in test `tearDown`. |
| Reset cached sessions | `provider reset` | Clears connection cache without dropping tables. |

The boolean environment variable `RDBMS_CREATE_EMPTY_DATABASE` (used by abbaco docker-compose stacks) is a convention for "call `prepareForInitialPersistence` at startup if true." Wire that yourself on the application — it is not built into Stargate or Sagan.

## 7. Repository CRUD surface

Once the mapping is applied, the management system uses the repository directly. The full surface is on `RDBMSRepository.class.st`:

| Selector | Purpose |
|---|---|
| `store: aDomainObject` | Insert. Runs the conflict-checking strategy first. Returns the stored object (now with `sequentialNumber` populated). |
| `update: original executing: monadicBlock` | Refresh `original` from the database, apply the block (typically `[ :stored | stored synchronizeWith: updated ]`), commit. Wraps in `transact:`. |
| `purge: aDomainObject` | Delete. Returns the purged object. |
| `purgeAllMatching: aCriteriaOrBlock` | Bulk delete by criteria. |
| `transact: aBlock` | Explicit transaction scope. Multiple `store:` / `update:` / `purge:` calls inside one block share a unit of work. |
| `findAll` | Return everything. |
| `findAllMatching: aCriteriaOrBlock` | Filter by block (`[ :p | p name = 'X' ]`) or by `criteriaBuilder`. |
| `findAllMatching: aCriteriaOrBlock limitedTo: aSize sortedByAscending: aSelector` | Pagination + ordering. |
| `findAllMatching: aCriteriaOrBlock sortedBy: aSortFunction` | Ordering without limit. |
| `withOneMatching: aCriteriaOrBlock do: foundBlock else: noneBlock` | Single-result lookup. The `foundBlock` is value:'d with the result; `noneBlock` is value:'d with no args (the canonical place to signal `ObjectNotFound`). |
| `withOneWhere: anAttributeName is: aValue do: foundBlock else: noneBlock` | Convenience around `withOneMatching:` for single-equality lookups. |
| `countAll`, `countMatching:` | Count without loading objects. |

**Avoid `update: original with: updated`** — the older API used in the abbaco subscription-api code. The latest Sagan API is `update:executing:` with a monadic block. Always pass a `synchronizeWith:` block:

```smalltalk
portfolios update: original executing: [ :stored | stored synchronizeWith: updated ]
```

## 8. Conflict-checking strategies

Pass one to `createRepositoryStoringObjectsOfType:checkingConflictsAccordingTo:` (or the Sagan-Kepler equivalent `createRepositoryFor:storingObjectsOfType:checkingConflictsAccordingTo:`):

| Strategy | When |
|---|---|
| `DoNotCheckForConflictsStrategy new` | The aggregate has no application-level uniqueness constraint (e.g. historical records, append-only logs). Most common for status-driven entities where the status itself encodes the uniqueness. |
| `CriteriaBasedConflictCheckingStrategy forSingleAspectMatching: <selectorOrBlock>` | A single attribute is unique (e.g. `#name`). The argument is a unary symbol or a 1-arg block that extracts the aspect. |
| `CriteriaBasedConflictCheckingStrategy forSingleAspectMatching: [ :p | p owner -> p name ]` | Compound uniqueness. Return an `Association` or `Array` from the block; Sagan compares them for equality. |
| `CriteriaBasedConflictCheckingStrategy forSingleAspectMatching: <…> explainingConflictWith: <stringBlock>` | Same as above but customizes the conflict-error message. The block receives the conflicting object. The message is plain English (no localization). |

The conflict check runs inside `store:` and `update:executing:` — it raises `ConflictingObjectFound` before the SQL hits the database.

## 9. SQL migrations live outside Sagan

Sagan does not ship a migration tool. `prepareForInitialPersistence` is a "drop and recreate" operation, suitable for fresh deploys but not for evolving production schemas.

Abbaco's existing convention (carried forward) is **per-change directories under `migrations/`**, each containing a sequence of shell scripts:

```
migrations/
├── add-portfolio-image-column/
│   ├── README.md
│   ├── step01.sh
│   ├── step02.sh
│   └── TESTING-PLAN.md
└── rename-position-fields/
    └── …
```

Each script is plain `psql`:

```bash
#!/usr/bin/env bash
set -e
psql -h "$PG_HOSTNAME" -U "$PG_USERNAME" -d "$PG_DATABASE_NAME" <<'SQL'
ALTER TABLE PORTFOLIO ADD COLUMN image VARCHAR(1024);
SQL
```


**The mapping configuration must always describe the post-migration schema.** When you change a table, change the `<Thing>RDBMSMappingConfiguration` in the same PR and add a migration script. CI integration tests (see `abbaco-api-testing`) catch drift by exercising the persist-query round-trip on a freshly recreated schema.

## 10. Common mistakes

- **Mismatched table names across the three declarations** — `'PORTFOLIO'` in the table, `'PORTFOLIOS'` in the descriptor, `'portfolio'` in a mapping. Sagan reports the failure as `descriptor not found for class IdentifiedPortfolio` at first query. Always go through a single accessor (`portfolioTableName`).
- **Forgetting `SequentialNumberMappingDefinition`** — `store:` works but the wrapper's `sequentialNumber` slot stays nil and subsequent `update:` calls fail to locate the row. Always pair the field definition with the mapping definition.
- **Mapping the value object instead of the wrapper** — descriptors must point at the class Sagan stores. The wrapper carries `sequentialNumber` and `uuid`; the value object does not.
- **Conversion blocks that crash on nil** — use `value ifNotNil: [ … ]` in both directions whenever the column is `nullableNamed:`. Reference the abbaco `expirationDateAllowingNilConversionDefinition` for the canonical pattern.
- **Running `prepareForInitialPersistence` in production** — drops every table. Gate it behind `RDBMS_CREATE_EMPTY_DATABASE` or equivalent, and never enable that in production environments.
- **Calling `RDBMSRepositoryProvider` directly inside the management system** — couples persistence choice to domain code. Always go through `RepositoryProviderSystem >> #mainDB` so the same management system code runs against `InMemoryRepositoryProvider` in tests and `RDBMSRepositoryProvider` in production.
- **Applying the mapping configuration once at module-install time** — Kepler restarts can recreate repositories. Apply the mapping inside `startUpWhenStopped` immediately after `createRepositoryFor:…` (see `abbaco-api-domain-model`).
- **Indexing nothing** — small tables work, large tables grind. Index every column used in a `withOneWhere:is:` or `findAllMatching:` filter expression, plus the foreign-key column on every owned-collection child table.
- **Using `repository configureMappingsIn: aConfig`** — this selector exists in older abbaco code (subscription-api), but the canonical Sagan API is `aConfig new cull: repository`. The configuration is `cull:`-applicable; treat it as a callable.
- **Writing a single mega-configuration for an entire API** — split per aggregate root. Each `<Thing>RDBMSMappingConfiguration` describes exactly one aggregate; the application or management system applies several to the same repository as needed.
- **Forgetting to add a migration script when changing a mapping** — production schema diverges from the mapping silently until a query fails. Pair every mapping change with a `migrations/<name>/step*.sh`.
