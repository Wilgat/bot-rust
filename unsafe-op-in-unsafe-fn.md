# unsafe_op_in_unsafe_fn warning

**Edition: 2024**

## Summary

In the **Rust 2024 Edition**, the `unsafe_op_in_unsafe_fn` lint is now **warn-by-default** (previously it was `allow` by default).

This lint triggers whenever an `unsafe` operation is performed directly inside an `unsafe fn` without being wrapped in an explicit `unsafe { ... }` block.

```rust
unsafe fn example(ptr: *mut i32) {
    // Pre-2024: no warning
    *ptr = 42;                    // unsafe op in unsafe fn → now warns

    // 2024 style (recommended / silences lint)
    unsafe { *ptr = 42; }
}
```

The lint encourages authors of `unsafe fn` to be explicit about which parts of the function body actually perform unsafe operations.

## Motivation / Problem (Pre-2024 Behavior)

Before the 2024 edition, it was very common (and completely allowed without warning) to write:

```rust
unsafe fn set(ptr: *mut i32, value: i32) {
    *ptr = value;               // unsafe operation
    // ... hundreds of lines of safe Rust code ...
}
```

**Key problems this caused**:

1. **Unsafe operations were easy to miss**  
   In a large `unsafe fn`, a single raw pointer dereference, FFI call, or other unsafe operation could be buried deep in the function. Reviewers and future maintainers often overlooked these lines because the surrounding code looked like normal safe Rust.

2. **False sense of encapsulation**  
   Developers sometimes believed that writing `unsafe fn` was enough to "contain" all unsafety — leading to code where unsafe operations were performed unintentionally or without proper justification.

3. **Poor auditability**  
   When auditing safety-critical code, it was hard to quickly answer the question:  
   *"Which exact lines in this function can cause undefined behavior?"*  
   You had to read the entire function body carefully. Explicit `unsafe { ... }` blocks act as visual boundaries that make unsafe operations stand out.

4. **Inconsistent style across the ecosystem**  
   Some crates/libraries used explicit `unsafe {}` even in `unsafe fn`, while others did not — leading to stylistic inconsistency and confusion when reading different codebases.

The community (and the language team) concluded that `unsafe fn` should signal *capability* (this function may perform unsafe operations), but should not silently allow unsafe actions without drawing attention to them.

## Details

In the **2024 Edition**:

- `unsafe_op_in_unsafe_fn` is now **`warn`** by default.
- Triggered when any of the following appear directly in an `unsafe fn` without an inner `unsafe` block:
  - Dereference of raw pointer (`*ptr`, `*mut T`, etc.)
  - Call to another `unsafe fn`
  - Inline assembly (`asm!`, `global_asm!`)
  - Use of `std::ptr` / `std::mem` unsafe APIs
  - Mutable static access
  - Other operations requiring `unsafe`

Recommended style (silences the lint):

```rust
unsafe fn write_through(ptr: *mut u8, byte: u8) {
    // Only this part is unsafe
    unsafe {
        *ptr = byte;
    }

    // Rest of the function can be safe Rust
    log!("Wrote byte {:02x}", byte);
}
```

or (when the entire function body is unsafe):

```rust
unsafe fn memcpy(dst: *mut u8, src: *const u8, len: usize) {
    unsafe {
        std::ptr::copy_nonoverlapping(src, dst, len);
    }
}
```

## Migration

1. **Automatic migration** (partial):

   ```bash
   cargo fix --edition
   ```

   `cargo fix` will **not** automatically wrap operations in `unsafe {}` (too semantic), but it can help apply `#![warn(unsafe_op_in_unsafe_fn)]` or fix trivial cases.

2. **Recommended workflow**:

   - Add `#![warn(unsafe_op_in_unsafe_fn)]` to crates **before** changing edition
   - Fix warnings gradually by wrapping unsafe operations
   - After switching to 2024 edition, the lint becomes `warn` automatically

3. **Silence lint selectively** (not recommended for new code):

   ```rust
   #[allow(unsafe_op_in_unsafe_fn)]
   unsafe fn legacy_code() { ... }
   ```

## Rationale

- Makes unsafe operations **visually distinct** even inside `unsafe fn`
- Improves code review and long-term maintainability
- Reduces risk of accidentally introducing UB in what looks like "mostly safe" code
- Brings Rust closer to the ideal: every place where UB can occur should be clearly marked

## See also

- [unsafe_op_in_unsafe_fn lint documentation](https://doc.rust-lang.org/rustc/lints/listing/allowed-by-default.html#unsafe-op-in-unsafe-fn)
- [Unsafe Rust chapter – The Rust Book](https://doc.rust-lang.org/book/ch19-01-unsafe-rust.html)
- Official page: [unsafe_op_in_unsafe_fn warning](https://doc.rust-lang.org/edition-guide/rust-2024/unsafe-op-in-unsafe-fn.html)
