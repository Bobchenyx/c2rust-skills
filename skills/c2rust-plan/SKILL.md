---
name: c2rust-plan
description: Create a conversion plan for C-to-Rust migration. Designs Rust crate structure, maps C library dependencies to Rust crates, and determines module conversion order. Use after c2rust-assess, or when the user mentions "conversion plan", "crate structure", "migration plan".
argument-hint: [--incremental|--full]
allowed-tools: [Read, Bash, Glob, Grep, Write, Agent]
---

# Conversion Planning

Design a concrete plan for converting the assessed C codebase to Rust.

## Arguments

The user invoked this with: $ARGUMENTS

- `--incremental`: Incremental conversion with FFI boundaries (default)
- `--full`: Full one-shot conversion (recommended only for small projects)

## Prerequisites

Read `c2rust-manifest.toml` — the assessment must be completed first.

```bash
cat c2rust-manifest.toml 2>/dev/null
```

If assessment is not completed, inform the user to run `/c2rust-assess` first.

Also read the assessment report:
```bash
cat c2rust-assessment.md 2>/dev/null
```

---

## Step 1: Define Rust Project Structure

Based on the module assessment, design the target Rust project:

### Workspace vs Single Crate Decision

- **< 5 modules or < 5000 LOC**: Single crate with modules
- **5+ modules or 5000+ LOC**: Cargo workspace with member crates

### Crate Design

For each C module, decide:
1. Does it become its own crate? (if large, reusable, or has distinct API)
2. Does it become a module within a parent crate? (if small or tightly coupled)
3. Does it stay as C with FFI bindings? (if too complex to convert now, or uses blocking patterns)

Design the directory structure:
```
rust-project/
├── Cargo.toml                # Workspace manifest
├── crates/
│   ├── core/                 # Core functionality
│   │   ├── Cargo.toml
│   │   └── src/
│   │       ├── lib.rs
│   │       └── module_a.rs
│   ├── utils/                # Utility functions
│   │   ├── Cargo.toml
│   │   └── src/lib.rs
│   └── app/                  # Binary crate
│       ├── Cargo.toml
│       └── src/main.rs
├── c-src/                    # Remaining C code (during transition)
│   └── unconverted.c
└── tests/
    └── integration/
```

---

## Step 2: Dependency Mapping

Using [references/crate-mapping.md](references/crate-mapping.md), map each external C library dependency to a Rust equivalent.

For each dependency, decide:
1. **Pure Rust replacement** — Preferred when mature and compatible
2. **-sys binding crate** — When exact C behavior compatibility needed
3. **Keep as C, use via FFI** — When no Rust equivalent exists
4. **Remove** — When the functionality is available in Rust std

Create a dependency mapping table:

```markdown
| C Library | C Usage | Rust Approach | Rust Crate | Notes |
|-----------|---------|--------------|------------|-------|
| libssl | TLS connections | Pure Rust | rustls | Safer, no OpenSSL dependency |
| zlib | Data compression | Pure Rust | flate2 | Drop-in replacement |
| libfoo | Custom protocol | FFI binding | foo-sys (create) | No Rust equivalent |
```

---

## Step 3: Determine Conversion Order

Build a dependency graph between modules and compute conversion order.

### Rules for ordering:
1. **Leaf modules first**: Modules with no internal dependencies
2. **Low risk first**: Among equally independent modules, convert lowest risk first
3. **High-value dependencies early**: Modules that many others depend on
4. **Blocking patterns last**: Modules with BLOCKING patterns may stay as C

### Algorithm:
1. Build adjacency list from module dependencies
2. Topological sort
3. Within each level, sort by risk score (ascending)
4. Mark CRITICAL-risk modules as "FFI boundary" (keep as C initially)

Present the conversion order as a numbered list:

```markdown
## Conversion Order

| Order | Module | Risk | Dependencies | Milestone |
|-------|--------|------|-------------|-----------|
| 1 | utils | LOW | none | Builds standalone |
| 2 | config | LOW | utils | Reads config files |
| 3 | parser | MEDIUM | utils | Parses input |
| 4 | core | HIGH | parser, config | Core logic works |
| 5 | network | MEDIUM | core | Network layer |
| 6 | main | LOW | all | Full binary |
| FFI | crypto | CRITICAL | none | Keep as C with bindings |
```

---

## Step 4: FFI Boundary Design (Incremental Mode)

For incremental conversion, define how C and Rust code will interact during the transition:

### For each conversion step:
1. Which module is being converted
2. What FFI interfaces does it need:
   - **Inbound**: C code calling into newly converted Rust
   - **Outbound**: Converted Rust calling remaining C code
3. What headers need to be generated:
   - `cbindgen` for Rust→C headers
   - `bindgen` for C→Rust bindings

### Build system integration:
- `build.rs` configuration for compiling remaining C code
- `cc` crate usage for C compilation
- Linker configuration for mixed binary

Reference [references/ffi-patterns.md](references/ffi-patterns.md) for FFI design patterns.

---

## Step 5: Build System Migration Plan

Design the transition from C build system to Cargo:

### Phase 1 (Start): Cargo wraps C build
```toml
# Cargo.toml
[build-dependencies]
cc = "1.0"
bindgen = "0.69"
```

### Phase 2 (During): Mixed build
- Cargo compiles Rust code
- `build.rs` compiles remaining C code via `cc`
- `bindgen` generates bindings for unconverted C headers

### Phase 3 (End): Pure Cargo
- All code is Rust
- No `build.rs` needed (or minimal)
- External C libraries through `-sys` crates

---

## Step 6: Define Success Criteria

For each module conversion milestone:
- [ ] `cargo check` passes (no compilation errors)
- [ ] `cargo test` passes (behavioral tests from c2rust-test)
- [ ] `cargo clippy` passes with no warnings
- [ ] Behavioral equivalence verified against C version
- [ ] unsafe block count is documented and justified
- [ ] FFI boundaries are clean and well-documented

---

## Output

### 1. Conversion Plan Document

Write `c2rust-plan.md` with the complete plan including:
- Target project structure (directory tree)
- Dependency mapping table
- Conversion order with milestones
- FFI boundary specifications
- Build system migration plan
- Success criteria per module

### 2. Manifest Update

Update `c2rust-manifest.toml`:
```toml
[plan]
status = "completed"
rust_project_name = "..."
crate_structure = "workspace"  # or "single"
target_dir = "rust/"
conversion_order = ["utils", "config", "parser", "core", "network", "main"]
plan_path = "c2rust-plan.md"

[dependencies_map]
openssl = "rustls"
zlib = "flate2"
pthread = "std::thread"
```

Update each `[[modules]]` entry with:
- `rust_crate_mapping` for external deps
- Conversion order position
- FFI boundary specification
