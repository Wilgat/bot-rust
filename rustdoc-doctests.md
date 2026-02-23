Here is a clear Markdown explanation of the **Rustdoc combined tests** change in the **Rust 2024 Edition**, based on the official Rust Edition Guide.

### Rustdoc combined tests

#### Summary
In the **Rust 2024 Edition**, `rustdoc` in test mode (`rustdoc --test` or `cargo test --doc`) now **combines compatible documentation tests (doctests)** into a single binary instead of compiling each doctest code block as a separate executable.

This results in a **significant performance improvement** for crates with many doctests.

#### Motivation / Problem (Pre-2024 Behavior)

Prior to the 2024 Edition, `rustdoc` compiled **each individual doctest code block** as its own separate executable:

- For a crate with 100 doctests → 100 separate compilations and binaries.
- Each test ran in its own process (good for isolation), but the compilation overhead was very high.

**Real problems this caused**:

1. **Slow documentation testing**
   - Running `cargo test --doc` on medium-to-large crates (hundreds of examples) could take **minutes** just for compilation — even if the tests themselves were trivial.
   - Painful in CI pipelines, local development, and when frequently checking docs.

2. **Scalability bottleneck**
   - Popular libraries (e.g., std, tokio, serde, clap) with extensive examples suffered the most.
   - Discouraged writing thorough doctests because testing them became too slow.

3. **Redundant overhead**
   - Each binary included full Rust standard library linking, metadata, etc. — massive duplication when tests were small `assert_eq!` snippets.

4. **No isolation penalty in practice**
   - Most doctests are pure functions or small examples with no global state interference.
   - The per-test process model was overkill for the majority of cases, yet expensive.

The old behavior was simple to implement but did not scale well as the ecosystem grew and crates added more examples.

#### What Changed in Rust 2024

Starting with the 2024 Edition:

- `rustdoc` **attempts to combine** all compatible doctests into **a single binary**.
- Each doctest is placed into its own **separate function** inside that binary.
- Tests still **run in independent processes** (via the test harness) → global state (statics, thread-locals) isolation is preserved.

**Compatibility rules** — a doctest is combinable unless it:

- Uses the `compile_fail` tag
- Specifies an `edition` tag
- Uses crate-wide attributes (`#![feature(...)]`, `#![global_allocator]`, etc.)
- Contains macros using `$crate`
- Other rare cases that would break in a shared crate context

If a doctest cannot be combined automatically, it falls back to the old **standalone binary** behavior.

**New escape hatch** — force standalone compilation with the `standalone_crate` tag:

```rust
/// ```
/// # standalone_crate
/// let loc = std::panic::Location::caller();
/// assert_eq!(loc.line(), 42);
/// ```
```

#### Benefits

- **Much faster `cargo test --doc`** — often 2–10× speedup (depending on number of tests).
- Compilation happens once instead of N times.
- No change to runtime isolation — tests still run separately.
- Encourages writing more and better doctests without performance penalty.

#### Migration Notes & Potential Incompatibilities

- **Edition-gated** — only active when the crate uses `edition = "2024"`.
- **No automatic migration** — `cargo fix` does not apply tags.
- Steps:
  1. Update to `edition = "2024"`.
  2. Run `cargo test --doc`.
  3. If failures occur → investigate and either:
     - Rewrite the test to be combinable
     - Add `standalone_crate` tag

- **Rare breaking cases to watch**:
  - `std::panic::Location::caller()` — line numbers may shift in the combined binary.
  - `std::any::type_name::<T>()` — module path may differ (e.g. `__doctest` instead of crate name).
  - Tests that rely on being the "main" crate or specific binary layout.

- Incompatibilities are expected to be **extremely rare** in practice.

#### Example

**Before (multiple binaries):**

```rust
/// Adds two numbers
///
/// ```
/// assert_eq!(add(1, 2), 3);
/// ```
pub fn add(a: i32, b: i32) -> i32 { a + b }

/// Subtracts two numbers
///
/// ```
/// assert_eq!(subtract(5, 2), 3);
/// ```
pub fn subtract(a: i32, b: i32) -> i32 { a - b }
```

→ Two separate compilations.

**After (2024 Edition — single binary):**

The two `assert_eq!` blocks become two functions in one executable → much faster testing.

#### Official Reference
https://doc.rust-lang.org/edition-guide/rust-2024/rustdoc-doctests.html

In short: A major quality-of-life win for documentation-heavy crates — doctests compile much faster in 2024 by being combined, with automatic fallback and an opt-out tag for the rare incompatible cases.