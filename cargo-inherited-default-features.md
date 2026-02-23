Here is a clear Markdown explanation of the **Cargo: Reject unused inherited default-features** change in the **Rust 2024 Edition**, based on the official Rust Edition Guide.

### Cargo: Reject unused inherited default-features

#### Summary
In the **Rust 2024 Edition**, Cargo now **rejects** (as a hard error) the use of `default-features = false` in an **inherited workspace dependency** when the workspace-level dependency does **not** explicitly set `default-features = false`.

- This prevents redundant and ineffective specifications that lead to confusion.
- `default-features = false` is only allowed on inherited dependencies if the workspace definition already disables default features.

#### Motivation / Problem (Pre-2024 Behavior)
Workspace inheritance lets you define dependencies once in the workspace root:

```toml
# workspace Cargo.toml
[workspace.dependencies]
regex = "1.10.4"   # implicitly default-features = true
```

A member crate could inherit it but try to disable defaults:

```toml
# member crate Cargo.toml (pre-2024)
[dependencies]
regex = { workspace = true, default-features = false }
```

**Issue**: This `default-features = false` was **silently ignored** (or warned in later versions), because workspace defaults are **unified/additive** across members. If any member (or the workspace) enables defaults, they stay enabled for everyone using that inherited dependency.

This led to:
- Surprising behavior
- Inconsistent feature activation
- Hard-to-debug feature mismatches

#### What Changed in Rust 2024
- It is now a **hard error** to write `default-features = false` on an inherited dependency unless the workspace explicitly sets `default-features = false`.
- Example that **now fails**:

```toml
# workspace Cargo.toml
[workspace.dependencies]
regex = "1.10.4"   # no default-features = false

# member Cargo.toml (edition = "2024")
[dependencies]
regex = { workspace = true, default-features = false }  # ERROR in 2024 Edition
```

Cargo will report an error like:

```
error: `default-features = false` is not allowed here because the workspace dependency does not disable default features
```

#### Correct Ways to Handle It

1. **Disable defaults at the workspace level** (recommended when you want control per-member):

```toml
# workspace Cargo.toml
[workspace.dependencies]
regex = { version = "1.10.4", default-features = false }
```

Then members can choose to re-enable them if needed:

```toml
# member Cargo.toml
[dependencies]
regex = { workspace = true, default-features = true }   # now allowed
```

2. **Remove the useless `default-features = false`** if you actually want the workspace defaults:

```toml
# member Cargo.toml
[dependencies]
regex = { workspace = true }   # clean & correct
```

**Warning**: If you build multiple members at once, features are unified — one member enabling defaults turns them on for all.

#### Migration Steps
- Run **`cargo fix --edition`** — Cargo will **automatically remove** useless `default-features = false` entries in inherited dependencies.
- Manually: Look for build warnings like:

```
warning: `default-features` is ignored for regex, since `default-features` was not specified for `workspace.dependencies.regex`, this could become a hard error in the future
```

Remove the `default-features = false` line where it appears.

#### Rationale
- Prevents confusion from ineffective settings.
- Makes feature control clearer and more predictable.
- Enforces intentional design: decide default-feature behavior **at the workspace level** when using inheritance.
- Aligns with Cargo's unification rules for workspace dependencies.

This is a **backwards-incompatible lint-turned-error** specifically for the 2024 Edition, helping large workspaces avoid subtle feature bugs.

For the full official text, see:  
https://doc.rust-lang.org/edition-guide/rust-2024/cargo-inherited-default-features.html