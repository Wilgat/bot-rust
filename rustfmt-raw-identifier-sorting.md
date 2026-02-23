
# Rustfmt: Raw identifier sorting

**Edition: 2024**

## Summary

`rustfmt` now properly sorts [raw identifiers](https://doc.rust-lang.org/reference/identifiers.html#raw-identifiers) by their actual identifier name instead of by their literal spelling including the `r#` prefix.

## Motivation / Problem (Pre-2024 Behavior)

Before the 2024 edition, `rustfmt` sorted items (especially `use` statements) using the **exact source text**, including the `r#` prefix of raw identifiers.  
Because `r#` comes before most letters in ASCII/Unicode sorting order, raw identifiers were consistently placed **before** normal identifiers — even when the logical name (without `r#`) should have come later alphabetically.

This produced unnatural and visually confusing ordering:

```rust
// Before 2024 edition (undesirable sorting)
use websocket::client::ClientBuilder;
use websocket::r#async::futures::Stream;     // ← appears too early
use websocket::result::WebSocketError;
```

Most developers expect items to be sorted by their semantic name (`async` should sort between `client` and `result`), not by whether the identifier needed to be escaped with `r#`.

## Details

In the **2024 Edition**, `rustfmt` now ignores the `r#` prefix when performing comparisons for sorting purposes and sorts based on the actual identifier text.

After applying formatting with the 2024 edition:

```rust
// After 2024 edition (correct alphabetical order)
use websocket::r#async::futures::Stream;
use websocket::client::ClientBuilder;
use websocket::result::WebSocketError;
```

This matches the expectation set by the [Rust Style Guide sorting rules](https://doc.rust-lang.org/style-guide/index.html#sorting).

## Migration

Just run:

```bash
cargo fmt
# or
rustfmt --edition 2024 **/*.rs
```

`rustfmt` will automatically apply the new sorting behavior when the crate is using the **2024 edition**.

See also: [Style edition](https://doc.rust-lang.org/edition-guide/rust-2024/rustfmt-style-edition.html) chapter for more information about style editions and migration strategies.
