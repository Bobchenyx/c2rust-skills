# c2rust-skills

A Claude Code plugin for systematic, repository-level C-to-Rust conversion.

## Overview

This plugin provides a complete skill set for converting C projects to idiomatic Rust, using Claude Sonnet 4.6 as the translation engine. It covers the full lifecycle:

1. **Assessment** — Analyze codebase complexity, dependencies, and conversion risks
2. **Planning** — Design Rust crate structure, map dependencies, determine conversion order
3. **Testing** — Build behavioral test suite before conversion for correctness verification
4. **Conversion** — Translate C to idiomatic Rust via Claude Sonnet 4.6 (no mechanical transpilation)
5. **Refinement** — Fix compilation errors + improve code idiomaticity
6. **Verification** — Validate converted code against behavioral tests and quality checks

## Skills

| Skill | Description |
|-------|-------------|
| `/c2rust` | Main orchestrator — start, resume, or check status of a conversion |
| `/c2rust-check-env` | Verify Rust toolchain (rustc, cargo, clippy) |
| `/c2rust-assess` | Assess C codebase for conversion risks and complexity |
| `/c2rust-plan` | Create conversion plan with crate structure and module ordering |
| `/c2rust-test` | Build behavioral test suite for pre/post-conversion verification |
| `/c2rust-convert` | Translate C to idiomatic Rust using Claude Sonnet 4.6 |
| `/c2rust-refine` | Fix compilation errors and improve code idiomaticity |
| `/c2rust-verify` | Run tests, quality checks, and generate verification report |

## Conversion Strategy

The default strategy is **incremental conversion**: modules are converted one at a time, with C and Rust code coexisting through FFI boundaries. This minimizes risk — each step produces a compilable, testable project.

Unlike mechanical transpilation tools, this plugin uses Claude Sonnet 4.6 to **directly translate** C source code into idiomatic Rust — with proper ownership, error handling, and Rust idioms from the start.

## Shared State

All skills coordinate through `c2rust-manifest.toml` in the project root, which tracks:
- Module list with per-module status and risk scores
- Conversion progress across all phases
- Dependency mappings (C libraries → Rust crates)
- Toolchain versions and readiness

## Installation

```bash
# Clone the plugin
git clone https://github.com/Bobchenyx/c2rust-skills.git ~/c2rust-skills

# Option A: Shell alias (simplest)
echo 'alias claude="claude --plugin-dir ~/c2rust-skills"' >> ~/.bashrc
source ~/.bashrc

# Option B: Register as local marketplace (persistent)
claude plugin marketplace add ~/c2rust-skills
claude plugin install c2rust-skills
```

See [USAGE.md](USAGE.md) for detailed installation options and full documentation (in Chinese).

## Quick Start

```
/c2rust-check-env          # Verify toolchain
/c2rust-assess --deep      # Deep analysis of your C project
/c2rust-plan               # Generate conversion plan
/c2rust-test               # Build test suite
/c2rust-convert --all      # Translate C → Rust via Claude Sonnet 4.6
/c2rust-refine --all       # Fix errors + improve idiomaticity
/c2rust-verify --all       # Validate correctness

# Or use the orchestrator:
/c2rust                    # Guided full pipeline
/c2rust status             # Check progress
/c2rust resume             # Resume from last phase
```

## Requirements

- `rustc` / `cargo` — Rust toolchain (>= 1.70.0)
- `clippy` — Rust linter (bundled with rustup)
- `gcc` / `cc` — C compiler (optional, for incremental FFI mode)
- `bindgen` — C headers → Rust FFI bindings (optional, for incremental mode)
- `cbindgen` — Rust → C headers (optional, for incremental mode)
