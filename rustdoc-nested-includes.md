Here is a clear Markdown explanation of the **Rustdoc nested include! change** in the **Rust 2024 Edition**, based on the official Rust Edition Guide.

### Rustdoc nested include! change

#### Summary
In the **Rust 2024 Edition**, the way `rustdoc` handles **nested `include!`** (or `include_str!` / `include_bytes!`) macros inside documentation comments has changed.

- **Pre-2024 editions**: Nested `include!` macros were expanded **at the crate root level** (as if the included file was part of the main crate).
- **Rust 2024 Edition**: Nested `include!` macros are now expanded **relative to the file that contains the doc comment** (i.e., the file where the `///` or `//!` comment appears).

This makes path resolution more intuitive and consistent with how developers expect file inclusion to work in documentation.

#### Motivation / Problem (Pre-2024 Behavior)

Before Rust 2024, when you wrote something like this in a module:

```rust
// src/lib.rs or src/module.rs
/// Some example that includes a file relative to *this module*:
///
/// ```
/// let config = include_str!("../../config.toml");
/// ```
pub fn load_config() { ... }
```

`rustdoc` would **not** resolve `../../config.toml` relative to `src/module.rs`.  
Instead, it resolved the path **relative to the crate root** (the directory containing `Cargo.toml`).

**Real problems this caused**:

1. **Broken relative paths in nested modules**
   - If the doc comment was in a deeply nested module (e.g. `src/api/v2/endpoints/user.rs`), then `include_str!("../../../fixtures/data.json")` would often point to the wrong file or fail entirely.
   - Developers had to use **absolute paths from crate root** (fragile, hard to move files) or copy-paste content into the doc comment.

2. **Inconsistent behavior between `cargo build` and `cargo doc`**
   - In normal compilation (`cargo build` / `cargo run`), `include!` resolves paths relative to the **file that contains the macro call**.
   - In `rustdoc`, the same macro call resolved paths relative to the **crate root** → surprising and error-prone difference.

3. **Painful documentation maintenance**
   - Moving a module to a different directory broke all relative `include_str!` / `include!` paths in its doc comments.
   - Common workaround: avoid relative paths entirely and use crate-root-relative paths (e.g. `include_str!("../../../../../fixtures/...")`) → ugly and very error-prone.

4. **Discouraged good documentation practices**
   - Writers avoided including external example files or config snippets because paths were unreliable.
   - Many crates ended up embedding small examples directly in the doc comment instead of using `include!` — leading to duplication and harder maintenance.

The old behavior was a historical artifact from early `rustdoc` implementation and never matched user expectations.

#### What Changed in Rust 2024

- Nested `include!`, `include_str!`, and `include_bytes!` inside documentation comments now resolve paths **relative to the source file that contains the doc comment** — exactly the same way they behave during normal compilation.
- This applies to both outer doc comments (`///`) and inner doc comments (`/*! ... */` / `//!`).

**Example – now works as expected**

```rust
// src/api/v2/endpoints/user.rs
/// Example configuration:
///
/// ```
/// let config = include_str!("../../../fixtures/user-config.toml");
/// println!("{}", config);
/// ```
pub fn load_user() { ... }
```

→ In 2024 edition, `../../../fixtures/user-config.toml` is resolved from `src/api/v2/endpoints/user.rs` → correct file is included.

#### Migration & Potential Incompatibilities

- **Edition-gated** — only active when the crate uses `edition = "2024"`.
- **Breaking change** — crates that relied on crate-root resolution will now look in the wrong place.
- **Common failure pattern**:
  ```rust
  /// ```
  /// let data = include_str!("fixtures/data.json");  // ← used to mean crate-root/fixtures/
  /// ```
  ```
  → Now looks for `fixtures/` next to the module file → usually fails.

- **Fix strategies**:
  1. Move the included file next to the documenting file (recommended when possible).
  2. Use crate-root relative paths consistently:
     ```rust
     include_str!("../../../../fixtures/data.json")   // count from module depth
     ```
  3. Embed small content directly in the doc comment (for tiny examples).
  4. Use absolute paths via `$CARGO_MANIFEST_DIR` in build scripts (more advanced).

- Run `cargo doc` after changing the edition and check for broken includes.

#### Rationale

- Makes `include!` behave **consistently** between normal compilation and documentation generation.
- Matches developer intuition: “the path should be relative to where I wrote it”.
- Removes a long-standing source of confusion and fragile paths in documentation.
- Encourages better-organized example files alongside modules.

#### Official Reference
https://doc.rust-lang.org/edition-guide/rust-2024/rustdoc-nested-includes.html

In short: Rust 2024 fixes one of the most annoying quirks in rustdoc — nested `include!` paths now work the way everyone expected them to, at the cost of a small breaking change for crates that relied on the old crate-root resolution.