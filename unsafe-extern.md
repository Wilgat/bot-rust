# Unsafe extern blocks

**Edition: 2024**

## Summary

In the **Rust 2024 Edition**, foreign function declarations inside `extern "C" { ... }` (and other ABI blocks) now require an explicit `unsafe` keyword:

```rust
// Allowed only in 2024 edition (and produces a lint warning in older editions)
unsafe extern "C" {
    fn printf(format: *const u8, ...) -> i32;
}

// Pre-2024 (still compiles in older editions, but now warned/denied)
extern "C" {
    fn printf(format: *const u8, ...) -> i32;
}
```

This change makes it immediately visible that calling such functions is unsafe and requires an `unsafe` block.

## Motivation / Problem (Pre-2024 Behavior)

Before the 2024 edition, you could declare foreign functions in safe code:

```rust
extern "C" {
    fn strlen(s: *const u8) -> usize;
    fn malloc(size: usize) -> *mut u8;
    fn free(ptr: *mut u8);
}

fn example(s: &str) -> usize {
    // Looks like safe code...
    unsafe { strlen(s.as_ptr()) }
}
```

**Key problems this caused**:

1. **Hidden unsafety at declaration site**  
   The fact that calling these functions was unsafe was **not visible** from the declaration itself. A reader had to know (or remember) that any function taking raw pointers, returning raw pointers, or interacting with C ABI was inherently unsafe — even though the declaration looked "safe."

2. **Misleading code appearance**  
   In large codebases, especially when foreign declarations were placed in a separate module or re-exported, it was easy to miss that a function required `unsafe` to call. This led to:
   - Accidental safe calls to FFI functions (causing UB)
   - Incorrect assumptions during code review
   - Difficulty teaching Rust's safety model to newcomers

3. **Inconsistency with Rust's safety philosophy**  
   Rust already requires `unsafe` for:
   - Dereferencing raw pointers
   - Calling other `unsafe fn`
   - Performing FFI calls that can cause UB  
   Yet the declaration of those very functions could be written in safe code — creating an unnecessary disconnect.

4. **Real-world confusion in libraries**  
   Many crates exposed `extern "C"` blocks in their public API without marking them `unsafe`, leading to frequent questions like:
   - "Why do I need `unsafe` to call this function?"
   - "Is this function actually safe to call?"

The change brings declaration-site safety in line with call-site safety.

## Details

In the **2024 Edition**:

- `extern "..." { ... }` blocks must be written as `unsafe extern "..." { ... }`
- The `unsafe` applies to the entire block — every function declared inside is treated as `unsafe fn`
- This is enforced by the new lint **`unsafe_extern`** (now `deny` by default in the 2024 edition)

Allowed forms in 2024:

```rust
unsafe extern "C" {
    fn c_func(ptr: *mut u8);
}

unsafe extern "system" fn callback(...) { ... }
```

## Migration

1. **Automatic migration** (recommended):

   ```bash
   cargo fix --edition
   ```

   Cargo’s edition fixer automatically adds the `unsafe` keyword to existing `extern` blocks.

2. **Manual fix** (if needed):

   - Search your codebase:

     ```bash
     rg '^extern\s+"[^"]+"\s*\{'
     ```

   - Add `unsafe` before every `extern` block (or before individual functions if using the function form).

3. **Pre-migration lint** (useful even before changing edition):

   ```rust
   #![deny(unsafe_extern)]
   ```

   This lint is `allow` by default in older editions but helps prepare code.

## Rationale

- Makes unsafety visible at both declaration **and** call sites.
- Reduces cognitive load: if you see `extern`, you immediately know `unsafe` is involved.
- Aligns foreign declarations with other unsafe constructs (e.g. `unsafe trait`, `unsafe impl`).
- Improves teachability: "anything involving raw pointers or C interop needs `unsafe`" becomes a simpler rule.

## See also

- [Unsafe extern lint](https://doc.rust-lang.org/rustc/lints/listing/allowed-by-default.html#unsafe-extern)
- [Extern blocks – Rust Reference](https://doc.rust-lang.org/reference/items/external-blocks.html)
- Official page: [Unsafe extern blocks](https://doc.rust-lang.org/edition-guide/rust-2024/unsafe-extern.html)
