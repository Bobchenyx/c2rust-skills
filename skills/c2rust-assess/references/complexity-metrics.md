# Complexity Metrics for C-to-Rust Conversion Assessment

## Metrics Overview

| Metric | What It Measures | How to Calculate | Impact on Conversion |
|--------|-----------------|------------------|---------------------|
| LOC | Codebase size | `wc -l` on .c/.h files | Linear: more code = more work |
| File count | Project breadth | Count .c and .h files | Affects module planning |
| Function count | API surface | grep for function definitions | Affects testing scope |
| Cyclomatic complexity | Control flow complexity | Count decision points | High CC = hard to refactor |
| Unsafe pattern density | Translation-blocking constructs | Count from pattern catalog | Determines manual work |
| Dependency depth | Module coupling | Build call/include graph | Affects conversion order |
| Global state count | Shared mutable state | Count file-scope variables | Hardest part of Rustification |

---

## LOC Counting

```bash
# Count C source lines (excluding blank lines and comments)
find . -name '*.c' -o -name '*.h' | xargs grep -v '^\s*$' | grep -v '^\s*//' | grep -v '^\s*\*' | wc -l

# Quick count including blanks
find . -name '*.c' | xargs wc -l
find . -name '*.h' | xargs wc -l

# Per-directory breakdown
find . -name '*.c' -exec dirname {} \; | sort -u | while read d; do
  echo "$d: $(find "$d" -maxdepth 1 -name '*.c' | xargs cat 2>/dev/null | wc -l) lines"
done
```

**Conversion effort estimation by LOC**:

| LOC Range | Project Size | Estimated Effort |
|-----------|-------------|-----------------|
| < 1,000 | Tiny | Single session |
| 1,000 - 5,000 | Small | A few sessions |
| 5,000 - 20,000 | Medium | Multi-day effort |
| 20,000 - 100,000 | Large | Multi-week effort |
| > 100,000 | Very Large | Multi-month effort, consider partial conversion |

---

## Cyclomatic Complexity

Cyclomatic complexity (CC) = number of independent paths through a function.

**Approximate calculation** (count decision points + 1):
```bash
# For each .c file, count decision keywords
grep -c '\b\(if\|else\|while\|for\|case\|&&\|||\|\?\)\b' file.c
```

**Interpretation for conversion**:

| CC | Complexity | Conversion Impact |
|----|-----------|-------------------|
| 1-5 | Low | Straightforward conversion |
| 6-10 | Moderate | May need control flow restructuring |
| 11-20 | High | Likely needs significant refactoring |
| 21-50 | Very High | Consider breaking into smaller functions |
| > 50 | Extreme | Must decompose before conversion |

Functions with CC > 20 should be flagged for manual review during assessment.

---

## Unsafe Pattern Density

Count occurrences of patterns from the C pattern catalog:

```bash
# Blocking patterns (must be manually rewritten)
echo "=== BLOCKING PATTERNS ==="
echo "Inline assembly:   $(grep -rn '\b__asm__\b\|asm\s*(' --include='*.c' --include='*.h' | wc -l)"
echo "Computed goto:     $(grep -rn 'goto\s*\*\|&&[a-zA-Z]' --include='*.c' | wc -l)"
echo "setjmp/longjmp:    $(grep -rn '\bsetjmp\b\|\blongjmp\b' --include='*.c' --include='*.h' | wc -l)"

# Hard patterns (significant manual work)
echo "=== HARD PATTERNS ==="
echo "Void pointers:     $(grep -rn 'void\s*\*' --include='*.c' --include='*.h' | wc -l)"
echo "Union types:       $(grep -rn '\bunion\b' --include='*.c' --include='*.h' | wc -l)"
echo "Bitfields:         $(grep -rn ':\s*[0-9]\+\s*;' --include='*.c' --include='*.h' | wc -l)"
echo "va_args:           $(grep -rn 'va_list\|va_start' --include='*.c' --include='*.h' | wc -l)"
echo "Global mutables:   $(grep -rn '^static\s\+[^c]' --include='*.c' | grep -v 'const' | wc -l)"

# Moderate patterns (mostly automatable with cleanup)
echo "=== MODERATE PATTERNS ==="
echo "goto:              $(grep -rn '\bgoto\b' --include='*.c' | wc -l)"
echo "Function pointers: $(grep -rn '(\*[a-zA-Z_]*)\s*(' --include='*.c' --include='*.h' | wc -l)"
echo "Pointer arithmetic:$(grep -rn '[a-zA-Z_]\+\s*+\+\|+\+\s*[a-zA-Z_]' --include='*.c' | wc -l)"
echo "Conditional comp:  $(grep -rn '#ifdef\|#if defined' --include='*.c' --include='*.h' | wc -l)"
```

**Unsafe Pattern Density Score**:

Patterns are weighted by actual translation difficulty, distinguishing **benign** patterns (that Claude handles trivially) from **truly dangerous** ones:

```
# Benign patterns (Claude handles well — low weight)
benign_moderate = forward_goto_cleanup + simple_char_cast + simple_pointer_deref
# Dangerous patterns (require real design work — full weight)  
dangerous_moderate = backward_goto + function_pointer + complex_conditional_comp

adjusted_moderate = benign_moderate * 0.5 + dangerous_moderate * 2

density = (blocking_count * 10 + hard_count * 5 + adjusted_moderate) / total_loc * 1000
```

**Key distinction**: A `goto cleanup;` (forward, cleanup pattern) is EASY for Claude — it maps directly to `?` + RAII. A computed `goto *table[i]` is BLOCKING. Both match `grep '\bgoto\b'` but have vastly different conversion difficulty. The assess skill should classify gotos by direction before counting.

Similarly, `void *` in a generic container API (like cJSON's allocator hooks) is HARD, but `void *` as an unused parameter in a callback typedef is TRIVIAL.

| Density | Rating | Meaning |
|---------|--------|---------|
| < 5 | Low | Straightforward translation |
| 5-15 | Medium | Some design decisions needed |
| 15-30 | High | Significant manual work and review required |
| > 30 | Critical | Consider partial conversion with FFI boundaries |

---

## Dependency Depth

### Internal dependencies (between modules)

```bash
# Map which .c files include which .h files
for f in $(find . -name '*.c'); do
  echo "=== $f ==="
  grep '#include\s*"' "$f" | sed 's/.*"\(.*\)".*/\1/'
done
```

**Conversion order priority**:
- Modules with 0 internal dependencies → convert first (leaf nodes)
- Modules depended on by many others → convert early (high-value)
- Modules with many dependencies → convert last (need others ready)

### External dependencies (third-party libraries)

```bash
# From linker flags
grep -rn '\-l[a-z]' Makefile CMakeLists.txt 2>/dev/null
# From pkg-config
grep -rn 'pkg-config\|pkg_check_modules' Makefile CMakeLists.txt configure.ac 2>/dev/null
# From includes
grep -rn '#include\s*<' --include='*.c' --include='*.h' | grep -v 'std\|string\|stdio\|stdlib' | sort -u
```

---

## Global State Analysis

```bash
# File-scope static variables (potentially global mutable state)
grep -rn '^static\b' --include='*.c' | grep -v 'const\|inline\|void\|int.*(' | head -50

# Global (extern) variables
grep -rn '^extern\b.*[^(;]*;$' --include='*.h'
grep -rn '^[a-zA-Z_].*[^)(]\s*=' --include='*.c' | grep -v 'static\|const\|main'
```

**Global state conversion difficulty**:

| Pattern | Difficulty | Rust Approach |
|---------|-----------|---------------|
| Read-only globals | TRIVIAL | `const` or `static` |
| Write-once (initialized at startup) | EASY | `LazyLock` / `OnceLock` |
| Thread-local state | MODERATE | `thread_local!` |
| Mutex-protected state | MODERATE | `Mutex<T>` / `RwLock<T>` |
| Unprotected mutable global | HARD | Must add synchronization or restructure |
| Complex interconnected globals | HARD | Pass state explicitly, or `Arc<Mutex<State>>` |

---

## Risk Score Calculation

Combine metrics into a per-module risk score:

```
risk_score = (
  unsafe_density_score * 0.35 +      # Unsafe patterns (most impactful)
  complexity_score * 0.25 +           # Cyclomatic complexity
  global_state_score * 0.20 +         # Global mutable state
  dependency_score * 0.10 +           # External dependencies
  size_score * 0.10                   # Raw size
)
```

| Risk Score | Rating | Recommendation |
|-----------|--------|----------------|
| 0-25 | LOW | Good candidate for early conversion |
| 26-50 | MEDIUM | Standard conversion, expect some manual work |
| 51-75 | HIGH | Plan significant manual work, convert later |
| 76-100 | CRITICAL | Consider keeping as C with FFI bindings |
