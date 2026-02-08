---
name: rust-macros
description: "Provides decision frameworks and patterns for Rust macro design, covering both declarative (macro_rules!) and procedural (proc_macro) macros including advanced patterns like TT-munching, push-down accumulation, internal rules, hygiene, testing with trybuild, and debugging with cargo-expand."
---

# Rust Macros Design Skill

Specialized guidance for designing, reviewing, and implementing Rust macros. Focuses on decision frameworks, common pitfalls, and patterns that go beyond standard Rust knowledge.

---

## Choosing the Right Macro Type

| Feature | `macro_rules!` | `proc_macro` |
| --- | --- | --- |
| **Best Use Case** | Small DSLs, DRY-ing repetitive logic | Complex codegen, custom derives, attributes |
| **Compile Time** | Faster | Slower (separate crate) |
| **Error Messages** | Limited | Full control over spans |
| **Hygiene** | Mixed (partial) | Configurable via `Span` |

**Decision Rule:** Start with `macro_rules!` unless you need:
- Custom derive implementations
- Attribute macros that transform items
- Complex syntax tree manipulation
- Rich error messages with precise spans
- Access to type information

## When NOT to Use Macros

Do not reach for a macro when:
- A function suffices
- A trait with a blanket impl works
- The macro harms IDE support (autocompletion, go-to-definition)
- The macro obscures intent
- There's only one call site — inline the code
- Error messages become cryptic

**Rule of thumb:** A macro is justified when it eliminates substantial boilerplate that cannot be addressed by generics, traits, or functions.

---

## Declarative Macro Quick Reference

Key mechanics:
- Rules are tried **top-to-bottom** — most specific first
- `$` is reserved for the macro engine, cannot be matched literally
- `tt` is the only specifier that losslessly captures arbitrary input
- Trailing comma support: `$($x:expr),* $(,)?`

---

## Procedural Macro Quick Reference

### Pipeline Architecture

Structure proc-macro crates as testable pipelines:

```
my_macro/src/
├── lib.rs       # Entry point
├── parse.rs     # TokenStream → AST (syn)
├── analyze.rs   # AST → Domain model
├── lower.rs     # Domain model → Codegen IR
└── codegen.rs   # IR → TokenStream (quote!)
```

Each stage is independently unit-testable. Keep codegen "dumb" — complex logic lives in analyze/lower.

### Span Best Practices

| Span Type | Use When |
|-----------|----------|
| `Span::mixed_site()` | **Default choice** — mirrors macro_rules! hygiene |
| `Span::call_site()` | Generated identifiers must be visible to caller |
| `Span::def_site()` | Maximum isolation (unstable) |

### Testing Strategy

- **Unit tests**: Test each pipeline stage with `syn::parse_quote!` inputs
- **Compile-fail tests**: Use `trybuild` with `.rs`/`.stderr` pairs
- **Hygiene tests**: Test with `#![no_implicit_prelude]`
- Regenerate `.stderr` files with `TRYBUILD=overwrite`, review with `git diff`

### Debugging

- `cargo expand` — the #1 macro debugging tool
- `eprintln!` inside proc-macros outputs during compilation
- `RUST_BACKTRACE=1` on unit tests for codegen panics
- Rust-Analyzer "Expand macro recursively" command

---

## Detailed References

For in-depth coverage of specific topics:

- **Declarative macro patterns** (TT munching, push-down accumulation, internal rules, callbacks): See [declarative-patterns.md](declarative-patterns.md)
- **Follow-set restrictions and repetition operators**: See [fragment-reference.md](fragment-reference.md)
- **Procedural macro nine rules and workspace patterns**: See [procedural-guide.md](procedural-guide.md)
- **Essential crates and learning resources**: See [resources.md](resources.md)
