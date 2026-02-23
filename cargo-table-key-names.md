Here is a clear Markdown explanation of the **Cargo: Table and key name consistency** change in the **Rust 2024 Edition**, based on the official Rust Edition Guide.

### Cargo: Table and key name consistency

#### Summary
In the **Rust 2024 Edition**, Cargo enforces stricter **consistency rules** for table and key names in `Cargo.toml` files.

Specifically, Cargo now **rejects** (as a hard error) the use of inconsistent casing or separator styles for the same logical key across different parts of the manifest.

The most visible and commonly affected rules are:

- Dependencies must use **hyphenated** names in `[dependencies]`, `[dev-dependencies]`, etc.
- But the same crate must be referenced with **consistent hyphenation** when used as a table key in `[package.metadata]`, `[features]`, or other sections.

In practice, this change primarily affects **hyphen vs underscore** usage in feature names and package metadata keys.

#### Motivation / Problem (Pre-2024 Behavior)

Before Rust 2024, Cargo was very lenient with key naming:

- Crate names in `[dependencies]` are **always hyphenated** (because they come from crates.io, which enforces hyphens).
  ```toml
  [dependencies]
  serde_json = "1.0"
  ```

- But when the same crate was used as a **feature name** or **metadata key**, people used **both** styles freely:
  ```toml
  [features]
  serde-json = ["serde_json"]     # hyphen
  # or
  serde_json = ["serde_json"]     # underscore — also accepted!
  ```

  ```toml
  [package.metadata.docs.rs]
  features = ["serde-json"]       # hyphen
  # or
  features = ["serde_json"]       # underscore — also worked
  ```

**Real-world problems this caused**:

1. **Inconsistent style across projects**
   - Some crates used `serde-json`, others `serde_json` — no single convention.

2. **Hard-to-catch typos / copy-paste errors**
   - Developers would write `serde_json` in features but `serde-json` in docs.rs metadata → features silently not enabled during `docs.rs` builds.

3. **Tooling confusion**
   - `cargo expand`, `cargo doc`, IDEs, and linters sometimes treated them differently or failed to autocomplete/validate correctly.

4. **Future-proofing pain**
   - Cargo wanted to reserve the right to give special meaning to certain key patterns in the future → allowing both styles made that harder.

#### What Changed in Rust 2024

Cargo now enforces **one canonical form** for the same logical identifier:

- **Hyphenated form is canonical** when the identifier comes from a crate name.
- When referring to a dependency (in features, package.metadata, etc.), you **must** use the **hyphenated** version that matches the `[dependencies]` key.

Allowed (2024 Edition):
```toml
[dependencies]
serde_json = "1.0"

[features]
default = ["serde-json"]

[package.metadata.docs.rs]
features = ["serde-json"]
```

Forbidden (hard error in 2024 Edition):
```toml
[features]
serde_json = ["serde_json"]      # ← error: should be serde-json
```

Error message looks something like:

```
error: feature name `serde_json` is not consistent with the dependency name `serde_json`
  --> Cargo.toml:8:1
   |
8  | serde_json = ["serde_json"]
   | ^^^^^^^^^^ help: use the hyphenated form: `serde-json`
```

#### Affected Sections

| Section                        | Must use hyphen form if it refers to a dependency |
|--------------------------------|----------------------------------------------------|
| `[features]`                   | Yes                                                |
| `[package.metadata.*]`         | Yes (especially `docs.rs`, `release`)              |
| `[target.*.dependencies]`      | Yes (when inheriting names)                        |
| `[workspace.dependencies]`     | Yes                                                |
| `[patch.crates-io]`            | Yes                                                |

Keys that are **not** derived from crate names (e.g. custom `[package.metadata.my-tool]`) are unaffected.

#### Migration Steps

1. Run **`cargo fix --edition`**  
   Cargo will automatically rewrite most inconsistent feature names and metadata keys to the hyphenated form.

2. Search your `Cargo.toml` for underscore usage in feature names:
   ```bash
   grep -r "_ =" Cargo.toml
   ```

3. Fix manually if needed (especially in complex `[package.metadata]` tables).

4. After migration, re-run `cargo check`, `cargo doc`, `cargo publish --dry-run` to verify.

#### Rationale

- Enforces a single, predictable convention (hyphens = crates.io style).
- Prevents silent misconfigurations (especially on docs.rs).
- Makes future Cargo features (e.g. smarter metadata parsing) safer.
- Aligns with how most Rust developers already write feature names.

#### Official Reference
https://doc.rust-lang.org/edition-guide/rust-2024/cargo-table-key-names.html

This is a small but important hygiene change that eliminates a long-standing source of subtle bugs in large projects and published crates.