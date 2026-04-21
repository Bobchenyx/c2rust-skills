# c2rust-skills

A Claude Code plugin for systematic, repository-level C-to-Rust conversion.

## Overview

This plugin provides a complete skill set for converting C projects to idiomatic Rust, using Claude Sonnet as the translation engine. It covers the full lifecycle:

1. **Assessment** — Analyze codebase complexity, dependencies, and conversion risks
2. **Planning** — Design Rust crate structure, map dependencies, determine conversion order
3. **Testing** — Build behavioral test suite before conversion for correctness verification
4. **Conversion** — Translate C to idiomatic Rust via Claude Sonnet (no mechanical transpilation)
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
| `/c2rust-convert` | Translate C to idiomatic Rust using Claude Sonnet |
| `/c2rust-refine` | Fix compilation errors and improve code idiomaticity |
| `/c2rust-verify` | Run tests, quality checks, and generate verification report |

## Conversion Strategy

The default strategy is **incremental conversion**: modules are converted one at a time, with C and Rust code coexisting through FFI boundaries. This minimizes risk — each step produces a compilable, testable project.

Unlike mechanical transpilation tools, this plugin uses Claude Sonnet to **directly translate** C source code into idiomatic Rust — with proper ownership, error handling, and Rust idioms from the start. All four subagents (translation, analysis, code review, debug) run on Sonnet.

## Shared State

All skills coordinate through `c2rust-manifest.toml` in the project root, which tracks:
- Module list with per-module status and risk scores
- Conversion progress across all phases
- Dependency mappings (C libraries → Rust crates)
- Toolchain versions and readiness

## Installation

```bash
# Clone the repo
git clone https://github.com/Bobchenyx/c2rust-skills.git ~/c2rust-skills

# Option A: User-level (available in all projects)
ln -s ~/c2rust-skills/skills/* ~/.claude/skills/

# Option B: Project-level (available only in one project)
cd /path/to/your-c-project
mkdir -p .claude/skills
ln -s ~/c2rust-skills/skills/* .claude/skills/
```

After installation, all `/c2rust-*` commands are available when you start Claude Code.

## Quick Start

```
/c2rust-check-env          # Verify toolchain
/c2rust-assess --deep      # Deep analysis of your C project
/c2rust-plan               # Generate conversion plan
/c2rust-test               # Build test suite
/c2rust-convert --all      # Translate C → Rust via Claude Sonnet
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

## FAQ

**The translated Rust code doesn't compile?**
This is normal. Use `/c2rust-refine` to auto-fix most mechanical errors. Semantic issues that require design decisions will be presented to you interactively.

**A module is rated CRITICAL risk?**
Keep it as C with FFI bindings during incremental conversion. Mark it as an "FFI boundary" in the plan phase. Rewrite it manually later.

**The refine phase is stuck in a loop?**
If errors stop decreasing after 3 iterations, switch to `--interactive` mode to inspect each issue individually.

**How do I handle BLOCKING patterns (inline assembly, computed goto, setjmp)?**
These cannot be auto-translated. Keep modules containing them as C via FFI, or manually rewrite: inline asm → `core::arch::asm!`, computed goto → match/enum state machine, setjmp/longjmp → `Result<T, E>`.

**Is `unsafe` in the translated code normal?**
Claude-translated Rust rarely uses `unsafe`. It mainly appears at FFI boundaries with unconverted C modules. The refine phase further minimizes `unsafe` usage and adds `// SAFETY:` comments.

**What C projects are supported?**
Any pure C project. The translation engine reads C source files directly — it doesn't depend on a specific build system or require compiling the C code. C++ is not supported.

**How do I re-convert a module?**
Set the module's `status` back to `"planned"` in `c2rust-manifest.toml`, delete the corresponding `.rs` file, and re-run `/c2rust-convert <module-name>`.

**Skills fail to write files in non-interactive (`-p`) mode?**
In `claude -p` mode, tool permissions are restricted by default. Add `--dangerously-skip-permissions` or use `--allowedTools "Read,Bash,Glob,Grep,Write,Edit,Agent"` to grant the skills the access they need. Interactive mode (the default) is unaffected — you'll be prompted for permission as usual.
