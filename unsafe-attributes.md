# Unsafe attributes

**Edition: 2024**

## Summary

In the **Rust 2024 Edition**, several attributes that can have a strong impact on program safety (especially memory safety and undefined behavior) are now **unsafe to use** outside of an `unsafe` context.

The affected attributes are:

- `#[target_feature(...)]`
- `#[naked]`
- `#[link_section = "..."]`
- `#[no_mangle]`
- `#[export_name = "..."]`

Trying to apply any of these attributes in safe code now produces a **hard error** in the 2024 edition:

```rust
#[target_feature(enable = "avx2")]   // error in 2024 edition
fn add(a: f32, b: f32) -> f32 {
    a + b
}
```

Correct usage (2024 edition):

```rust
unsafe {
    #[target_feature(enable = "avx2")]
    fn add(a: f32, b: f32) -> f32 {
        a + b
    }
}
```

or (preferred style):

```rust
#[target_feature(enable = "avx2")]
unsafe fn add(a: f32, b: f32) -> f32 {
    a + b
}
```

## Motivation / Problem (Pre-2024 Behavior)

Before the 2024 edition, all of the above attributes could be applied in **safe** code:

```rust
#[target_feature(enable = "avx2")]
fn fast_add(a: f32, b: f32) -> f32 {
    // uses AVX2 instructions — UB if CPU doesn't support it
    a + b
}

#[naked]
fn custom_asm() {
    // no prologue/epilogue — UB if ABI is violated
    asm!("...");
}

#[no_mangle]
pub static mut GLOBAL: u32 = 0;   // safe code can create aliasing issues
```

**Key problems this caused**:

1. **Hidden undefined behavior**  
   Attributes like `#[target_feature]` can cause **undefined behavior** (e.g., illegal instruction exceptions, segfaults, or worse) if the feature is not available on the target CPU. Yet the function itself could be marked `safe` and called from safe code — violating Rust’s core guarantee that safe code cannot cause UB.

2. **False sense of safety**  
   Developers (and reviewers) could easily miss that a function or static was unsafe to call just by looking at the signature — the danger was hidden in an attribute far from the function body.

3. **Inconsistent safety model**  
   Rust already requires `unsafe` for operations that can cause UB (raw pointer dereference, FFI calls, etc.). Allowing safety-impacting attributes in safe code created an inconsistency: some ways to introduce UB were guarded by `unsafe`, others were not.

4. **Particularly dangerous with generics / inlining**  
   A generic function with `#[target_feature]` could be instantiated and inlined into safe code paths, silently introducing UB without any `unsafe` keyword visible at the call site.

The community consensus (backed by multiple soundness issues and discussions) was that these attributes should be treated the same way as other UB-risky constructs — they belong inside `unsafe` blocks or on `unsafe fn`.

## Details

In the **2024 Edition**:

- The listed attributes are now **forbidden** outside of `unsafe` contexts.
- You can place them:
  - Inside an `unsafe { ... }` block (affects the inner item)
  - On an `unsafe fn` / `unsafe static` / `unsafe extern` item
- The change is enforced by the new lint **`unsafe_attributes`** (now `deny` by default in the 2024 edition).

## Migration

1. **Automatic migration** (recommended):

   ```bash
   cargo fix --edition
   ```

   Cargo’s edition fixer will wrap affected attributes in `unsafe { ... }` blocks or move them onto `unsafe fn` declarations where possible.

2. **Manual review** (strongly recommended):

   - After running `cargo fix`, check each change:
     - Does the function really need to be `unsafe` now?
     - Are the preconditions (e.g. CPU feature availability) documented and checked at runtime?
   - Search your codebase for the affected attributes:

     ```bash
     rg '#\[(target_feature|naked|link_section|no_mangle|export_name)'
     ```

3. **Pre-migration lint** (to prepare):

   ```rust
   #![deny(unsafe_attributes)]
   ```

   This lint is `allow` by default in older editions but very useful during transition.

## Rationale

- Aligns attribute usage with Rust’s safety model: if something can introduce **undefined behavior**, it should require `unsafe`.
- Makes dangerous code more visible during code review — every UB-risky attribute is now near an `unsafe` keyword.
- Prevents accidental UB from safe-looking code.
- Prepares the language for stricter future checks (e.g., runtime feature detection requirements).

## See also

- [Unsafe attributes lint](https://doc.rust-lang.org/rustc/lints/listing/allowed-by-default.html#unsafe-attributes)
- [Target feature – Rust Reference](https://doc.rust-lang.org/reference/abi.html#the-target_feature-attribute)
- Official page: [Unsafe attributes](https://doc.rust-lang.org/edition-guide/rust-2024/unsafe-attributes.html)
