# Claude Skills vs Agents

## When to Use Agents (Subagents)

Agents are for **parallel execution and context isolation**, not strategic guidance. Here are the real use cases:

### 1. **Parallel Task Execution** ⚡

Running multiple independent tasks **simultaneously**:

```
User: "Review all 5 modules in parallel - check security, performance, 
      and test coverage for each"

Main Claude:
  → Spawns 5 review agents in parallel
  → Each runs independently with isolated context
  → All complete simultaneously
  → Main agent aggregates results
```

**Why agent not skill:**
- Need 5 separate contexts running at once
- Each agent processes different module
- Results come back in parallel (much faster)

### 2. **Cost Optimization** 💰

Using cheaper models for simple, repetitive tasks:

```yaml
---
name: code-formatter
tools: Read, Write, Edit
model: haiku  # ← Cheap! ~$0.25 per million tokens vs $3 for Sonnet
---

Format all code files according to style guide.
```

**Why agent not skill:**
- Haiku is 10x cheaper than Sonnet
- Formatting doesn't need Sonnet's intelligence
- Can spawn many Haiku agents for batch operations
- Main agent (Sonnet) orchestrates

**Example:**
```
User: "Format all 50 Python files in this repo"

Main Claude (Sonnet):
  → Identifies all files
  → Spawns 10 Haiku formatter agents in parallel
  → Each formats 5 files
  → Cost: ~$0.10 instead of $1.00
```

### 3. **Context Isolation** 🔒

Keeping exploratory work separate from main conversation:

```
User: "Explore this legacy codebase and figure out how auth works"

Main Claude:
  → Spawns Explore agent
  → Agent digs through 100+ files
  → Agent's context fills up with exploration details
  → Returns summary to main agent
  → Main agent's context stays clean!
```

**Why agent not skill:**
- Don't want main context polluted with exploration details
- Exploration might require reading 50+ files
- Main agent only needs the summary
- Can throw away agent's context after

### 4. **Enforcing Security Constraints** 🛡️

Limiting tool access for dangerous operations:

```yaml
---
name: sql-query-runner
tools: Read  # ← ONLY Read, no Write/Edit/Bash
model: sonnet
---

Execute SQL queries from user, but:
- Read-only access
- No file modifications
- No shell commands
- No web access
```

**Why agent not skill:**
- Hard constraint on available tools
- Can't accidentally modify files
- Can't run dangerous commands
- Security boundary enforced by system

### 5. **Built-in Agents** 🏗️

Claude Code has built-in agents you use constantly:

```
Explore Agent:
  - Read-only tools (Read, Grep, Glob)
  - Fast codebase exploration
  - Doesn't clutter main context
  
Plan Agent:
  - Creates implementation plans
  - Separate planning context
  - Main agent executes the plan
```

## Real-World Example: When Each Makes Sense

### Scenario: Code Review System

```yaml
# ✅ USE SKILL: Strategic guidance
---
name: rust-reviewer
description: Expert Rust code review strategies...
---

Teaches main agent:
- What to look for in Rust code
- Common anti-patterns
- Best practices
- Review structure

# ✅ USE AGENT: Parallel execution
---
name: module-reviewer
tools: Read, Grep, Glob
model: haiku  # Cheap!
---

Reviews a single module in isolation.
Spawned in parallel for multiple modules.
```

**Usage:**
```
User: "Review all 10 modules in the workspace"

Main Claude (using rust-reviewer skill):
  1. Loads rust-reviewer skill for strategy
  2. Identifies all 10 modules
  3. Spawns 10 module-reviewer agents (Haiku, parallel)
  4. Each agent reviews one module independently
  5. Collects all reviews
  6. Synthesizes findings using rust-reviewer guidance
  7. Presents consolidated report
```

**Cost & Speed:**
- Without agents: 10 minutes, $2.00 (sequential Sonnet)
- With agents: 2 minutes, $0.50 (parallel Haiku)

## Decision Matrix

| Use Case | Use Skill | Use Agent |
|----------|-----------|-----------|
| Teach strategies | ✅ | ❌ |
| Provide domain knowledge | ✅ | ❌ |
| Reusable patterns | ✅ | ❌ |
| Run tasks in parallel | ❌ | ✅ |
| Keep context clean | ❌ | ✅ |
| Use cheaper model | ❌ | ✅ |
| Enforce tool restrictions | ❌ | ✅ |
| Need continuous conversation | ✅ | ❌ |
| One-shot independent task | ❌ | ✅ |

## Your Rust Agents: Should They Be Agents or Skills?

Let's evaluate what you had:

### ✅ rust-reviewer.md → **SKILL** (Correct!)
**Why:**
- Teaches review strategies
- Main agent needs this guidance
- No need for isolation
- Not for parallel execution
- Should be available continuously

### ✅ rust-macros.md → **SKILL** (Correct!)
**Why:**
- Domain expertise about macros
- Reference material
- Main agent needs this when working with macros
- Not a task to run in isolation

### ❓ lsp-navigator.md → **SKILL** (I converted it)
**Why:**
- Strategic navigation guidance
- Main agent needs LSP access
- Not for parallel execution
- Continuous code exploration

**HOWEVER**, you COULD have both:

```yaml
# SKILL: Strategic guidance
~/.claude/skills/lsp-navigation/SKILL.md
  → Teaches when/how to use LSP
  → Available to main agent always

# AGENT: Parallel deep-dive research (optional)
~/.claude/agents/code-researcher.md
---
name: code-researcher
tools: Read, Grep, Glob  # No Write
model: haiku  # Cheap for research
---
Deep research into codebase structure.
Used when spawning multiple parallel research agents.
```

**Usage:**
```
User: "Research how authentication, logging, and database 
      access work in this codebase"

Main Claude (using lsp-navigation skill):
  1. Spawns 3 code-researcher agents (Haiku, parallel)
     - Agent 1: Authentication research
     - Agent 2: Logging research  
     - Agent 3: Database research
  2. Each digs deep independently
  3. Main agent synthesizes findings
  4. Presents consolidated architecture overview
```

## The Golden Rule

**Skills = Teaching**
- "Here's HOW to do X"
- "Here's WHEN to do Y"
- Available continuously to main agent

**Agents = Doing**
- "Go DO this specific task"
- "Run in parallel"
- "Keep context separate"
- Often cheaper/faster

## Summary

Agents aren't "worse" - they're **different tools for different jobs**:

- **Limitation (subset of tools)** = **Feature for safety/cost**
- **Context isolation** = **Feature for keeping main context clean**
- **Separate execution** = **Feature for parallelization**

You don't use agents much in simple workflows, but they're critical for:
1. Large-scale operations (review 100 files)
2. Parallel execution (speed)
3. Cost optimization (Haiku vs Sonnet)
4. Security boundaries (read-only access)

For your LSP navigation case: **Skill is correct** because you want strategic guidance available to the main agent, not parallel execution or isolation.