# Rustc Error Fix Catalog for Translated Rust Code

Common compiler errors encountered in C-to-Rust translated code and their fixes.

---

## Error Classification

| Category | Auto-fixable? | Description |
|----------|:------------:|-------------|
| MECHANICAL | Yes | Syntax, imports, simple type mismatches |
| STRUCTURAL | Partially | Requires restructuring but follows clear rules |
| SEMANTIC | No | Requires design decisions, ask user |

---

## MECHANICAL Errors (Auto-Fix)

### E0432: unresolved import

**Cause**: Translation may produce `use` statements that don't match the Rust module structure.

```
error[E0432]: unresolved import `libc::types`
```

**Fix**: Update import paths to match actual module layout.
```rust
// BEFORE
use libc::types::os::arch::c95::*;
// AFTER
use libc::*;
```

Also check: module might need `mod module_name;` declaration in parent.

---

### E0308: mismatched types

**Cause**: C's implicit integer conversions don't exist in Rust.

```
error[E0308]: mismatched types
  expected `u32`, found `i32`
```

**Fixes**:
```rust
// Integer widening: add explicit cast
let x: u32 = y as u32;

// Bool to integer
let x: i32 = if flag { 1 } else { 0 };
// Or: let x: i32 = flag as i32;

// Integer to bool
let flag: bool = x != 0;

// Pointer to integer (rare, keep unsafe)
let addr: usize = ptr as usize;
```

---

### E0425: cannot find value/function

**Cause**: Missing function/constant definition, or not imported.

```
error[E0425]: cannot find function `helper_func` in this scope
```

**Fixes**:
- Add `use crate::module::helper_func;`
- Add `mod module;` if the module isn't declared
- If from C: add `extern "C" { fn helper_func(...); }`
- If a macro: ensure `#[macro_use]` or `use module::macro_name;`

---

### E0412: cannot find type

**Cause**: Type not in scope or not yet defined.

```
error[E0412]: cannot find type `size_t` in this scope
```

**Common fixes for C-originated types**:
```rust
use libc::{size_t, ssize_t, c_int, c_char, c_void, c_uint, c_long};
// Or use Rust equivalents:
// size_t → usize
// ssize_t → isize
// c_int → i32 (platform-dependent, but usually)
```

---

### E0277: trait not satisfied

**Cause**: Translated structs missing common trait derives.

```
error[E0277]: `MyStruct` doesn't implement `Debug`
```

**Fix**: Add derive macros.
```rust
#[derive(Debug, Clone, Default)]
#[repr(C)]
struct MyStruct { ... }

// For Copy types (no heap allocation):
#[derive(Debug, Clone, Copy, Default, PartialEq)]
#[repr(C)]
struct SimpleStruct { ... }
```

---

### E0382: use of moved value

**Cause**: C allows reusing variables after "logical move"; Rust doesn't.

```
error[E0382]: use of moved value: `data`
```

**Fixes**:
```rust
// If type is Copy-able, add #[derive(Copy, Clone)]
// If not, use .clone() at the move point:
let data2 = process(data.clone());
use_again(&data);

// Or restructure to use references:
let result = process(&data);
use_again(&data);
```

---

### E0061: wrong number of arguments

**Cause**: Function signature mismatch, often from variadic C functions.

```
error[E0061]: this function takes 2 arguments but 3 arguments were supplied
```

**Fix**: Check if the C function was variadic (`...`). Variadic functions in Rust require special handling:
```rust
// If truly variadic, keep as extern "C"
extern "C" {
    fn printf(fmt: *const c_char, ...) -> c_int;
}
```

---

### E0599: no method named X on type Y

**Cause**: Translation sometimes produces method calls that don't exist on the target type.

**Fix**: Usually needs `as` cast or explicit trait method:
```rust
// BEFORE: x.wrapping_add(1)  (if x is wrong type)
// AFTER: (x as u32).wrapping_add(1)
```

---

### E0658: feature not stable

**Cause**: Translation sometimes uses nightly-only features.

```
error[E0658]: `extern_types` is experimental
```

**Fixes**:
- Add `#![feature(feature_name)]` to lib.rs (requires nightly)
- Or replace with stable alternatives:
  - `extern type` → opaque struct: `#[repr(C)] pub struct OpaqueType { _private: [u8; 0] }`
  - `c_variadic` → keep as extern "C" fn

---

## STRUCTURAL Errors (Partially Auto-fixable)

### E0499: multiple mutable borrows

**Cause**: C code routinely aliases mutable pointers; Rust forbids this.

```
error[E0499]: cannot borrow `*self` as mutable more than once at a time
```

**Fixes (in order of preference)**:
```rust
// 1. Split borrows: borrow different fields
let a = &mut self.field_a;
let b = &mut self.field_b;  // OK: different fields

// 2. Restructure: do one operation at a time
let temp = self.get_value();
self.set_other(temp);

// 3. Use Cell/RefCell for interior mutability
use std::cell::RefCell;
struct MyStruct {
    data: RefCell<Vec<i32>>,
}

// 4. Use indices instead of references
let idx = find_index(&self.items, key);
self.items[idx].modify();
```

---

### E0502: borrow conflict (immutable + mutable)

**Cause**: Reading and writing same data simultaneously.

```
error[E0502]: cannot borrow `v` as mutable because it is also borrowed as immutable
```

**Fixes**:
```rust
// 1. Clone the immutable borrow
let val = v[0].clone();
v.push(val);

// 2. Use indices
let val = v[0];  // Copy if Copy type
v.push(val);

// 3. Split the operation
let items_to_add: Vec<_> = v.iter().filter(|x| x.needs_copy()).cloned().collect();
v.extend(items_to_add);
```

---

### E0106: missing lifetime specifier

**Cause**: Functions returning references without explicit lifetimes.

```
error[E0106]: missing lifetime specifier
```

**Fixes**:
```rust
// Simple case: input lifetime = output lifetime
fn get_name(item: &Item) -> &str {
    &item.name
}

// Multiple inputs: specify which input the output borrows from
fn longest<'a>(a: &'a str, b: &str) -> &'a str {
    a  // output borrows from first input
}

// Struct with references:
struct Parser<'a> {
    input: &'a str,
}
```

---

### E0515: cannot return reference to local variable

**Cause**: Functions that return pointers to stack-local data (valid in C if caller copies, invalid in Rust).

```
error[E0515]: cannot return reference to local variable `result`
```

**Fixes**:
```rust
// Return owned data instead of reference
fn compute() -> String {  // not &str
    let result = format!("computed");
    result  // ownership transferred to caller
}

// Or return through output parameter
fn compute(output: &mut String) {
    *output = format!("computed");
}
```

---

## SEMANTIC Errors (Require User Decision)

### Ownership model for shared data

**Symptom**: Multiple errors about moves, borrows, lifetimes around shared data structures.

**Options to present to user**:
1. `Rc<T>` / `Arc<T>` — Reference counted (single-thread / multi-thread)
2. Clone the data — Simple but potentially expensive
3. Redesign with indices — Use arena + index pattern
4. Keep raw pointers — Maintain unsafe, add safety documentation

---

### Error handling strategy

**Symptom**: Many functions return `c_int` error codes, needs unified approach.

**Options to present to user**:
1. `anyhow::Result` — Quick, flexible, good for applications
2. `thiserror` custom errors — Type-safe, good for libraries
3. Manual enum — Maximum control, no dependencies
4. Keep C-style codes — At FFI boundaries, convert at wrapper layer

---

### Global state management

**Symptom**: `static mut` variables causing unsafe access warnings everywhere.

**Options to present to user**:
1. Pass state explicitly — Cleanest but biggest refactor
2. `Mutex<T>` / `RwLock<T>` — Thread-safe global, minimal refactor
3. `thread_local!` — If state is thread-specific
4. State struct + dependency injection — Balance of clean design and effort

---

### Container type choice

**Symptom**: C linked list / tree converted to raw pointer chains.

**Options to present to user**:
1. `Vec<T>` — Usually the best default, cache-friendly
2. `VecDeque<T>` — If front insertion/removal is common
3. `HashMap<K, V>` — If lookup by key is primary use
4. `BTreeMap<K, V>` — If ordered iteration is needed
5. Custom with `unsafe` — If pointer-based structure is performance-critical

---

## Linker Errors

### undefined reference to `c_function_name`

**Cause**: C function not linked.

**Fixes**:
```rust
// In build.rs, ensure C files are compiled:
cc::Build::new()
    .file("src/missing_module.c")
    .compile("missing_module");

// Or link system library:
println!("cargo:rustc-link-lib=name_of_lib");

// Or add to Cargo.toml:
// [dependencies]
// name-sys = "version"
```

### multiple definition of `symbol`

**Cause**: Same C function defined in multiple translation units (common with static inline in headers).

**Fix**: Make the function `static` in the C source, or consolidate to single definition. In Rust: ensure only one `#[no_mangle]` definition exists.

### undefined reference to `__rust_alloc` (or other Rust symbols)

**Cause**: Linking Rust staticlib without the Rust standard library.

**Fix**: Link with `+whole-archive` or ensure `libstd` is linked:
```bash
# In build script or Makefile:
RUSTFLAGS="-C prefer-dynamic" cargo build
```
