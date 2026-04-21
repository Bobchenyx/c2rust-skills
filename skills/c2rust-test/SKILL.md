---
name: c2rust-test
description: Build behavioral test suite for C-to-Rust conversion verification. Creates tests that capture C code behavior before conversion, enabling correctness validation of the Rust output. Use after c2rust-plan, or when user mentions "build tests", "test suite", "behavioral tests", "verification tests".
argument-hint: [module-name|--all]
allowed-tools: [Read, Bash, Glob, Grep, Write]
---

# Behavioral Test Suite Builder

Build a test suite that captures the behavior of the C codebase BEFORE conversion, so we can verify the Rust code behaves identically.

## Arguments

The user invoked this with: $ARGUMENTS

- `module-name`: Build tests for a specific module
- `--all`: Build tests for all modules
- No argument: Build tests for the next untested module in conversion order

## Prerequisites

Read `c2rust-manifest.toml` — the plan must be completed first.

```bash
cat c2rust-manifest.toml 2>/dev/null
```

---

## Step 1: Discover Existing Tests

Scan the C project for existing test infrastructure:

```bash
# Look for test directories
find . -type d -name 'test*' -o -name 'tests' -o -name 'check' -o -name 'spec' 2>/dev/null

# Look for test files
find . -name '*test*.[ch]' -o -name '*check*.[ch]' -o -name '*spec*.[ch]' 2>/dev/null

# Detect test frameworks
grep -rl 'CU_\|START_TEST\|TEST_F\|cmocka\|unity\|mu_assert\|assert_true' --include='*.c' --include='*.h' 2>/dev/null

# Check for test targets in build system
grep -i 'test\|check' Makefile CMakeLists.txt 2>/dev/null | head -20
```

If existing tests are found:
1. Document what they test
2. Try to run them: `make test` or `ctest` or `make check`
3. Record which tests pass — these become our regression baseline

---

## Step 2: Identify Public API Functions

For each target module, identify the functions that form its public API:

```bash
# Functions declared in headers (public API)
grep -n '^[a-zA-Z_].*(' module_header.h | grep -v '//' | grep -v '#'

# Functions defined in source (check which are static = private)
grep -n '^[a-zA-Z_].*(' module.c | grep -v 'static ' | grep -v '//' | grep -v '#'

# Static functions (internal, test through public API)
grep -n '^static.*(' module.c | grep -v '//'
```

For each public function, analyze:
- **Input types**: What parameters does it take?
- **Output type**: What does it return?
- **Side effects**: Does it modify global state? Write to files? Print output?
- **Error conditions**: What error codes/behaviors exist?
- **Edge cases**: NULL inputs, empty strings, zero values, overflow

---

## Step 3: Build Integration Tests

For each public API function, create integration tests that:

### A. Behavioral Equivalence Tests

Test that the function produces the same output for the same input in both C and Rust.

Use the template from [templates/integration-test.rs](templates/integration-test.rs):

```rust
#[test]
fn test_function_name_basic() {
    // Test basic/happy path
    let result = module::function_name(input);
    assert_eq!(result, expected_output);
}

#[test]
fn test_function_name_edge_cases() {
    // Test edge cases: empty input, max values, etc.
    assert_eq!(module::function_name(""), expected_empty);
    assert_eq!(module::function_name(i32::MAX), expected_max);
}

#[test]
fn test_function_name_error_cases() {
    // Test error conditions
    assert!(module::function_name(invalid_input).is_err());
}
```

### B. Golden Output Tests

For functions with complex output, capture the C version's output as golden data:

```bash
# Compile and run C code to generate golden output
gcc -o test_runner test_golden.c module.c -lm
./test_runner > golden/module_output.txt
```

```rust
#[test]
fn test_module_golden_output() {
    let result = module::process(test_input);
    let golden = include_str!("../golden/module_output.txt");
    assert_eq!(result, golden);
}
```

### C. FFI Boundary Tests (for incremental conversion)

Use the template from [templates/ffi-test.rs](templates/ffi-test.rs):

```rust
#[test]
fn test_ffi_round_trip() {
    // Call Rust → C → Rust and verify data integrity
    let input = TestData::new();
    let c_result = unsafe { ffi::c_process(input.as_ptr()) };
    let rust_result = process(&input);
    assert_eq!(c_result, rust_result);
}
```

---

## Step 4: Property-Based Tests (for algorithmic code)

For functions with mathematical or algorithmic behavior, generate property-based tests:

```rust
use proptest::prelude::*;

proptest! {
    #[test]
    fn test_sort_is_sorted(mut v in prop::collection::vec(any::<i32>(), 0..1000)) {
        sort_function(&mut v);
        for window in v.windows(2) {
            assert!(window[0] <= window[1]);
        }
    }
    
    #[test]
    fn test_encode_decode_round_trip(data in prop::collection::vec(any::<u8>(), 0..10000)) {
        let encoded = encode(&data);
        let decoded = decode(&encoded);
        assert_eq!(data, decoded);
    }
}
```

---

## Step 5: Test Harness Setup

Create the test directory structure:

```
tests/
├── common/
│   ├── mod.rs              # Shared test utilities
│   └── golden_data/        # Golden output files from C version
│       ├── module_a.txt
│       └── module_b.bin
├── integration/
│   ├── test_module_a.rs    # Per-module integration tests
│   ├── test_module_b.rs
│   └── test_ffi.rs         # FFI boundary tests
└── fixtures/
    ├── sample_input.txt    # Test input data
    └── config.toml         # Test configuration
```

Add test dependencies to Cargo.toml:
```toml
[dev-dependencies]
proptest = "1.0"        # Property-based testing (optional)
tempfile = "3"          # Temporary files for I/O tests
assert_cmd = "2"        # CLI testing (if applicable)
```

---

## Step 6: Generate Golden Data

For each module with testable I/O:

1. Compile the C code with test driver
2. Run with known inputs
3. Capture output (stdout, files, return codes)
4. Store as golden data in `tests/common/golden_data/`

```bash
# Example: compile test driver
gcc -o golden_gen tests/golden_gen.c src/module.c -I include
./golden_gen > tests/common/golden_data/module_output.txt
echo $? > tests/common/golden_data/module_exit_code.txt
```

---

## Step 7: Identify Test Coverage Gaps

Not all C behavior can be captured by automated tests. Read `c2rust-assessment.md` (if it exists) and cross-reference each module's hard/blocking patterns against the list below.

### Patterns outside test coverage

| Pattern | Why It's Hard to Test | Mitigation |
|---------|----------------------|------------|
| **Global mutable state** | Tests may pass or fail depending on execution order; state leaks between tests | Note affected modules; recommend `#[serial_test::serial]` after conversion |
| **Signal handlers** (`signal`, `sigaction`) | Cannot reliably trigger and capture signal behavior in unit tests | Document expected signal behavior; verify manually |
| **Filesystem side effects** | Tests depend on filesystem state; non-deterministic across platforms | Use `tempfile` crate for isolation; note as partial coverage |
| **setjmp/longjmp** | Non-local control flow cannot be captured in input→output behavioral tests | Skip behavioral tests for these paths; verify via code review after conversion |
| **Thread-dependent behavior** | Race conditions and lock ordering are non-deterministic | Note as untestable; recommend `loom` or `shuttle` for post-conversion verification |
| **Hardware/platform-specific** (`ioctl`, inline asm, SIMD) | Behavior depends on specific hardware or OS | Document platform assumptions; test only on target platform |

### Per-module coverage flags

For each module, check whether it contains any of the above patterns (from the assessment's hard/blocking pattern list). If so, add a `coverage_gaps` note:

```
Module "core": 8 global mutable variables → test coverage is PARTIAL (execution-order-dependent state)
Module "network": signal handlers × 3 → test coverage EXCLUDES signal behavior
Module "crypto": inline asm × 2 → BLOCKING pattern, no behavioral tests generated
```

Include these flags in the test manifest output so users know exactly what the tests guarantee and what they don't.

---

## Output

### 1. Test Files

Write test files to the `tests/` directory following the structure above.

### 2. Test Manifest

Write a summary of all tests created, **including coverage gap flags**:
```markdown
# Test Manifest

| Module | Tests | Type | Coverage Gaps | Status |
|--------|-------|------|---------------|--------|
| utils | 12 | unit | none | ready |
| parser | 8 | integration + golden | none | ready |
| core | 15 | integration + property | global state (8 vars) — partial | ready |
| network | 5 | FFI boundary | signal handlers (3) — excluded | ready |
| crypto | 0 | — | inline asm — BLOCKING | skipped |
```

### 3. Manifest Update

Update `c2rust-manifest.toml`:
```toml
[tests]
status = "completed"
test_count = 40
test_dir = "tests/"
golden_data_dir = "tests/common/golden_data/"
```

Update each `[[modules]]` entry that had tests created:
```toml
[[modules]]
name = "utils"
status = "tested"
```
