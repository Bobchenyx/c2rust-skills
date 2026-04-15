---
name: c2rust-verify
description: Verify converted Rust code against behavioral tests and quality checks. Runs test suite, compares with C implementation, checks code quality, and generates verification report. Use after c2rust-refine, or when user mentions "verify", "validate", "check conversion", "run tests".
argument-hint: [module-name|--all] [--quick|--full]
allowed-tools: [Read, Bash, Glob, Grep, Write, Agent]
---

# Conversion Verification

Verify that the converted Rust code is correct, safe, and meets quality standards.

## Arguments

The user invoked this with: $ARGUMENTS

- `module-name`: Verify a specific module
- `--all`: Verify all refined modules
- `--quick`: Only run tests, skip deep analysis
- `--full`: Full verification with agent-assisted review (default)

## Prerequisites

Read `c2rust-manifest.toml` — refinement must be completed for target modules.

```bash
cat c2rust-manifest.toml 2>/dev/null
```

---

## Step 1: Compilation Check

Verify clean compilation:

```bash
cd <target_dir>

# Check compilation
cargo check 2>&1
echo "Exit code: $?"

# Check for warnings
cargo check 2>&1 | grep -c 'warning'
```

If compilation fails, stop and advise running `/c2rust-refine` first.

---

## Step 2: Run Test Suite

### Run unit tests
```bash
cargo test 2>&1 | tee /tmp/test-results.txt

echo "=== Test Summary ==="
grep -E 'test result:|running [0-9]+ test' /tmp/test-results.txt
```

### Run integration tests (from c2rust-test phase)
```bash
cargo test --test '*' 2>&1 | tee /tmp/integration-results.txt
```

### Run with golden data comparison
If golden data exists:
```bash
# Run test binary with golden data comparison
cargo test golden 2>&1
```

Record:
- Total tests run
- Tests passed
- Tests failed (with details)
- Tests skipped

---

## Step 3: Clippy Analysis

```bash
cargo clippy -- -W clippy::all -W clippy::pedantic 2>&1 | tee /tmp/clippy-results.txt

echo "=== Clippy Summary ==="
echo "Warnings: $(grep -c 'warning:' /tmp/clippy-results.txt)"
echo "Errors: $(grep -c 'error:' /tmp/clippy-results.txt)"

# Show top issues
grep 'warning:' /tmp/clippy-results.txt | sed 's/.*warning: //' | sort | uniq -c | sort -rn | head -10
```

---

## Step 4: Unsafe Audit

```bash
echo "=== Unsafe Usage ==="

# Count unsafe blocks
echo "Unsafe blocks: $(grep -rn 'unsafe\s*{' --include='*.rs' | grep -v test | grep -v '// ' | wc -l)"

# Count unsafe functions
echo "Unsafe fns: $(grep -rn 'unsafe\s\+fn' --include='*.rs' | grep -v test | wc -l)"

# List all unsafe usages with context
grep -rn 'unsafe' --include='*.rs' | grep -v test | grep -v '// '
```

Categorize each unsafe usage:
- **FFI boundary** — Necessary for calling C code
- **Raw pointer deref** — Could potentially be made safe
- **Static mut access** — Should be replaced with safe alternatives
- **Justified** — Has `// SAFETY:` comment explaining why

---

## Step 5: Miri Check (optional, for UB detection)

```bash
# Only if miri is installed
if rustup component list --installed | grep -q miri; then
    echo "Running Miri for undefined behavior detection..."
    cargo +nightly miri test 2>&1 | tee /tmp/miri-results.txt
    
    if grep -q 'error' /tmp/miri-results.txt; then
        echo "WARNING: Miri found potential undefined behavior!"
        grep 'error' /tmp/miri-results.txt
    else
        echo "Miri: No undefined behavior detected"
    fi
else
    echo "Miri not installed. Skipping UB detection."
    echo "Install with: rustup +nightly component add miri"
fi
```

---

## Step 6: Code Quality Review (--full mode)

Launch `rust-reviewer` agent for deep code quality analysis:

**Agent tasks**:
1. Review all converted Rust code for idiomatic patterns
2. Identify any C-isms that survived translation (C-style loops, manual null checks, etc.)
3. Check error handling consistency across modules
4. Verify public API design is clean and ergonomic
5. Assess documentation coverage on public items
6. Check for performance anti-patterns (unnecessary allocations, missing `with_capacity`, etc.)

---

## Step 7: Behavioral Comparison

If the original C code can still be compiled and run:

```bash
# Build C version
make -C <original_c_dir> 2>/dev/null

# Build Rust version
cargo build --release

# Compare outputs for known inputs
echo "=== Behavioral Comparison ==="
for input in test_inputs/*; do
    c_output=$(<original_c_dir>/program < "$input" 2>&1)
    rust_output=$(target/release/program < "$input" 2>&1)
    
    if [ "$c_output" = "$rust_output" ]; then
        echo "PASS: $(basename $input)"
    else
        echo "FAIL: $(basename $input)"
        diff <(echo "$c_output") <(echo "$rust_output")
    fi
done
```

---

## Step 8: Generate Verification Report

Write `c2rust-verification-report.md`:

```markdown
# Conversion Verification Report

## Summary
- **Date**: [date]
- **Project**: [name]
- **Modules verified**: [list]
- **Overall status**: PASS / PARTIAL / FAIL

## Compilation
- Status: OK / WARNINGS / ERRORS
- Warnings: [count]
- Errors: [count]

## Test Results
| Category | Passed | Failed | Skipped | Total |
|----------|--------|--------|---------|-------|
| Unit tests | X | X | X | X |
| Integration | X | X | X | X |
| Golden data | X | X | X | X |
| Property | X | X | X | X |
| **Total** | **X** | **X** | **X** | **X** |

### Failed Tests
[Details of each failed test with error message]

## Code Quality
- Clippy warnings: [count]
- Unsafe blocks: [count]
  - FFI boundary: [count]
  - Raw pointer: [count]
  - Justified: [count]
  - Needs attention: [count]
- SAFETY comments: [count present] / [count needed]

## Miri Analysis
- Status: CLEAN / ISSUES FOUND / NOT RUN
- [Details if issues found]

## Behavioral Equivalence
- Test cases compared: [count]
- Matches: [count]
- Differences: [count]
[Details of any behavioral differences]

## Code Review Findings
[Summary from rust-reviewer agent]

### Critical Issues
[Any issues that must be fixed]

### Recommendations
[Improvements that could be made]

## Remaining Work
- [ ] [List any remaining items]

## Metrics
- Original C LOC: [count]
- Converted Rust LOC: [count]
- Unsafe blocks: [count]
- Test coverage: [estimate]
```

---

## Output

### 1. Verification Report
`c2rust-verification-report.md` with full analysis.

### 2. Manifest Update
```toml
[verification]
status = "completed"
tests_passed = 38
tests_failed = 2
tests_total = 40
report_path = "c2rust-verification-report.md"
```

Update module statuses:
```toml
[[modules]]
name = "utils"
status = "verified"
```

### 3. Next Steps
Based on results, recommend:
- If all pass: Conversion complete! Consider further optimization.
- If some fail: Identify root causes, suggest running `/c2rust-refine` on specific issues.
- If behavioral differences: Flag for investigation.
