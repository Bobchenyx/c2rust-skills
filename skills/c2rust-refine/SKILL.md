---
name: c2rust-refine
description: Fix compilation errors and transform translated Rust code into idiomatic safe Rust. Uses smart hybrid approach — auto-fixes mechanical errors, pauses for design decisions. Use after c2rust-convert, or when user mentions "fix errors", "refine rust", "make idiomatic", "fix compilation", "debug rust code".
argument-hint: [module-name|--all] [--auto|--interactive]
allowed-tools: [Read, Bash, Glob, Grep, Write, Edit, Agent]
---

# Compilation Debug & Idiomatic Refinement

Fix compilation errors in translated Rust code and progressively improve it toward idiomatic, safe Rust.

## Arguments

The user invoked this with: $ARGUMENTS

- `module-name`: Refine a specific module
- `--all`: Refine all converted modules
- `--auto`: Only do mechanical auto-fixes, skip semantic decisions
- `--interactive`: Pause for every change, even mechanical ones
- Default: Smart hybrid mode

## Prerequisites

Read `c2rust-manifest.toml` — conversion must be completed for target modules.

```bash
cat c2rust-manifest.toml 2>/dev/null
```

---

## Overview: Smart Hybrid Workflow

```
Pre-check: Is code already clean?
  ↓ yes → Skip to summary
  ↓ no  → Phase A
Phase A: Mechanical Auto-Fix Loop
  ↓ (compilation errors reach 0 or only semantic issues remain)
Phase B: Semantic Issue Consultation
  ↓ (all design decisions made)
Phase C: Idiomatic Refinement
  ↓ (clippy clean, unsafe minimized)
Done
```

---

## Pre-check: Skip If Clean

Claude-based translation often produces code that compiles and passes clippy on the first try. Check before entering the fix loop:

```bash
cd <target_dir>

# Quick health check
cargo check 2>&1 | tail -5
errors=$(cargo check 2>&1 | grep -c '^error')
warnings=$(cargo clippy -- -W clippy::all 2>&1 | grep -c '^warning')
unsafe_count=$(grep -rn --exclude-dir=tests --exclude-dir=benches 'unsafe' src/ --include='*.rs' | grep -v '// ' | wc -l)

echo "Errors: $errors | Clippy warnings: $warnings | Unsafe blocks: $unsafe_count"
```

**If errors=0 AND warnings=0 AND unsafe_count=0**: The code is already clean. Skip directly to output — update the manifest and report "no refinement needed." This is expected for well-structured C code translated by Claude.

**If errors=0 but warnings>0 or unsafe_count>0**: Skip Phase A/B, go directly to Phase C (idiomatic refinement).

**If errors>0**: Proceed to Phase A.

---

## Phase A: Mechanical Error Auto-Fix Loop

**Goal**: Get the code to compile by fixing errors that have clear, unambiguous solutions.

### Iteration Loop (max 20 iterations):

1. **Run cargo check, capture all errors**:
```bash
cd <target_dir>
cargo check 2>&1 | tee /tmp/cargo-errors.txt

# Summary
echo "=== Error Summary ==="
grep '^error\[E' /tmp/cargo-errors.txt | sed 's/error\[E\([0-9]*\)\].*/E\1/' | sort | uniq -c | sort -rn
echo "Total errors: $(grep -c '^error' /tmp/cargo-errors.txt)"
```

2. **Classify each error**:

**MECHANICAL (auto-fix)**:
| Error Code | Description | Fix Strategy |
|-----------|-------------|-------------|
| E0432 | Unresolved import | Fix `use` path |
| E0308 | Mismatched types | Add `as` cast or `.into()` |
| E0425 | Cannot find value/fn | Add `use`, `mod`, or `extern` |
| E0412 | Cannot find type | Add `use libc::*` or type alias |
| E0277 | Trait not satisfied | Add `#[derive(...)]` |
| E0382 | Use of moved value | Add `.clone()` or `Copy` derive |
| E0061 | Wrong arg count | Fix function signature |
| E0599 | No method on type | Add explicit type cast |
| E0658 | Unstable feature | Replace with stable alternative |
| E0463 | Can't find crate | Add to Cargo.toml dependencies |
| E0433 | Failed to resolve | Fix module path |

**STRUCTURAL (partially auto-fixable)**:
| Error Code | Description | Fix Strategy |
|-----------|-------------|-------------|
| E0499 | Multiple mut borrows | Split borrows or use indices |
| E0502 | Borrow conflict | Clone or restructure |
| E0106 | Missing lifetime | Add lifetime annotations |
| E0515 | Return ref to local | Return owned data |
| E0507 | Move out of borrowed | Clone or restructure |

**SEMANTIC (require user input)**:
| Error Code | Description | Requires |
|-----------|-------------|----------|
| Complex E0499 | Fundamental aliasing | Ownership redesign |
| Complex E0106 | Non-trivial lifetimes | Lifetime design decision |
| E0277 (complex) | Custom trait impl | Design decision |
| Multiple related | Interconnected errors | Architectural decision |

3. **Apply mechanical fixes**:

Read the error-fix-catalog reference: [references/error-fix-catalog.md](references/error-fix-catalog.md)

For each mechanical error:
- Read the file and error location
- Apply the fix (edit the file)
- Move to next error

4. **Re-check compilation**:
```bash
cargo check 2>&1 | grep -c '^error'
```

5. **Loop decision**:
- If 0 errors → proceed to Phase B (or C if no semantic issues)
- If errors decreased → continue loop (next iteration)
- If errors unchanged for 3 iterations → stop loop, report remaining errors
- If max iterations reached → stop, report remaining

### Common Batch Fixes

For efficiency, check these common issues before the iteration loop:

```bash
# Verify all dependencies in Cargo.toml match what the translated code uses
cargo check 2>&1 | grep 'can.t find crate' | sort -u

# Check for any remaining libc type usage that should be native Rust types
grep -rn 'c_int\|c_char\|c_void' --include='*.rs' | grep -v 'ffi' | head -20
```

---

## Phase B: Semantic Issue Consultation

**Goal**: Resolve design decisions that require human judgment.

For each semantic issue found during Phase A or remaining after it:

### Present the issue to the user:

```
## Design Decision Needed: [Issue Title]

**File**: path/to/file.rs:42
**Error**: E0499 — cannot borrow `self.data` as mutable because it is also borrowed as immutable

**Context**: 
The original C code accesses `self.data` through two pointers simultaneously.
In Rust, this violates borrowing rules.

**Options**:

1. **Split into separate fields** — Restructure the data so the two accesses target different fields
   - Pro: Zero-cost, safe
   - Con: Requires API changes

2. **Use RefCell for interior mutability** — Wrap the field in `RefCell<T>`
   - Pro: Minimal code changes
   - Con: Runtime borrow checking overhead, panics on misuse

3. **Use index-based access** — Store index instead of reference, access through the container
   - Pro: Safe, no runtime overhead
   - Con: Less ergonomic

**Recommendation**: Option 1 if feasible, Option 2 as fallback
```

Wait for user's choice, then apply it.

### Common semantic decisions:

1. **Error handling strategy**: Present options from error-fix-catalog
2. **Ownership model**: Box vs Rc vs Arc for shared data
3. **Global state**: Mutex vs explicit state passing
4. **Container choice**: Vec vs HashMap vs custom
5. **Unsafe justification**: Keep unsafe with comment vs restructure

---

## Phase C: Idiomatic Refinement

**Goal**: Transform compiling but C-like Rust into idiomatic Rust. Only proceed after code compiles.

Reference: [references/unsafe-to-safe.md](references/unsafe-to-safe.md)

### Refinement passes (in order):

#### Pass 1: Remove unnecessary unsafe blocks
```bash
# Find all unsafe blocks
grep -rn 'unsafe\s*{' --include='*.rs' | head -50
```
For each unsafe block, check if the contents actually require unsafe. If not, remove the `unsafe` wrapper.

#### Pass 2: Replace raw pointers with references
Where pointer is known non-null and lifetime is clear, replace `*mut T` / `*const T` with `&mut T` / `&T`.

#### Pass 3: Replace manual memory management
- `malloc` → `Box::new()` / `Vec::with_capacity()`
- `free` → let RAII handle it (remove explicit free)
- `realloc` → `Vec::resize()`

#### Pass 4: Replace C string handling
- `*mut c_char` → `String` / `&str` (internal)
- Keep `CString` / `CStr` only at FFI boundaries

#### Pass 5: Replace null checks with Option
- `if ptr.is_null() { return -1; }` → `Option<&T>` + `?` operator

#### Pass 6: Replace error codes with Result
- `return -1;` → `return Err(Error::...);`
- `return 0;` → `return Ok(result);`

#### Pass 7: Run clippy and apply suggestions
```bash
cargo clippy -- -W clippy::all 2>&1 | head -100

# Auto-fix what clippy can
cargo clippy --fix --allow-dirty -- -W clippy::all
```

#### Pass 8: Final unsafe audit
```bash
# Count remaining unsafe
echo "Unsafe blocks remaining:"
grep -rn 'unsafe' --include='*.rs' | grep -v '// ' | grep -v 'test' | wc -l

# List each for documentation
grep -rn 'unsafe' --include='*.rs' | grep -v '// ' | grep -v 'test'
```

For each remaining unsafe block, add a `// SAFETY: ` comment explaining why it's necessary.

---

## Output

### 1. Refined Rust Code
All modified .rs files with compilation fixes and idiomatic improvements.

### 2. Refinement Log
Summary of changes made:
- Mechanical fixes applied (count by error type)
- Semantic decisions made (list)
- Idiomatic improvements (list)
- Remaining unsafe blocks (count + justification)

### 3. Manifest Update

```toml
[refinement]
status = "completed"
iteration_count = 7
errors_remaining = 0
unsafe_blocks_remaining = 12
```

Update module statuses:
```toml
[[modules]]
name = "utils"
status = "refined"
```

### 4. Next Step
Recommend running `/c2rust-verify` to validate correctness.
