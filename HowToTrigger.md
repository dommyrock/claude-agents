# How to Trigger Agents & Skills

This guide shows example prompts for the agents and skills included in this repo.

## Agents

Claude automatically delegates to the right agent based on its `description` field. You can also ask for a specific agent by name. Use `/agents` to manage agents (list, create, edit, delete).

### lsp-navigator

Navigates codebases using LSP for semantic code intelligence. Falls back to Grep/Glob when LSP cannot resolve symbols.

| # | Example Prompt |
|---|----------------|
| 1 | "Where is the `Handler` trait defined, and who implements it?" |
| 2 | "Use the lsp-navigator agent to trace the call chain from `main()` to the database query." |

### rust-reviewer

Reviews Rust code for quality, idiomaticity, safety, and best practices. Has the `rust-macros` skill attached, so macro knowledge is loaded automatically.

| # | Example Prompt |
|---|----------------|
| 1 | "Use the rust-reviewer agent to review `src/lib.rs` for safety issues." |
| 2 | "Have the rust-reviewer audit error handling in `src/server/routes.rs` — are there any `unwrap()` calls that could panic in production?" |

## Skills

Skills activate automatically when Claude detects a matching context. You can also invoke them directly as slash commands with `/skill-name`.

### rust-macros

Decision frameworks and patterns for Rust macro design — both `macro_rules!` and proc macros.

| # | Example |
|---|---------|
| 1 | `/rust-macros` — invoke directly as a slash command |
| 2 | "I need a `derive` macro that generates a Builder pattern — should this be `macro_rules!` or a proc macro?" — Claude loads the skill automatically |
