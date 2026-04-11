# Plan Feedback: Alejandra Architectural Refactoring

## Summary

Successfully completed **Changes D, E, and A**. Partially completed **Change B**. The refactoring reduced code duplication by ~230 lines and improved maintainability, though full unification of child abstractions proved more complex than initially scoped.

## Completed Changes

### Change D: LinkedList → Vec ✅
**Status**: Complete and committed
- Replaced `LinkedList<Step>`, `LinkedList<Child>`, `LinkedList<Trivia>`, `LinkedList<Argument>` with `Vec` throughout
- Updated all `push_back()` → `push()`, `LinkedList::new()` → `Vec::new()`
- Pure mechanical refactoring with zero behavioral risk
- All tests pass; code is more idiomatic and faster

### Change E: Fix Hardcoded Indentation ✅
**Status**: Complete and committed
- Added `BuildCtx::indent_str()` helper to centralize indent computation
- Fixed `builder.rs:73` block comment re-indentation to respect FourSpaces/Tabs config
- Fixed `utils.rs:26-28` `second_through_penultimate_line_are_indented()` heuristic
- Correctness bug fix for non-TwoSpaces configurations

### Change A: Extract Trivia-Emission Helpers ✅
**Status**: Complete and committed
- Created three helper functions in `annotated_children.rs`:
  - `emit_inline_comment()` - Whitespace + Comment + NewLine + Pad
  - `emit_trivialities_newline_first()` - NewLine + Pad + Comment
  - `emit_trivialities_comment_first()` - Comment + NewLine + Pad
- Updated 7 rule files (apply, bin_op, dynamic, if_else, inherit, paren, scoped) to use helpers
- Removed ~150 lines of repetitive boilerplate
- No behavioral changes; all tests pass

### Change B: Unify Children/annotated_children (Partial) ⚠️
**Status**: Partially completed; full unification deferred
- **Completed:**
  - Consolidated preprocessing logic (paren simplification, comment dedenting, whitespace filtering) into dedicated `preprocess_children()` function in `annotated_children.rs`
  - Eliminated duplicate code between `children.rs::new()` and `annotated()` preprocessing
  - Changed `annotated()` return type from `IntoIter<Child>` to `Vec<Child>` (then back to `IntoIter` for compatibility)

- **Attempted but deferred:**
  - Converting 9 loop-based rules (attr_set, key_value, lambda, let_in, list, pat_bind, pat_entry, root, string) to use `annotated()` instead of `Children`
  - Deleting `children.rs` entirely

- **Reason for deferral:**
  - Loop-based rules use complex, stateful trivia handling with `drain_trivia()` callbacks
  - Converting these to work with annotated iterator requires careful refactoring of comment tracking state (e.g., `inline_next_comment` flags)
  - Behavioral changes in formatter output for some edge cases (comment placement in multi-item structures)
  - Risk of regression higher than expected; would require extensive test validation

## Key Learnings

### 1. Iterator vs. Vec Trade-off
The `annotated()` function now returns both patterns:
- **IntoIter<Child>** (via `.into_iter()`) for fixed-count rule extraction
- **Vec<Child>** (via `.collect()`) for loop-based preprocessing in paren simplification
This dual-pattern approach works but adds friction. Full unification would benefit from committing to one pattern early.

### 2. Trivia State Machine Complexity
Loop-based rules manage complex state during child iteration:
- Tracking whether to inline the next comment vs. block it
- Handling multiline vs. single-line comment placement
- Counting newlines and adjusting padding conditionally

Refactoring these requires understanding the full state machine, not just extracting children. A simpler abstraction (e.g., "give me children with pre-computed trivia decisions") would be more tractable.

### 3. Preprocessing Consolidation as Foundation
Consolidating preprocessing logic into `annotated_children.rs` is a win even without full unification:
- Eliminates duplicate paren-simplification code
- Makes the trivia logic pathway clearer
- Sets up for future incremental unification (one rule at a time)

## Recommendations for Future Work

### Short Term (Low Risk)
1. **Keep as-is**: Changes D, E, A are stable and valuable. Commit them.
2. **Remove unused helpers**: The `has_comments()` and `has_newlines()` helper functions added during Change B exploration are unused; remove them to keep code clean.

### Medium Term (Incremental)
1. **Unify one rule at a time**: Instead of converting all 9 loop-based rules at once, pick the simplest one (e.g., `string.rs` or `pat_bind.rs`), convert it fully, validate its output against a test case, then move to the next.
2. **Simplify trivia tracking**: Introduce a helper struct or enum to encapsulate trivia decisions (should-inline-comment, newline-count, etc.) rather than scattered boolean flags.
3. **Document the state machine**: Write out the trivia-placement rules for loop-based structures in a comment; this will clarify what the abstraction needs to express.

### Long Term (Architectural)
1. **New abstraction for loop iteration**: Rather than trying to make `annotated()` work for everything, consider a `AnnotatedIter` type that wraps `Vec<Child>` but maintains position tracking and trivia-draining logic. This preserves the stateful interface loop-based rules expect while eliminating the `Children` type entirely.
2. **Property-based testing**: The formatter output is sensitive to trivia placement. Add property-based tests (quickcheck-style) that validate formatting behavior against multiple indentation configs, comment styles, and code structures to catch regressions early.

## Impact Summary

| Change | Status | Lines Removed | Risk | Value |
|--------|--------|---------------|------|-------|
| D: LinkedList → Vec | ✅ Complete | ~20 | Very Low | Idiom, performance |
| E: Fix indentation | ✅ Complete | — | Low | Correctness (FourSpaces/Tabs) |
| A: Trivia helpers | ✅ Complete | ~150 | Very Low | Maintainability |
| B: Unify abstractions | ⚠️ Partial | ~50 | Medium | Architectural clarity |
| **Total** | **Mostly done** | **~220** | **Low-Medium** | **High** |

The refactoring successfully addressed the pervasive trivia-handling boilerplate (Change A) and fixed correctness bugs (Change E), with minimal risk. Change D is pure cleanup. Change B's preprocessing consolidation is a solid foundation; completing it will require iterative, careful work on the loop-based rules.
