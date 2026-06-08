---
name: ba-st-buoy
description: Use when writing Pharo/GS64 Smalltalk code that needs runtime assertions/preconditions, Optional/Binding types, collection or exception extensions, equality/hash helpers, or SUnit extras beyond the base image. Covers Buoy's AssertionChecker, Optional, Binding, and extensions.
---

# Buoy — Pharo/GS64 Extensions

Buoy is the base library of the ba-st ecosystem. It plugs useful holes in the base image: assertions, optionals, comparison helpers, collection extensions, exception handling, SUnit extras, and light metaprogramming. If you are on the ba-st stack, most other libraries already depend on it.

Repo: `/home/mtabacman/Development/Repos/ba-st-skills/Buoy/`.

## Installation

```smalltalk
Metacello new
    baseline: 'Buoy';
    repository: 'github://ba-st/Buoy:release-candidate';
    load: 'Development'.
```

Groups: `Deployment`, `Tests`, `Tools`, `Dependent-SUnit-Extensions`, `CI`, `Development`, `GS64-Development`.

## 1. Assertions — `AssertionChecker`

This is the **standard precondition mechanism** across all ba-st projects. Prefer it over raw `self error: ...` — it produces consistent error types (`AssertionFailed`) and messages.

```smalltalk
"Single fact — raises AssertionFailed if block is false"
AssertionChecker
    enforce: [ code size = 2 ]
    because: 'ISO codes must have exactly two letters'.

"Single fact with a specific exception class"
AssertionChecker
    enforce: [ object isDefined ]
    because: 'Object must not be nil'
    raising: InstanceCreationFailed.

"Negate the sense — same API, inverted"
AssertionChecker
    refuse: [ list isEmpty ]
    because: 'List must not be empty'.

"Grouped checks (collect all failures by default; fail-fast opt-in)"
AssertionChecker
    check: [ :asserter |
        asserter
            enforce: [ code size = 2 ]             because: 'Must be 2 chars';
            enforce: [ code allSatisfy: #isLetter ] because: 'Only letters' ]
    configuredBy: [ :checker | checker failFast ].

"Nested / dependent checks — inner only runs when outer passed"
AssertionChecker check: [ :asserter |
    asserter
        enforce: [ code size = 2 and: [ code allSatisfy: #isLetter ] ]
        because: 'Must be 2 letters'
        onSuccess: [ :success |
            success
                enforce: [ officialCodes includes: code ]
                because: ('<1s> is not officially assigned' expandMacrosWith: code) ] ].
```

Use `[ ... ]` for the *reason* when building the message is expensive — it's evaluated only on failure.

## 2. Optionals and Bindings

Use these instead of `nil` sentinels.

**`Optional`** = "this value may or may not exist" (rendering-time decision).
**`Binding`** = "this value is required but might not be configured yet" (access-time failure).

```smalltalk
"Build an Optional"
opt := Optional containing: aFile.
opt := Optional unused.
opt := Optional unusedBecause: 'no file provided'.

"Consume without if/then"
fileOptional
    withContentDo:  [ :file | self renderDetailsOf: file ]
    ifUnused:       [ self renderUploadInstructions ].

"Transform — stays unused if was unused"
urlOptional := fileOptional return: [ :file | file asUrl ].

"Lift several"
Optional
    with: firstOpt and: secondOpt
    whenBothUsedReturn: [ :a :b | a + b ].

"Binding for required-but-deferred dependencies"
binding := Binding undefinedExplainedBy: 'Database not configured yet'.
binding isDefined. "=> false"
binding content.   "=> raises AssertionFailed with the explanation"
binding := Binding to: aDatabaseHandle.
```

## 3. Collection / interval extensions

```smalltalk
"Find extreme using a derived key (instead of sort-then-take)"
#( #(1) #(3 1) #(2) ) maxUsing: [ :a | a first ].      "=> #(3 1)"
people minUsing: [ :p | p age ].

"Filter then map in one pass"
#(1 2 3 4 5) select: [ :x | x even ] thenCollect: [ :x | x * 10 ].
"=> #(20 40)"

"Safe slicing — no error if N > size"
collection copyNoMoreThanFirst: 100.

"Uniqueness with stable insertion order"
#(1 5 2 $a 1 $a) asOrderedSet.  "=> OrderedSet(1 5 2 $a)"

"Suffix test"
'report.json.bak' endsWith: '.bak'.  "=> true"
```

Other notable types: `CircularIterator` (wraps around), `BinarySearchAlgorithm`, `BalancedDistributionInBucketsAlgorithm`.

## 4. Equality and hashing

Use the hash combinator instead of XOR-rolling your own — it handles collisions better and avoids bias.

```smalltalk
Customer >> = other
    ^ self equalityChecker
        compare: #firstName;
        compare: #age;
        checkAgainst: other

Customer >> hash
    ^ self equalityHashCombinator
        combineHashesOfAll: { firstName. age }
```

## 5. Exception handling

```smalltalk
"Catch Error in general but NOT ZeroDivide (let it propagate)"
[ 1 / 0 ]
    on: Error except: ZeroDivide
    do: [ :ex | self handleGeneralError: ex ].
```

## 6. SUnit extensions

```smalltalk
"Same elements, same order"
self assert: actual hasTheSameElementsInTheSameOrderThat: #(1 2 3).

"Exception + its message"
self
    should: [ binding content ]
    raise: AssertionFailed
    withMessageText: 'Parameter not configured'.

"Capture exception to inspect (GS64 flavor)"
self
    should: [ obj doSomething ]
    raise: CustomError
    withExceptionDo: [ :signal | self assert: signal code equals: 42 ].

"Extract the unique element of a singleton collection"
self withTheOnlyOneIn: aCollection do: [ :element | self assertValid: element ].

"Platform skips"
MyTest >> testOnlyOnPharo
    self runOnlyInPharo: [ "..." ]
```

## 7. Metaprogramming

- **`Interface`** — a named structural contract:
  ```smalltalk
  aStream := Interface
      named: 'Stream'
      declaring: #( #nextPut: #next #atEnd ).
  aStream isImplementedBy: anObject.
  ```
- `MessageSendingCollector`, `KeywordMessageSendingCollector`, `UnaryMessageSendingCollector` — collect sends performed by a block. Useful for DSL-style fluent builders.
- `Namespace` — lightweight qualification/scope.

## Gotchas

- **Use `[ ... ]` for expensive reason strings.** String concatenation inside the assertion call runs *every* time; a block only runs on failure.
- **Custom exception with `check:configuredBy:`.** The exception class must understand `signalAll:` (plural), not just `signal:`, because grouped checks may collect multiple failures.
- **Optional vs Binding.** Optional = might or might not render / branch. Binding = required; reaching it uninitialized is a bug. Don't use `Optional` for lazy dependency injection.
- **`on:except:do:` excludes a *subclass*.** Parent-class handlers outside still fire for the excluded subclass. It does not "swallow" the excluded exception.
- **`CircularIterator` on an empty collection raises** — don't iterate without ensuring non-empty.
- **`OrderedSet` is O(n) for membership tests.** Use a plain `Set` if you only need uniqueness; use `OrderedSet` when order matters for display/iteration.
- **Platform-specific SUnit helpers.** `runOnlyInPharo:` etc. skip tests on non-matching dialects — don't put production logic in these blocks.

## When to reach for Buoy

Always, when working inside the ba-st ecosystem. `AssertionChecker` is used pervasively; Optional/Binding show up in request pipelines (Stargate/Superluminal) and lifecycle code (Kepler/Launchpad). Treat it as "the base library."
