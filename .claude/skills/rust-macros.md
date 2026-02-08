---
name: rust-macros
description: "Use this skill when you need to design, review, or implement Rust macros. Covers both declarative (macro_rules!) and procedural (proc_macro) macros including advanced patterns like TT-munching, push-down accumulation, internal rules, hygiene, testing with trybuild, and debugging with cargo-expand."
---

# Rust Macros Design skill

You are a specialized skill for designing, reviewing, and implementing Rust macros. You have deep knowledge of both declarative (`macro_rules!`) and procedural (`proc_macro`) macros. Use this reference to guide all macro-related work.

---

## Table of Contents

1. [Choosing the Right Macro Type](#1-choosing-the-right-macro-type)
2. [Declarative Macros (`macro_rules!`)](#2-declarative-macros-macro_rules)
3. [Fragment Specifiers Reference](#3-fragment-specifiers-reference)
4. [Follow-Set Restrictions](#4-follow-set-restrictions)
5. [Repetition Operators](#5-repetition-operators)
6. [Declarative Macro Patterns](#6-declarative-macro-patterns)
7. [Macro Hygiene](#7-macro-hygiene)
8. [Rules for Writing Hygienic Macros](#8-rules-for-writing-hygienic-macros)
9. [Procedural Macros](#9-procedural-macros)
10. [Procedural Macro Project Structure](#10-procedural-macro-project-structure)
11. [Nine Rules for Procedural Macros](#11-nine-rules-for-procedural-macros)
12. [Procedural Macro Hygiene and Spans](#12-procedural-macro-hygiene-and-spans)
13. [Testing Macros](#13-testing-macros)
14. [Debugging Macros](#14-debugging-macros)
15. [Performance Considerations](#15-performance-considerations)
16. [When NOT to Use Macros](#16-when-not-to-use-macros)
17. [Reference Crates and Resources](#17-reference-crates-and-resources)

---

## 1. Choosing the Right Macro Type

| Feature | Declarative (`macro_rules!`) | Procedural (`proc_macro`) |
| --- | --- | --- |
| **Paradigm** | Pattern matching (DSL-like) | Imperative (Rust code) |
| **Input** | Token streams via matchers | Token streams via `TokenStream` type |
| **Best Use Case** | Small DSLs, DRY-ing up repetitive logic | Complex code generation, custom Attributes, Derives |
| **Tooling** | Built-in to the language | Requires `syn` and `quote` crates |
| **Compile Time** | Faster | Slower (requires compiling a separate macro crate) |
| **Error Messages** | Limited control | Full control over spans and diagnostics |
| **Hygiene** | Mixed (partial) | Configurable via `Span` types |

**Decision Rule:** Start with `macro_rules!` unless you need one of:
- Custom derive implementations
- Attribute macros that transform items
- Complex syntax tree manipulation
- Rich, user-friendly error messages with precise spans
- Access to type information or complex parsing

Avoid procedural macros unless declarative macros truly cannot address the problem, as procedural macros increase compile complexity and maintenance cost.

---

## 2. Declarative Macros (`macro_rules!`)

### Basic Structure

```rust
macro_rules! my_macro {
    // Rule 1: base case
    () => { /* expansion */ };

    // Rule 2: pattern with captures
    ($x:expr) => { /* use $x */ };

    // Rule 3: pattern with repetition
    ($($item:expr),* $(,)?) => {
        // use $($item)*
    };
}
```

### Key Mechanics

- **Rules are tried top-to-bottom.** The first matching rule wins. Place the most specific (or most common) rules first.
- **Outer delimiters are interchangeable.** `my_macro!()`, `my_macro![]`, and `my_macro!{}` all work — the outer delimiters in the definition match any pair.
- **The `$` character cannot be matched or transcribed literally** — it is reserved for the macro engine.
- **Metavariable references:** In the transcriber (right side of `=>`), use `$name` to substitute captured fragments.

### Macro Export and Visibility

```rust
// To export from a crate:
#[macro_export]
macro_rules! public_macro {
    () => {};
}

// In 2018+ edition, exported macros appear at crate root regardless of where they are defined.
// Users import with: use my_crate::public_macro;
```

---

## 3. Fragment Specifiers Reference

Fragment specifiers define what kind of syntax a metavariable captures:

| Specifier | Matches | Description |
|-----------|---------|-------------|
| `block` | `{ ... }` | A block expression |
| `expr` | Any expression | Full expression (2024 edition adds `_` and `const {}`) |
| `expr_2021` | Expression | Backwards-compatible expression (excludes `_` and `const {}`) |
| `ident` | Identifier/keyword | An identifier or keyword (except bare `_`), raw identifiers, or `$crate` |
| `item` | Item | Function, struct, enum, impl, trait, use, extern, etc. |
| `lifetime` | `'label` | A lifetime token |
| `literal` | `-`? literal | A literal value, optionally negated |
| `meta` | Attribute contents | Inner contents of an attribute |
| `pat` | Pattern | A pattern (2021+: includes top-level or-patterns `A \| B`) |
| `pat_param` | Pattern | A pattern without top-level or-alternatives |
| `path` | Type path | A path like `std::collections::HashMap` |
| `stmt` | Statement | A statement (no trailing semicolon, except for items requiring one) |
| `tt` | Token tree | A single token OR a balanced group in `()`, `[]`, or `{}` — the most flexible specifier |
| `ty` | Type | Any type expression |
| `vis` | Visibility | A possibly empty visibility qualifier (`pub`, `pub(crate)`, etc.) |

**Critical:** `tt` is the only specifier that can losslessly capture arbitrary input. This is why `$($tail:tt)*` is the standard idiom for capturing "the rest" of the input in recursive macros.

---

## 4. Follow-Set Restrictions

Fragment specifiers can only be followed by specific tokens. Violating these rules causes a compile error:

| Specifier | Allowed Followers |
|-----------|-------------------|
| `expr`, `stmt` | `=>`, `,`, `;` |
| `pat_param` | `=>`, `,`, `=`, `\|`, `if`, `in` |
| `pat` | `=>`, `,`, `=`, `if`, `in` |
| `path`, `ty` | `=>`, `,`, `=`, `\|`, `;`, `:`, `>`, `>>`, `[`, `{`, `as`, `where`, or a `block` metavariable |
| `vis` | `,`, an identifier (except raw `priv`), any type-starting token, or an `ident`/`ty`/`path` metavariable |
| `ident`, `block`, `item`, `lifetime`, `literal`, `meta`, `tt` | **No restrictions** |

---

## 5. Repetition Operators

```rust
$( ... )*      // Zero or more
$( ... )+      // One or more
$( ... )?      // Zero or one (optional — cannot use a separator)
```

**Separator rules:**
- A separator token can appear between any repeated elements.
- Common separators: `,` and `;`.
- The separator cannot be a delimiter (`(`, `)`, `[`, `]`, `{`, `}`) or a repetition operator.
- The `?` operator does not support separators.
- Example: `$( $i:ident ),*` matches `a, b, c`.

**Transcription rules:**
- A metavariable must appear at the same nesting depth of repetitions in the transcriber as in the matcher.
- Each repetition in the transcriber must contain at least one metavariable to determine the expansion count.
- Multiple metavariables in the same repetition must bind to the same number of fragments.

```rust
// Valid: replaces commas with semicolons
macro_rules! transform {
    ($( $i:ident ),*) => { $( $i );* };
}

// Valid: trailing comma support
macro_rules! my_vec {
    ($( $x:expr ),* $(,)?) => {
        {
            let mut v = Vec::new();
            $( v.push($x); )*
            v
        }
    };
}
```

---

## 6. Declarative Macro Patterns

### 6.1 TT Muncher (Incremental Token Tree Munching)

A recursive `macro_rules!` macro that processes input one token tree at a time. At each step it:
1. Matches and removes tokens from the start of input
2. Generates intermediate output
3. Recurses on the remaining tail captured as `$($tail:tt)*`

```rust
macro_rules! count_exprs {
    // Base case: no input left
    () => { 0 };
    // Recursive case: consume one expression, recurse on rest
    ($head:expr $(, $tail:expr)*) => {
        1 + count_exprs!($($tail),*)
    };
}
```

**When to use:** When you need to process a heterogeneous sequence of tokens where standard repetition (`*`, `+`) is insufficient because each element requires different handling.

**Performance:** TT munchers are **inherently quadratic**. Processing N token trees requires N + (N-1) + (N-2) + ... + 1 matching operations.

**Best practices:**
- Place frequently-matched rules first to minimize failed match attempts.
- Prefer standard repetition (`*` or `+`) whenever possible instead of recursion.
- Keep the number of rules small.
- Be aware of the `recursion_limit` (default 128, adjustable with `#![recursion_limit = "256"]`).

### 6.2 Internal Rules

Internal rules consolidate multiple macro helpers into a single macro definition using a prefix token (conventionally `@`) as a dispatcher:

```rust
#[macro_export]
macro_rules! foo {
    // Internal rule: prefixed with @
    (@as_expr $e:expr) => { $e };

    // Public-facing rule: dispatches to internal rule
    ($($tts:tt)*) => {
        foo!(@as_expr $($tts)*)
    };
}
```

**Why `@`?** The `@` token is not used in prefix position in Rust syntax, so it cannot conflict with user input.

**Rule ordering:** Internal `@`-prefixed rules must come before bare rules to prevent the compiler from misparsing internal invocations.

**Benefits:**
- Eliminates namespace pollution (no need to export helper macros).
- Avoids macro name collisions.
- Works naturally with TT muncher patterns.

**Trade-off:** Internal rules can hurt compile times because the compiler attempts to match all rules sequentially and the `@identifier` prefix adds matching overhead.

### 6.3 Push-Down Accumulation

Build up a sequence of tokens incrementally using an accumulator parameter, emitting the complete construct only when a termination condition is met:

```rust
macro_rules! reverse {
    // Termination: input empty, emit accumulator
    ([] -> [$($acc:tt)*]) => {
        $($acc)*
    };
    // Recursive: move one token from input to accumulator
    ([$first:tt $($rest:tt)*] -> [$($acc:tt)*]) => {
        reverse!([$($rest)*] -> [$first $($acc)*])
    };
}

// Usage: reverse!([a b c] -> []) expands to c b a
```

**Convention:** Use `($input) -> ($output)` syntax to clarify the flow.

**Critical:** The accumulator must be captured as `$($acc:tt)*` (token trees), not as any other fragment specifier — only `tt` allows lossless intermediate storage.

**Performance:** Push-down accumulation is **inherently quadratic**. Combined with TT munching, it becomes **doubly quadratic**.

**Optimization:** Place the accumulator at the **end** of rules. If a rule fails to match, the compiler won't waste time matching the (potentially long) accumulator before reaching the failing part.

### 6.4 Callbacks

Pass the name of another macro as a parameter, calling it with the result once processing is done:

```rust
macro_rules! process {
    ($callback:ident -> ($($input:tt)*)) => {
        $callback!($($input)*)
    };
}
```

Useful for composing macros and building pipelines.

---

## 7. Macro Hygiene

`macro_rules!` macros in Rust are **partially hygienic** ("mixed-site hygiene"):

**Hygienic (definition-site resolution):**
- Local variables (let bindings)
- Labels (loop labels, block labels)
- `$crate` metavariable

**Unhygienic (call-site resolution):**
- Function names
- Type names
- Module paths
- Trait names
- All other items

### How It Works

Every identifier carries an invisible "syntax context." Two identifiers are equal only if both their textual name AND syntax context match. Each macro expansion gets a unique syntax context. Tokens substituted into the expansion (from captured metavariables) retain their original syntax context.

```rust
macro_rules! using_a {
    ($e:expr) => {
        {
            let a = 42;  // This 'a' has the macro's syntax context
            $e           // This uses the caller's syntax context
        }
    };
}

let a = 10;
let result = using_a!(a * 2);
// result = 20, NOT 84 — because the caller's `a` (10) is different
// from the macro's internal `a` (42) due to hygiene.
```

### The `$crate` Metavariable

`$crate` resolves to the crate where the macro is defined, providing a stable path regardless of how the macro is invoked:

```rust
// In crate `my_lib`:
#[macro_export]
macro_rules! use_my_type {
    () => {
        $crate::MyType::new()
    };
}
```

- `$crate` is only available in `macro_rules!` (not in procedural macros).
- For procedural macros, use re-exports or accept a crate path parameter.

---

## 8. Rules for Writing Hygienic Macros

Follow these rules to ensure macros work correctly in all contexts:

### Rule 1: Use Fully Qualified Paths for Standard Library

Always start standard library paths with `::` to avoid conflicts with user-defined modules.

```rust
// BAD
let foo: Option<()>;

// GOOD
let foo: ::core::option::Option<()>;
```

### Rule 2: Use `$crate` for Your Own Crate (Declarative Macros)

Never hardcode your crate name — it may be renamed by users.

```rust
// BAD
::my_crate::do_things()

// GOOD
$crate::do_things()
```

### Rule 3: Use `self::` for Items Defined in the Same Module

```rust
// BAD (could conflict with a crate named Foo)
let foo: Foo;

// GOOD
let foo: self::Foo;
```

### Rule 4: Never Use Trait Methods Directly

Trait methods depend on which traits are in scope at the call site. Always use fully qualified syntax (UFCS):

```rust
// BAD
let s = "Hello".to_owned();

// GOOD
let s = ::std::borrow::ToOwned::to_owned("Hello");

// BAD
let v = vec.into_iter();

// GOOD
let v = ::std::iter::IntoIterator::into_iter(vec);
```

### Rule 5: Qualify Primitive Types

Primitives are not keywords and can be shadowed. Use absolute paths:

```rust
// BAD
let number: i32 = 5;

// GOOD
let number: ::core::primitive::i32 = 5;
```

### Rule 6: Re-export `core` and `alloc` for `no_std` Support

```rust
// In your crate root:
#[doc(hidden)]
pub use ::core;
#[doc(hidden)]
pub use ::alloc;

// In your macro:
$crate::core::option::Option::Some(value)
```

### Rule 7: For Procedural Macros — Support Crate Renaming

Accept a parameter allowing users to specify the crate path:

```rust
#[my_derive(crate = "my_crate_renamed")]
struct Foo { ... }
```

### Rule 8: For Procedural Macros — Use `Span::mixed_site()`

Prefer `Span::mixed_site()` over `Span::call_site()` to protect against variable and item interference. This mirrors `macro_rules!` hygiene behavior.

### Rule 9: Test with `#![no_implicit_prelude]`

Include tests that use this attribute to verify your macros don't accidentally depend on the prelude:

```rust
#![no_implicit_prelude]

// Test your macro invocations here — if they compile, they're hygienic.
```

---

## 9. Procedural Macros

There are three types of procedural macros:

### 9.1 Derive Macros

Automatically implement traits on structs/enums:

```rust
use proc_macro::TokenStream;
use quote::quote;
use syn::{parse_macro_input, DeriveInput};

#[proc_macro_derive(MyTrait, attributes(my_attr))]
pub fn my_derive(input: TokenStream) -> TokenStream {
    let input = parse_macro_input!(input as DeriveInput);
    let name = &input.ident;

    let expanded = quote! {
        impl MyTrait for #name {
            fn my_method(&self) -> &str {
                stringify!(#name)
            }
        }
    };

    TokenStream::from(expanded)
}
```

### 9.2 Attribute Macros

Transform the item they annotate:

```rust
#[proc_macro_attribute]
pub fn my_attribute(attr: TokenStream, item: TokenStream) -> TokenStream {
    // attr = arguments to the attribute
    // item = the annotated item
    // Return the (possibly modified) item
    item
}
```

### 9.3 Function-like Macros

Invoked like declarative macros but with full procedural power:

```rust
#[proc_macro]
pub fn my_macro(input: TokenStream) -> TokenStream {
    // Parse and transform input
    input
}
```

### Essential Crates

- **`syn`** (v2.0+): Parses `TokenStream` into a Rust AST. Enable the `"full"` feature for complete syntax support.
- **`quote`**: The `quote!` macro generates `TokenStream` from Rust-like syntax. Use `#variable` for interpolation and `#(#iter)*` for repetition.
- **`proc-macro2`**: Wrapper around `proc_macro` types that works outside of procedural macro context (enables unit testing).

```toml
[dependencies]
syn = { version = "2", features = ["full"] }
quote = "1"
proc-macro2 = "1"
```

---

## 10. Procedural Macro Project Structure

### Recommended Pipeline Architecture

Structure your proc-macro crate as a pipeline with distinct, testable stages:

```
my_macro/
├── Cargo.toml          # proc-macro = true
├── src/
│   ├── lib.rs          # Entry point: #[proc_macro_derive(...)]
│   ├── parse.rs        # Stage 1: TokenStream → AST (using syn)
│   ├── analyze.rs      # Stage 2: AST → Domain-specific model
│   ├── lower.rs        # Stage 3: Domain model → Codegen IR
│   └── codegen.rs      # Stage 4: IR → TokenStream (using quote!)
└── tests/
    ├── integration.rs  # End-to-end tests
    └── ui/             # trybuild compile-fail tests
        ├── fail_01.rs
        └── fail_01.stderr
```

**Why this structure?**
- Each stage can be unit-tested independently.
- The codegen stage is kept "dumb as possible" — it just maps IR to tokens.
- Complex logic lives in `analyze` and `lower`, which are ordinary Rust code testable with standard `#[test]`.
- Panics in codegen can be diagnosed with `RUST_BACKTRACE=1` on unit tests.

### Workspace Pattern

Use a Rust workspace where:
- The proc-macro crate lives in a sub-crate
- The main crate re-exports the macro
- A `tests` crate exercises the macro as a user would

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

---

## 11. Nine Rules for Procedural Macros

(Based on Carl Kadie's "Nine Rules for Creating Procedural Macros in Rust")

1. **Use a workspace and `proc_macro2`** — Develop, debug, and unit-test your macro logic in a normal (non-macro) crate using `proc_macro2` types. This gives you a standard development experience.

2. **Convert freely between code representations** — Use `syn`, `proc_macro2`, and `quote` to move between literal code, tokens, syntax trees, and strings as needed.

3. **Create debuggable unit tests** — Write unit tests that clearly report differences between what your macro produces and what you expect. Compare token streams as strings for readable diffs.

4. **Understand syntax trees** — Use the `syn` documentation and AST explorer tools to understand Rust syntax tree structures. Destructure them with Rust's pattern matching and struct/enum field access.

5. **Construct syntax trees with `parse_quote!`** — Use `parse_quote!` to build syntax tree nodes from quasi-quoted Rust syntax, and use struct update syntax for modifications.

6. **Use `syn`'s `Fold` trait** — Recursively traverse, destructure, and reconstruct syntax trees using the Fold visitor pattern when you need to transform nested structures.

7. **Handle errors ergonomically** — Return user-friendly, testable error messages. Use `syn::Error` for errors with span information that points to the problematic source code.

8. **Create integration tests with `trybuild`** — Write both passing and failing integration tests. Use `trybuild` for UI tests that verify your macro produces the correct compiler errors on invalid input.

9. **Follow API design principles** — Eat your own dogfood (use your macro yourself), run Clippy on both the macro crate and generated code, and write thorough documentation.

---

## 12. Procedural Macro Hygiene and Spans

Every token in a `TokenStream` carries an associated `Span` that encodes source location and hygiene information.

### Three Span Types

| Span Type | Hygiene | Use When |
|-----------|---------|----------|
| `Span::call_site()` | **Unhygienic** — resolves at the invocation site | You want generated identifiers to be accessible to the caller (e.g., a new function the user should call) |
| `Span::mixed_site()` | **Mixed** — mirrors `macro_rules!` behavior | **Default choice** — local variables are hygienic, items resolve at call site |
| `Span::def_site()` | **Fully hygienic** — resolves at macro definition (**unstable**) | Maximum isolation; generated names invisible to caller |

**Best practice:** Use `Span::mixed_site()` as your default. Only use `Span::call_site()` when you specifically need generated identifiers to be visible to the caller.

### Setting Spans

```rust
use proc_macro2::{Ident, Span};

// Create an identifier with mixed-site hygiene
let my_ident = Ident::new("my_var", Span::mixed_site());

// Create an identifier visible at call site
let user_facing = Ident::new("new_function", Span::call_site());

// Copy span from user input for good error messages
let derived_name = Ident::new(&format!("{}Builder", name), name.span());
```

---

## 13. Testing Macros

### Unit Tests (Procedural Macros)

Test individual pipeline stages with synthesized inputs:

```rust
#[test]
fn test_parse_stage() {
    let input: syn::DeriveInput = syn::parse_quote! {
        #[my_attr(option = "value")]
        struct Foo {
            bar: String,
        }
    };
    let model = analyze(input);
    assert_eq!(model.fields.len(), 1);
    assert_eq!(model.options.get("option"), Some(&"value".to_string()));
}
```

### Compile-Fail Tests with `trybuild`

```rust
// tests/ui.rs
#[test]
fn ui_tests() {
    let t = trybuild::TestCases::new();
    t.pass("tests/ui/pass_*.rs");       // These should compile
    t.compile_fail("tests/ui/fail_*.rs"); // These should fail with expected errors
}
```

Each `fail_*.rs` file has a corresponding `fail_*.stderr` file with the expected compiler output. Run with `TRYBUILD=overwrite` to regenerate `.stderr` files, then review with `git diff`.

### Testing Declarative Macros

```rust
#[test]
fn test_my_macro() {
    // Test expansion result
    assert_eq!(my_macro!(1, 2, 3), expected_value);
}

// Test hygiene with no_implicit_prelude
#[test]
fn test_hygiene() {
    mod inner {
        #![no_implicit_prelude]
        // Invoke macro here — if it compiles, it's hygienic
        super::my_macro!();
    }
}
```

---

## 14. Debugging Macros

### `cargo-expand`

The single most important tool for macro debugging. Shows the generated code after all macro expansion:

```bash
# Install
cargo install cargo-expand

# Expand all macros in the crate
cargo expand

# Expand a specific module
cargo expand module_name

# Expand a specific test
cargo expand --test test_name

# Expand with specific features
cargo expand --features my_feature
```

**Note:** Macro expansion to text is lossy — the expanded output may not compile identically, but it reveals the structure of generated code.

### Rust-Analyzer

Use the "Expand macro recursively" command (requires experimental proc-macro support enabled) to inspect expansion with correct span information directly in your IDE.

### Debugging Panics in Proc-Macros

When a procedural macro panics during integration tests:
1. Run the unit tests for the codegen stage with `RUST_BACKTRACE=1`.
2. Since codegen is ordinary Rust code (thanks to the pipeline architecture), you get meaningful stack traces.

### `eprintln!` in Proc-Macros

You can use `eprintln!` inside procedural macros — output appears during compilation:

```rust
#[proc_macro_derive(MyDerive)]
pub fn my_derive(input: TokenStream) -> TokenStream {
    let input = parse_macro_input!(input as DeriveInput);
    eprintln!("Generating code for: {}", input.ident);
    // ...
}
```

---

## 15. Performance Considerations

### Declarative Macros

- **TT munchers are quadratic** in the number of token trees. Avoid them for inputs that could be large.
- **Push-down accumulation is quadratic** in the accumulator size. Combined with TT munching, it's **doubly quadratic**.
- **Optimization:** Place accumulators at the end of rules so failed matches short-circuit before matching the long accumulator.
- **Rule ordering:** Place the most common rules first. The compiler tries rules top-to-bottom.
- **Prefer repetition over recursion:** `$($x:expr),*` is O(n); recursive TT munching is O(n^2).

### Procedural Macros

- **Separate crate compilation:** Procedural macros require compiling a separate crate, adding to build time.
- **`syn` parse features:** Only enable the `syn` features you need. `features = ["full"]` parses all Rust syntax; `features = ["derive"]` is lighter for derive-only macros.
- **Minimize re-parsing:** Parse the input once, transform the AST, then generate output. Don't round-trip through strings.
- **`proc-macro2` wrapping:** The overhead of `proc-macro2` is negligible — always use it for testability.

### General

- Avoid generating massive amounts of code in a macro — it all has to be parsed and type-checked by the compiler.
- Consider whether a trait with a blanket impl could replace a derive macro.
- Profile compile times with `cargo build --timings` to identify macro-heavy bottlenecks.

---

## 16. When NOT to Use Macros

(From "Rust for Rustaceans" by Jon Gjengset)

Do not reach for a macro when:

- **A function suffices.** If your macro just wraps a function call, use the function directly.
- **A trait with a blanket impl works.** Generic traits often eliminate the need for derive macros.
- **The macro harms IDE support.** Macros can break autocompletion, go-to-definition, and refactoring tools. If DX suffers, reconsider.
- **The macro obscures intent.** Code readers should understand what happens without expanding the macro mentally.
- **Compile time is critical.** Every macro invocation costs compile time. In hot paths of large projects, this adds up.
- **The macro is used once.** If there's only one call site, inline the code.
- **Error messages become cryptic.** If the macro produces incomprehensible compiler errors, users will struggle.

**Rule of thumb:** A macro is justified when it eliminates substantial boilerplate that cannot be addressed by generics, traits, or functions, and the generated code remains predictable.

---

## 17. Reference Crates and Resources

### Essential Crates

| Crate | Purpose |
|-------|---------|
| [`syn`](https://docs.rs/syn/) | Parse `TokenStream` into Rust AST |
| [`quote`](https://docs.rs/quote/) | Generate `TokenStream` from quasi-quoted Rust code |
| [`proc-macro2`](https://docs.rs/proc-macro2/) | Wrapper types enabling unit testing of proc-macros |
| [`cargo-expand`](https://github.com/dtolnay/cargo-expand) | Show macro expansion output — the #1 debugging tool |
| [`trybuild`](https://github.com/dtolnay/trybuild) | Test harness for compile-fail and compile-pass tests |

### Study These Crates for Architecture Patterns

| Crate | What It Teaches |
|-------|-----------------|
| [`thiserror`](https://github.com/dtolnay/thiserror) | Clean, attribute-based proc-macro architecture |
| [`serde`](https://github.com/serde-rs/serde) | Handling complex generics and trait bounds at scale |
| [`clap` (derive)](https://github.com/clap-rs/clap) | Attribute macro design for CLI argument parsing |

### Learning Resources

| Resource | Focus |
|----------|-------|
| [The Little Book of Rust Macros](https://lukaswirth.dev/tlborm/) | The "bible" for `macro_rules!` — advanced patterns, TT munching, push-down accumulation, internal rules |
| [David Tolnay's Proc-Macro Workshop](https://github.com/dtolnay/proc-macro-workshop) | Best hands-on guide for procedural macros — build a Builder derive, custom Debug, syntax parsers |
| [The Rust Reference: Macros](https://doc.rust-lang.org/reference/macros.html) | Authoritative specification for hygiene, expansion, and fragment specifier rules |
| [The Rust Book Ch. 19](https://doc.rust-lang.org/book/ch19-06-macros.html) | Conceptual overview of all macro types |
| [Rust for Rustaceans](https://nostarch.com/rust-rustaceans) | When to use (and not use) macros; impact on compilation and IDE support |
| [Ferrous Systems: Testing Proc-Macros](https://ferrous-systems.com/blog/testing-proc-macros/) | Pipeline architecture, unit testing each stage, debugging techniques |
| [Nine Rules for Proc-Macros](https://users.rust-lang.org/t/nine-rules-for-creating-procedural-macros-in-rust/83080) | Practical workflow rules for proc-macro development |
| [Hygienic Macro Guide (Kestrer)](https://gist.github.com/Kestrer/8c05ebd4e0e9347eb05f265dfb7252e1) | Complete rules for writing macros that work in every context |
