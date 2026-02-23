Here is a clear Markdown explanation of the **Newly unsafe functions** change in the **Rust 2024 Edition**, based on the official Rust Edition Guide.

### Newly unsafe functions

#### Summary
In the **Rust 2024 Edition**, several functions in the standard library that were previously **safe** are now marked **`unsafe`** and require an `unsafe` block to call.

This change affects functions that can cause **undefined behavior (UB)** if misused (e.g., passing invalid pointers, violating aliasing rules, or dereferencing null/raw pointers without proper checks).

The affected functions now require explicit `unsafe { ... }` wrapping, making their potential danger more visible in the code.

#### Motivation / Problem (Pre-2024 Behavior)

Before Rust 2024, a number of standard library functions were declared **safe** even though they could easily lead to **undefined behavior** when called with invalid arguments.

The most prominent examples are raw pointer methods:

```rust
// Pre-2024: these were safe functions
ptr.is_null()                    // safe
ptr.as_ref()                     // safe
ptr.as_mut()                     // safe
ptr.offset(isize)                // safe
ptr.add(usize)                   // safe
ptr.sub(usize)                   // safe
ptr.wrapping_offset(isize)       // safe
ptr.wrapping_add(usize)          // safe
ptr.wrapping_sub(usize)          // safe
(non_null_ptr).as_ptr()          // safe (on NonNull<T>)
```

**Key problems this caused**:

1. **False sense of safety**
   - Developers assumed that because the function was marked `safe`, calling it could never cause UB.
   - In reality, many of these operations produce **UB** if:
     - The pointer is null (for `as_ref`/`as_mut`)
     - The offset goes out of bounds of the allocated object
     - The pointer is dangling / invalid / misaligned
     - Aliasing rules are violated

2. **Hidden undefined behavior**
   - Code like this compiled without `unsafe` but was still unsound:
     ```rust
     let ptr: *const i32 = std::ptr::null();
     let r = unsafe { ptr.as_ref().unwrap() };  // UB! null dereference
     // or even worse:
     let r = ptr.as_ref();                      // UB if null (pre-2024 accepted)
     ```

3. **Inconsistent safety story**
   - Rust's safety philosophy: **safe Rust should not cause UB**.
   - But these functions allowed safe code to trigger UB → contradiction.
   - Made it harder to reason about soundness audits (tools like Miri, unsafe analyses).

4. **Blocked stricter aliasing and pointer rules**
   - Future improvements to strict provenance, pointer aliasing rules, or `noalias` metadata were blocked because safe code could violate them.

5. **Real bugs in the wild**
   - Many crates and applications had subtle UB bugs hidden behind "safe" pointer methods.
   - Miri often caught these only when run in strict mode → not caught in normal builds.

By making these operations **unsafe**, Rust forces developers to acknowledge the danger and either:
- Use safe alternatives (`Option<&T>`, slices, etc.)
- Prove correctness inside `unsafe` blocks

#### What Changed in Rust 2024

The following functions (and their variants) are now **`unsafe`**:

| Function / Method                  | Old (pre-2024) | New (2024) | Main reason for unsafe |
|------------------------------------|----------------|------------|------------------------|
| `*const T::as_ref(&self) -> Option<&T>` | safe          | unsafe    | UB if pointer is null or invalid |
| `*mut T::as_mut(&mut self) -> Option<&mut T>` | safe      | unsafe    | UB if null, dangling, or aliasing violation |
| `*const T::offset(self, count: isize) -> *const T` | safe | unsafe    | UB if offset out of bounds of allocation |
| `*const T::add(self, count: usize) -> *const T` | safe   | unsafe    | Same as offset |
| `*mut T::add` / `sub` / `offset`   | safe          | unsafe    | Same |
| `*const T::wrapping_offset` / `wrapping_add` / `wrapping_sub` | safe | **still safe** | No UB — just wrapping arithmetic |
| `NonNull<T>::as_ptr(self) -> *const T` | safe      | unsafe    | UB if provenance is lost or pointer is invalid |
| `NonNull<T>::as_ref(&self) -> &T`  | safe          | unsafe    | UB if pointer was dangling/invalid |
| `NonNull<T>::as_mut(&mut self) -> &mut T` | safe   | unsafe    | Same |

**Important exceptions that remain safe**:
- `wrapping_*` methods (no UB, just modular arithmetic)
- `read` / `write` / `copy` / `copy_nonoverlapping` (already unsafe)
- `addr()` / `expose_addr()` / `from_exposed_addr()` (provenance-related, unsafe where appropriate)

#### Migration & Fixes

- **`cargo fix --edition`** does **not** automatically wrap calls in `unsafe` — too semantic.
- Common migration patterns:
  ```rust
  // Old (pre-2024)
  let r = ptr.as_ref().unwrap();

  // New (2024) – safe way
  let r = unsafe { ptr.as_ref().unwrap_unchecked() };  // if you proved non-null

  // Better: use safe patterns
  let Some(r) = (unsafe { ptr.as_ref() }) else { unreachable!() };
  ```

- Use Miri (`cargo miri test`) to find remaining UB after migration.
- Prefer safe abstractions (`&[T]`, `Option<&T>`, `NonNull` with checked methods) when possible.

#### Rationale

- **Honest safety guarantees**: If a function can cause UB, it must be `unsafe`.
- Clears the way for stricter pointer provenance, aliasing, and future soundness improvements.
- Makes code more auditable — every `unsafe` block is a place to check invariants.
- Aligns with Rust’s core principle: **safe code cannot cause undefined behavior**.

#### Official Reference
https://doc.rust-lang.org/edition-guide/rust-2024/newly-unsafe-functions.html

In short: The 2024 edition removes the last major exceptions where **safe Rust could trigger UB** — a philosophically important cleanup that makes Rust’s safety story stronger and more trustworthy.