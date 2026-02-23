# Rustfmt: Version sorting

**Edition: 2024**

## Summary

`rustfmt` utilizes a new sorting algorithm that applies a version-sort-like comparison when reordering items (especially imports) according to the Rust Style Guide rules.

## Motivation / Problem (Pre-2024 Behavior)

In previous editions, `rustfmt` (and the corresponding Rust Style Guide sorting rules) used a simple **ASCIIbetical** (lexicographic) sorting approach based on Unicode character order.  

This produced suboptimal results when sorting identifiers or paths that contain numeric components — especially version-like or size-like names (e.g. `NonZeroU8`, `NonZeroU16`, `NonZeroU32`, `NonZeroU64`).  
Because it compares characters directly, "U16" sorts before "U8" (since '1' < '8'), even though semantically 8 < 16.

This led to unnatural and visually confusing ordering in sorted lists:

```rust
// Pre-2024 editions (ASCIIbetical sorting)
use std::num::{NonZeroU16, NonZeroU32, NonZeroU64, NonZeroU8};
```

Most developers expect numeric parts to be compared numerically (8 < 16 < 32 < 64), not as strings — particularly for types, versions, or numbered items where logical progression improves readability.

## Details

In the **2024 Edition**, `rustfmt` switches to a **version-sort-like algorithm** when applying the [Rust Style Guide sorting rules](https://doc.rust-lang.org/style-guide/index.html#sorting).  

It continues to compare Unicode characters lexicographically in general, but provides **better numeric handling** for digit sequences embedded in identifiers. This results in more natural ordering for many common cases (especially numeric suffixes/prefixes).

Example with unsorted input:

```rust
use std::num::{NonZeroU32, NonZeroU16, NonZeroU8, NonZeroU64};
use std::io::{Write, Read, stdout, self};
```

**Pre-2024 output** (ASCIIbetical):

```rust
use std::io::{self, stdout, Read, Write};
use std::num::{NonZeroU16, NonZeroU32, NonZeroU64, NonZeroU8};
```

**2024 Edition output** (improved numeric-aware sorting):

```rust
use std::io::{self, Read, Write, stdout};
use std::num::{NonZeroU8, NonZeroU16, NonZeroU32, NonZeroU64};
```

## Migration

The change is applied automatically when formatting with the 2024 edition. Simply run:

```bash
cargo fmt
# or
rustfmt --edition 2024 **/*.rs
```

`rustfmt` will use the new sorting behavior for crates using the **2024 edition**.

See also: [Style edition](https://doc.rust-lang.org/edition-guide/rust-2024/rustfmt-style-edition.html) chapter for more information about style editions and migration strategies.

## References

- [Rust Style Guide – Sorting rules](https://doc.rust-lang.org/style-guide/index.html#sorting)
- [Rustfmt: Style edition](https://doc.rust-lang.org/edition-guide/rust-2024/rustfmt-style-edition.html)
- Official page: [Rustfmt: Version sorting](https://doc.rust-lang.org/edition-guide/rust-2024/rustfmt-version-sorting.html)
