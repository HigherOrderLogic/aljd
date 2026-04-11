# Multiline String Correctness

## Status
`in-progress`

## Priority
`P0-critical`

## Context

Alejandra's multiline string handling (`'' ... ''`) can modify the *semantic content* of strings,
not just their formatting. This is the most severe class of bug a formatter can have — the formatted
code behaves differently from the original. Users who discover this lose trust immediately.

**Issues**:
- [#409](https://github.com/kamadorueda/alejandra/issues/409) — alejandra modifies logical contents of multiline strings
- [#442](https://github.com/kamadorueda/alejandra/issues/442) — string escapes like `''${1+x}` should not be touched

## Scope

**In scope:**
- Audit `src/alejandra/src/rules/string.rs` for content-modifying behavior
- Fix any case where string semantics change after formatting
- Add regression tests for known-broken cases from issues

**Out of scope:**
- Indentation style within strings (cosmetic, not semantic) — separate concern
- String interpolation formatting improvements

## Implementation

### Architecture Understanding

The string rule (`src/alejandra/src/rules/string.rs`) handles two cases:
1. **Double-quoted strings** (`"..."`): Children are iterated and formatted. Straightforward.
2. **Indented strings** (`''...''`): Complex processing:
   - Interpolations (`${...}`) are replaced with a placeholder hash
   - Content is split by newlines
   - Lines are trimmed of trailing whitespace (when last line is whitespace-only)
   - Minimum indentation is computed and stripped ("dedent")
   - New indentation is applied based on `build_ctx.indentation` and config
   - Placeholders are replaced back with formatted interpolations

**Root causes of semantic changes:**

1. **Trailing whitespace trimming** (lines 50-62): The `should_trim_end` logic trims trailing
   whitespace from ALL lines if the last line is whitespace-only. This changes string content
   if trailing spaces are semantically significant.

2. **Dedent/re-indent** (lines 64-113): The algorithm strips the minimum common indentation
   then applies new indentation. For strings where indentation is meaningful (e.g., Makefiles,
   Python snippets, heredocs), this corrupts content.

3. **String escape handling**: Escape sequences like `''${...}` and `'''` inside indented strings
   may be mishandled during the split/rejoin process.

### Step 1: Add semantic preservation tests

**Directory**: `src/alejandra/tests/cases/default/string_semantic/`

Create `in.nix` and `out.nix` with test cases from the issues:

```nix
{
  # Trailing whitespace in multiline string should be preserved
  # when it's semantically significant
  makefileContent = ''
    target:
    \techo "hello"
  '';

  # String escapes must not be modified
  escaped = ''
    ''${1+x}
    '''
    ''''
  '';

  # Interpolation positions must be preserved exactly
  withInterp = ''
    before ${expr} after
    ${"complex"}
  '';
}
```

### Step 2: Evaluate current behavior

Run the test and capture actual output. Compare the Nix evaluation of `in.nix` and `out.nix`
to detect any semantic differences:

```bash
nix eval --file tests/cases/default/string_semantic/in.nix
nix eval --file tests/cases/default/string_semantic/out.nix
# Outputs must be identical
```

### Step 3: Fix the string rule

**File**: `src/alejandra/src/rules/string.rs`

Key fixes needed:

**3a. Trailing whitespace**: The `should_trim_end` heuristic (line 50-51) should ONLY trim
trailing whitespace that Nix's `''` string semantics would already strip. Nix's indented strings
strip the common leading indentation, but trailing whitespace on lines is preserved.
The formatter should not strip trailing whitespace from string content at all — only
surrounding formatting whitespace.

**3b. Dedent safety**: The dedent algorithm must match Nix's own indentation stripping exactly.
Nix strips the minimum indentation from all non-empty lines (excluding the first).
Alejandra must do the same — no more, no less.

**3c. Re-indent correctness**: When re-applying indentation (lines 92-113), the indentation
added must exactly correspond to `build_ctx.indentation` levels so that the Nix evaluator
sees the same final string value.

### Step 4: Add evaluation-based test

Add a test that evaluates both `in.nix` and `out.nix` with Nix and asserts identical results.
This could be a separate integration test or a shell script in CI:

```bash
#!/usr/bin/env bash
set -e
for case in tests/cases/default/string_semantic; do
  before=$(nix eval --raw --file "$case/in.nix" 2>/dev/null || echo "SKIP")
  after=$(nix eval --raw --file "$case/out.nix" 2>/dev/null || echo "SKIP")
  if [ "$before" != "$after" ] && [ "$before" != "SKIP" ]; then
    echo "SEMANTIC CHANGE in $case"
    diff <(echo "$before") <(echo "$after")
    exit 1
  fi
done
```

## Branch
`fix/multiline-string-correctness`

## Verification

1. `cargo test` — all tests pass
2. New `string_semantic` test case passes
3. Nix evaluation of `in.nix` and `out.nix` produces identical values
4. Idempotency: formatting the output file again produces no changes
5. Existing string test cases still pass (no regressions)

## Current Implementation Status (git diff analysis)

### What has been changed so far

The diff to `string.rs` made three changes:

1. **Replaced `!portions[0].is_empty()` with `!portions[0].trim().is_empty()`** — Separates
   "has content" from "is last line" into distinct branches. The intent is to avoid adding
   `content_pad` before the last whitespace-only line (which corresponds to where the closing
   `''` lands). This is the right direction.

2. **Added `should_add_pad` guard for interpolation lines** — Same logic: don't pad the last
   line if it's whitespace-only and contains interpolations. Correct intent.

3. **Changed newline emission to `index + 1 < lines.len() || has_content`** — Emits a trailing
   newline after the last line if it has content. This is needed so that the closing `''` appears
   on its own line after a content-bearing last line.

### What is still broken

**Problem 1: `has_content` uses `trim()` which drops semantically significant whitespace lines**

Lines in the middle of a string that consist only of spaces (e.g., intentional blank lines with
trailing spaces, which Nix preserves) will hit the `!is_last_line` branch and emit `portions[0]`
unchanged — which is correct for those. But the `trim()` check means a line like `"  "` (two
spaces, not empty after dedent) is treated as "no content" and falls through to the empty-line
branches. For a middle line, it emits `portions[0]` (the spaces) — OK. For the last line, it
emits `String::new()` — stripping those spaces. This is a semantic change if the last content
line (before the `''`) had trailing spaces.

**Problem 2: Double indentation of content lines**

The re-indent step (lines 98-118) already prepends `string_content_indent_unit` to every
non-empty line. Then, at lines 133-143, `content_pad` (= `indent_unit.repeat(build_ctx.indentation)`)
is prepended again. This means content gets:
```
content_pad + indent_unit + original_dedented_content
```
But what Nix sees as the string value after stripping the closing `''`'s indentation is:
```
indent_unit + original_dedented_content
```
This is only correct if `content_pad` equals the indentation that Nix strips (i.e., the closing
`''`'s column). For this to work, the closing `''` must be at column `len(content_pad)`, not at
column `len(content_pad) + len(indent_unit)`. The closing `''` must sit at the outer indentation
(`content_pad` level), not at the content indentation level.

### Key insight: closing `''` placement is cosmetic, not semantic

The closing `''` of a Nix indented string determines the indentation Nix strips from all lines.
Its position does not affect the string's value *directly* — only what gets stripped from content
lines does. So:

```nix
# These are semantically equivalent:
x = '' a
    '';        # '' at outer indent, content gets "a\n" after stripping 4 spaces

x = ''  a
      '';      # '' indented further, content still "a\n" after stripping 6 spaces
```

Both forms are valid and produce the same value **as long as** the content indentation and
closing `''` indentation are consistent. The formatter's goal should be:

- Place the closing `''` at **outer indentation** (no indent unit added — just `content_pad`
  which equals `indent_unit.repeat(build_ctx.indentation)`)
- Place content lines at **outer indentation + one indent unit**
- Nix then strips `outer indentation` from all lines, leaving `indent_unit + content`

This is the correct mental model. The closing `''` dedenting to outer indentation is **cosmetic**
(it makes the code more readable) but it also ensures the semantic stripping is correct.

### Concrete correct behavior

For a string like `'' a\n''` appearing at indentation level 1 (one indent unit = two spaces):

```
''
    a       ← outer_indent(2 spaces) + indent_unit(2 spaces) + "a"
  ''        ← outer_indent(2 spaces) only — no indent unit
```

Nix strips 2 spaces (the closing `''`'s column), so the value is `"  a\n"` — one indent unit
worth of leading spaces on the content, then a newline. That matches the original.

The current implementation adds `content_pad` (2 spaces) to content lines in addition to the
already-prepended `indent_unit` (2 spaces), yielding 4 spaces before `a`. Then the closing `''`
must also be at 4 spaces for semantics to hold — but if it's placed at 2 spaces (outer indent),
Nix only strips 2 spaces, leaving `"    a\n"` (4 spaces) instead of `"  a\n"` (2 spaces).
This is a semantic corruption.

### Required fix

The re-indent step (lines 98-118) should be removed or reworked. The correct approach is:

1. Dedent all lines by the minimum indentation (as currently done).
2. Re-indent content lines to `indent_unit.repeat(build_ctx.indentation + 1)` — outer + one level.
3. Emit the closing `''` at `indent_unit.repeat(build_ctx.indentation)` — outer level only,
   with no extra indent unit. Do not emit any token before the closing `''` token itself.
4. The newline before the closing `''` comes from the last line's newline emission (step 3 in
   the current diff is correct).

The `content_pad` added at lines 133-143 should equal `indent_unit.repeat(build_ctx.indentation)`
(outer indentation) and the `indent_unit` from the re-indent step together equal
`indent_unit.repeat(build_ctx.indentation + 1)`. The re-indent step + content_pad approach is
functionally equivalent to just doing it in one place — the bug is that the re-indent step adds
ONE `indent_unit` but `content_pad` adds `build_ctx.indentation` copies, making total indentation
`(build_ctx.indentation + 1)` units, which is correct for content BUT the closing `''` is being
emitted without any pad token, landing at column 0 rather than `build_ctx.indentation` units.

**The actual bug**: the closing `''` (the last element, `elements[elements.len() - 1]`) is
emitted by the caller after the `rule()` function returns its steps — or it falls through
somewhere without a pad. Verify where the closing `''` token is emitted and ensure a
`content_pad` (= `indent_unit.repeat(build_ctx.indentation)`) token is emitted before it.

## Dependencies
None. (But plan 003-idempotency's test harness improvement will help catch regressions.)