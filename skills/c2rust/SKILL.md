---
name: c2rust
description: Main pipeline guide for C-to-Rust repository conversion. Interactively walks through the full conversion pipeline from assessment to verification, prompting the user to invoke each phase skill. Use to start a new conversion, check progress, or resume from where you left off. Triggers on "convert C to Rust", "c2rust pipeline", "start conversion", "conversion status".
argument-hint: [status|resume|assess|plan|test|convert|refine|verify]
allowed-tools: [Read, Bash, Glob, Grep, Write, Edit, Agent]
---

# C-to-Rust Conversion Pipeline Guide

Main entry point for repository-level C-to-Rust conversion. This skill tracks progress and guides the user through each phase interactively, prompting them to invoke the corresponding phase skill (`/c2rust-assess`, `/c2rust-plan`, etc.) at each step.

## Arguments

The user invoked this with: $ARGUMENTS

### Commands:
- **(no argument)**: Start new conversion or resume from last checkpoint
- **status**: Show current conversion progress
- **resume**: Resume from the last completed phase
- **assess**: Jump to assessment phase
- **plan**: Jump to planning phase
- **test**: Jump to test building phase
- **convert**: Jump to conversion phase
- **refine**: Jump to refinement phase
- **verify**: Jump to verification phase

---

## Step 1: Check Current State

```bash
# Check for existing manifest
if [ -f "c2rust-manifest.toml" ]; then
    echo "=== Existing conversion project found ==="
    cat c2rust-manifest.toml
else
    echo "=== No conversion project found — starting fresh ==="
fi
```

---

## Step 2: Route Based on State and Arguments

### If `status` was requested:

Display a summary dashboard:

```markdown
# C-to-Rust Conversion Status

## Project: [name]
## Strategy: [incremental/full]

| Phase | Status | Details |
|-------|--------|---------|
| Toolchain | [ready/not ready] | rustc [version], cargo [version] |
| Assessment | [status] | [total_loc] LOC, [risk_level] risk |
| Planning | [status] | [n] modules, [strategy] |
| Testing | [status] | [n] tests created |
| Conversion | [status] | [converted]/[total] modules |
| Refinement | [status] | [errors] errors, [unsafe] unsafe blocks |
| Verification | [status] | [passed]/[total] tests |

## Module Progress
| Module | Risk | Status | Notes |
|--------|------|--------|-------|
| [name] | [risk] | [status] | [notes] |
| ... | ... | ... | ... |

## Overall Progress: [X]%
## Next Step: /c2rust-[next_phase]
```

### If starting fresh (no manifest):

Guide through the full pipeline:

1. **Welcome message**:
```
Starting C-to-Rust conversion for this project.

I'll guide you through 7 phases:
  1. Environment check — verify Rust toolchain
  2. Assessment — analyze your C codebase
  3. Planning — design the Rust project structure
  4. Testing — build behavioral test suite
  5. Conversion — translate C to idiomatic Rust (via Claude Sonnet 4.6)
  6. Refinement — fix compilation errors and polish
  7. Verification — validate correctness

Let's begin with checking your toolchain.
```

2. **Run toolchain check** (inline, don't invoke separate skill):
```bash
echo "=== Checking Toolchain ==="
echo "rustc:     $(rustc --version 2>/dev/null || echo 'NOT FOUND')"
echo "cargo:     $(cargo --version 2>/dev/null || echo 'NOT FOUND')"
echo "clippy:    $(cargo clippy --version 2>/dev/null || echo 'NOT FOUND')"
echo "bindgen:   $(bindgen --version 2>/dev/null || echo 'NOT FOUND (optional)')"
echo "cbindgen:  $(cbindgen --version 2>/dev/null || echo 'NOT FOUND (optional)')"
```

3. **Initialize manifest** with project defaults

4. **Ask user to confirm** before proceeding to assessment

5. **Phase progression**:
   After each phase completes:
   - Summarize what was accomplished
   - Show what's next
   - Ask: "Proceed to [next phase]? (or use /c2rust-[phase] to run it separately)"

### If resuming (manifest exists):

Read the manifest and determine the next phase:

```
Phase determination logic:
1. If [toolchain].ready == false → run check-env
2. If [assessment].status != "completed" → run assess
3. If [plan].status != "completed" → run plan
4. If [tests].status != "completed" → run test
5. If [conversion].status != "completed" → run convert (next unconverted module)
6. If [refinement].status != "completed" → run refine (next unrefined module)
7. If [verification].status != "completed" → run verify
8. All completed → report success
```

### If specific phase requested:

Validate prerequisites:
- `plan` requires assessment completed
- `test` requires plan completed
- `convert` requires plan completed
- `refine` requires conversion completed (at least partially)
- `verify` requires refinement completed (at least partially)

If prerequisites not met, inform user and suggest the prerequisite phase.

---

## Step 3: Phase Execution

For each phase, prompt the user to invoke the corresponding skill (e.g., "Run `/c2rust-assess` to begin assessment"). Provide context continuity between phases by summarizing what was accomplished and what comes next:

### Phase transitions:

After **assess** completes:
```
Assessment complete.
- [X] source files analyzed
- [Y] modules identified
- Overall risk: [LEVEL]
- Key concerns: [list top 3 concerns]

Next: Planning phase will design the Rust project structure.
Proceed? (y/n, or /c2rust-plan to run separately)
```

After **plan** completes:
```
Conversion plan ready.
- Target: [workspace/single] crate
- [N] modules in conversion order
- [M] external dependencies mapped

Next: Test building phase will create behavioral tests.
Proceed? (y/n, or /c2rust-test to run separately)
```

After **test** completes:
```
Test suite built.
- [N] tests created across [M] modules
- Golden data captured for [K] functions

Next: Conversion phase will translate C to Rust via Claude Sonnet 4.6.
Proceed? (y/n, or /c2rust-convert to run separately)
```

After **convert** completes:
```
Translation complete.
- [N]/[M] modules translated to idiomatic Rust
- [E] initial compilation errors
- [U] design notes logged

Next: Refinement phase will fix errors and improve code quality.
Proceed? (y/n, or /c2rust-refine to run separately)
```

After **refine** completes:
```
Refinement complete.
- [E] compilation errors fixed
- [D] design decisions made
- [U] unsafe blocks remaining (down from [U0])

Next: Verification phase will validate correctness.
Proceed? (y/n, or /c2rust-verify to run separately)
```

After **verify** completes:
```
Verification complete!
- Tests: [P]/[T] passed
- Clippy: [W] warnings
- Unsafe: [U] blocks (all justified)
- Behavioral equivalence: [status]

Conversion project status: [COMPLETE / NEEDS ATTENTION]
[Summary of any remaining work]
```

---

## Progress Tracking

Calculate overall progress percentage:

```
Phase weights:
  toolchain:    5%
  assessment:  10%
  planning:    10%
  testing:     15%
  conversion:  25%
  refinement:  25%
  verification:10%

progress = sum of (weight * phase_completion_ratio)
```

Where `phase_completion_ratio`:
- `pending` = 0.0
- `in_progress` = 0.5
- `completed` = 1.0

For conversion/refinement/verification, calculate per-module:
```
phase_ratio = completed_modules / total_modules
```

---

## Error Recovery

If a phase fails:
1. Record the error state in manifest
2. Provide diagnostic information
3. Suggest remediation:
   - Tool not found → `/c2rust-check-env --install`
   - Translation error → review C source and agent prompt context
   - Compilation error → `/c2rust-refine`
   - Test failure → investigate specific test

Never leave the manifest in an inconsistent state. If a phase fails partway through, record partial progress so `resume` can pick up correctly.
