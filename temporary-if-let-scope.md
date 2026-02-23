# if let temporary scope

**Edition: 2024**

## Summary

In the **Rust 2024 Edition**, the temporary values created when evaluating the scrutinee expression (`$expr`) in an `if let $pat = $expr { ... } else { ... }` are now dropped **before** entering the `else` branch (or immediately after the `then` block finishes if the pattern matches).

This shortens the lifetime of those temporaries compared to previous editions, reducing surprising borrow-checker behavior and resource lifetime issues.

## Motivation / Problem (Pre-2024 Behavior)

Before the 2024 edition, temporaries produced by the scrutinee expression in an `if let` were extended to live until **after** the entire `if let` expression — including through the entire `else` block.

This could lead to unexpected lifetime extensions, borrow checker errors, deadlocks, or resource leaks — especially when the scrutinee expression created a temporary with a non-trivial `Drop` implementation (e.g., RAII guards like locks, file handles, or scoped threads).

Classic real-world example (deadlock):

```rust
use std::sync::RwLock;

fn f(value: &RwLock<Option<bool>>) {
    if let Some(x) = *value.read().unwrap() {
        println!("value is {x}");
    } else {
        // Pre-2024: read lock is still held here → deadlock!
        let mut v = value.write().unwrap();
        if v.is_none() {
            *v = Some(true);
        }
    }
    // Read lock dropped here (pre-2024)
}
```

The temporary `RwLockReadGuard` returned by `value.read()` lived until the end of the entire `if let` statement — even if the pattern matched and the `then` block ran.  
In the `else` case, attempting to acquire a write lock while still holding the read lock caused a **deadlock**.

This behavior was unintuitive: developers expected the temporary to be dropped as soon as the pattern match decision was made, not held across the `else` branch.

Other common pain points included:
- Borrow checker complaints when later code tried to borrow the same value mutably
- Unnecessary resource retention (files, connections, scopes) longer than needed
- Subtle differences between `if let` and equivalent `match` expressions

## Details

In the **2024 Edition**, the drop scope of temporaries in the scrutinee of `if let` is shortened:

- If the pattern matches → temporaries are dropped **after** the `then` block finishes
- If the pattern does **not** match → temporaries are dropped **before** entering the `else` block

Fixed version (no deadlock):

```rust
use std::sync::RwLock;

fn f(value: &RwLock<Option<bool>>) {
    if let Some(x) = *value.read().unwrap() {
        println!("value is {x}");
    } // Read lock dropped here in 2024 edition
    else {
        // Now safe to acquire write lock
        let mut v = value.write().unwrap();
        if v.is_none() {
            *v = Some(true);
        }
    }
}
```

This aligns `if let` more closely with intuitive expectations and matches the behavior of many other pattern-matching constructs.

See also: [Temporary scope rules](https://doc.rust-lang.org/reference/destructors.html#temporary-scopes) in the Rust Reference.

## Migration

The change is **edition-gated** and can break code that relied on the old, longer temporary lifetime.

**Automatic migration**:
- Run `cargo fix --edition`
- The `if_let_rescope` lint (part of the `rust-2024-compatibility` group) detects problematic cases and suggests rewriting `if let` → `match`, which preserves the old (longer) temporary scope:

```rust
match *value.read().unwrap() {
    Some(x) => {
        println!("value is {x}");
    }
    _ => {
        let mut v = value.write().unwrap();
        if v.is_none() {
            *v = Some(true);
        }
    }
}
// Read lock dropped here (same as pre-2024 behavior)
```

**Important**:
- After migration, **review** each suggested change.
- If your code actually needed the temporary to live past the `else` block (rare), keep the `match` version.
- If the shorter lifetime is correct/safer, you can revert to `if let` manually.

**Manual inspection** (without migrating edition yet):

```rust
#![warn(if_let_rescope)]
```

This lint is `allow` by default but very useful during transition.

## See also

- [if_let_rescope lint documentation](https://doc.rust-lang.org/rustc/lints/listing/allowed-by-default.html#if_let_rescope)
- [Tail expression temporary scope](https://doc.rust-lang.org/edition-guide/rust-2024/temporary-tail-expr-scope.html) (related change)
- Official page: [if let temporary scope](https://doc.rust-lang.org/edition-guide/rust-2024/temporary-if-let-scope.html)
