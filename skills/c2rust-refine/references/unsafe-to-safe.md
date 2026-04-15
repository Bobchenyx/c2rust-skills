# Unsafe-to-Safe Rust Conversion Patterns

Reference patterns for reviewing and improving translated Rust code. While Claude-based translation typically produces idiomatic safe Rust from the start, these patterns are useful as a review checklist and for cases where FFI boundaries or complex C patterns result in remaining unsafe code.

---

## Priority Order

Apply these patterns in this order for best results:
1. Remove trivially unnecessary unsafe blocks
2. Replace raw pointers with references
3. Replace manual memory management with RAII
4. Replace C strings with Rust strings
5. Replace null checks with Option
6. Replace error codes with Result
7. Replace global mutable state
8. Replace C-style iteration with iterators
9. Reduce remaining unsafe scope

---

## Pattern 1: Remove Unnecessary Unsafe

Translated code may occasionally wrap operations in `unsafe` that don't actually need it, especially at FFI boundaries.

```rust
// BEFORE
unsafe {
    let x: i32 = 42;
    let y = x + 1;
    println!("{}", y);
}

// AFTER
let x: i32 = 42;
let y = x + 1;
println!("{}", y);
```

**Rule**: If a block contains no unsafe operations (raw pointer deref, FFI call, `union` field access, `static mut` access), remove the `unsafe` wrapper.

---

## Pattern 2: Raw Pointer → Reference

```rust
// BEFORE: raw pointer dereference
unsafe fn process(ptr: *mut Data) {
    (*ptr).field = 42;
    let val = (*ptr).other_field;
}

// AFTER: reference
fn process(data: &mut Data) {
    data.field = 42;
    let val = data.other_field;
}
```

**When safe to convert**:
- Pointer is known non-null (caller guarantees)
- Pointer is properly aligned (struct pointer from C allocation)
- No aliasing violations (no other `&mut` to same data)

**When NOT safe**:
- Pointer may be null → use `Option<&T>` or check first
- Pointer may be dangling → need lifetime analysis
- Multiple mutable pointers to same data → use `Cell`/`RefCell` or restructure

---

## Pattern 3: malloc/free → Box/Vec

### Single allocation

```rust
// BEFORE
let ptr = unsafe { libc::malloc(std::mem::size_of::<Node>()) as *mut Node };
unsafe { (*ptr).value = 42; }
// ... later ...
unsafe { libc::free(ptr as *mut libc::c_void); }

// AFTER
let mut node = Box::new(Node { value: 42, ..Default::default() });
// Automatically freed when `node` goes out of scope
```

### Array allocation

```rust
// BEFORE
let n = 100;
let arr = unsafe { libc::malloc(n * std::mem::size_of::<i32>()) as *mut i32 };
for i in 0..n {
    unsafe { *arr.add(i) = i as i32; }
}
// ... later ...
unsafe { libc::free(arr as *mut libc::c_void); }

// AFTER
let mut arr: Vec<i32> = (0..100).map(|i| i as i32).collect();
// Or if you need uninitialized memory:
let mut arr: Vec<i32> = Vec::with_capacity(100);
```

### realloc

```rust
// BEFORE
let new_ptr = unsafe {
    libc::realloc(ptr as *mut libc::c_void, new_size) as *mut i32
};

// AFTER
vec.resize(new_len, 0);  // or vec.reserve(additional)
```

---

## Pattern 4: C Strings → Rust Strings

### Internal string handling

```rust
// BEFORE
let s: *mut i8 = unsafe {
    libc::malloc(256) as *mut i8
};
unsafe {
    libc::strcpy(s, b"hello\0".as_ptr() as *const i8);
    let len = libc::strlen(s);
}

// AFTER
let s = String::from("hello");
let len = s.len();
```

### String comparison

```rust
// BEFORE
if unsafe { libc::strcmp(s1, s2) } == 0 { ... }

// AFTER
if s1 == s2 { ... }
```

### String formatting

```rust
// BEFORE
let buf: [i8; 256] = [0; 256];
unsafe { libc::snprintf(buf.as_ptr() as *mut i8, 256, fmt, args...) };

// AFTER
let s = format!("pattern {}", value);
```

### At FFI boundaries (keep C strings)

```rust
// When passing to C functions, still need CString
let c_str = CString::new(rust_string)?;
unsafe { c_function(c_str.as_ptr()) };

// When receiving from C functions
let rust_str = unsafe { CStr::from_ptr(c_ptr) }.to_str()?;
```

---

## Pattern 5: Null Checks → Option

```rust
// BEFORE
let ptr = find_item(key);
if ptr.is_null() {
    return -1;  // not found
}
let item = unsafe { &*ptr };

// AFTER
let item = match find_item(key) {
    Some(item) => item,
    None => return Err(Error::NotFound),
};

// The function signature changes:
// BEFORE: fn find_item(key: i32) -> *mut Item
// AFTER:  fn find_item(key: i32) -> Option<&Item>
```

### Nullable struct fields

```rust
// BEFORE
struct Node {
    next: *mut Node,  // NULL means end of list
    data: *mut Data,  // NULL means no data
}

// AFTER
struct Node {
    next: Option<Box<Node>>,
    data: Option<Box<Data>>,
}
```

---

## Pattern 6: Error Codes → Result

```rust
// BEFORE (C-style)
fn process_file(path: *const c_char) -> c_int {
    let fd = unsafe { libc::open(path, libc::O_RDONLY) };
    if fd < 0 { return -1; }
    
    let ret = unsafe { do_something(fd) };
    if ret < 0 {
        unsafe { libc::close(fd); }
        return -2;
    }
    
    unsafe { libc::close(fd); }
    return 0;
}

// AFTER (Rust-style)
fn process_file(path: &str) -> Result<(), ProcessError> {
    let mut file = File::open(path).map_err(ProcessError::Io)?;
    do_something(&mut file).map_err(ProcessError::Processing)?;
    Ok(())
    // file is automatically closed via Drop
}

#[derive(Debug, thiserror::Error)]
enum ProcessError {
    #[error("I/O error: {0}")]
    Io(#[from] std::io::Error),
    #[error("processing failed: {0}")]
    Processing(String),
}
```

### Propagation pattern (replacing goto cleanup)

```rust
// BEFORE (C goto cleanup pattern)
int process() {
    int ret = -1;
    Resource *r1 = acquire_r1();
    if (!r1) goto cleanup;
    
    Resource *r2 = acquire_r2();
    if (!r2) goto cleanup;
    
    ret = do_work(r1, r2);
    
cleanup:
    if (r2) release_r2(r2);
    if (r1) release_r1(r1);
    return ret;
}

// AFTER (Rust RAII + Result)
fn process() -> Result<i32, Error> {
    let r1 = acquire_r1()?;  // Auto-released via Drop
    let r2 = acquire_r2()?;  // Auto-released via Drop
    do_work(&r1, &r2)
    // r2 dropped, then r1 dropped — reverse order, automatic
}
```

---

## Pattern 7: Global Mutable State

### Read-only global → const/static

```rust
// BEFORE
static mut CONFIG_PATH: *const c_char = b"/etc/app.conf\0".as_ptr() as *const c_char;

// AFTER
const CONFIG_PATH: &str = "/etc/app.conf";
```

### Write-once global → OnceLock/LazyLock

```rust
// BEFORE
static mut GLOBAL_CONFIG: *mut Config = std::ptr::null_mut();
fn init() { unsafe { GLOBAL_CONFIG = Box::into_raw(Box::new(Config::load())); } }
fn get_config() -> &Config { unsafe { &*GLOBAL_CONFIG } }

// AFTER (Rust 1.80+)
static GLOBAL_CONFIG: LazyLock<Config> = LazyLock::new(|| Config::load());
fn get_config() -> &'static Config { &GLOBAL_CONFIG }

// Or with OnceLock for fallible init:
static GLOBAL_CONFIG: OnceLock<Config> = OnceLock::new();
fn init() -> Result<(), Error> {
    GLOBAL_CONFIG.set(Config::load()?).map_err(|_| Error::AlreadyInit)
}
```

### Shared mutable state → Mutex/RwLock

```rust
// BEFORE
static mut COUNTER: i32 = 0;
fn increment() { unsafe { COUNTER += 1; } }
fn get_count() -> i32 { unsafe { COUNTER } }

// AFTER
static COUNTER: Mutex<i32> = Mutex::new(0);
fn increment() { *COUNTER.lock().unwrap() += 1; }
fn get_count() -> i32 { *COUNTER.lock().unwrap() }

// For simple counters, AtomicI32 is better:
static COUNTER: AtomicI32 = AtomicI32::new(0);
fn increment() { COUNTER.fetch_add(1, Ordering::SeqCst); }
fn get_count() -> i32 { COUNTER.load(Ordering::SeqCst); }
```

### State struct pattern (eliminate multiple globals)

```rust
// BEFORE: scattered globals
static mut DB_CONN: *mut Connection = null_mut();
static mut CACHE: *mut HashMap = null_mut();
static mut LOGGER: *mut Logger = null_mut();

// AFTER: single state struct
struct AppState {
    db: Connection,
    cache: HashMap<String, Value>,
    logger: Logger,
}

// Pass state explicitly instead of using globals
fn process(state: &mut AppState, input: &str) -> Result<(), Error> {
    state.logger.info("processing");
    let cached = state.cache.get(input);
    // ...
}
```

---

## Pattern 8: C-style Iteration → Iterators

### Array iteration

```rust
// BEFORE
for i in 0..n {
    let item = unsafe { *arr.add(i) };
    process(item);
}

// AFTER
for item in arr.iter() {
    process(*item);
}
// Or with map/filter:
let results: Vec<_> = arr.iter().filter(|x| x.is_valid()).map(|x| x.transform()).collect();
```

### Linked list traversal

```rust
// BEFORE
let mut current = head;
while !current.is_null() {
    let node = unsafe { &*current };
    process(node.data);
    current = node.next;
}

// AFTER (if converted to Option<Box<Node>>)
let mut current = &head;
while let Some(node) = current {
    process(&node.data);
    current = &node.next;
}
```

### Index-based string iteration

```rust
// BEFORE
let mut i = 0;
while unsafe { *s.add(i) } != 0 {
    let ch = unsafe { *s.add(i) } as u8 as char;
    process(ch);
    i += 1;
}

// AFTER
for ch in s.chars() {
    process(ch);
}
```

---

## Pattern 9: Reduce Unsafe Scope

When some unsafe is truly needed, minimize its scope:

```rust
// BEFORE (entire function is unsafe)
unsafe fn big_function(ptr: *mut Data) {
    let data = &mut *ptr;
    let processed = data.field * 2;  // safe operation
    let result = format!("{}", processed);  // safe operation
    external_c_function(result.as_ptr());
}

// AFTER (only unsafe where needed)
fn big_function(ptr: *mut Data) -> Result<(), Error> {
    let data = unsafe { &mut *ptr };  // Minimal unsafe: just the deref
    let processed = data.field * 2;
    let result = CString::new(format!("{}", processed))?;
    unsafe { external_c_function(result.as_ptr()) };  // Minimal unsafe: FFI call
    Ok(())
}
```

**Principle**: Each `unsafe` block should contain exactly one unsafe operation, with a comment explaining why it's safe.
