# Rustfmt: Style edition

**Edition: 2024**

## Summary

Users can now control which version (edition) of the Rust Style Guide `rustfmt` should follow when formatting code.  
This is called the **Style Edition**, and it can be set independently of the language edition in some cases.

## Motivation / Problem (Pre-2024 Behavior)

Before the 2024 edition, Rustfmt faced a difficult tension between two strong goals:

1. **Following the latest Rust Style Guide recommendations** (to produce modern, consistent, idiomatic formatting)
2. **Formatting stability guarantee** — once code was successfully formatted by a particular `rustfmt` version, newer versions were not allowed to change that output (to prevent massive, noisy diffs every time someone updated their toolchain)

This created a de-facto freeze:

- The default formatting behavior could not meaningfully improve or fix long-standing quirks.
- The Rust Style Guide could not evolve without breaking existing formatted code.
- Legacy formatting bugs and awkward choices (e.g. strange nested tuple formatting, inconsistent spacing in some constructs) were effectively permanent.
- Style improvements discovered after 2018 had almost no path to reach most users.

As a result, both the community and the language team were stuck: either accept stagnant formatting forever, or cause painful widespread reformatting churn — neither option was satisfactory.

[RFC 3338](https://rust-lang.github.io/rfcs/3338-style-evolution.html) solved this impasse by **aligning style evolution to the edition system**, allowing the Style Guide (and thus `rustfmt` defaults) to advance in controlled steps — just like the language itself.

## Details

In the **2024 Edition**, `rustfmt` gains support for a **Style Edition** concept:

- By default, the Style Edition matches the crate’s language `edition` (set in `Cargo.toml`).
- You can override it separately via configuration.

Ways to use the 2024 Style Edition:

**1. Via Cargo (recommended for most projects)**

```toml
# Cargo.toml
[package]
edition = "2024"
```

```bash
cargo fmt
```

→ automatically uses 2024 style rules

**2. Via command line**

```bash
rustfmt src/lib.rs --edition 2024
```

**3. Via rustfmt.toml / .rustfmt.toml (persistent project setting)**

```toml
style_edition = "2024"
```

Then just run:

```bash
rustfmt src/lib.rs
# or let your editor run rustfmt on save
```

**4. One-off override**

```bash
rustfmt src/lib.rs --style-edition 2024
```

The 2024 Style Edition includes several formatting improvements (such as better raw identifier sorting, version sorting in attributes, and various small fixes) described in other sections of this guide.

## Migration

Migrating is usually automatic:

```bash
cargo fmt          # if edition = "2024" in Cargo.toml
# or
rustfmt --edition 2024 **/*.rs
```

This will reformat your code using the 2024 style rules.

**Important recommendation for teams / open-source projects:**

Add a `rustfmt.toml` file to your repository:

```toml
# rustfmt.toml
style_edition = "2024"
```

This helps ensure that:

- `cargo fmt` (CI, manual runs) and editor format-on-save produce **the same output**
- Contributors using different editors / toolchains don’t fight over formatting
- People who run `rustfmt` directly (many editors do this by default) still get 2024-style formatting instead of falling back to 2015 defaults

Without this file, editors that invoke `rustfmt` without `--edition` will often apply outdated 2015-era formatting, creating inconsistent diffs.

See also: the individual 2024 formatting changes (raw identifier sorting, version sorting, etc.) documented in this guide.
