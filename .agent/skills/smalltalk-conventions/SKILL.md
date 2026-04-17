---
name: smalltalk-conventions
description: Global coding standards for Smalltalk (Pharo). Defines naming, architecture, and syntax constraints emphasizing immutability and valid object creation. Also covers the foundational OOP principles (messaging, bijection, simple design), polymorphism over conditionals, code generation conventions for MCP eval, and code review checklists.
---

# Smalltalk Code Conventions

This skill outlines mandatory conventions and the lessons learned for writing robust Smalltalk code.

## 🌟 Core Principles

These principles sit *above* every individual rule below. When in doubt, fall back to them.

### Messaging Over Objects

> "I'm sorry that I long ago coined the term 'objects' for this topic because it gets many people to focus on the lesser idea. The big idea is messaging." — Alan Kay

Objects are autonomous computers that communicate only by sending messages. The quality of a design is determined by *how* modules communicate, not by what their internal structure looks like. The essential triad:

1. **Message passing** — the primary mechanism of computation.
2. **Encapsulation** — local retention, protection, and hiding of state.
3. **Dynamic binding** — extreme late binding of all things.

Classes, inheritance, and static types are tools within this paradigm; they are not the paradigm itself.

### The Bijection Principle

Every domain concept maps to exactly one object in the code, and every object in the code maps to exactly one domain concept (Wilkinson, Contieri).

Common violations:

- **One object, multiple entities.** `10` used to represent both 10 meters and 10 inches (the Mars Climate Orbiter loss).
- **One entity, multiple objects.** The same person modeled as both an `Athlete` and a `Judge` with no shared identity.
- **Missing entities.** A `String` where `EmailAddress` belongs; a raw `Number` where `Money` belongs.
- **Extra entities.** A `CustomerManager` with no real-world counterpart — the behavior belongs on `Customer` itself.

If you cannot find the real-world counterpart of a code object (or vice versa), the model is broken.

### Simple Design

Kent Beck's four rules, in strict priority order:

1. **Passes the tests.**
2. **Reveals intention.**
3. **No duplication.**
4. **Fewest elements.**

> "Make it work, make it right, make it fast." — Kent Beck (in that order, always)

Three similar lines are better than a premature abstraction; add abstractions only once the duplication pattern is clear.

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

    ####  Pattern: Domain Validation
    Use this pattern to prevent invalid data from ever entering your system.

    **Class Side (Guard Clause):**
    ```smalltalk
    User class >> email: aString age: anAmountOfYears
        anAmountOfYears < 0 ifTrue: [ Error signal: 'Age cannot be negative' ].
        ^ self new initializeEmail: aString age: anAmountOfYears
    ```

* **Creation-method selectors must read like a sentence.**
    *Good:* `Game playedBy: somePlayers on: aBoard rolling: someDice`.
    *Bad:* `Game players: somePlayers board: aBoard dice: someDice`.
    Prefer verbs in the infinitive (`rolling`, `showing`, `providing`) or meaningful connectors (`on`, `for`, `by`). Never use a single bare connector as the only keyword — if the selector has several keywords, at least one should carry a verb.

* **Initialization method naming:** the initializer is always `initialize` followed by the creation selector with the first letter capitalized. `playedBy:on:rolling:` calls `initializePlayedBy:on:rolling:`.

* **Fail fast.** If the domain rejects the input (November 31st, negative age, empty player list), signal an error in the creation method. Never create a broken object and hope a downstream caller fixes it.

### 2. Dependency Injection & Composition
* **Inject Collaborators:** Pass all dependencies (collaborators) via the constructor/creation method.
* **Composition over Inheritance:** Prefer composing small, specialized objects rather than creating large classes with complex inheritance hierarchies or many instance variables.
* **Inheritance is ontology, not reuse.** Use inheritance only for genuine IS-A domain relationships (`SavingsAccount` IS-A `Account`). A `Stack` is *not* a kind of `OrderedCollection` — it *uses* one.
* **Watch hierarchy depth.** More than 2–3 levels is a warning sign. Flatten with composition.
* **One domain per object.** A `Customer` does not know about SQL. An `Invoice` does not know about HTTP. Each object lives in one problem domain and talks to other domains through messages.

### 3. Mocking Policy (Strict)
* **No Mocks for Internals:** Do not use mock objects for internal domain logic.
* **Boundaries Only:** Mocks/Stubs are permitted *only* for strict external boundaries (e.g., HTTP calls, Database drivers, FFI).
* **Prefer Stubs:** Even for boundaries, prefer passing a stubbed collaborator over a complex mock framework.
    * *Example:* Do not mock the `ZnClient` class methods. Instead, design your object to accept a client instance (or a polymorphic stub) as a parameter.

    ```smalltalk
    "Good: inject the collaborator"
    WeatherService class >> using: anHttpClient
        ^ self new initializeUsing: anHttpClient

    "In tests, pass a stub:"
    StubHttpClient >> get: aUrl
        ^ '{"temp": 20}' "controlled response"

    WeatherService using: StubHttpClient new
    ```

### 4. Immutability

Favor immutable objects. Immutable objects:

- Eliminate temporal coupling — order of operations does not matter.
- Can be shared freely between collaborators without defensive copying.
- Simplify reasoning about correctness and enable referential transparency.

When mutation is necessary, confine it to well-defined boundaries, make changes atomic (all required fields change together), and keep the object valid after every change.

```smalltalk
"Immutable: return a new date"
tomorrow := today nextDay.

"If mutation is necessary, make it atomic:"
account transferAmount: 100 to: savings.
"Not: account setBalance: account balance - 100."
```

### 5. Polymorphism Over Conditionals

Never check the type of an object in domain logic. Do not use `isKindOf:`, `isMemberOf:`, `class`, or `respondsTo:` to decide what to do.

> "Don't check who they are. Ask them to do instead." — Maxi Contieri

Give different objects different implementations of the same message.

**Null Object pattern.** Instead of returning `nil` and forcing every caller to check, return an object that responds to the same protocol with safe defaults.

```smalltalk
"❌ Bad: nil-checks spread through the code"
customer address
    ifNil: [ 'No address' ]
    ifNotNil: [ :addr | addr printString ].

"✅ Good: NullAddress responds to the same protocol"
customer address printString.
"where a missing address is a MissingAddress that prints 'No address'"
```

`nil` is not a polymorphic object. Every `ifNil:` in domain code is a design smell signaling a missing abstraction.

### 6. Encapsulation

Hide all implementation details. Expose **behavior**, never structure. "Behavior is essential, data is accidental" (Contieri).

If you find yourself writing `customer name` to build a display string elsewhere, ask whether `customer` should instead understand `printOn:` / `displayStringOn:`. The caller must not need to know what data the object holds — only what the object can do.

### 7. Coupling & Cohesion

- **Maximize cohesion:** every method and instance variable in a class should relate to the class's single responsibility.
- **Minimize coupling:** objects couple only through the messages they exchange. Keep that set small and intentional.

> "Coupling is software's fundamental problem." — Maxi Contieri

### 8. Metaphors

Name objects after the roles they play in the domain. A `Ledger` records transactions. A `Cashier` processes sales. A `Warehouse` manages inventory. A good metaphor makes "where would I find the logic for X?" obvious.

## 🏷️ Naming Conventions (English CamelCase)

- **Variables (Role-Based):** Use `a` or `an` followed by the **role** of the object, not just its type.
  - *Bad:* `aString`, `aNumber`
  - *Good:* `aName`, `anAmountOfLaps`, `aClient`
- **Collections:** **Never** use `a`, `an`, or bare plural names. Always use the prefix `some` followed by the plural noun.
  - *Bad:* `aList`, `anArray`, `accounts`
  - *Good:* `someStocks`, `someNumbers`, `someAccounts`
- **Block Parameters:** Do not use articles. Use precise nouns.
  - *Examples:* `[:item | ... ]`, `[:node | ... ]`

### Class Names

A class name synthesizes the meaning of all messages its instances understand. Reading the name should give a reasonable expectation of what instances can do.

- **Use domain language.** If domain experts say "Invoice," do not call it `BillingDocument`. If they say "Policy," do not call it `RuleSet`.
- **Avoid implementation-oriented suffixes.** `Manager`, `Helper`, `Handler`, `Processor`, `Data`, `Info`, `Utils` almost always hide a missing domain concept. A `TransactionManager` is probably just a `Ledger`.
- **"And" signals two responsibilities.** `ReaderAndWriter` should be `Reader` and `Writer`.
- **Placeholder over misleading name.** If you do not yet understand a concept well enough to name it, use `Xyz` or `TODO`. A deliberately wrong name invites correction; a subtly wrong name invites bugs.

### Method Names (Selectors)

- **Intention-revealing.** Name messages for WHAT, not HOW. `sortByDate` over `quickSort`. `includesElement:` over `linearSearch:`.
- **Leverage keyword messages — they should read as prose.**

    ```smalltalk
    "Good: reads as prose"
    account transferAmount: 100 to: savings.
    collection detect: [ :each | each isOverdue ] ifNone: [ NullInvoice new ].

    "Bad: positional arguments obscure meaning"
    account transfer: 100 and: savings.
    ```

- **No abbreviations.** Write `numberOfElements`, not `numElems`. Smalltalk has no line-length pressure.
- **Composed Method.** Break methods into small pieces, each with a name that explains its role in the larger computation.

### Instance Variables

- No articles. Use precise nouns: `name`, `client`, `stocks`, `accounts`.
- Describe the **relationship** with the collaborator, not its type: `owner` over `person`; `paymentStrategy` over `strategy`.

### Temporary Variables

Describe the role of the value. `highestBid` over `temp` or `b`.

## 🛠️ Syntax, Style & Best Practices

- **Nil Usage:** `nil` is **forbidden** except for lazy initialization logic.
- **Collections:** Prefer immutable literal Arrays (e.g., `#(1 2 3)`) for static collections unless readability specifically suffers.
- **Blocks & Cull:** When using `cull:`, remember it allows the block to accept *fewer* arguments than the sender provides. Do not flag this mismatch as an error; it is a feature.
- **Cascades:** Prefer cascades (`;`) when sending multiple messages to the same receiver.
- **Forbidden Selectors:** Code must never contain `halt`, `haltOnce`, or `flag:` in production/committed code.
- **Line Endings:** When compiling strings (e.g., in tools/importers), always use `.withInternalLineEndings` to handle Pharo's CR requirements.
- **Type Checking:** Avoid `isKindOf:` or `isMemberOf:`. Use polymorphism or double dispatch.

## 🧪 Testing Guidelines

- **Existence:** Verify functional scenarios are covered. Do not fixate on calculated coverage percentages.
- **Behavior, not state.** Assert on what instances respond to, not on their instance variables.
- **Scenario names.** `testTransferReducesSourceBalance` over `testTransfer`. The name should describe the stimulus and the expected outcome.
- **Invalid scenarios.** Every creation method's guard clause deserves a `should:raise:withMessageText:` test. See `create-smalltalk-code` §4.1.
- **Independent tests.** Each test sets up its own world. Tests must be runnable in any order.
- **Real objects for domain logic.** See the Mocking Policy above — mocks/stubs only at strict I/O boundaries.

## 🦨 Code Smells Quick Reference

| Smell | Remedy |
|---|---|
| Anemic Model | Move behavior into the object, remove accessors |
| Nil Returns / Nil Checks | Null Object pattern or `detect:ifNone:` |
| Type Checking (`isKindOf:`) | Polymorphic message |
| Long Method | Composed Method — extract with intention-revealing names |
| Feature Envy | Move method to the envied object |
| Primitive Obsession | Create value object (`EmailAddress`, `Money`, `DateRange`) |
| Singleton Abuse | Pass collaborators explicitly through constructors |
| Incomplete Object | Require all collaborators in constructor |
| Deep Inheritance | Flatten with composition |
| God Class | Identify distinct responsibilities; split into cohesive objects |
| Overengineering | Apply Beck's "fewest elements"; build only what you need today |
| Coupled Tests | Each test sets up its own world; no shared mutable state |
| Premature Optimization | "Make it work, make it right, make it fast." Measure first |

## ✅ Code Review Checklist

Run through this list on every review — each item should have a clear "yes."

### Design

- [ ] Each class maps to exactly one domain concept (Bijection)?
- [ ] Objects complete and valid at creation?
- [ ] Behavior lives in objects, not in callers extracting data through accessors?
- [ ] Type conditionals replaced with polymorphic messages?
- [ ] `nil` used as a sentinel? Null Object instead?
- [ ] Objects immutable where possible?

### Naming

- [ ] Class names in domain language (not implementation roles)?
- [ ] Method names reveal intention?
- [ ] Collections use the `some` prefix? Variables use role-based names?
- [ ] Any `Manager` / `Helper` / `Handler` / `Utils` hiding a missing concept?

### Testing

- [ ] Each new behavior has a test?
- [ ] Test names describe scenarios and outcomes?
- [ ] Tests written before the implementation (TDD evidence)?
- [ ] Tests independent — runnable in any order?
- [ ] Tests assert on behavior, not internal state?
- [ ] Invalid-scenario guards covered by `should:raise:withMessageText:`?
- [ ] Mocks only at I/O boundaries?

### Coupling

- [ ] New dependencies justified and minimal?
- [ ] Inheritance only for genuine IS-A domain relationships?
- [ ] Different domains kept separate? (No SQL in domain objects, no HTTP in models)

### Simplicity

- [ ] Beck's four rules satisfied?
- [ ] No unnecessary abstraction?
- [ ] No premature optimization?
