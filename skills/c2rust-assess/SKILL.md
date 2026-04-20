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
# Exclude directories that are not part of the core source
EXCLUDE="-not -path '*/.git/*' -not -path '*/test*' -not -path '*/tests/*' -not -path '*/bench*' -not -path '*/fuzz*' -not -path '*/example*' -not -path '*/vendor*' -not -path '*/third_party/*' -not -path '*/unity/*'"

# Count C/C++ source files and headers
echo "=== Source Files ==="
c_count=$(eval "find . -name '*.c' $EXCLUDE" | wc -l)
cpp_count=$(eval "find . \( -name '*.cpp' -o -name '*.cc' -o -name '*.cxx' \) $EXCLUDE" | wc -l)
h_count=$(eval "find . \( -name '*.h' -o -name '*.hpp' \) $EXCLUDE" | wc -l)
echo "C files: $c_count"
if [ "$cpp_count" -gt 0 ]; then echo "C++ files: $cpp_count"; fi
echo "H files: $h_count"

# Header-only library detection
if [ "$c_count" -eq 0 ] && [ "$h_count" -gt 0 ]; then
  echo ""
  echo "WARNING: No .c files found — this may be a header-only library."
  echo "Checking for implementation in headers..."
  # Headers with function bodies (not just declarations) indicate header-only
  impl_lines=$(eval "find . -name '*.h' $EXCLUDE" | xargs grep -l 'static.*{$\|^{' 2>/dev/null | head -5)
  if [ -n "$impl_lines" ]; then
    echo "Header-only implementation found in:"
    echo "$impl_lines"
    echo "Counting .h LOC as source LOC."
    echo ""
  fi
fi

# LOC count
echo "=== Lines of Code ==="
eval "find . -name '*.c' $EXCLUDE" | xargs wc -l 2>/dev/null | tail -1
eval "find . -name '*.h' $EXCLUDE" | xargs wc -l 2>/dev/null | tail -1

# Per-directory breakdown (identify modules)
echo "=== Module Breakdown ==="
eval "find . -name '*.c' $EXCLUDE" -exec dirname {} \; | sort | uniq -c | sort -rn
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

**Important**: All pattern greps MUST exclude test, bench, fuzz, vendor, and example directories. Use this exclude pattern on every grep:

```bash
EXCL="--exclude-dir=test --exclude-dir=tests --exclude-dir=bench --exclude-dir=fuzz --exclude-dir=fuzzing --exclude-dir=fuzzers --exclude-dir=example --exclude-dir=examples --exclude-dir=vendor --exclude-dir=third_party --exclude-dir=unity --exclude-dir=contrib --exclude-dir=.git"
# Also filter out root-level test/bench files via pipe when counting:
EXCL_FILES='grep -v "test[^/]*\.c:\|bench[^/]*\.c:\|_test\.c:\|_bench\.c:"'

echo "=== BLOCKING PATTERNS ==="
echo "Inline assembly:     $(grep -rn $EXCL '\b__asm__\b\|\b__asm\b\|asm\s*volatile\|asm\s*(' --include='*.c' --include='*.h' 2>/dev/null | eval $EXCL_FILES | wc -l)"
echo "Computed goto:       $(grep -rn $EXCL 'goto\s*\*\|&&[a-zA-Z_]' --include='*.c' 2>/dev/null | eval $EXCL_FILES | wc -l)"
echo "setjmp/longjmp:      $(grep -rn $EXCL '\bsetjmp\b\|\blongjmp\b\|\bsigsetjmp\b' --include='*.c' --include='*.h' 2>/dev/null | eval $EXCL_FILES | wc -l)"

echo ""
echo "=== HARD PATTERNS ==="
echo "Void pointers:       $(grep -rn $EXCL 'void\s*\*' --include='*.c' --include='*.h' 2>/dev/null | eval $EXCL_FILES | wc -l)"
echo "Unions:              $(grep -rn $EXCL '\bunion\s\+{\\|\bunion\s\+[a-zA-Z]' --include='*.c' --include='*.h' 2>/dev/null | eval $EXCL_FILES | wc -l)"
echo "Bitfields:           $(grep -rn $EXCL ':\s*[0-9]\+\s*;' --include='*.h' 2>/dev/null | eval $EXCL_FILES | wc -l)"
echo "Variadic functions:  $(grep -rn $EXCL 'va_list\|va_start' --include='*.c' --include='*.h' 2>/dev/null | eval $EXCL_FILES | wc -l)"
echo "Global mutable:      $(grep -rn $EXCL '^static\b' --include='*.c' 2>/dev/null | eval $EXCL_FILES | grep -v 'const\|inline' | grep -v '(.*)' | grep '[;={]' | wc -l)"
echo "Signal handlers:     $(grep -rn $EXCL '\bsignal\b\|\bsigaction\b' --include='*.c' --include='*.h' 2>/dev/null | eval $EXCL_FILES | wc -l)"

echo ""
echo "=== MODERATE PATTERNS ==="
echo "goto (total):        $(grep -rn $EXCL '\bgoto\b' --include='*.c' 2>/dev/null | eval $EXCL_FILES | wc -l)"
echo "Fn pointers (inline):$(grep -rn $EXCL '(\*[a-zA-Z_]\+)\s*(' --include='*.c' --include='*.h' 2>/dev/null | eval $EXCL_FILES | wc -l)"
echo "Fn pointers (typedef):$(grep -rn $EXCL 'typedef.*(\*' --include='*.c' --include='*.h' 2>/dev/null | eval $EXCL_FILES | wc -l)"
echo "Pointer arithmetic:  $(grep -rn $EXCL '[a-zA-Z_]\+\s*++\|++\s*[a-zA-Z_]' --include='*.c' 2>/dev/null | eval $EXCL_FILES | wc -l)"
echo "#ifdef blocks:       $(grep -rn $EXCL '#ifdef\|#if\s\+defined' --include='*.c' --include='*.h' 2>/dev/null | eval $EXCL_FILES | wc -l)"
echo "Casts:               $(grep -rn $EXCL '([a-zA-Z_]\+\s*\*)' --include='*.c' 2>/dev/null | eval $EXCL_FILES | wc -l)"
```

### Benign vs Dangerous Pattern Classification

Raw grep counts over-inflate risk. After scanning, **classify** patterns to separate benign (Claude handles trivially) from dangerous (require manual design work):

**Goto classification** — for each `goto` match, check direction:
```bash
# List all goto targets with their label definitions
# Forward goto to cleanup/end/fail/error labels = BENIGN (maps to ? + RAII)
grep -rn $EXCL '\bgoto\b' --include='*.c' 2>/dev/null | while read line; do
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

**Void pointer classification** — distinguish generic-container void* from callback/context usage:
```bash
# void* as callback user data or context = BENIGN
# void* as return type of allocator/generic container = HARD
# void* as function pointer storage (e.g. `const void *function`) = HARD
grep -rn $EXCL 'void\s*\*' --include='*.c' --include='*.h' 2>/dev/null | \
  grep -c 'unused\|userdata\|user_data\|\buser\b\|context\|ctx\|cookie\|opaque\|cb_data\|closure\|arg\|priv\|data'
# These are typically benign callback data — subtract from hard pattern count
```

Use these classifications in the risk formula (Phase 7) to avoid over-inflating scores for projects with many forward-goto cleanup patterns or callback-style void pointers.

### Structural Redesign Detection

Some C patterns don't match any single grep pattern but require **complete data structure redesign** in Rust. These are invisible to pattern-count risk scoring but have high conversion impact. Scan for:

```bash
echo ""
echo "=== STRUCTURAL REDESIGN INDICATORS ==="
echo "Packed structs:      $(grep -rn $EXCL '__attribute__.*packed\|#pragma\s*pack' --include='*.c' --include='*.h' 2>/dev/null | wc -l)"
echo "Flexible array:      $(grep -rn $EXCL '\[\]\s*;' --include='*.h' 2>/dev/null | grep -v '//' | wc -l)"
echo "Container intrusion: $(grep -rn $EXCL '\bcontainer_of\b\|offsetof' --include='*.c' --include='*.h' 2>/dev/null | wc -l)"
echo "Negative indexing:   $(grep -rn $EXCL '\[-[0-9]\]' --include='*.c' --include='*.h' 2>/dev/null | wc -l)"
echo "Token-paste macros:  $(grep -rn $EXCL '##' --include='*.h' 2>/dev/null | grep -v '//' | wc -l)"
echo "Fn ptr in union:     $(grep -rn $EXCL 'void\s*\*.*function\|void\s*\*.*callback\|void\s*\*.*handler' --include='*.c' --include='*.h' 2>/dev/null | wc -l)"

echo ""
echo "=== PLATFORM / SYSTEM PATTERNS ==="
echo "Terminal/ioctl:      $(grep -rn $EXCL 'ioctl\|termios\|tcsetattr\|tcgetattr\|TIOCGWINSZ\|raw\s*mode' --include='*.c' --include='*.h' 2>/dev/null | eval $EXCL_FILES | wc -l)"
echo "File I/O (low-level): $(grep -rn $EXCL '\bopen\b\|\bread\b\|\bwrite\b\|\bclose\b\|\bfcntl\b\|\bmmap\b' --include='*.c' 2>/dev/null | eval $EXCL_FILES | wc -l)"
echo "Socket/network:      $(grep -rn $EXCL '\bsocket\b\|\bbind\b\|\blisten\b\|\baccept\b\|\bconnect\b\|\bsend\b\|\brecv\b' --include='*.c' --include='*.h' 2>/dev/null | eval $EXCL_FILES | wc -l)"
echo "Bitwise ops density: $(grep -rn $EXCL '>>\|<<\|&\s*0x\||\s*0x' --include='*.c' 2>/dev/null | eval $EXCL_FILES | wc -l)"
```

**What to flag**:
- If a project uses packed structs + flexible array members + negative pointer indexing together (e.g., sds, Redis objects), the core data structure is designed around C-specific memory layout tricks that must be completely reimagined in Rust.
- If a project has high terminal/ioctl or socket usage, it likely needs platform-specific Rust crates (e.g., `termion`/`crossterm` for terminal, `std::net` or `tokio` for networking).
- High bitwise operation density (>100 in a single file) indicates crypto or protocol code where the Rust translation is mostly mechanical (`>>`, `<<`, `&`, `|` map directly) but needs careful constant/type matching.

---

## Phase 5: Module Boundary Identification

Identify logical modules using the following strategies.

### Fast path: Single-file and header-only projects

If the project has 0-1 source files, skip module detection entirely:

```bash
c_file_count=$(eval "find . -name '*.c' $EXCLUDE" | wc -l)
if [ "$c_file_count" -le 1 ]; then
  echo "Single-module project — no decomposition needed"
fi
# Also handle header-only libraries (0 .c files, implementation in .h)
if [ "$c_file_count" -eq 0 ]; then
  h_with_impl=$(eval "find . -name '*.h' $EXCLUDE" | xargs grep -cl '{' 2>/dev/null | wc -l)
  if [ "$h_with_impl" -gt 0 ]; then
    echo "Header-only library detected — treat .h files as source"
  fi
fi
```

Record as 1 module with all LOC (including .h LOC for header-only projects). Proceed directly to risk scoring. No dependency graph, no ordering needed.

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
