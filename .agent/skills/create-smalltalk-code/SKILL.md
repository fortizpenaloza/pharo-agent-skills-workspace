---
name: create-smalltalk-code
description: Guide for creating Pharo Smalltalk (Pharo) classes, methods, and tests. Covers fluid class definition, method compilation, test creation, deterministic testing, validation of invalid scenarios, and troubleshooting. Invoke before writing any Smalltalk code through the pharo-smalltalk MCP.
---

# Create Smalltalk Class Skill

## ⚠️ Standards Compliance
**Before generating any code**, you must review and apply the rules defined in the `smalltalk-conventions` skill (naming, object design, TDD).

This skill outlines the standard procedure for modeling domain objects and creating tests in Pharo Smalltalk using the MCP server.

## 1. Class Creation

Use the **Fluid Class Definition** syntax (`<<`). This is the modern and preferred way to define classes in Pharo. Always 
verify the class was created successfully immediately after installation.

**Template:**

```smalltalk
(Superclass << #ClassName
  slots: { #slot1 . #slot2 };
  package: 'PackageName') install.
```

**Example:**

```smalltalk
(Object << #FVector
  slots: { #x . #y };
  package: 'FVectorModel') install.
```

**Verification:**
Always verify the class was created successfully immediately after installation.

```smalltalk
(Object << #FVector slots: { #x . #y }; package: 'FVectorModel') install.
'Class <1s> installed successfully' expandMacrosWith: (Smalltalk at: #FVector)
```

## 2. Method Compilation

Compile methods using `compile:classified:`. Ensure the class exists before compiling methods for it.

**🚨 Tip:** To prevent "Linefeed" errors, always append withInternalLineEndings to the source string.

**Template:**

```smalltalk
ClassName compile: 'methodSelector: argument
  ^ return' withInternalLineEndings classified: 'protocol'.
```

**Example (Accessing):**

```smalltalk
IS2Player compile: 'name
  ^ name' withInternalLineEndings classified: 'accessing'.
```

**Example (Logic):**

```smalltalk
FVector compile: '+ aVector
  ^ FVector new
    x: (x + aVector x);
    y: (y + aVector y);
    yourself' withInternalLineEndings classified: 'arithmetic'.
```

### 2.1 Class-Side Creation Methods

Follow the **Creation Method Pattern** from `smalltalk-conventions`: a public class-side method that reads as a sentence, delegating to a private instance-side `initialize...` method. Use `ClassName class compile:` for class-side methods.

**Class-side (public API):**

```smalltalk
FVector class compile: 'x: anXCoordinate y: aYCoordinate

  ^ self new initializeX: anXCoordinate y: aYCoordinate' withInternalLineEndings classified: 'instance creation'.
```

**Instance-side (private initialization):**

```smalltalk
FVector compile: 'initializeX: anXCoordinate y: aYCoordinate

  x := anXCoordinate.
  y := aYCoordinate' withInternalLineEndings classified: 'initialization'.
```

**Accessing method (no setter — read-only):**

```smalltalk
FVector compile: 'x
  ^ x' withInternalLineEndings classified: 'accessing'.
```

**Refactored logic using the creation method (no public setters, immutable result):**

```smalltalk
FVector compile: '+ aVector
  ^ self class x: x + aVector x y: y + aVector y' withInternalLineEndings classified: 'arithmetic'.
```

### 2.2 Indentation and Line Endings (MCP eval)

When submitting code via `mcp__smalltalk-interop__eval`:

- Use **LF only (`\n`)**. CRLF (`\r\n`) passes through the eval handler verbatim and corrupts Tonel exports.
- Use **2-space indentation** in method bodies — this matches Pharo's formatter and what `TonelWriter` exports.
- Avoid **trailing whitespace** on any line.
- Always append **`withInternalLineEndings`** to source strings.

## 3. Test Creation

Create a test class subclassing `TestCase`.

**Template:**

```smalltalk
(TestCase << #ClassNameTest
  slots: {};
  package: 'PackageName') install.
```

**Example:**

```smalltalk
(TestCase << #FVectorTest
  slots: {};
  package: 'FVectorTest') install.
```

A common convention is to place tests in a sibling package named `PackageName-Tests`:

```smalltalk
(TestCase << #FVectorTest
  slots: {};
  package: 'FVectorModel-Tests') install.
```

## 4. Test Implementation

Write test methods ensuring they start with `test`.

**Template:**

```smalltalk
ClassNameTest compile: 'testFeature

  | instance result |
  instance := ClassName new.
  result := instance someOperation.
  self assert: result equals: expectedValue.' withInternalLineEndings classified: 'tests'.
```

**Example (behavioral scenario, using the creation method):**

```smalltalk
FVectorTest compile: 'testVectorAddition

  | result |
  result := (FVector x: 1 y: 2) + (FVector x: 3 y: 4).
  self assert: result x equals: 4.
  self assert: result y equals: 6' withInternalLineEndings classified: 'tests'.
```

### 4.1 Testing Invalid Scenarios

For every creation method, add tests that check invalid cases are rejected (e.g., negative values, empty collections, incompatible parameters). Use `should:raise:withMessageText:` to assert both the exception class and the message.

**Example (guarding number of sides):**

```smalltalk
DieTest compile: 'testNegativeFacesAreNotAllowed

  self should: [ Die sided: -5 ] raise: InstanceCreationFailed withMessageText: ''Number of sides must be strictly positive''' withInternalLineEndings classified: 'tests - invalid scenarios'.
```

**Example (guarding player list):**

```smalltalk
GameTest compile: 'testAtLeastOnePlayerInTheGame

  self should: [ Game playedBy: {} ] raise: InstanceCreationFailed withMessageText: ''There must be at least one player''' withInternalLineEndings classified: 'tests - invalid scenarios'.
```

**Example (guarding user age):**

```smalltalk
UserTest compile: 'testNegativeAgeIsRejected

  self should: [ User email: ''alice@example.com'' age: -1 ] raise: Error withMessageText: ''Age cannot be negative''' withInternalLineEndings classified: 'tests - invalid scenarios'.
```

Pair every happy-path test with at least one invalid-scenario test covering each guard clause in the creation method.

## 5. Deterministic Testing (Mocking)

When dealing with randomness (e.g., dice rolls), create a subclass or a mock object to control the output.

**Example (Mock Die):**

```smalltalk
(IS2Die << #IS2LoadedDie
	slots: { #rollResult };
(IS2Die << #IS2LoadedDie
  slots: { #rollResult };
  package: 'IS2GameTest') install.

IS2LoadedDie compile: 'roll
  ^ rollResult' withInternalLineEndings classified: 'action'.
```

**Example (Loaded die using the full creation method pattern):**

```smalltalk
(Die << #LoadedDie
    slots: { #rollResult };
    package: 'Game-Tests') install.

LoadedDie class compile: 'rolling: aResult

  ^ self new initializeRolling: aResult' withInternalLineEndings classified: 'instance creation'.

LoadedDie compile: 'initializeRolling: aResult

  rollResult := aResult' withInternalLineEndings classified: 'initialization'.

LoadedDie compile: 'roll

  ^ rollResult' withInternalLineEndings classified: 'action'.
```

Reminder (from `smalltalk-conventions`): mocks/stubs are permitted **only at strict external boundaries** (HTTP, DB, FFI) or to make non-deterministic collaborators deterministic in tests. Never mock internal domain logic.

## 6. Execution & Verification

Run tests using the `run_class_test` command (the canonical MCP tool may be exposed as `mcp__pharo-smalltalk__run_class_test`).

```json
{
  "class_name": "ClassNameTest"
}
```

Verify that the test actually ran:

- **Red** before implementation: `MessageNotUnderstood` (class/method missing) or assertion failure for the right reason.
- **Green** after implementation: all tests pass and the Transcript is clean.

### 6.1 Reformat Before Export

Before exporting a package to git, run a reformat pass on all classes touched in the session so the Tonel export matches Pharo's canonical formatting:

```smalltalk
(Smalltalk packageOrganizer packageNamed: 'YourPackage') definedClasses
    do: [ :cls |
        cls methods do: [ :m | m reformat ].
        cls class methods do: [ :m | m reformat ] ]
```

## Troubleshooting

| Problem | Cause | Fix |
|---|---|---|
| `OCUndeclaredVariableNotice` | Referenced class doesn't exist yet | Create and install the referenced class before compiling methods that use it |
| `KeyNotFound` on test run | Test class not registered in `SystemEnvironment` | Re-run the class creation `install` command |
| `HTTP 500` on `Smalltalk snapshot:andQuit:` | `SnapshotOperation` is not JSON-serializable | Expected — the snapshot succeeded. Verify by querying image state |
| `HTTP 500` on class/package removal | `Metaclass` / `Package` is not JSON-serializable | Expected — the operation succeeded. Verify by re-querying |
| "Linefeed" errors | Missing `withInternalLineEndings` | Append `withInternalLineEndings` to the source string |
| CRLF characters appear in exported Tonel | Source string submitted with `\r\n` | Resubmit using LF-only (`\n`) line endings |
| Method body indentation differs from Tonel output | Wrong indent width or tabs in source | Use 2-space indentation; run a reformat pass before export |
