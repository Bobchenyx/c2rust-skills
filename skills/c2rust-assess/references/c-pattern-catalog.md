# C Pattern Catalog for Conversion Assessment

Catalog of C patterns with conversion difficulty ratings and recommended Rust approaches.

## Difficulty Scale

| Rating | Meaning |
|--------|---------|
| TRIVIAL | Claude translates automatically, output is idiomatic |
| EASY | Straightforward translation, minor review needed |
| MODERATE | Requires design decisions during translation |
| HARD | Complex translation, may produce suboptimal code that needs manual review |
| BLOCKING | Cannot be auto-translated, requires fundamental manual redesign |

---

## Memory Management Patterns

### malloc/calloc/realloc/free
- **Difficulty**: EASY
- **Detection**: `grep -rn '\b(malloc|calloc|realloc|free)\s*(' *.c`
- **Translation approach**: `Box::new()`, `Vec::with_capacity()`, RAII
- **Risk**: Memory leaks if free paths are complex; double-free if aliased

### Custom allocators / memory pools
- **Difficulty**: HARD
- **Detection**: Custom `alloc`/`pool_alloc` functions, slab allocators
- **Translation approach**: Custom `Allocator` trait impl, `bumpalo`, or `typed-arena`
- **Risk**: Complex lifetime relationships

### Stack-allocated variable-length arrays (VLA)
- **Difficulty**: MODERATE
- **Detection**: `int arr[n]` where n is not const
- **Translation approach**: `Vec` on heap, or `smallvec` for optimization

---

## Pointer Patterns

### Simple pointer dereference
- **Difficulty**: TRIVIAL
- **Detection**: Standard `*ptr` usage
- **Translation approach**: References `&T` / `&mut T`

### Pointer arithmetic
- **Difficulty**: MODERATE
- **Detection**: `ptr + offset`, `ptr++`, `ptr[i]`
- **Translation approach**: Slices, iterators, `.get()` with bounds checking

### Double/triple pointers (`**ptr`, `***ptr`)
- **Difficulty**: MODERATE
- **Detection**: Multi-level pointer declarations
- **Translation approach**: `&mut &mut T`, `Box<Box<T>>`, or redesign with proper ownership

### Void pointers (`void *`)
- **Difficulty**: HARD
- **Detection**: `void *` parameters, generic data structures
- **Translation approach**: Generics `<T>`, trait objects `dyn Trait`, `Any`

### Function pointers
- **Difficulty**: MODERATE
- **Detection**: `typedef void (*callback)(int)`, function pointer fields, `(*fp)(...)`
- **Translation approach**: `fn(...)` types, `Fn`/`FnMut`/`FnOnce` trait objects, closures

### Pointer casts between types
- **Difficulty**: HARD
- **Detection**: `(struct foo *)ptr`, type-punning through unions
- **Translation approach**: `transmute` (still unsafe) or redesign to avoid type punning

---

## Control Flow Patterns

### goto statements (forward / cleanup)
- **Difficulty**: EASY
- **Detection**: `grep -rn '\bgoto\b' *.c`
- **Translation approach**: `?` operator + RAII, early return, labeled blocks
- **Note**: Forward gotos for cleanup are **benign** — Claude handles them well with `?` and `Result`. Do not weight these heavily in risk scoring.

### goto statements (backward / loop)
- **Difficulty**: HARD
- **Detection**: goto targets that appear before the goto statement
- **Translation approach**: `loop` / `while` / labeled loops

### Computed goto / labels-as-values
- **Difficulty**: BLOCKING
- **Detection**: `&&label`, `goto *ptr`
- **Translation approach**: Manual rewrite to match/enum state machine

### setjmp / longjmp
- **Difficulty**: BLOCKING
- **Detection**: `#include <setjmp.h>`, `setjmp()`, `longjmp()`
- **Translation approach**: `Result<T, E>` for error propagation, `panic!`/`catch_unwind` as last resort
- **Risk**: Used for error handling OR for coroutine-like patterns — different solutions needed

### Signal handlers
- **Difficulty**: HARD
- **Detection**: `signal()`, `sigaction()`, `sig_atomic_t`
- **Translation approach**: `signal-hook` crate, or `ctrlc` for simple cases

---

## String Handling

### C strings (null-terminated)
- **Difficulty**: EASY
- **Detection**: `char *`, `strlen`, `strcpy`, `strcat`, `strcmp`
- **Translation approach**: `String`, `&str`, `CStr`, `CString` at FFI boundaries

### String formatting (sprintf, snprintf)
- **Difficulty**: MODERATE
- **Detection**: `sprintf`, `snprintf`, `fprintf`
- **Translation approach**: `format!()`, `write!()` macros

### Character-by-character manipulation
- **Difficulty**: EASY
- **Detection**: `s[i]`, `*s++` character iteration
- **Translation approach**: `.chars()`, `.bytes()` iterators, `char` methods

---

## Preprocessor / Macros

### Simple constant macros
- **Difficulty**: TRIVIAL
- **Detection**: `#define FOO 42`
- **Translation approach**: `const FOO: i32 = 42;`

### Function-like macros
- **Difficulty**: MODERATE
- **Detection**: `#define MAX(a,b) ((a)>(b)?(a):(b))`
- **Translation approach**: `inline fn`, `macro_rules!`, or generic function

### X-macros / token pasting
- **Difficulty**: HARD
- **Detection**: `#define X(name) name##_init`, X-macro tables
- **Translation approach**: `macro_rules!` with `paste` crate, or proc macros

### Conditional compilation (#ifdef)
- **Difficulty**: MODERATE
- **Detection**: `#ifdef`, `#if defined()`
- **Translation approach**: `cfg` attributes, feature flags
- **Note**: Only one configuration is active at a time; translations target the primary platform

### Include guards / header structure
- **Difficulty**: TRIVIAL
- **Detection**: `#ifndef _FOO_H_`
- **Translation approach**: Rust module system handles this natively

---

## Concurrency Patterns

### pthreads
- **Difficulty**: MODERATE
- **Detection**: `pthread_create`, `pthread_mutex_lock`, `pthread_cond_wait`
- **Translation approach**: `std::thread`, `Mutex`, `Condvar`, `Arc`

### Atomic operations
- **Difficulty**: MODERATE
- **Detection**: `__atomic_*`, `__sync_*`, `<stdatomic.h>`
- **Translation approach**: `std::sync::atomic` types

### Thread-local storage
- **Difficulty**: MODERATE
- **Detection**: `__thread`, `_Thread_local`, `pthread_key_create`
- **Translation approach**: `thread_local!` macro, `std::cell::RefCell`

### Global mutable state
- **Difficulty**: HARD
- **Detection**: `static` variables modified by multiple functions
- **Translation approach**: `Mutex<T>`, `RwLock<T>`, `OnceCell`/`LazyLock`, or redesign to pass state explicitly

---

## Data Structure Patterns

### Structs with flexible array members
- **Difficulty**: HARD
- **Detection**: `struct { int len; char data[]; }`
- **Translation approach**: `Vec<u8>` field, or custom DST with `#[repr(C)]`

### Unions
- **Difficulty**: HARD
- **Detection**: `union { ... }`
- **Translation approach**: `enum` with variants where possible, or keep as union with safe accessor methods

### Bitfields
- **Difficulty**: HARD
- **Detection**: `struct { unsigned int flag : 1; }`
- **Translation approach**: `bitflags` crate, manual bit manipulation, or `modular-bitfield`

### Linked lists / intrusive data structures
- **Difficulty**: HARD
- **Detection**: `struct node { struct node *next; }`, Linux-style `list_head`
- **Translation approach**: `Vec`, `VecDeque`, `LinkedList`, or unsafe with `Pin`

### Opaque types (incomplete struct declarations)
- **Difficulty**: EASY
- **Detection**: `struct foo;` (forward declaration only)
- **Translation approach**: Newtype wrapper or `extern type` (nightly)

---

## I/O and System Patterns

### File I/O (fopen, fread, fwrite)
- **Difficulty**: EASY
- **Detection**: `fopen`, `fread`, `fwrite`, `fclose`
- **Translation approach**: `std::fs::File`, `BufReader`, `BufWriter`

### Socket programming
- **Difficulty**: MODERATE
- **Detection**: `socket()`, `bind()`, `listen()`, `accept()`
- **Translation approach**: `std::net::TcpListener/TcpStream`, or `tokio` for async

### mmap / shared memory
- **Difficulty**: HARD
- **Detection**: `mmap`, `shm_open`, `shmget`
- **Translation approach**: `memmap2` crate, careful lifetime management

### ioctl / system calls
- **Difficulty**: HARD
- **Detection**: `ioctl()`, `syscall()`
- **Translation approach**: `nix` crate or `libc` with safe wrappers

---

## Inline Assembly

### Basic inline asm
- **Difficulty**: BLOCKING
- **Detection**: `asm`, `__asm__`, `__asm`
- **Translation approach**: `core::arch::asm!` macro (requires manual rewrite)
- **Note**: Often used for performance-critical code, SIMD, or hardware access

### SIMD intrinsics
- **Difficulty**: HARD
- **Detection**: `#include <immintrin.h>`, `_mm_*` functions
- **Translation approach**: `core::arch::x86_64` intrinsics, or `packed_simd2` / `std::simd` (nightly)

---

## Variadic Functions

### va_list / va_start / va_end
- **Difficulty**: HARD
- **Detection**: `va_list`, `va_start`, `...` in function parameters
- **Translation approach**: Generics, trait-based dispatch, builder pattern, or macro_rules!
- **Note**: Rust does not natively support variadic functions except with `extern "C"`
