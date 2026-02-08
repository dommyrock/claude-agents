# Declarative Macro Patterns

## TT Muncher (Incremental Token Tree Munching)

A recursive `macro_rules!` macro that processes input one token tree at a time:
1. Match and remove tokens from the start of input
2. Generate intermediate output
3. Recurse on the remaining tail captured as `$($tail:tt)*`

```rust
macro_rules! count_exprs {
    () => { 0 };
    ($head:expr $(, $tail:expr)*) => {
        1 + count_exprs!($($tail),*)
    };
}
```

**When to use:** When you need to process a heterogeneous sequence where standard repetition is insufficient.

**Performance:** Inherently **quadratic** — N + (N-1) + ... + 1 matching operations.

**Best practices:**
- Place frequently-matched rules first
- Prefer standard repetition (`*` or `+`) whenever possible
- Be aware of `recursion_limit` (default 128, adjustable with `#![recursion_limit = "256"]`)

## Internal Rules

Consolidate multiple macro helpers into a single macro using a `@` prefix as dispatcher:

```rust
#[macro_export]
macro_rules! foo {
    (@as_expr $e:expr) => { $e };
    ($($tts:tt)*) => {
        foo!(@as_expr $($tts)*)
    };
}
```

**Why `@`?** Not used in prefix position in Rust syntax, so it cannot conflict with user input.

**Rule ordering:** `@`-prefixed rules must come before bare rules.

**Benefits:** Eliminates namespace pollution, avoids macro name collisions, works naturally with TT munchers.

## Push-Down Accumulation

Build up tokens incrementally using an accumulator parameter:

```rust
macro_rules! reverse {
    ([] -> [$($acc:tt)*]) => {
        $($acc)*
    };
    ([$first:tt $($rest:tt)*] -> [$($acc:tt)*]) => {
        reverse!([$($rest)*] -> [$first $($acc)*])
    };
}
// Usage: reverse!([a b c] -> []) expands to c b a
```

**Convention:** Use `($input) -> ($output)` syntax.

**Critical:** Accumulator must be `$($acc:tt)*` — only `tt` allows lossless intermediate storage.

**Performance:** Inherently **quadratic**. Combined with TT munching → **doubly quadratic**.

**Optimization:** Place accumulator at the **end** of rules for faster short-circuiting on failed matches.

## Callbacks

Pass another macro's name as a parameter, calling it with the result:

```rust
macro_rules! process {
    ($callback:ident -> ($($input:tt)*)) => {
        $callback!($($input)*)
    };
}
```

Useful for composing macros and building pipelines.
