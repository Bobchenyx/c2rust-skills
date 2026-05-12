# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Is

A Claude Code plugin providing a complete skill set for repository-level C-to-Rust conversion. It uses Claude Sonnet as the translation engine — no mechanical transpilation. The plugin is all markdown (SKILL.md files and agent definitions); there is no compiled code or build system.

## Working on this repo

This repo has no compiled code, no test suite, and no lint config of its own — it is pure markdown (SKILL.md and agent definitions). Validation is end-to-end: install the skills into `~/.claude/skills/` (see README for symlink commands) and exercise the `/c2rust-*` commands against a real C project. The `cargo` / `clippy` / `cargo test` commands referenced inside the skills run in the **target** C project, not here.

## Project Structure

```
.claude-plugin/plugin.json   — Plugin manifest (name, version, description)
skills/                      — Slash-command skills (each dir has a SKILL.md)
  c2rust/                    — Main orchestrator (/c2rust)
  c2rust-check-env/          — Toolchain verification
  c2rust-assess/             — C codebase analysis + risk scoring
  c2rust-plan/               — Conversion plan + crate structure design
  c2rust-test/               — Behavioral test suite builder
  c2rust-convert/            — C→Rust translation via Claude Sonnet
  c2rust-refine/             — Compilation fix + idiomatic refinement
  c2rust-verify/             — Correctness validation + quality report
agents/                      — Subagent definitions (all model: sonnet)
  c-to-rust-translator.md    — Translation agent
  c-analyzer.md              — Deep analysis agent (--deep mode)
  rust-reviewer.md           — Code quality reviewer
  debug-assistant.md         — Compilation error fixer
```

Each skill directory may contain a `references/` subdirectory with lookup tables (e.g., `c-pattern-catalog.md`, `crate-mapping.md`, `error-fix-catalog.md`, `unsafe-to-safe.md`).

## Pipeline Order

assess → plan → test → convert → refine → verify

The `/c2rust` pipeline guide walks the user through all phases interactively. Each phase can also run standalone. Prerequisites are enforced: plan requires assessment, convert requires plan, refine requires conversion, verify requires refinement.

## Shared State: c2rust-manifest.toml

All skills coordinate through `c2rust-manifest.toml` in the target project root. The manifest schema is strictly enforced — use **exactly** these section names:

```
[project], [assessment], [plan], [tests], [conversion], [refinement], [verification], [toolchain], [[modules]], [dependencies_map]
```

Never create custom sections like `[package]`, `[source]`, `[target]`, `[[output]]`, or `[notes]`. Extra conversion data goes in the `notes` field inside `[conversion]`.

## SKILL.md Format

Each skill is defined by its SKILL.md with YAML frontmatter:

```yaml
---
name: skill-name
description: Trigger description and keyword matches
argument-hint: [args]
allowed-tools: [Read, Bash, Glob, Grep, Write, Edit, Agent]
---
```

`$ARGUMENTS` in the body is replaced with the user's input when the skill is invoked.

## Agent Definitions

Agent markdown files in `agents/` have frontmatter specifying `name`, `description`, `model` (optional), and `tools`. All four agents set `model: sonnet`. Each skill provides a detailed playbook with reference catalogs and structured decision trees, so no component requires a specific session model tier.

## Key Conventions

- **Convert completion checklist** (enforced in c2rust-convert; do not report success until all four pass): `cargo check` clean → `cargo clippy -- -W clippy::all` 0 warnings → `cargo test` green (inline unit tests from the translator; integration tests from `/c2rust-test` are validated later in `/c2rust-verify`) → `c2rust-manifest.toml` written with the exact section names listed above. The clippy gate is re-checked in c2rust-refine (Phase C).
- **BLOCKING patterns stay as C**: inline assembly, computed `goto`, and `setjmp`/`longjmp` cannot be auto-translated. Modules containing them must be kept as C across an FFI boundary; do not attempt automatic conversion. Manual rewrites map to `core::arch::asm!`, a match/enum state machine, and `Result<T, E>` respectively.
- **Directory exclusions**: All grep-based pattern scanning in assess must exclude test, bench, fuzz, vendor, example, third_party, unity, and contrib directories. Use the `$EXCL` and `$EXCL_FILES` variables defined in the assess skill.
- **Pattern classification**: Raw grep counts for goto and void* over-inflate risk. The assess skill classifies patterns as benign vs dangerous (e.g., forward-goto-to-cleanup is benign; backward goto is dangerous) before computing risk scores.
- **Small project fast path**: For projects with ≤ 5 files AND < 2,000 LOC, assess skips module decomposition and convert uses a single agent call instead of the multi-phase foundation-first strategy.
- **FFI glue ownership**: The convert skill manages FFI boundaries during incremental conversion (Step 5b). After each module is converted, it regenerates bindings, updates build.rs, and verifies the mixed build.
- **Generated artifacts** (not tracked in git): `c2rust-manifest.toml`, `c2rust-assessment.md`, `c2rust-plan.md`, `c2rust-verification-report.md`.

## End-user docs

`README.md` is the user-facing reference: installation (user-level vs project-level symlinks), quick-start command sequence, requirements (`bindgen`/`cbindgen` are only needed for incremental FFI mode), and FAQ (BLOCKING patterns, refine-loop fallback, `-p` mode permissions, re-converting a module). When changing any user-visible behavior — command names, argument flags, pipeline order, or the manifest schema — update `README.md` alongside the affected SKILL.md.
