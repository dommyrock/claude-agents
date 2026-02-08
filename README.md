# Claude Code: Skills, Agents, and Plugins

A reference project demonstrating the correct architecture for extending Claude Code with custom skills, agents, and LSP integration.

## Directory Structure

```
.claude/
├── agents/
│   ├── lsp-navigator.md              # Agent: LSP-powered code navigation
│   └── rust-reviewer.md              # Agent: isolated code review
└── skills/
    └── rust-macros/
        ├── SKILL.md                  # Skill: macro design guidance
        ├── declarative-patterns.md   # Reference: TT munching, push-down, etc.
        ├── fragment-reference.md     # Reference: specifiers, follow-set rules
        ├── procedural-guide.md       # Reference: proc-macro pipeline, nine rules
        └── resources.md              # Reference: crates and learning links
```

## Skills vs Agents

Skills and agents serve fundamentally different purposes. Choosing wrong wastes tokens, context, or money.

### Skills = Knowledge Injection

Skills are **reusable instruction sets** that load into the main conversation when relevant. They teach Claude *how* to approach a task — conventions, patterns, decision frameworks.

```
User: "Help me write a proc-macro for this struct"

Main Claude:
  → Loads rust-macros skill (triggered by description match)
  → Skill content enters the main context window
  → Claude applies the guidance while working in your conversation
  → Full access to conversation history and all tools
```

**Use a skill when:**
- Providing domain knowledge or conventions
- Teaching strategies the main agent should follow inline
- The guidance needs access to conversation context
- You want Claude to apply it while working alongside you

**Key properties:**
- Runs **inline** in the main conversation (unless `context: fork` is set)
- Loaded **on-demand** when the description matches the task
- Only metadata (name + description) costs tokens at startup
- Full content loads only when triggered (~5k tokens)
- Can bundle reference files for progressive disclosure

### Agents (Subagents) = Isolated Workers

Agents are **specialized sub-processes** that run in their own context window with their own system prompt, tool access, and permissions. They work independently and return a summary.

```
User: "Review the authentication module"

Main Claude:
  → Delegates to rust-reviewer agent
  → Agent runs in separate context (doesn't pollute main conversation)
  → Agent reads 50+ files, builds detailed analysis
  → Returns summary to main conversation
  → Main context stays clean
```

**Use an agent when:**
- The task produces verbose output you don't need in main context
- You want to run multiple tasks in parallel
- You need to enforce tool restrictions (e.g., read-only)
- You want to use a cheaper/faster model for routine work
- The work is self-contained and can return a summary

**Key properties:**
- Runs in **isolated context** — separate from main conversation
- Has its own **system prompt** (the markdown body)
- Can restrict **tools** (e.g., read-only agents)
- Can use a different **model** (Haiku for cheap tasks, Sonnet for complex ones)
- Can **preload skills** via the `skills` frontmatter field
- Cannot spawn other subagents (no nesting)

### Decision Matrix

| Need | Skill | Agent |
|------|-------|-------|
| Domain knowledge / conventions | Yes | |
| Strategic guidance for main agent | Yes | |
| Runs inline with conversation context | Yes | |
| Parallel task execution | | Yes |
| Context isolation (keep main clean) | | Yes |
| Cost optimization (cheaper model) | | Yes |
| Tool restrictions (read-only, no bash) | | Yes |
| Self-contained one-shot task | | Yes |
| Continuous back-and-forth needed | Yes | |

### Combining Skills and Agents

Skills and agents work together. An agent can preload skills to get domain knowledge within its isolated context:

```yaml
# .claude/agents/rust-reviewer.md
---
name: rust-reviewer
skills:
  - rust-macros    # Preloaded: full skill content injected at startup
---
```

This means the rust-reviewer agent has macro expertise without the main conversation loading that content. The skill content is injected into the agent's context, not just made available for invocation.

## Configuration Reference

### Skill Frontmatter (SKILL.md)

All fields are optional. Only `description` is recommended.

| Field | Purpose |
|-------|---------|
| `name` | Display name (defaults to directory name). Lowercase, hyphens, max 64 chars |
| `description` | What the skill does and when to use it. **Write in third person.** Max 1024 chars |
| `disable-model-invocation` | `true` = only user can invoke via `/name`. Default: `false` |
| `user-invocable` | `false` = hidden from `/` menu, only Claude can load it. Default: `true` |
| `allowed-tools` | Tool allowlist when skill is active (e.g., `Read, Grep, Glob`) |
| `model` | Model override when skill is active |
| `context` | `fork` = run in a subagent instead of inline |
| `agent` | Which agent type to use when `context: fork` (e.g., `Explore`, `Plan`) |

### Agent Frontmatter (.md files)

`name` and `description` are required.

| Field | Required | Purpose |
|-------|----------|---------|
| `name` | Yes | Unique identifier (lowercase, hyphens) |
| `description` | Yes | When Claude should delegate. **Write in third person** |
| `tools` | No | Tool allowlist. Inherits all if omitted |
| `disallowedTools` | No | Tool denylist |
| `model` | No | `sonnet`, `opus`, `haiku`, or `inherit` (default) |
| `skills` | No | Skills to preload into agent context at startup |
| `permissionMode` | No | `default`, `acceptEdits`, `dontAsk`, `bypassPermissions`, `plan` |
| `maxTurns` | No | Max agentic turns before stopping |
| `mcpServers` | No | MCP servers available to this agent |
| `hooks` | No | Lifecycle hooks scoped to this agent |
| `memory` | No | Persistent memory: `user`, `project`, or `local` |
| `color` | No | UI background color for identification |

### Description Best Practices

From the official docs:

- **Always write in third person.** The description is injected into the system prompt.
  - Good: `"Reviews Rust code for safety and best practices"`
  - Bad: `"Use this when you need to review Rust code"`
- **Be specific and include trigger keywords** so Claude picks the right skill/agent
- **Include both what it does and when to use it**
- Max 1024 characters for skills

### Progressive Disclosure for Skills

Keep `SKILL.md` under 500 lines. Move detailed reference material to separate files:

```
my-skill/
├── SKILL.md           # Overview + decision frameworks (~100-200 lines)
├── reference.md       # Detailed reference (loaded when needed)
├── examples.md        # Usage examples (loaded when needed)
└── scripts/
    └── helper.py      # Utility script (executed, not loaded into context)
```

Reference supporting files from SKILL.md so Claude knows when to load them. Keep references **one level deep** — avoid chains of files referencing other files.

## LSP Integration

Claude Code supports the Language Server Protocol (LSP) via **plugins**, giving Claude real-time code intelligence: diagnostics, go-to-definition, find-references, and hover information.

### Setup for Rust

1. **Install rust-analyzer:**
   ```bash
   rustup component add rust-analyzer
   ```

2. **Install the LSP plugin** (inside Claude Code):
   ```
   /plugin install rust-analyzer-lsp@claude-plugins-official
   ```

   Or from the community marketplace ([Piebald-AI](https://github.com/Piebald-AI)):
   ```
   /plugin install rust-lsp@claude-code-lsps
   ```

   Other community marketplaces may also provide LSP plugins — check the `/plugin` Discover tab.

3. **Verify** by asking Claude to use LSP operations on your Rust code.

### How It Works

LSP plugins configure an `.lsp.json` that tells Claude Code how to connect to a language server:

```json
{
  "rust": {
    "command": "rust-analyzer",
    "extensionToLanguage": {
      ".rs": "rust"
    }
  }
}
```

The language server binary must be installed separately — plugins only configure the connection.

### LSP Operations Available

| Operation | What It Does |
|-----------|-------------|
| `goToDefinition` | Jump to where a symbol is defined |
| `findReferences` | Find every usage of a symbol |
| `hover` | Get type signatures and documentation |
| `documentSymbol` | List all symbols in a file |
| `goToImplementation` | Find trait/interface implementations |
| `diagnostics` | Real-time errors and warnings after edits |

### LSP Access from Skills and Agents

**Main conversation (skills running inline):** Full LSP access. Skills that run in the main context can use LSP operations directly because they share the main agent's tool set.

**Subagents:** Subagents inherit tools from the parent conversation by default, including LSP tools provided by plugins. However, if an agent restricts its `tools` field, LSP tools may not be included unless explicitly allowed.

**Recommendation:** For code navigation skills that need LSP, run them inline (default) rather than in a forked context. If you need LSP in an agent, don't restrict the `tools` field or ensure LSP tools are included.

### Alternative: MCP-Based LSP

You can also access language servers through MCP (Model Context Protocol) instead of the native LSP plugin system. Community projects like [mcp-language-server](https://github.com/isaacphi/mcp-language-server) act as a bridge between Claude and language servers. Configure in `~/.claude/settings.json` under `mcpServers`.

### Available LSP Plugins

| Plugin | Language Server | Install Binary |
|--------|----------------|----------------|
| `rust-analyzer-lsp@claude-plugins-official` | rust-analyzer | `rustup component add rust-analyzer` |
| `typescript-lsp@claude-plugins-official` | TypeScript Language Server | `npm install -g typescript-language-server typescript` |
| `pyright-lsp@claude-plugins-official` | Pyright | `pip install pyright` or `npm install -g pyright` |

For languages not covered by existing plugins, create your own with an `.lsp.json` file. See the [Plugins reference](https://code.claude.com/docs/en/plugins-reference#lsp-servers).

## What This Project Demonstrates

### `rust-macros` (Skill)
Domain knowledge for Rust macro design. Uses progressive disclosure: concise SKILL.md (~118 lines) with four reference files for detailed coverage. Claude loads the overview when macros come up, and drills into specific reference files only when needed.

### `lsp-navigator` (Agent)
Lightweight code navigation worker that prefers LSP operations over text search. Runs on Haiku for cost efficiency. Demonstrates how to give an agent LSP tool access and encode LSP-specific strategies (blind spots, locate-then-query workflow) without duplicating knowledge Claude already has.

### `rust-reviewer` (Agent)
Isolated code review worker. Runs in its own context to keep review details out of the main conversation. Uses Sonnet model with Glob, Grep, Read, and Bash tools. Preloads the `rust-macros` skill for macro-aware reviews.

## Resources

- [Skills documentation](https://code.claude.com/docs/en/skills)
- [Subagents documentation](https://code.claude.com/docs/en/sub-agents)
- [Plugins documentation](https://code.claude.com/docs/en/plugins)
- [Plugins reference](https://code.claude.com/docs/en/plugins-reference)
- [Skills best practices](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices)
- [Skills overview](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview)
