# Proposed Improvements

## 1. Create `CLAUDE.md` (minimal bootstrap)

**File:** `CLAUDE.md`

No duplicated rules — skills are the authoritative source. CLAUDE.md just bootstraps the session:
- Workspace description
- "Read these skills before any Pharo work" with paths to the three skills
- MCP server notice and a quick tool reference table (read-only vs. package-I/O distinction, which isn't documented anywhere else)

## 2. Expand MCP permissions

**File:** `.claude/settings.local.json`

Currently only `eval` is auto-permitted. Add all read-only and test-execution tools to the `allow` list so Claude can explore and run tests without interrupting the user for approval on every operation.

Add to `"allow"`:
- `mcp__pharo-smalltalk__get_class_source`
- `mcp__pharo-smalltalk__get_method_source`
- `mcp__pharo-smalltalk__get_class_comment`
- `mcp__pharo-smalltalk__list_packages`
- `mcp__pharo-smalltalk__list_classes`
- `mcp__pharo-smalltalk__list_methods`
- `mcp__pharo-smalltalk__list_extended_classes`
- `mcp__pharo-smalltalk__search_classes_like`
- `mcp__pharo-smalltalk__search_methods_like`
- `mcp__pharo-smalltalk__search_implementors`
- `mcp__pharo-smalltalk__search_references`
- `mcp__pharo-smalltalk__search_references_to_class`
- `mcp__pharo-smalltalk__search_traits_like`
- `mcp__pharo-smalltalk__get_settings`
- `mcp__pharo-smalltalk__run_class_test`
- `mcp__pharo-smalltalk__run_package_test`

Keep requiring approval: `import_package`, `export_package`, `install_project`, `apply_settings`, `read_screen` — these mutate the image or filesystem and warrant explicit confirmation.

## 3. Fix `create-smalltalk-code/SKILL.md`

**File:** `.agent/skills/create-smalltalk-code/SKILL.md`

Confirmed issues:
- **Setter contradiction:** Section 2 includes a setter example (`x: anInteger / x := anInteger`) that directly violates the conventions skill ("No Public Setters"). Replace with a proper `initialize` method example.
- **Wrong tool name:** Section 6 references `mcp_pharo_run_class_test` (wrong format). Correct to `mcp__pharo-smalltalk__run_class_test`.
- **Missing class-side methods:** Add a section showing `ClassName class compile: '...' withInternalLineEndings classified: 'instance creation'` — required for the creation method pattern but completely absent.
- **Test template uses bare `new`:** Section 4 test template calls `ClassName new` directly, violating the conventions. Fix to use a class-side creation method.

## 4. Complete `smalltalk-conventions/SKILL.md`

**File:** `.agent/skills/smalltalk-conventions/SKILL.md`

The Testing Guidelines section ends abruptly at one bullet (line 103). Complete it with:
- Assertion selectors: `assert:equals:` (preferred), `assert:` (boolean), `deny:`, `should:raise:`
- Test method naming: past-tense behavioral (`testRejectsNegativeAge`, `testReturnsSumOfAllElements`)
- `setUp` guidance: minimal fixture, no shared mutable state between tests

Also add a `printOn:` convention to the Architecture section — every domain object should define it for debugging.

## 5. Fix `test-driven-development/SKILL.md`

**File:** `.agent/skills/test-driven-development/SKILL.md`

- Line 114: vague "calling the pharo mcp command run_class_test" — use the full tool name `mcp__pharo-smalltalk__run_class_test`
- REFACTOR phase: add `mcp__pharo-smalltalk__run_package_test` for regression checking after structural changes
- The `.dot` Graphviz diagram won't render in any Claude context — replace with an ASCII table

## 6. Add `explore-pharo-image` skill

**File:** `.agent/skills/explore-pharo-image/SKILL.md`

Biggest behavioral gap: Claude's default instinct is to create new classes rather than check if they exist. This skill teaches:
- Exploration workflow: `list_packages` → `list_classes` → `get_class_source` → `get_class_comment`
- Finding existing collaborators: `search_implementors`, `search_references`
- Common Pharo idioms to recognize when reading code
- The rule: **always read before writing**

## 7. Add `debugging-in-pharo` skill

**File:** `.agent/skills/debugging-in-pharo/SKILL.md`

When tests fail unexpectedly, no skill currently guides Claude. Cover:
- How to read `run_class_test` failure output: `MessageNotUnderstood`, `AssertionFailure`, `Error`, `OCUndeclaredVariableNotice`, `KeyNotFound`
- Recovery recipes for each failure type
- Using `eval` to inspect state: `SomeClass new printString`
- Common failure patterns and their causes

---

## Final File Tree

```
.
├── CLAUDE.md                                   <- NEW (minimal bootstrap)
├── .claude/
│   └── settings.local.json                     <- UPDATE (16 more tools auto-permitted)
├── .mcp.json                                   <- unchanged
└── .agent/skills/
    ├── create-smalltalk-code/SKILL.md          <- UPDATE (fix setter, add class-side, fix tool name)
    ├── smalltalk-conventions/SKILL.md          <- UPDATE (complete testing section, add printOn:)
    ├── test-driven-development/SKILL.md        <- UPDATE (fix tool name, add run_package_test, fix diagram)
    ├── explore-pharo-image/SKILL.md            <- NEW
    └── debugging-in-pharo/SKILL.md             <- NEW
```
