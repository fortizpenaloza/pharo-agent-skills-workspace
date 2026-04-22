---
name: create-smalltalk-code
description: Guide for creating Smalltalk (Pharo or GS64) classes, methods, and tests using the Fluid syntax.
---

# Create Smalltalk Class Skill

## ⚠️ Standards Compliance
**Before generating any code**, you must review and apply the rules defined in the `smalltalk-conventions` skill.

This skill outlines the standard procedure for modeling domain objects and creating tests in Pharo Smalltalk using the MCP server.

## 1. Class Creation

Use the **Fluid Class Definition** syntax (`<<`). This is the modern and preferred way to define classes in Pharo. Always 
verify the class was created successfully immediately after installation.

**Template:**

```smalltalk
(Superclass << #ClassName slots: { #slot1 . #slot2 }; package: 'PackageName') install.
'Class <1s> installed successfully' expandMacrosWith: Smalltalk at: #ClassName
```

**Example:**

```smalltalk
(Object << #FVector slots: { #x . #y }; package: 'FVectorModel') install.
'Class <1s> installed successfully' expandMacrosWith: Smalltalk at: #FVector
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

**Example (Setter):**

```smalltalk
FVector compile: 'x: anInteger
    x := anInteger' withInternalLineEndings classified: 'accessing'.
```

**Example (Logic):**

```smalltalk
FVector compile: '+ aVector
    ^ FVector new
        x: (x + aVector x);
        y: (y + aVector y);
        yourself' withInternalLineEndings classified: 'arithmetic'.
```

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

## 4. Test Implementation

Write test methods ensuring they start with `test`.

**Template:**
```smalltalk
ClassNameTest compile: 'testFeature
	| instance result |
	instance := ClassName new.
	"Setup"
	result := instance someOperation.
	self assert: result equals: expectedValue.' withInternalLineEndings classified: 'tests'.
```

## 5. Deterministic Testing (Mocking)

When dealing with randomness (e.g., dice rolls), create a subclass or a mock object to control the output.

**Example (Mock Die):**
```smalltalk
(IS2Die << #IS2LoadedDie
	slots: { #rollResult };
	package: 'IS2GameTest') install.

IS2LoadedDie compile: 'roll
	^ rollResult' withInternalLineEndings classified: 'action'.
```

## 6. Execution & Verification

Run tests using `mcp_pharo_run_class_test`.

```json
{
  "class_name": "ClassNameTest"
}
```

## Troubleshooting

-   **Variable Undeclared**: If you see `OCUndeclaredVariableNotice`, ensure the class you are referencing (like `IS2Die` inside `IS2GameTest`) is fully created and installed *before* you try to compile methods that reference it.
-   **KeyNotFound**: If `mcp_pharo_run_class_test` fails with `KeyNotFound`, it usually means the test class itself was not registered in the `SystemEnvironment`. Re-run the class creation `install` command.
