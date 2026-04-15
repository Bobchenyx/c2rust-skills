---
name: c2rust-check-env
description: Check and setup Rust toolchain for C-to-Rust conversion. Use when starting a conversion or when build tools are missing.
argument-hint: [--install]
allowed-tools: [Read, Bash, Write, Glob, Grep]
---

# Rust Toolchain Environment Check

Verify that all required tools for C-to-Rust conversion are installed.

## Arguments

$ARGUMENTS

If `--install` is specified, attempt to install missing tools automatically.

## Required Tools

### 1. Rust Toolchain (rustc + cargo)

```bash
rustc --version
cargo --version
rustup show active-toolchain 2>/dev/null
```

- **Required**: rustc >= 1.70.0
- **Install if missing**: `curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh`

### 2. Clippy (Rust linter)

```bash
cargo clippy --version 2>/dev/null
```

- **Required**: yes (bundled with rustup)
- **Install if missing**: `rustup component add clippy`

## Optional but Recommended

### For incremental conversion (C/Rust coexistence)

These are only needed if converting incrementally with FFI boundaries:

```bash
# C compiler (for compiling remaining C code via cc crate)
gcc --version 2>/dev/null | head -1 || cc --version 2>/dev/null | head -1

# bindgen (C headers → Rust FFI bindings)
bindgen --version 2>/dev/null

# cbindgen (Rust → C headers)
cbindgen --version 2>/dev/null
```

- `gcc`/`cc`: Usually pre-installed on Linux/macOS
- `bindgen`: `cargo install bindgen-cli`
- `cbindgen`: `cargo install cbindgen`

### For advanced verification

```bash
cargo miri --version 2>/dev/null   # Requires nightly
```

- `miri`: `rustup +nightly component add miri`

## Output

Generate a status table:

```
Tool            Status    Version         Notes
────────────────────────────────────────────────────
rustc           OK        1.94.1          Required
cargo           OK        1.94.1          Required
clippy          OK        0.1.94          Required
gcc/cc          OK        11.4.0          For incremental mode
bindgen         OK        0.69.4          Optional: cargo install bindgen-cli
cbindgen        OK        0.29.2          Optional: cargo install cbindgen
miri            MISSING   -               Optional: rustup +nightly component add miri
```

## Manifest Update

If `c2rust-manifest.toml` exists in the project root, update the `[toolchain]` section:

```toml
[toolchain]
rustc_version = "1.94.1"
cargo_version = "1.94.1"
clippy_version = "0.1.94"
ready = true  # true if rustc + cargo are present
```

If the manifest doesn't exist yet, create it with the `[toolchain]` and `[project]` sections.
