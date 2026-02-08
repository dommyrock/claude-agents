# Procedural Macro Guide

## Workspace Pattern

```
my_project/
├── Cargo.toml            # [workspace]
├── my_lib/
│   ├── Cargo.toml        # depends on my_lib_macros
│   └── src/lib.rs        # pub use my_lib_macros::MyDerive;
├── my_lib_macros/
│   ├── Cargo.toml        # [lib] proc-macro = true
│   └── src/lib.rs
└── my_lib_tests/
    ├── Cargo.toml        # depends on my_lib
    └── tests/
```

## Nine Rules for Procedural Macros

(Based on Carl Kadie's guide)

1. **Use a workspace and `proc_macro2`** — develop and unit-test macro logic in a normal crate
2. **Convert freely between code representations** — use `syn`, `proc_macro2`, and `quote`
3. **Create debuggable unit tests** — compare token streams as strings for readable diffs
4. **Understand syntax trees** — use syn docs and AST explorer tools
5. **Construct syntax trees with `parse_quote!`** — build nodes from quasi-quoted Rust
6. **Use `syn`'s `Fold` trait** — recursively traverse and reconstruct syntax trees
7. **Handle errors ergonomically** — use `syn::Error` for errors with span information
8. **Create integration tests with `trybuild`** — compile-fail and compile-pass tests
9. **Follow API design principles** — eat your own dogfood, run Clippy, write docs

## Hygiene Rules for Procedural Macros

These complement the Nine Rules above with proc-macro-specific hygiene guidance (from the Hygienic Macro Guide):

- **Support crate renaming** — accept `#[my_derive(crate = "renamed")]`
- **Use `Span::mixed_site()`** — prefer over `call_site()` for protection against variable interference
- **Test with `#![no_implicit_prelude]`** — verify macros don't depend on the prelude

## Performance Notes

### Declarative
- TT munchers: O(n^2). Avoid for large inputs.
- Push-down accumulation: O(n^2). Combined with TT munching: O(n^4).
- Rule ordering matters — compiler tries top-to-bottom.

### Procedural
- Separate crate compilation adds build time.
- Only enable needed `syn` features (`"derive"` is lighter than `"full"`).
- Parse once, transform AST, generate output. Don't round-trip through strings.
- Profile with `cargo build --timings`.
