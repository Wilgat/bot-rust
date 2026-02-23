# Disallow references to static mut

**Edition: 2024**

## Summary

In the **Rust 2024 Edition**, it is now a **hard error** to take a reference (`&` or `&mut`) to a `static mut` item **outside** of an `unsafe` block.

```rust
static mut COUNTER: u32 = 0;

// 2021 edition: allowed (but very dangerous)
let r = &COUNTER;           // error in 2024 edition
let rm = &mut COUNTER;      // error in 2024 edition

// Only allowed form in 2024 edition
let r = unsafe { &COUNTER };
let rm = unsafe { &mut COUNTER };
```

This forces developers to be explicit about the unsafety of accessing mutable statics, aligning with Rust’s safety guarantees.

## Motivation / Problem (Pre-2024 Behavior)

Before the 2024 edition, Rust allowed taking shared (`&`) and mutable (`&mut`) references to `static mut` items **in safe code**:

```rust
static mut GLOBAL: i32 = 42;

fn example() {
    let safe_ref: &i32 = &GLOBAL;           // allowed in 2015–2021 editions
    let safe_mut_ref: &mut i32 = &mut GLOBAL; // allowed in 2015–2021 editions
}
```

**This was extremely dangerous** because:

1. **No data-race protection**  
   `static mut` has no inherent synchronization. Taking `&mut` in safe code allowed multiple threads (or even reentrant calls on the same thread) to create aliased mutable references without any `unsafe` marker — directly violating Rust’s aliasing rules.

2. **False sense of safety**  
   Because the reference was created in safe code, the compiler did not require an `unsafe` block. This hid the fact that almost any subsequent use of the reference (dereference, store through it, pass it elsewhere) would require `unsafe` anyway — leading to subtle bugs and misleading code reviews.

3. **Encouraged bad patterns**  
   Libraries and tutorials sometimes showed `&static mut` in safe contexts, normalizing unsafe aliasing and making it harder for beginners to understand when `unsafe` is actually required.

4. **Soundness holes in generic / higher-rank code**  
   In some cases (especially with generic code or function pointers), safe references to `static mut` could flow into contexts where they caused undefined behavior without an obvious `unsafe` boundary.

The community consensus (and multiple soundness issues filed against the language) was that allowing safe references to `static mut` was a historical design mistake.

## Details

In the **2024 Edition**:

- Taking `&static mut T` or `&mut static mut T` **outside an `unsafe` block** is a **hard error**.
- The only legal way to obtain a reference remains:

```rust
unsafe {
    let r: &i32    = &GLOBAL;
    let rm: &mut i32 = &mut GLOBAL;
}
```

- This applies to both shared and mutable references.
- Raw pointers (`*const T`, `*mut T`) are unaffected — they still require `unsafe` to dereference, as before.

## Migration

1. **Search your codebase** for references to `static mut`:

```bash
rg '&\s*(mut\s+)?static\s+mut' src/
# or look for &GLOBAL, &mut GLOBAL where GLOBAL is static mut
```

2. **Wrap in `unsafe`**:

```rust
// Before (2021 edition)
let r = &GLOBAL;

// After (2024 edition)
let r = unsafe { &GLOBAL };
```

3. **`cargo fix --edition`** does **not** automatically wrap these — the change is too semantic and context-dependent.

4. **Consider safer alternatives** (strongly recommended):

   - Use `static` with `Mutex`, `RwLock`, `Atomic*`, or `UnsafeCell` + manual synchronization
   - Replace `static mut` with thread-local storage (`thread_local!`) when appropriate
   - Avoid global mutable state entirely when possible

## Rationale

- Makes it impossible to create aliased mutable references to `static mut` without an explicit `unsafe` block.
- Improves code auditability: every place that touches mutable static state is now visibly marked `unsafe`.
- Closes a long-standing soundness loophole and aligns `static mut` with Rust’s core aliasing rules.
- Prepares the language for potential future deprecation or redesign of `static mut`.

## See also

- [Unsafe blocks and operations](https://doc.rust-lang.org/book/ch19-01-unsafe-rust.html)
- [Static variables – Rust Reference](https://doc.rust-lang.org/reference/items/static-items.html)
- Official page: [Disallow references to static mut](https://doc.rust-lang.org/edition-guide/rust-2024/static-mut-references.html)
