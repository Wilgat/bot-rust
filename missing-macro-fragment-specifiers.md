Here is a clear Markdown explanation of the **Missing macro fragment specifiers** change in the **Rust 2024 Edition**, based on the official Rust Edition Guide.

### Missing macro fragment specifiers

#### Summary
In the **Rust 2024 Edition**, it is now a **hard error** to define a `macro_rules!` rule that uses a metavariable (`$name`) **without** specifying a fragment specifier (e.g. `:ident`, `:expr`, `:ty`, `:item`, etc.).

Before Rust 2024, such rules were allowed (though they triggered a warning in recent versions). Now they are rejected at compile time.

**Old (allowed pre-2024):**

```rust
macro_rules! bad_macro {
    () => { ... };
    ($name) => { println!("{}", $name); };   // ← no :fragment specifier
}
```

**New (hard error in 2024 Edition):**

```rust
macro_rules! bad_macro {
    () => { ... };
    ($name) => { println!("{}", $name); };   // error[E0306]: missing fragment specifier
}
```

Correct fix:

```rust
macro_rules! good_macro {
    () => { ... };
    ($name:expr) => { println!("{}", $name); };   // must specify :expr, :ident, etc.
}
```

#### Motivation / Problem (Pre-2024 Behavior)

In editions before 2024, `macro_rules!` allowed metavariables **without** a fragment specifier:

```rust
macro_rules! example {
    ($x) => { let _ = $x; };   // no :??? — silently accepted
}
```

This was technically legal, but **extremely dangerous and almost always a bug**:

**Major problems this caused:**

1. **Silent misbehavior / hygiene issues**
   - The metavariable `$x` was treated as if it had **no type constraint**.
   - It could match **literally anything** (even invalid syntax in that position), leading to confusing parse errors much later in expansion.

2. **Very misleading error messages**
   When expansion failed, the compiler pointed to the **expanded code** (not the macro definition), making debugging extremely hard.

   Example:

   ```rust
   example!(let y = 10);
   // → error: expected expression, found `let`
   //     points inside the expanded macro, not at the call site or definition
   ```

3. **Common copy-paste / beginner mistakes**
   - New macro authors frequently forgot to add `:expr`, `:ident`, etc.
   - Tutorials and old Stack Overflow answers sometimes showed incomplete specifiers → perpetuated the bug.

4. **No static checking of macro hygiene / correctness**
   - Without a specifier, the macro could accidentally capture or interfere with identifiers in ways that were hard to predict.
   - It bypassed much of the intent of fragment specifiers (which exist to guide parsing and hygiene).

5. **Lint was too weak**
   - The `missing_fragment_specifier` lint existed since ~Rust 1.30, but was:
     - `allow` by default for a long time
     - Later upgraded to `warn` in recent 1.x releases
     - Still only a warning → many crates shipped with these bugs

**Real-world impact:**
- Large codebases had hundreds of such rules hidden in utility macros.
- When people tried to enable stricter lints or update editions, they got flooded with warnings — but fixing them was manual and error-prone.

By making it a **hard error** in the 2024 edition, Rust forces all macro authors to write correct, explicit fragment specifiers — improving macro reliability, debuggability, and long-term maintainability.

#### What Changed in Rust 2024

- The lint `missing_fragment_specifier` is now a **hard deny-by-default error** in the 2024 edition.
- Any macro arm that uses `$var` without `:fragment` fails to compile.

**Allowed fragment specifiers** (quick reference):

| Specifier     | Matches                                      | Common use case              |
|---------------|----------------------------------------------|------------------------------|
| `:expr`       | Any expression                               | Most values                  |
| `:ident`      | Identifier                                   | Variable/function names      |
| `:ty`         | Type                                         | In generics, let bindings    |
| `:pat`        | Pattern (2021+)                              | match arms                   |
| `:pat_param`  | Pattern in function params (2024+)           | fn arguments                 |
| `:item`       | Item (fn, struct, mod, etc.)                 | Generating modules/code      |
| `:block`      | { ... } block                                | Control flow                 |
| `:stmt`       | Statement                                    | In blocks                    |
| `:literal`    | Literal (numbers, strings, etc.)             | Constants                    |
| `:path`       | Path (Type::Variant, mod::func)              | Paths                        |
| `:meta`       | Attribute contents                           | #[derive(...)]               |
| `:tt`         | Single token tree                            | Very generic / low-level     |

#### Migration Steps

1. **Search your codebase** for macros missing specifiers:
   ```bash
   rg '\$\w+\)'  # looks for ) right after $var
   ```

2. **`cargo fix --edition`** does **not** automatically add specifiers — it's too semantic (Rust can't guess whether you meant `:expr`, `:ident`, etc.).

3. **Manually review and add the correct fragment**:
   - Most common fix: add `:expr` or `:ident`
   - For patterns: use `:pat` or `:pat_param` (new in recent editions)
   - For very generic macros: use `:tt` (but sparingly)

4. **Test thoroughly** — adding a specifier can change what the macro accepts.

5. **Enable the lint early** (even before migrating edition):
   ```rust
   #![deny(missing_fragment_specifier)]
   ```

#### Rationale

- Macros are a core part of Rust's extensibility → they should be as safe and predictable as possible.
- Fragment specifiers are there for a reason: parsing guidance, hygiene, error location.
- Forcing explicit specifiers eliminates a long-standing footgun.
- Aligns with Rust's philosophy of making invalid code unrepresentable.

#### Official Reference
https://doc.rust-lang.org/edition-guide/rust-2024/missing-macro-fragment-specifiers.html

In short: The 2024 edition **closes a dangerous loophole** in declarative macros — no more silent `$var` without `:type`. A small but very valuable safety improvement for macro-heavy code.