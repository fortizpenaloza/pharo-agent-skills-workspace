---
name: smalltalk-conventions
description: Global coding standards for Smalltalk (Pharo). Defines naming, architecture, and syntax constraints emphasizing immutability and valid object creation.
---

# Smalltalk Code Conventions

This skill outlines mandatory conventions and the lessons learned for writing robust Smalltalk code.

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

### 2. Dependency Injection & Composition
* **Inject Collaborators:** Pass all dependencies (collaborators) via the constructor/creation method.
* **Composition over Inheritance:** Prefer composing small, specialized objects rather than creating large classes with complex inheritance hierarchies or many instance variables.

### 3. Mocking Policy (Strict)
* **No Mocks for Internals:** Do not use mock objects for internal domain logic.
* **Boundaries Only:** Mocks/Stubs are permitted *only* for strict external boundaries (e.g., HTTP calls, Database drivers, FFI).
* **Prefer Stubs:** Even for boundaries, prefer passing a stubbed collaborator over a complex mock framework.
    * *Example:* Do not mock the `ZnClient` class methods. Instead, design your object to accept a client instance (or a polymorphic stub) as a parameter.

## 🏷️ Naming Conventions (English CamelCase)

* **Variables (Role-Based):** Use `a` or `an` followed by the **role** of the object, not just its type.
    * *Bad:* `aString`, `aNumber`
    * *Good:* `aName`, `anAmountOfLaps`, `aClient`
* **Collections:** **Never** use `a`, `an`, or bare plural names. Always use the prefix `some` followed by the plural noun.
    * *Bad:* `aList`, `anArray`, `accounts`
    * *Good:* `someStocks`, `someNumbers`, `someAccounts`
* **Block Parameters:** Do not use articles. Use precise nouns.
    * *Examples:* `[:item | ... ]`, `[:node | ... ]`


## 🛠️ Syntax, Style & Best Practices

* **Nil Usage:** `nil` is **forbidden** except for lazy initialization logic.
* **Collections:** Prefer immutable literal Arrays (e.g., `#(1 2 3)`) for static collections unless readability specifically suffers.
* **Blocks & Cull:** When using `cull:`, remember it allows the block to accept *fewer* arguments than the sender provides. Do not flag this mismatch as an error; it is a feature.
* **Cascades:** Prefer cascades (`;`) when sending multiple messages to the same receiver.
* **Forbidden Selectors:** Code must never contain `halt`, `haltOnce`, or `flag:` in production/committed code.
* **Line Endings:** When compiling strings (e.g., in tools/importers), always use `.withInternalLineEndings` to handle Pharo's CR requirements.
* **Type Checking:** Avoid `isKindOf:` or `isMemberOf:`. Use polymorphism or double dispatch.


## 🧪 Testing Guidelines
* **Existence:** Verify functional scenarios are covered. Do not fixate on calculated coverage percentages.