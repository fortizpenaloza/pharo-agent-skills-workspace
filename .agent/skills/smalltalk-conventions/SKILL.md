---
name: smalltalk-conventions
description: Global coding standards for Smalltalk. Defines naming, architecture, and syntax constraints.
---

# Smalltalk Code Conventions

This skill outlines the lessons learned and conventions followed to write Smalltalk code.

## Code Style & Best Practices
* **Blocks & Cull:** Remember that `cull:` (and `cull:cull:`) allows blocks to accept *fewer* arguments than the selector implies. Do not flag this as an error.
* **Nil:** Never use `nil` except for lazy initialization.
* **Collections:** Prefer immutable Arrays for static/fixed collections unless readability suffers.
* **Forbidden:** Code must never contain `halt` senders.
* **Enforce only the creation of valid objects:** This has an immediate consequence of avoiding the creation of setters.

    * **Example:**
    ```smalltalk
    ArithmeticCondition toBeDifferentTo: 1
    ```
    instead of 
    ```smalltalk
    ArithmeticCondition new toBeDifferentTo: 1
    ```

* **Instance Creation:** Again, avoid the use of `new`, usually means an anemic model or the use of setters. 

    * **Example:**
    ```smalltalk
    CircularIterator class >> cyclingOver: aSequence
        ^ self new initializeCyclingOver: aSequence
    ```

    ```smalltalk 
    CircularIterator >> initializeCyclingOver: aSequence
        options := aSequence
    ```

* **Avoid isKindOf: or isMemberOf:** Use polymorphism or double dispatch to handle type-specific behavior.

## 🏷️ Naming (English CamelCase)
* **Variables/Arguments:** Use `a` or `an` for single objects (e.g., `aClient`, `anAccount`). Do not use the `the` prefix. Also use the role of the variable (e.g., `aName` instead of `aString` if the variable is used as a client or `anAmountOfLaps` instead of `aNumber`).
* **Method Arguments (Collections):** Always use the prefix `some` followed by the plural noun. **Never** use `a` or `an` for collections.
    * *Bad:* `aList`, `anArray`, `aCollectionOfNumbers`.
    * *Good:* `someStocks`, `someNumbers`, `someAccounts`.
* **Block Parameters:** Do not use articles. Use precise nouns (e.g., `[:item | ... ]`, `[:node | ... ]`).

## 🛠️ Syntax & Formatting
* **Line Endings:** Always use `.withInternalLineEndings` when compiling strings to handle Pharo's CR requirement.
* **Cascades:** Prefer cascades (`;`) for multiple messages to the same receiver.
* **Nil:** Only allowed for lazy initialization.

## 🏗️ Architecture
* **Dependency Injection:** Favor passing collaborators in constructors/initializers.
* **Object Composition:** Prefer composing small, specialized objects over creating large classes with many instance variables.
* **Mocking Policy (Strict):**
    * **Avoid Mocks for internals:** Do not use mock objects for internal domain logic.
    * **Dependency Injection:** Objects must accept enough collaborators in their creation/configuration to be tested without external information.
    * **Exception:** Mocks are allowed only for strict external boundaries (HTTP calls, Database drivers, FFI).
    * *Example:* Do not mock `ZnClient`. Instead, pass the client (or a stubbed collaborator) as a parameter to the method or object being tested.
* **Test Data:** Do not critique the language used in test assertion strings or text messages (English or Spanish is acceptable in tests).
* **Tests:** Verify tests exist. Do not calculate coverage, but ensure functional scenarios are covered.