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


**Example 2 from real project**

Prompt

```sh
Use the lsp-navigator agent to trace the call chain of collect_with_timeout fn
```

Claude tool use:

```sh
⏺ lsp-navigator(Trace collect_with_timeout call chain)
  ⎿  Done (7 tool uses · 17.0k tokens · 24s)
```

Final output:

```sh
⏺ Here's the full call chain for collect_with_timeout:

  Upstream (callers → entry points)

  main()                                          src/main.rs:11
  ├── search_interactive()                        src/search/service.rs:98
  │   └── run_interactive()                       src/cli/interactive.rs:12
  │       └── SearchService::search()             src/search/service.rs:44
  │           └── search_all_with_cache()         src/scraping/sites/mod.rs:91  (macro-generated)
  │               └── collect_with_timeout()      src/scraping/sites/mod.rs:124
  │
  ├── search_dedup()                              src/main.rs:35
  │   └── SearchService::search_dedup()           src/search/service.rs:75
  │       └── SearchService::search()             src/search/service.rs:44
  │           └── search_all_with_cache()         ...
  │               └── collect_with_timeout()
  │
  └── search_single()                             src/main.rs:44
      └── SearchService::search()                 src/search/service.rs:44
          ├── search_all_with_cache()             src/scraping/sites/mod.rs:91
          │   └── collect_with_timeout()
          └── search_all_direct()                 src/scraping/sites/mod.rs:76
              └── collect_with_timeout()

  Downstream (what it calls)

  collect_with_timeout()                          src/scraping/sites/mod.rs:124
  ├── join_all()                                  futures crate — runs all 14 site futures concurrently
  │   └── timeout(SITE_TIMEOUT, fut)              tokio::time — 15s per-site deadline
  │       └── search_<site>_direct/with_cache()   src/scraping/sites/<site>.rs (14 sites)
  ├── warn!()                                     tracing — logs failures and timeouts
  └── SearchError::validation()                   on timeout — now includes site name

  Two direct callers — both macro-generated aggregators (search_all_direct, search_all_with_cache). Three entry paths from main() —
  interactive, dedup, and single search modes. All paths funnel through SearchService::search() into one of the two aggregators, then into
  collect_with_timeout.
```

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
