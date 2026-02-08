# How to Trigger Agents & Skills

This guide shows example prompts for the agents and skills included in this repo.

## Agents

Invoke agents with `/agent <name>` inside Claude Code, then provide your prompt.

### lsp-navigator

Navigates codebases using LSP for semantic code intelligence. Falls back to Grep/Glob when LSP cannot resolve symbols.

| # | Example Prompt |
|---|----------------|
| 1 | "Where is the `Handler` trait defined, and who implements it?" |
| 2 | "Trace the call chain from `main()` to the database query in this project." |

### rust-reviewer

Reviews Rust code for quality, idiomaticity, safety, and best practices. Has the `rust-macros` skill attached, so macro knowledge is loaded automatically.

| # | Example Prompt |
|---|----------------|
| 1 | "Review `src/lib.rs` for safety issues and idiomatic Rust." |
| 2 | "Audit the error handling in `src/server/routes.rs` — are there any `unwrap()` calls that could panic in production?" |

## Skills

Skills activate automatically when Claude detects a matching task. You can also reference them directly in your prompt.

### rust-macros

Decision frameworks and patterns for Rust macro design — both `macro_rules!` and proc macros.

| # | Example Prompt |
|---|----------------|
| 1 | "I need a `derive` macro that generates a Builder pattern — should this be `macro_rules!` or a proc macro?" |
| 2 | "Review my `macro_rules!` definition in `src/macros.rs` for hygiene issues and suggest improvements." |
