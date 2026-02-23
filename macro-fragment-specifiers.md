Here is a clear Markdown explanation of the **Macro fragment specifiers** change in the **Rust 2024 Edition**, based on the official Rust Edition Guide.

### Macro fragment specifiers

#### Summary
In the **Rust 2024 Edition**, the `expr` fragment specifier in `macro_rules!` macros has been **extended** to also match:
- `const { ... }` expressions (const blocks, stabilized in Rust 1.79)
- `_` expressions (underscore expressions, stabilized in Rust 1.59)

To preserve the old (pre-2024) behavior of `expr` (which did **not** match these forms), a new fragment specifier **`expr_2021`** was introduced. It behaves exactly like the old `expr`.

This change allows macros to naturally support newer expression syntax without requiring edition-specific macro definitions.

#### Motivation / Problem (Pre-2024 Behavior)

In editions before 2024 (including 2021), the `expr` fragment specifier was deliberately **restricted** — it did **not** match top-level `const { ... }` blocks or `_` expressions.

This restriction was intentional to prevent **silent breakage** when new expression kinds were added to the language.

**Example of the problem (pre-2024 / 2021 Edition behavior):**

```rust
macro_rules! example {
    ($e:expr)          => { ("first rule") };
    (const $e:expr)    => { ("second rule") };
}

fn main() {
    // In 2021 Edition: matches the SECOND rule
    println!("{}", example!(const { 1 + 1 }));     // → "second rule"

    // Normal block expression matches the first rule
    println!("{}", example!({ 1 + 1 }));           // → "first rule"
}
```

If the language team had simply updated `expr` to match `const` expressions when they were added (in 1.79), the macro above would have **silently changed behavior**:
- `const { 1 + 1 }` would suddenly match the **first** rule instead of the second → **breaking change** for any macro that had a specific `const` arm.

This created a recurring problem:
- New syntax additions (`const` blocks, `_` expr, future additions) could not easily be supported by existing `expr`-based macros.
- Macro authors had to either:
  - Avoid using `expr` for patterns that might include new syntax
  - Add edition-specific macro variants
  - Use more restrictive specifiers like `block` or `item`
- Users of macros got confusing errors or unexpected matches when passing new expression kinds.

The solution is deferred to editions: change `expr` behavior only in a new edition, and provide an escape hatch (`expr_2021`) for compatibility.

#### What Changed in Rust 2024

- **`expr`** now matches **all** expressions, including:
  - `const { ... }` blocks
  - `_` expressions
  - (and all previous expression kinds)

- New fragment specifier **`expr_2021`** added:
  - Matches exactly the same things as `expr` did before 2024
  - Does **not** match top-level `const { ... }` or `_`

**New behavior example (2024 Edition):**

```rust
macro_rules! example {
    ($e:expr)          => { ("first rule") };
    (const $e:expr)    => { ("second rule") };
}

fn main() {
    // Now matches the FIRST rule (because expr includes const)
    println!("{}", example!(const { 1 + 1 }));     // → "first rule"

    // Also matches first rule
    println!("{}", example!(_));                   // → "first rule"

    // Normal expressions still match first rule
    println!("{}", example!({ 1 + 1 }));           // → "first rule"
}
```

To keep the old behavior, change to `expr_2021`:

```rust
macro_rules! example {
    ($e:expr_2021)     => { ("old behavior") };
    (const $e:expr)    => { ("const rule") };
}
```

#### Migration Steps

- The **`edition_2024_expr_fragment_specifier`** lint (part of `rust-2024-compatibility`) detects uses of `expr` that might change meaning.
- Run **`cargo fix --edition`** — it automatically replaces `expr` with `expr_2021` in macro definitions to preserve existing behavior.
- After migration:
  - Review each changed macro.
  - If the macro **should** support `const {}` or `_`, revert the fix back to `expr`.
  - If backward compatibility is needed (e.g., supporting older editions via cfg), keep `expr_2021`.
- To see warnings without migrating yet:
  ```rust
  #![warn(edition_2024_expr_fragment_specifier)]
  ```

#### Related Change: Missing macro fragment specifiers
A separate but related hardening in 2024:
- The `missing_fragment_specifier` lint (previously warn/allow) is now a **hard error** in all editions (finalized since Rust 1.89).
- Example that now fails to compile:
  ```rust
  macro_rules! foo {
      () => {};
      ($name) => {};     // ERROR: missing fragment specifier (e.g. $name:ident)
  }
  ```
- Migration: manually remove or fix unused rules that lack `:fragment` specifiers. No automatic fix.

#### Rationale

- Enables macro authors to support new language syntax (`const`, `_`, future additions) without edition gymnastics.
- Preserves compatibility via `expr_2021`.
- Follows Rust's edition philosophy: evolve the language while giving time to migrate.

#### Official Reference
https://doc.rust-lang.org/edition-guide/rust-2024/macro-fragment-specifiers.html

This is a forward-compatibility improvement for the declarative macro system — small but important for macro-heavy codebases and future language evolution.