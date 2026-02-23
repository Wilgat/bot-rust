# Tail expression temporary scope

**Edition: 2024**

## Summary

In the **Rust 2024 Edition**, the drop scope (lifetime) of temporaries created in **tail expressions** has been adjusted so that they are dropped **at the end of the enclosing block**, rather than extending to the end of the containing function or larger scope.

This change primarily affects:
- The final expression in a block (`{ ...; expr }`)
- The value expression in `match` arms
- The body of closures and immediately-invoked closures
- The return expression of functions (when written as a tail expression)

The goal is to make temporary lifetimes more predictable and to fix long-standing borrow checker frustrations in common patterns.

## Motivation / Problem (Pre-2024 Behavior)

Before the 2024 edition, temporaries created as part of a **tail expression** were extended to live until the **end of the enclosing function** (or sometimes even longer in closure contexts).  
This led to many surprising borrow checker errors, especially when:

- A temporary was created in the final expression of a block
- That temporary held a borrow of something declared earlier in the same block
- The block was used as the tail expression of a function, closure, or match arm

Classic example that failed to compile pre-2024:

```rust
fn example(v: &mut Vec<i32>) -> &i32 {
    let last = v.last().unwrap();           // temporary borrow created here

    if some_condition() {
        return last;                        // ok — borrow lives long enough
    }

    // Pre-2024: temporary from `v.last()` lives until end of function
    // → cannot borrow `v` mutably again here
    v.push(42);

    last                                    // tail expression
}
```

**Error (pre-2024)**:

```
error[E0502]: cannot borrow `*v` as mutable because it is also borrowed as immutable
  --> src/lib.rs:8:5
   |
4  |     let last = v.last().unwrap();
   |                - immutable borrow occurs here
...
8  |     v.push(42);
   |     ^ mutable borrow occurs here
...
11 | }
   | - immutable borrow might be used here
```

The borrow checker saw that the temporary borrow created by `v.last()` could "flow" into the return value of the function, so it forced the borrow to live until the end of the function — blocking the later mutable borrow of `v`.

This pattern appeared frequently in:
- Builder patterns
- Conditional returns with fallbacks
- Match expressions returning references
- Closures that sometimes early-return and sometimes mutate captured state

It was unintuitive because most developers expected the temporary to be dropped as soon as the value was no longer needed (i.e., after the block finished evaluating), not held artificially long.

## Details

In the **2024 Edition**, the temporary is now dropped **at the end of the smallest enclosing block** that contains the tail expression, rather than extending to the function scope.

Fixed version (compiles in 2024):

```rust
fn example(v: &mut Vec<i32>) -> &i32 {
    let last = v.last().unwrap();

    if some_condition() {
        return last;
    }

    v.push(42);           // now allowed — temporary dropped at end of block

    last
}
```

The same improvement applies to:
- `match` arms
- Closure bodies
- Block expressions used as function returns

## Migration

The change is **edition-gated** and can cause borrow checker errors to disappear (i.e., previously failing code now compiles).

**Automatic migration**:
- Run `cargo fix --edition`
- The `tail-expr-temporary-scope` lint (part of `rust-2024-compatibility`) detects cases where code **relied** on the old longer lifetime and suggests changes (rare).

**Most common outcome**:
- Code that previously failed to compile now succeeds without modification.
- Very few cases actually break (usually code that was unsound or relied on very unusual lifetime extension).

**Manual check** (before changing edition):

```rust
#![warn(tail_expr_temporary_scope)]
```

## See also

- Related change: [if let temporary scope](https://doc.rust-lang.org/edition-guide/rust-2024/temporary-if-let-scope.html)
- [Temporary scope rules](https://doc.rust-lang.org/reference/destructors.html#temporary-scopes) in the Rust Reference
- Official page: [Tail expression temporary scope](https://doc.rust-lang.org/edition-guide/rust-2024/temporary-tail-expr-scope.html)
