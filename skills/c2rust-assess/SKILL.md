---
name: c2rust-assess
description: Assess a C codebase for Rust conversion. Analyzes build system, dependencies, code complexity, and conversion risks. Use when starting a C-to-Rust conversion, when the user mentions "assess", "analyze C code", "conversion difficulty", or "risk assessment".
argument-hint: [--quick|--deep] [path]
allowed-tools: [Read, Bash, Glob, Grep, Write, Agent]
---

# C Codebase Assessment for Rust Conversion

Perform a thorough assessment of the C codebase to understand its structure, complexity, and conversion risks before attempting any C-to-Rust conversion.

## Arguments

The user invoked this with: $ARGUMENTS

Parse arguments:
- `--quick`: Fast overview mode (default if no flag given)
- `--deep`: Full deep analysis with agent-assisted call graph tracing
- `path`: Optional path to the C source directory (default: current directory)

## Prerequisites

Read `c2rust-manifest.toml` if it exists to check current state. If it doesn't exist, this is a fresh assessment — create the manifest at the end.

---

## Phase 1: Build System Detection

Identify the project's build system by scanning for build files:

```bash
# Check for common build system files
ls -la Makefile CMakeLists.txt configure.ac configure.in meson.build SConstruct BUILD.gn Makefile.am 2>/dev/null

# For cmake, check top-level CMakeLists.txt
head -50 CMakeLists.txt 2>/dev/null

# For autotools, check configure.ac
head -50 configure.ac 2>/dev/null

# For make, check Makefile structure
head -80 Makefile 2>/dev/null
```

Record: build system type, version requirements, build targets, build flags.

---

## Phase 2: Source File Inventory

```bash
# Count C source files and headers
echo "=== Source Files ==="
find . -name '*.c' -not -path '*/test*' -not -path '*/.git/*' | wc -l
find . -name '*.h' -not -path '*/.git/*' | wc -l

# LOC count
echo "=== Lines of Code ==="
find . -name '*.c' -not -path '*/test*' -not -path '*/.git/*' | xargs wc -l 2>/dev/null | tail -1
find . -name '*.h' -not -path '*/.git/*' | xargs wc -l 2>/dev/null | tail -1

# Per-directory breakdown (identify modules)
echo "=== Module Breakdown ==="
find . -name '*.c' -not -path '*/.git/*' -exec dirname {} \; | sort | uniq -c | sort -rn
```

---

## Phase 3: Dependency Analysis

### External library dependencies

```bash
# From linker flags in build files
grep -rn '\-l[a-z]' Makefile CMakeLists.txt 2>/dev/null
grep -rn 'target_link_libraries\|find_package\|pkg_check_modules' CMakeLists.txt 2>/dev/null
grep -rn 'PKG_CHECK_MODULES\|AC_CHECK_LIB' configure.ac 2>/dev/null

# From system includes
grep -rh '#include\s*<' --include='*.c' --include='*.h' | sort -u | grep -v 'std\|assert\|errno\|limits\|ctype'
```

### Internal module dependencies

```bash
# Map internal #include relationships
for f in $(find . -name '*.c' -not -path '*/.git/*'); do
  deps=$(grep '#include\s*"' "$f" | sed 's/.*"\(.*\)".*/\1/' | tr '\n' ', ')
  if [ -n "$deps" ]; then
    echo "$(basename $f) -> $deps"
  fi
done
```

---

## Phase 4: Unsafe Pattern Scanning

Scan for patterns from [references/c-pattern-catalog.md](references/c-pattern-catalog.md).

```bash
echo "=== BLOCKING PATTERNS ==="
echo "Inline assembly:     $(grep -rn '\b__asm__\b\|\b__asm\b\|asm\s*volatile\|asm\s*(' --include='*.c' --include='*.h' 2>/dev/null | grep -v '//' | wc -l)"
echo "Computed goto:       $(grep -rn 'goto\s*\*\|&&[a-zA-Z_]' --include='*.c' 2>/dev/null | wc -l)"
echo "setjmp/longjmp:      $(grep -rn '\bsetjmp\b\|\blongjmp\b\|\bsigsetjmp\b' --include='*.c' --include='*.h' 2>/dev/null | wc -l)"

echo ""
echo "=== HARD PATTERNS ==="
echo "Void pointers:       $(grep -rn 'void\s*\*' --include='*.c' --include='*.h' 2>/dev/null | wc -l)"
echo "Unions:              $(grep -rn '\bunion\s\+{\\|\bunion\s\+[a-zA-Z]' --include='*.c' --include='*.h' 2>/dev/null | wc -l)"
echo "Bitfields:           $(grep -rn ':\s*[0-9]\+\s*;' --include='*.h' 2>/dev/null | wc -l)"
echo "Variadic functions:  $(grep -rn 'va_list\|va_start\|\.\.\.)' --include='*.c' --include='*.h' 2>/dev/null | wc -l)"
echo "Global mutable:      $(grep -rn '^static\b' --include='*.c' 2>/dev/null | grep -v 'const\|inline' | grep -v '(.*)' | wc -l)"
echo "Signal handlers:     $(grep -rn '\bsignal\b\|\bsigaction\b' --include='*.c' --include='*.h' 2>/dev/null | wc -l)"

echo ""
echo "=== MODERATE PATTERNS ==="
echo "goto (total):        $(grep -rn '\bgoto\b' --include='*.c' 2>/dev/null | wc -l)"
echo "Function pointers:   $(grep -rn '(\*[a-zA-Z_]\+)\s*(' --include='*.c' --include='*.h' 2>/dev/null | wc -l)"
echo "Pointer arithmetic:  $(grep -rn '[a-zA-Z_]\+\s*++\|++\s*[a-zA-Z_]' --include='*.c' 2>/dev/null | wc -l)"
echo "#ifdef blocks:       $(grep -rn '#ifdef\|#if\s\+defined' --include='*.c' --include='*.h' 2>/dev/null | wc -l)"
echo "Casts:               $(grep -rn '([a-zA-Z_]\+\s*\*)' --include='*.c' 2>/dev/null | wc -l)"
```

### Benign vs Dangerous Pattern Classification

Raw grep counts over-inflate risk. After scanning, **classify** patterns to separate benign (Claude handles trivially) from dangerous (require manual design work):

**Goto classification** — for each `goto` match, check direction:
```bash
# List all goto targets with their label definitions
# Forward goto to cleanup/end/fail/error labels = BENIGN (maps to ? + RAII)
grep -rn '\bgoto\b' --include='*.c' 2>/dev/null | while read line; do
  file=$(echo "$line" | cut -d: -f1)
  lineno=$(echo "$line" | cut -d: -f2)
  label=$(echo "$line" | sed 's/.*goto\s*\([a-zA-Z_]*\).*/\1/')
  # Check if label is defined AFTER the goto (forward = benign)
  label_line=$(grep -n "^${label}:" "$file" 2>/dev/null | head -1 | cut -d: -f1)
  if [ -n "$label_line" ] && [ "$label_line" -gt "$lineno" ]; then
    echo "BENIGN (forward): $line"
  else
    echo "DANGEROUS (backward/unknown): $line"
  fi
done
```

**Void pointer classification** — distinguish generic-container void* from trivial/unused:
```bash
# void* in function parameter that is cast immediately or unused = BENIGN
# void* as return type of allocator/generic container = HARD (already counted above)
# Quick heuristic: void* in typedef for callbacks is usually benign
grep -rn 'void\s*\*' --include='*.c' --include='*.h' 2>/dev/null | \
  grep -c 'unused\|userdata\|user_data\|context\|ctx\|cookie\|opaque\|cb_data'
# These are typically benign — subtract from hard pattern count
```

Use these classifications in the risk formula (Phase 7) to avoid over-inflating scores for projects with many forward-goto cleanup patterns or callback-style void pointers.

### Structural Redesign Detection

Some C patterns don't match any single grep pattern but require **complete data structure redesign** in Rust. These are invisible to pattern-count risk scoring but have high conversion impact. Scan for:

```bash
echo ""
echo "=== STRUCTURAL REDESIGN INDICATORS ==="
echo "Packed structs:      $(grep -rn '__attribute__.*packed\|#pragma\s*pack' --include='*.c' --include='*.h' 2>/dev/null | wc -l)"
echo "Flexible array:      $(grep -rn '\[\]\s*;' --include='*.h' 2>/dev/null | grep -v '//' | wc -l)"
echo "Container intrusion: $(grep -rn '\bcontainer_of\b\|offsetof' --include='*.c' --include='*.h' 2>/dev/null | wc -l)"
echo "Negative indexing:   $(grep -rn '\[-[0-9]\]' --include='*.c' --include='*.h' 2>/dev/null | wc -l)"
echo "Token-paste macros:  $(grep -rn '##' --include='*.h' 2>/dev/null | grep -v '//' | wc -l)"
```

**What to flag**: If a project uses packed structs + flexible array members + negative pointer indexing together (e.g., sds, Redis objects), the core data structure is likely designed around C-specific memory layout tricks that must be completely reimagined in Rust (typically as a newtype over `Vec<T>` or similar). Note this in the assessment as a **structural redesign required** finding — it won't inflate the pattern count but must be communicated in the plan.

---

## Phase 5: Module Boundary Identification

Identify logical modules using the following strategies.

### Fast path: Single-file projects

If the project has only 1 source file (e.g., `sds.c` + `sds.h`), skip module detection entirely:

```bash
c_file_count=$(find . -name '*.c' -not -path '*/test*' -not -path '*/.git/*' | wc -l)
if [ "$c_file_count" -le 1 ]; then
  echo "Single-module project — no decomposition needed"
fi
```

Record as 1 module with all LOC. Proceed directly to risk scoring. No dependency graph, no ordering needed.

### For multi-file projects

For multi-directory projects, directory structure is usually sufficient. **For flat projects** (all .c files in one directory), use naming and dependency analysis.

### Strategy 1: Directory-based (multi-directory projects)

Each directory with .c files is a candidate module. Works well when the project already has a directory hierarchy.

### Strategy 2: Naming-prefix grouping (flat projects)

When all .c files live in one directory, group by naming prefix:

```bash
# Extract common prefixes from .c filenames
find . -maxdepth 1 -name '*.c' -exec basename {} .c \; | \
  sed 's/\(_[a-z]\)/\n\1/' | head -1 | sort -u

# Example: cJSON.c + cJSON_Utils.c → module "cJSON" (core) + module "cJSON_Utils" (utils)
# Example: http_parser.c + http_client.c + http_server.c → 3 modules sharing "http" prefix
```

Heuristic: if `<prefix>.c` and `<prefix>.h` both exist, that's a core module. If `<prefix>_<suffix>.c` exists with a matching `<prefix>_<suffix>.h`, that's an extension module depending on the core.

### Strategy 3: Header-dependency clustering

Group .c files that include the same project header:

```bash
# For each project header, list which .c files include it
for h in $(find . -name '*.h' -not -path '*/.git/*' -exec basename {} \;); do
  consumers=$(grep -rl "#include\s*\"$h\"" --include='*.c' 2>/dev/null | tr '\n' ' ')
  if [ -n "$consumers" ]; then
    echo "$h -> $consumers"
  fi
done
```

Files that share a private header (not included by other modules) belong to the same module. Files that provide a public header used by others form a dependency relationship.

### Strategy 4: Build target groupings

From Makefile/CMake, identify which .c files are compiled into which library or executable targets. Each target is a natural module boundary.

```bash
# CMake: look for add_library / add_executable with source lists
grep -A 20 'add_library\|add_executable' CMakeLists.txt 2>/dev/null

# Make: look for object file lists
grep '\.o' Makefile 2>/dev/null | head -20
```

### Module record

For each identified module, record:
- Name
- Path
- File list (.c and .h)
- LOC
- Internal dependencies (which other modules it uses)
- External dependencies (which libraries)
- Hard pattern count
- Preliminary risk rating

---

## Phase 6: Deep Analysis (--deep mode only)

**Only execute this phase if `--deep` flag is present.**

Launch up to 3 `c-analyzer` agents in parallel for comprehensive analysis:

### Agent 1: Call Graph & Control Flow
Prompt: Trace the call graph starting from main entry points. Identify:
- Function call depth chains
- Recursive functions
- Functions with highest cyclomatic complexity
- Control flow patterns (state machines, event loops)

### Agent 2: Data Flow & State
Prompt: Analyze data flow patterns. Identify:
- Global mutable state and which functions access it
- Struct ownership patterns (who allocates, who frees)
- Data that flows between modules
- Thread safety patterns (mutexes, atomics)

### Agent 3: Macro & Type Analysis
Prompt: Analyze preprocessor and type usage. Identify:
- Complex macros (X-macros, recursive, token pasting)
- Union types and how they're used
- Bitfield usage
- Type-punning patterns
- Conditional compilation paths

After agents complete, read their key findings and incorporate into the assessment.

---

## Phase 7: Risk Score Calculation

Using the methodology from [references/complexity-metrics.md](references/complexity-metrics.md), calculate per-module risk scores.

**Important**: Use the benign/dangerous classification from Phase 4. Do NOT treat all moderate patterns equally.

```
# Split moderate patterns into benign vs dangerous
benign_moderate = forward_goto_count + trivial_void_ptr + simple_casts
dangerous_moderate = backward_goto_count + function_pointers + complex_ifdef

# Weighted density
adjusted_moderate = benign_moderate * 0.5 + dangerous_moderate * 2

density = (blocking_count * 10 + hard_count * 5 + adjusted_moderate) / module_loc * 1000
```

Classify each module:
- 0-25: LOW risk — good candidate for early conversion
- 26-50: MEDIUM risk — standard conversion, some manual work
- 51-75: HIGH risk — significant manual work, convert later
- 76-100: CRITICAL risk — consider keeping as C with FFI bindings

---

## Output

### 1. Assessment Report

Write `c2rust-assessment.md` with:

```markdown
# C-to-Rust Conversion Assessment

## Project Overview
- **Project**: [name]
- **Build system**: [type]
- **Total LOC**: [number] (.c) + [number] (.h)
- **Source files**: [number] .c files, [number] .h files
- **Assessment mode**: quick|deep
- **Overall risk**: LOW|MEDIUM|HIGH|CRITICAL

## External Dependencies
| Library | Usage | Rust Equivalent | Status |
|---------|-------|----------------|--------|
| ... | ... | ... | available/needs-binding/no-equivalent |

## Module Assessment
| Module | LOC | Risk | Blocking | Hard | Moderate | Dependencies |
|--------|-----|------|----------|------|----------|-------------|
| ... | ... | ... | ... | ... | ... | ... |

## Blocking Issues
[List any patterns that CANNOT be auto-converted and require manual rewrite]

## Recommended Conversion Order
[Topologically sorted module list, factoring in risk]

## Estimated Effort
[Based on LOC and risk level]
```

### 2. Manifest Update

Update `c2rust-manifest.toml` using **exactly** these section names and field names. Do NOT create custom sections or rename fields:

```toml
[project]
name = "<project name>"
source_dir = "."
build_system = "make"                # make | cmake | autotools | meson | custom
total_loc = 1700
total_files = 1
total_headers = 3

[assessment]
status = "completed"                 # pending | in_progress | completed
date = "2026-04-15"
mode = "quick"                       # quick | deep
feasibility = "HIGH"                 # LOW | MEDIUM | HIGH | CRITICAL
complexity = "low"                   # low | low-medium | medium | high | critical
report = "c2rust-assessment.md"

[plan]
status = "pending"

[conversion]
status = "pending"

[toolchain]
rustc_version = ""
cargo_version = ""
ready = false

[[modules]]
name = "core"
path = "."
status = "assessed"                  # pending | assessed | planned | converted | verified
risk = "low"                         # low | medium | high | critical
loc = 1700
dependencies = []
hard_patterns = []
notes = ""

[dependencies_map]
# C library = "Rust equivalent"
# e.g. pthreads = "std::thread"
```

**Important**: You MUST use these exact section names (`[project]`, `[assessment]`, `[plan]`, `[conversion]`, `[toolchain]`, `[[modules]]`, `[dependencies_map]`). Do not create custom sections like `[package]`, `[source]`, or `[target]`.
