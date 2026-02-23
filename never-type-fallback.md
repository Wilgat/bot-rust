Here is a clear Markdown explanation of the **Never type fallback change** in the **Rust 2024 Edition**, based on the official Rust Edition Guide.

### Never type fallback change

#### Summary
In the **Rust 2024 Edition**, the fallback behavior for the **never type** (`!`) during type coercion and inference has changed:

- **Pre-2024 editions**: When `!` appeared at a coercion site and the target type could not be inferred, it fell back to the **unit type** `()`.
- **Rust 2024 Edition**: The fallback is now **`!`** itself.

This prevents spurious and confusing coercions of `!` → `()` and makes the never type behave more predictably and intuitively. The change is currently edition-specific (2024 only), with plans to apply it across all editions later.

Additionally, the lint **`never_type_fallback_flowing_into_unsafe`** is now **`deny`** by default in the 2024 edition (previously `warn`), to catch cases where the new fallback could lead to `!` flowing into `unsafe` code and potentially cause undefined behavior.

#### Motivation / Problem (Pre-2024 Behavior)

Historically, when the compiler saw a value of type `!` at a **coercion site** (e.g., in a function return, generic argument, or expression position), it inserted an implicit helper like this:

```rust
fn absurd<T>(x: !) -> T { x }  // hypothetical internal helper
```

If the target type `T` could not be inferred from context, the compiler arbitrarily chose **`T = ()`** as the fallback. This caused `!` to **spontaneously coerce to `()`** even in places where `()` would **not** have been inferred naturally without the fallback rule.

**Key problems this caused**:

1. **Confusing and non-intuitive inference**
   - Code that "looked like" it should diverge (never return) suddenly inferred as returning `()` — breaking the semantic meaning of `!` ("computation that never produces a value").
   - Example that previously compiled but was surprising:
     ```rust
     if true {
         Default::default()
     } else {
         return   // or panic!(), unreachable!(), etc.
     };
     // → Inferred as () due to ! → () fallback
     ```
     The `return` branch coerced to `()` instead of staying `!`.

2. **Blocked stabilization of the never type `!`**
   - The never type has been unstable for years partly because of this fallback rule.
   - Allowing `!` to silently become `()` made it hard to reason about divergence and made `!` less useful in generic/trait contexts.

3. **Subtle bugs in generic code**
   - Patterns relying on inference of `()` from diverging expressions broke when combined with generics or traits that didn't expect `()`.
   - Example (pre-2024 accepted, but confusing):
     ```rust
     fn f<T: Default>() -> Result<T, ()> {
         Ok(T::default())
     }

     f()?;  // inferred T = () because of ! fallback from ? on diverging error path
     ```

4. **Safety risk in unsafe code**
   - In rare cases, `!` falling back to `()` could flow into `unsafe` blocks or FFI, potentially hiding divergence or causing UB if assumptions about non-returning paths were violated.

The fallback to `()` was arbitrary, confusing, and a long-standing footgun — it prevented making `!` a truly first-class type.

#### What Changed in Rust 2024

- The fallback type for `!` coercions is now **`!`** itself.
- If no concrete type can be inferred, the expression stays `!` instead of becoming `()`.
- This preserves divergence semantics: a diverging expression remains diverging unless context explicitly requires another type.

**Examples of changed behavior**:

1. **Conditional with diverging branch**
   ```rust
   if true {
       Default::default()
   } else {
       return   // or panic!()
   };
   // Pre-2024: inferred ()
   // 2024: inferred ! → error: the trait `Default` is not implemented for `!`
   ```
   **Fix**:
   ```rust
   () = if true {
       Default::default()
   } else {
       return
   };
   // or
   if true {
       <() as Default>::default()
   } else {
       return
   };
   ```

2. **Generic `?` operator**
   ```rust
   fn f<T: Default>() -> Result<T, ()> { Ok(T::default()) }

   f()?;  // Pre-2024: inferred T = ()
          // 2024: inferred T = ! → error (! does not implement Default)
   ```
   **Fix**:
   ```rust
   f::<()>()?;    // or () = f()?;
   ```

3. **Closure expecting `()`**
   ```rust
   trait Unit { }
   impl Unit for () { }

   fn run<R: Unit>(f: impl FnOnce() -> R) { f(); }

   run(|| panic!());  // Pre-2024: ! → () OK
                      // 2024: ! does not implement Unit → error
   ```
   **Fix**:
   ```rust
   run(|| -> () { panic!() });
   ```

#### Migration Notes

- **No automatic `cargo fix`** — changes are semantic; you must manually annotate types.
- In **pre-2024 editions**, the compiler emits warnings via the lint **`dependency_on_unit_never_type_fallback`** to highlight code that relies on the old fallback.
- Common places to fix:
  - Diverging expressions in generic/trait bounds expecting `()`
  - `?` on `Result<T, E>` where `T` was implicitly `()`
  - Conditionals or closures ending in `return`/`panic!()`/`unreachable!()`
- The lint **`never_type_fallback_flowing_into_unsafe`** is now **`deny`** by default in 2024 to protect against potential UB.

#### Rationale

- Makes `!` behave consistently with its meaning: "never produces a value" — no more silent conversion to `()`.
- Removes a major blocker to stabilizing the never type as a first-class citizen.
- Improves predictability in type inference, especially in generic, async, and diverging code.
- The edition-based rollout gives the ecosystem time to migrate before the change becomes universal.

#### Official Reference
https://doc.rust-lang.org/edition-guide/rust-2024/never-type-fallback.html

This change is subtle but important for code relying on diverging expressions — it prevents hidden `()` inference and makes Rust's type system cleaner and more honest about non-returning paths.