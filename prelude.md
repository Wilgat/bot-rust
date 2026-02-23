Here is a clear Markdown explanation of the **Changes to the prelude** in the **Rust 2024 Edition**, based on the official Rust Edition Guide.

### Changes to the prelude

#### Summary
In the **Rust 2024 Edition**, the **standard library prelude** has been updated with the following additions:

- **`std::iter::from_fn`** is now automatically in scope (previously required explicit `use`).
- **`std::iter::from_generator`** is now automatically in scope (previously required explicit `use`).

These are the **only two items** added to the prelude in the 2024 edition.

No items were removed from the prelude.

#### Motivation / Problem (Pre-2024 Behavior)

Before Rust 2024, the prelude contained a carefully curated set of the most commonly used items from `std`, such as:

- `Option`, `Result`, `Vec`, `String`, `Box`, `AsRef`, `Into`, `From`, `Iterator`, `IntoIterator`, `Clone`, `Debug`, `Default`, etc.

However, two very useful iterator construction functions were **not** included:

```rust
use std::iter::{from_fn, from_generator};
```

**Real problems this caused**:

1. **Common friction point when writing iterator pipelines**
   - `from_fn` is one of the most natural ways to create custom iterators on-the-fly:
     ```rust
     let mut count = 0;
     let naturals = std::iter::from_fn(move || {
         count += 1;
         Some(count)
     });
     ```
   - Developers frequently had to add an extra `use` statement — especially annoying in short scripts, examples, playground code, or small functions.

2. **Generators (when stabilized) would feel second-class**
   - The `gen` keyword and generator syntax (future feature) produce values of type `impl Iterator<Item = T>`.
   - `std::iter::from_generator` is the canonical way to turn a generator into an iterator.
   - Without it in the prelude, code using generators would require an extra `use` just to consume the result → inconsistent ergonomics.

3. **Cognitive overhead in teaching and quick prototyping**
   - Beginners and experienced users alike often forget to import `from_fn` → leads to unnecessary errors or verbose code.
   - In Rust Playground / examples / blog posts / quick scripts → extra line of `use` clutters otherwise clean code.

4. **Prelude philosophy mismatch**
   - The prelude is meant to include items that are **almost always needed** when writing idiomatic Rust.
   - `from_fn` is used very frequently in real-world iterator-heavy code (combinators, lazy sequences, stateful iterators).
   - `from_generator` is expected to become similarly common once generators are stabilized.

Adding these two items was a low-risk, high-value ergonomic win:
- They are **very unlikely to conflict** with user-defined names (unlike e.g. adding more traits).
- They are **purely additive** — no breaking change for existing code.

#### What Changed in Rust 2024

The prelude now includes:

```rust
// Newly available without explicit use
pub use std::iter::{from_fn, from_generator};
```

**Example – before vs after**

**Pre-2024 (2021 Edition):**

```rust
use std::iter::from_fn;

fn main() {
    let mut n = 0;
    let evens = from_fn(move || {
        n += 2;
        Some(n)
    });

    for x in evens.take(5) {
        println!("{}", x);
    }
}
```

**Rust 2024 Edition:**

```rust
fn main() {
    let mut n = 0;
    let evens = from_fn(move || {     // no use needed!
        n += 2;
        Some(n)
    });

    for x in evens.take(5) {
        println!("{}", x);
    }
}
```

Same applies to `from_generator` once generators land.

#### Migration Impact

- **Purely additive** — no breaking changes.
- Old code continues to compile unchanged.
- New code becomes slightly cleaner (one less `use` line in many cases).
- No `cargo fix` needed — the change is automatic when you set `edition = "2024"`.

#### Rationale

- Improves **day-to-day ergonomics** for iterator-heavy code (very common in Rust).
- Prepares the prelude for the upcoming generator feature without requiring a later edition bump.
- Keeps the prelude small and focused — only two carefully chosen additions.
- Follows the principle: **if something is used very frequently and has almost no name-conflict risk, it belongs in the prelude**.

#### Official Reference
https://doc.rust-lang.org/edition-guide/rust-2024/prelude.html

In short: A small, delightful quality-of-life improvement — `from_fn` and `from_generator` are now always available, reducing boilerplate in one of Rust’s most common idioms.