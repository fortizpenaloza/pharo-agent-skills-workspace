---
name: smalltalk-conventions
description: Global coding standards for Smalltalk (Pharo). Mandatory rules on naming, immutability, encapsulation, method size, nil avoidance, and the SUnit-flavored testing idioms used by every project in this workspace. Load this first, before any other skill that produces code.
---

# Smalltalk Code Conventions

The mandatory conventions for every Smalltalk file in this workspace. Other skills (`create-smalltalk-code`, `test-driven-development`, the `abbaco-api-*` family) build on top of these — they may add specifics, they never override.

## 🏗️ Architecture & Object Design

### 1. Instance Creation & Validity
* **Enforce Valid Objects:** Objects must be fully initialized upon creation. Never create an "empty" object and populate it later.
* **No Public Setters:** This follows the valid object rule. Avoid creating setters; pass all required state during initialization.
* **Avoid explicit `new` in client code:** Client and domain code should almost never send `new` directly; instead, use class-side creation methods that define the "shape" of the object. Using `self new` *inside* those class-side creation methods (as shown below) is the standard pattern.

    ####  ❌ Bad Pattern (Anemic/Mutable)
    *Risk: The object exists in an invalid state (missing currency) and relies on side-effects.*
    ```smalltalk
    wallet := Money new.
    wallet amount: 100.
    ```

    ####  ✅ Good Pattern (Immutable/Valid)
    *Benefit: The object is guaranteed to be valid from the moment it is created.*

    **Class-side:**
    ```smalltalk
    Money class >> amount: anAmount currency: aCurrency
        ^ self new initializeAmount: anAmount currency: aCurrency
    ```

    **Instance-side:**
    ```smalltalk
    Money >> initializeAmount: anAmount currency: aCurrency
        amount := anAmount.
        currency := aCurrency
    ```

    **Client Usage:**
    ```smalltalk
    wallet := Money amount: 100 currency: 'USD'.
    ```

* **Creation Method Pattern:** The object is guaranteed to be valid from the moment it is created.

    **Class-side:**
    ```smalltalk
    CircularIterator class >> cyclingOver: aSequence
        ^ self new initializeCyclingOver: aSequence
    ```

    **Instance-side:**
    ```smalltalk
    CircularIterator >> initializeCyclingOver: aSequence
        sequence := aSequence.
    ```

### 2. Domain Preconditions (model layer only)

Factories enforce **business rules**, not types. The parser/decoder layer (HTTP body decoder, persistence mapping, anything at a system boundary) has already committed to the types by the time a value reaches a domain factory. The factory's job is to verify what the parser **can't know** — closed enumerations, length limits, no-duplicate invariants, state-machine guards.

* **Use `AssertionChecker enforce:because:raising:`** (from Buoy) and raise `InstanceCreationFailed`.
* **Don't assert types** — no `isString`, no `isKindOf:`, no `class =`, no `isNotNil` guards at the model layer. If a precondition's *only* check is a type/nil shape, delete the precondition entirely.
* If a non-string slips through where a string was expected, a downstream `notEmpty` / `size <= N` call will fail naturally at the boundary — that's the parser's contract violation, not the domain's.

```smalltalk
Portfolio class >> assertNameIsValid: aName
    AssertionChecker
        enforce: [ aName notEmpty and: [ aName size <= 120 ] ]
        because: 'The portfolio name must be a non-empty string of at most 120 characters'
        raising: InstanceCreationFailed
```

### 3. Encapsulation — no metaprogramming state access

Never reach into another object's private state with reflection. **`instVarNamed:` / `instVarAt:` / `readSlotNamed:` are banned everywhere** — production code, helpers, tests, REPL one-offs. There is no test-only carve-out.

* If a piece of state must be observable, expose a proper public selector for it. In a Kepler-backed system, also list the selector in `registerInterfaces`.
* The only exception is when the user **explicitly authorises** a metaprogramming expression for a specific debug/research task.

####  ❌ Bad Pattern
```smalltalk
"In a test or helper, anywhere"
storedCount := (rootSystem >> #PortfolioManagementSystem) instVarNamed: 'portfolios'
```

####  ✅ Good Pattern
```smalltalk
"On the system:"
PortfolioManagementSystem >> portfolios
    ^ portfolios findAll

"In registerInterfaces, declare it."
"At the call site:"
storedCount := self systemUnderTest portfolios size
```

### 4. Dependency Injection & Composition
* **Inject Collaborators:** Pass all dependencies (collaborators) via the constructor/creation method.
* **Composition over Inheritance:** Prefer composing small, specialized objects rather than creating large classes with complex inheritance hierarchies or many instance variables.

### 5. Mocking Policy (Strict)
* **No Mocks for Internals:** Do not use mock objects for internal domain logic.
* **Boundaries Only:** Mocks/Stubs are permitted *only* for strict external boundaries (e.g., HTTP calls, Database drivers, FFI).
* **Prefer Stubs:** Even for boundaries, prefer passing a stubbed collaborator over a complex mock framework.
    * *Example:* Do not mock the `ZnClient` class methods. Instead, design your object to accept a client instance (or a polymorphic stub) as a parameter.

## 🏷️ Naming Conventions (English CamelCase)

* **Variables (Role-Based):** Use `a` or `an` followed by the **role** of the object, not just its type. Strip suffixes that just announce the class (`Url`, `Timestamp`, `String`, `Object`). The reader should learn what the value *means*, not what its class is.
    * *Bad:* `aString`, `aNumber`, `aliceProfileUrl`, `marketOpenTimestamp`, `usdSettlement` (when the meaning is already in the segment name)
    * *Good:* `aName`, `anAmountOfLaps`, `aClient`, `alice`, `marketOpen`, `localUSD`
* **Collections:** **Never** use `a`, `an`, or bare plural names. Always use the prefix `some` followed by the plural noun.
    * *Bad:* `aList`, `anArray`, `accounts`
    * *Good:* `someStocks`, `someNumbers`, `someAccounts`
* **Block Parameters:** Do not use articles. Use precise nouns.
    * *Examples:* `[:item | ... ]`, `[:node | ... ]`
* **Method parameters follow the same role-based rule.** `aPortfolio` / `newPortfolio` over `original` / `replacement`. The domain type tells the reader what's in the variable; "original" only tells them it's the first argument.

## 📏 Method Size

* **Methods should be short.** A method that does one thing reads like a short paragraph: the top-level body is a list of verbs naming the workflow, not a wall of mechanics.
* **Extract a private helper as soon as you notice:**
    * The method has two-plus distinct phases ("look up, then build, then sort").
    * A block grows large enough that the reader has to mentally label its sections.
    * You're about to write a comment naming a region of the method — that region wants to be a method whose selector *is* the name.
    * A temp variable lives several lines away from its only use.
    * A `transact:` / `do:` / `inject:into:` block spans more than a handful of lines of real logic.
* **The top-level method orchestrates; private helpers do the mechanics.** Helpers go under categories like `private - management`, `private - querying`. The class browser surfaces them as implementation details, callers read them as named verbs.

####  ❌ Bad Pattern (one method holds the whole story)
```smalltalk
updatePortfolio: original with: replacement positions: somePositions on: aDate
    ^ portfolios transact: [
        | existingPositions newPositions |
        newPositions := somePositions asOrderedCollection.
        existingPositions := self positionsOf: original on: aDate ifNone: [ nil ].
        (existingPositions isNil
            or: [ existingPositions asSet ~= newPositions asSet ])
            ifTrue: [ positionsAudit store: (...) ].
        original name = replacement name ifFalse: [
            portfolios update: original executing: [ :stored | stored synchronizeWith: replacement ] ].
        original ]
```

####  ✅ Good Pattern (top-level is a paragraph; mechanics live in helpers)
```smalltalk
updatePortfolio: aPortfolio with: newPortfolio positions: somePositions on: aDate
    ^ portfolios transact: [
        self
            ifExistingPositionsOf: aPortfolio
            on: aDate
            haveBeenChangedBy: somePositions
            do: [ :newPositions | self recordPositionsOf: aPortfolio as: newPositions on: aDate ].
        self synchronizePortfolio: aPortfolio with: newPortfolio.
        aPortfolio ]
```

### Helpers without unused returns

A helper that builds-and-returns a structure callers throw away is dead weight. Either embrace the data (and use it) or embrace the side effect (and don't fake-return).

####  ❌ Bad Pattern (returns a Dictionary every caller discards)
```smalltalk
seedSeveralPortfolios
    | seeded |
    seeded := Dictionary new.
    seeded
        at: #alice put: (self systemUnderTest startManagingPortfolio: (... alice ...));
        at: #bob   put: (self systemUnderTest startManagingPortfolio: (... bob ...)).
    ^ seeded
```

####  ✅ Good Pattern (small, specifically-named helpers; each returns only what callers use)
```smalltalk
startManagingAlicesLongTermPortfolio
    ^ self systemUnderTest startManagingPortfolio: self alicesLongTermPortfolio

startManagingBobsRetirementPortfolio
    ^ self systemUnderTest startManagingPortfolio: self bobsRetirementPortfolio
```

## 🛠️ Syntax, Style & Best Practices

* **Nil Usage:** `nil` is **forbidden** except for lazy initialization logic.
* **No `isNil` / `notNil` tests.** When an answer might not exist, design a block-based API instead of returning `nil` and asking the caller to test for it. Mirror the SUnit / Sagan pattern: a `with<Thing>do:ifNone:` (or `do:else:`) selector takes a found-block and a none-block, and the caller never sees `nil`. The `ifNone:` / `else:` block is often empty when there's nothing to do.

    ####  ❌ Bad Pattern (nil leaks out and gets tested)
    ```smalltalk
    existing := self positionsOf: aPortfolio on: aDate ifNone: [ nil ].
    (existing isNil or: [ existing asSet ~= newPositions asSet ])
        ifTrue: [ aBlock value: newPositions ]
    ```

    ####  ✅ Good Pattern (block-based, no nil)
    ```smalltalk
    self
        withPositionsOf: aPortfolio
        on: aDate
        do: [ :existing |
            existing asSet = newPositions asSet
                ifFalse: [ aBlock value: newPositions ] ]
        ifNone: [ aBlock value: newPositions ]
    ```

    Sagan's `RepositoryBehavior >> withOneMatching:do:else:` already follows this shape. Apply the same pattern to your own `<Thing>System` lookups, helpers, and any place a value might be absent.

* **`withAll:` for collection arguments.** Smalltalk distinguishes `with:` (one element) from `withAll:` (a collection). Honour that distinction in your own selectors.
    * *Bad:* `startManagingPortfolio: aPortfolio with: somePositions` (the keyword `with:` is taking a collection)
    * *Good:* `startManagingPortfolio: aPortfolio withAll: somePositions`
    * The same applies to instance variables and parameters: `someTradableSecurities` (plural) when it's a collection, not `tradableSecurity`.
* **Collections:** Prefer immutable literal Arrays (e.g., `#(1 2 3)`) for static collections unless readability specifically suffers.
* **Blocks & Cull:** When using `cull:`, remember it allows the block to accept *fewer* arguments than the sender provides. Do not flag this mismatch as an error; it is a feature.
* **Cascades:** Prefer cascades (`;`) when sending multiple messages to the same receiver.
* **Forbidden Selectors:** Code must never contain `halt`, `haltOnce`, or `flag:` in production/committed code.
* **Line Endings:** When compiling strings (e.g., in tools/importers), always use `.withInternalLineEndings` to handle Pharo's CR requirements.
* **Type Checking:** Avoid `isKindOf:` or `isMemberOf:`. Use polymorphism or double dispatch.

## 🧪 Testing Guidelines

* **Existence:** Verify functional scenarios are covered. Do not fixate on calculated coverage percentages.
* **Domain-specific fixture names.** Test fixtures, helpers, and constants are named for the concrete thing they represent — never `sampleX`, `aThing`, `urlA`/`urlB`, `someValidThing`. A reader who jumps into a failing test should see what scenario it's testing without chasing helper definitions.
    * *Bad:* `sampleOwner`, `samplePortfolio`, `urlA`, `aTimestamp`
    * *Good:* `aliceFreelanceProfile` (the role + the specific person), `alicesLongTermPortfolio` (owner + curve identity), `acmeOrder42` (a specific external resource), `marketOpenOnMayFirst2026` (a specific moment)
    * Owners use the person's name; settings/segments use the segment; URLs/IDs anchor to whatever the upstream actually calls them; timestamps name the moment.
* **`assert:equals:`, never `assert: (x = y)`.** SUnit's structured assertions print both sides on failure; the plain `assert:` form only reports "false was true". For identity comparisons (`==`), use `assert:identicalTo:`.

    ####  ❌ Bad Pattern
    ```smalltalk
    self assert: (portfolio name = 'Long-term holdings')
    self assert: (portfolio identifier == originalIdentifier)
    ```

    ####  ✅ Good Pattern
    ```smalltalk
    self assert: portfolio name equals: 'Long-term holdings'.
    self assert: portfolio identifier identicalTo: originalIdentifier.
    ```

    Boolean predicates (`isEmpty`, `hasAnyValue`, …) stay single-argument: `self assert: portfolio isEmpty`.

* **`assertCollection:hasSameElements:` for set semantics.** Use it whenever the assertion is about which elements are in the collection, not their order. Reserve `assertCollection:equals:` for assertions where the order is part of what's being verified (e.g. "rows are sorted by modifiedDuration ascending").

    Ask: "If the elements came back in a different order, would the SUT still be correct for this assertion?" If yes → `hasSameElements:`. If no → `equals:` and consider whether the test name should mention the ordering.

## 📚 Related skills

* `create-smalltalk-code` — practical Fluid class definition + MCP compile patterns. Load after this skill.
* `test-driven-development` — the Red-Green-Refactor loop these testing guidelines support. Load alongside this skill for any new code.
* `abbaco-api-house-style` and the `abbaco-api-*` family — specifics for the Abbaco REST API surface; build on these conventions, never override them.
