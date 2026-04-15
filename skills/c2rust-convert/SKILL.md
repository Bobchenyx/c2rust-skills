---
name: c2rust-convert
description: Translate C source code to idiomatic Rust using Claude. Reads C source files and produces equivalent Rust code with proper ownership, error handling, and Rust idioms. Use after c2rust-plan, or when the user mentions "translate", "convert to rust", "rewrite in rust", "execute conversion".
argument-hint: <module-name|--all>
allowed-tools: [Read, Bash, Glob, Grep, Write, Edit, Agent]
---

# C-to-Rust Translation

Translate C source code to idiomatic Rust using the `c-to-rust-translator` agent (Claude Sonnet 4.6). This produces clean, safe Rust code directly — no mechanical transpilation, no unsafe-heavy output to clean up later.

## Arguments

The user invoked this with: $ARGUMENTS

- `module-name`: Convert a specific module
- `--all`: Convert all modules in planned order
- No argument: Convert the next unconverted module in planned order

## Prerequisites

1. Read `c2rust-manifest.toml` — plan must be completed
2. Verify Rust toolchain readiness (check `[toolchain].ready`)

```bash
cat c2rust-manifest.toml 2>/dev/null
```

If toolchain is not ready, prompt user to run `/c2rust-check-env` first.

---

## Step 1: Verify Toolchain

```bash
rustc --version
cargo --version
```

Only `rustc` and `cargo` are required. No need for `c2rust`, `clang`, `bear`, or `compile_commands.json`.

---

## Step 2: Prepare Target Directory

Set up the Rust project structure from the plan:

### For workspace:
```bash
mkdir -p <target_dir>
cd <target_dir>
cat > Cargo.toml << 'TOML'
[workspace]
resolver = "2"
members = [
    "crates/*",
]
TOML
mkdir -p crates
```

### For single crate:
```bash
mkdir -p <target_dir>
cd <target_dir>
cargo init --lib
```

Set up Cargo.toml with dependencies from the plan's dependency mapping:
```toml
[package]
name = "<project_name>"
version = "0.1.0"
edition = "2021"

[dependencies]
# Add mapped Rust crate dependencies from plan
# (many C stdlib deps map to Rust std and need no crate)

[dev-dependencies]
# For testing
```

---

## Step 3: Read and Understand C Source

Before launching translation agents, thoroughly read the C source to prepare context.

For each module in conversion order:

1. **Read the header file** — understand the public API:
```
Read the module's .h file(s) to extract:
- Public type definitions (structs, enums, typedefs)
- Public function declarations (the API surface)
- Macros that define constants or behavior
- Documentation comments
```

2. **Read the source file** — understand the implementation:
```
Read the module's .c file(s) to understand:
- Internal/static helper functions
- Data structure usage patterns
- Memory management patterns
- Error handling patterns
- Algorithm logic
```

3. **Cross-reference with assessment** — check the manifest for:
- Risk level and hard patterns for this module
- External dependencies and their Rust mappings
- Internal dependencies (which other modules does this use?)

---

## Step 4: Launch Translation Agent(s)

For each module, launch a `c-to-rust-translator` agent with comprehensive context.

**Agent prompt template**:

```
Translate the following C module to idiomatic Rust.

## Project Context
- Project: [name]
- This module: [module_name] ([description])
- Dependencies on other modules: [list]
- External C libraries used: [list → Rust crate mappings]

## Target Rust Structure
- Output file(s): [planned .rs file paths from the plan]
- Module role in crate: [how this fits into the crate structure]

## C Source Code

### Header ([header_file.h]):
[paste full header content]

### Source ([source_file.c]):
[paste full source content]

## Design Decisions from Plan
- [Key decisions, e.g., "use enum JsonValue instead of struct with type tag"]
- [Data structure choices, e.g., "Vec<T> for children instead of linked list"]
- [Error handling, e.g., "Result<T, ParseError> instead of global error"]

## Translation Requirements
1. Write idiomatic Rust — NOT a mechanical translation
2. Use the type mappings and design decisions above
3. All public items need /// doc comments
4. Add #[cfg(test)] mod tests { } with basic unit tests
5. No unsafe code unless absolutely necessary
6. Write output to: [target file path]
```

### Translation Strategy

Choose the strategy based on project size:

#### Small project fast path (≤ 1 module OR < 2,000 LOC)

For single-module or very small projects, use a single agent call with comprehensive context. The foundation-first multi-phase strategy adds unnecessary overhead here:

```
Single agent receives:
- Full C header + source content
- All design decisions from the plan
- Target Rust file path
- Complete API mapping
```

One agent, one call, one output file. No foundation types to write first, no parallelism needed.

#### Foundation-First Parallel (multi-module projects)

For projects with 2+ modules, use a three-phase approach for optimal speed and consistency:

**Phase 1 — Foundation modules** (write directly, no agent needed):
- Error types (`error.rs`) — defines `ParseError`, `JsonError`, etc.
- Core data types (`value.rs` or equivalent) — defines the central data structure

These are typically small, design-driven modules that set the type contracts for everything else. Write them first based on the plan's design decisions.

**Phase 2 — Independent consumers** (launch agents in parallel):
- Modules that depend on the foundation but NOT on each other
- Example: parser and printer both use the value type but are independent
- Launch one agent per module simultaneously for maximum speed
- Each agent receives: full C source, the foundation Rust types, and design decisions from the plan

**Phase 3 — Dependent modules** (sequential or batch):
- Modules that depend on Phase 2 outputs
- Example: utils that operate on parsed values
- Can often be batched into a single agent if they share the same dependency

### Key Principle

Provide the **already-written foundation types** as context to every translation agent. This ensures:
- Consistent type usage across all modules
- No duplicate/conflicting type definitions
- Agents can write code that compiles together on the first try

### For incremental conversion with FFI

If some modules remain as C during transition:

1. Create a `ffi.rs` module with `extern "C"` declarations for unconverted C functions
2. Create a `build.rs` to compile remaining C code via `cc` crate:
```rust
fn main() {
    cc::Build::new()
        .file("c-src/unconverted.c")
        .include("c-src")
        .compile("c_remaining");
}
```

---

## Step 5: Write Output Files

After each agent completes, write the translated Rust code to the planned file locations.

Ensure the module structure matches the plan:
```
src/
├── lib.rs          # Module declarations, public re-exports
├── module_a.rs     # Translated from module_a.c
├── module_b.rs     # Translated from module_b.c
└── ...
```

Update `src/lib.rs` with proper module declarations:
```rust
pub mod module_a;
pub mod module_b;
// Re-exports for convenience
pub use module_a::PublicType;
```

---

## Step 6: Initial Compilation Check

```bash
cd <target_dir>
cargo check 2>&1 | tee /tmp/cargo-check.txt

echo "=== Summary ==="
echo "Errors: $(grep -c '^error' /tmp/cargo-check.txt 2>/dev/null || echo 0)"
echo "Warnings: $(grep -c '^warning' /tmp/cargo-check.txt 2>/dev/null || echo 0)"
```

### If compilation succeeds:
- Run `cargo clippy` for quality check
- Run `cargo test` if tests were included
- Report success

### If compilation fails:
Analyze errors and attempt to fix them inline:

1. **Simple fixes** (import paths, type mismatches): Fix directly
2. **Cross-module issues** (type mismatch between modules): Align types
3. **Complex issues**: Note for `/c2rust-refine` phase

Iterate: fix → `cargo check` → fix → until clean or issues need user input.

---

## Step 7: Quality Checks

After successful compilation:

```bash
cd <target_dir>

# Clippy
cargo clippy -- -W clippy::all 2>&1 | head -50

# Run any generated tests
cargo test 2>&1

# Count unsafe usage (should be minimal or zero for non-FFI code)
echo "Unsafe blocks: $(grep -rn 'unsafe' src/ --include='*.rs' | grep -v test | grep -v '// ' | wc -l)"
```

---

## Output

### 1. Translated Rust Code

All .rs files in the target directory, written as idiomatic Rust.

### 2. Compilation Results

Summary of `cargo check` / `cargo clippy` / `cargo test` output.

### 3. Translation Notes

Document any:
- Intentional deviations from C behavior (with justification)
- C patterns that were redesigned (e.g., linked list → Vec)
- Remaining issues for `/c2rust-refine`
- Assumptions made during translation

### 4. Manifest Update

Update `c2rust-manifest.toml`:
```toml
[conversion]
status = "completed"
method = "claude-sonnet-4.6"
modules_converted = 5
modules_total = 5
```

Update each converted module's status:
```toml
[[modules]]
name = "utils"
status = "converted"
```

Report to the user:
- Modules translated
- Compilation status (pass/fail, error count)
- Test results (if tests were generated)
- Any issues requiring `/c2rust-refine`
