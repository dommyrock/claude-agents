# Macro Resources and Reference Crates

> **Note:** These are recommendations for the user to explore outside of Claude sessions. Claude cannot access external URLs or install crates during a session — use these as starting points for your own learning and project setup.

## Essential Crates

| Crate | Purpose |
|-------|---------|
| [`syn`](https://docs.rs/syn/) | Parse `TokenStream` into Rust AST |
| [`quote`](https://docs.rs/quote/) | Generate `TokenStream` from quasi-quoted Rust |
| [`proc-macro2`](https://docs.rs/proc-macro2/) | Wrapper types enabling unit testing of proc-macros |
| [`cargo-expand`](https://github.com/dtolnay/cargo-expand) | Show macro expansion — the #1 debugging tool |
| [`trybuild`](https://github.com/dtolnay/trybuild) | Compile-fail and compile-pass test harness |

## Cargo.toml for Proc-Macro Crates

```toml
[dependencies]
syn = { version = "2", features = ["full"] }
quote = "1"
proc-macro2 = "1"
```

## Study These Crates

| Crate | What It Teaches |
|-------|-----------------|
| [`thiserror`](https://github.com/dtolnay/thiserror) | Clean attribute-based proc-macro architecture |
| [`serde`](https://github.com/serde-rs/serde) | Handling complex generics and trait bounds at scale |
| [`clap` (derive)](https://github.com/clap-rs/clap) | Attribute macro design for CLI parsing |

## Learning Resources

| Resource | Focus |
|----------|-------|
| [The Little Book of Rust Macros](https://lukaswirth.dev/tlborm/) | The "bible" for `macro_rules!` — advanced patterns |
| [David Tolnay's Proc-Macro Workshop](https://github.com/dtolnay/proc-macro-workshop) | Best hands-on proc-macro guide |
| [The Rust Reference: Macros](https://doc.rust-lang.org/reference/macros.html) | Authoritative specification |
| [Rust for Rustaceans](https://nostarch.com/rust-rustaceans) | When to use (and not use) macros |
| [Nine Rules for Proc-Macros](https://users.rust-lang.org/t/nine-rules-for-creating-procedural-macros-in-rust/83080) | Practical workflow rules |
| [Hygienic Macro Guide](https://gist.github.com/Kestrer/8c05ebd4e0e9347eb05f265dfb7252e1) | Complete hygiene rules |
