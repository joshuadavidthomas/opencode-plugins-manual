# About This Manual

- **Date**: 2025-12-09
- **Model**: Claude Sonnet 4.5 (anthropic/claude-sonnet-4.5-20250929)
- **Source commit**: 3efc95b15

## Generation

This manual was created through a three-phase workflow documented in [`artifacts/`](../artifacts/).

### Research

- **Command**: [`/research-codebase`](../artifacts/commands/research-codebase.md)
- **Skill**: [`researching-codebases`](../artifacts/skills/researching-codebases/)
- **Query**: "What does OpenCode support for custom plugins compared to Claude Code? What parity exists and where are the gaps?"
- **Agents**: [`code-locator`](../artifacts/agents/code-locator.md), [`code-analyzer`](../artifacts/agents/code-analyzer.md), [`code-pattern-finder`](../artifacts/agents/code-pattern-finder.md)
- **Output**: [research.md](../artifacts/research.md)

### Planning (2025-12-09)

- **Plan document**: [plan.md](../artifacts/plan.md)
- **Strategy**: 4-phase parallel subagent execution with detailed prompts and constraints

### Execution (2025-12-09)

- **Model**: Claude Sonnet 4.5 (anthropic/claude-sonnet-4.5-20250929)
- **Skill**: [`writing-clearly-and-concisely`](artifacts/skills/writing-clearly-and-concisely/)
- **Subagent type**: `general` (4 parallel instances)
- **Code exploration agents**: [`code-locator`](../artifacts/agents/code-locator.md), [`code-analyzer`](../artifacts/agents/code-analyzer.md), [`code-pattern-finder`](../artifacts/agents/code-pattern-finder.md)

**Phases executed:**

- Phase 1 (Foundations): 3 files - overview, quick start, plugin context
- Phase 2 (Core Capabilities): 3 files - hooks reference, config injection, custom tools
- Phase 3 (Advanced Topics): 3 files - events, SDK client, auth hooks
- Phase 4 (Reference & Migration): 4 files - migration guide + appendices

Each subagent explored the codebase using specialized agents and verified all code examples against source at commit 3efc95b15.

## Contents

- **[00-about.md](00-about.md)** - Generation process and artifacts (this file)
- **[01-overview.md](01-overview.md)** - What plugins are, capabilities, minimal examples
- **[02-quick-start.md](02-quick-start.md)** - Plugin locations, basic structure, step-by-step tutorial
- **[03-plugin-context.md](03-plugin-context.md)** - Deep dive on PluginInput (client, project, $, directory, worktree)
- **[04-hooks-reference.md](04-hooks-reference.md)** - All 10 hooks with signatures and examples
- **[05-config-hook.md](05-config-hook.md)** - Injecting commands, agents, MCP servers (undocumented feature)
- **[06-custom-tools.md](06-custom-tools.md)** - Tool helper, Zod patterns, execution context
- **[07-events.md](07-events.md)** - Event system deep dive with 33+ event types
- **[08-sdk-client.md](08-sdk-client.md)** - Client API overview with source pointers
- **[09-auth-hook.md](09-auth-hook.md)** - Auth providers (OAuth/API key)
- **[10-claude-code-migration.md](10-claude-code-migration.md)** - Feature mapping for Claude Code plugin authors

### Appendices

- **[appendix/config-schemas.md](appendix/config-schemas.md)** - Full command/agent/MCP schemas
- **[appendix/event-schemas.md](appendix/event-schemas.md)** - Complete event payload schemas
- **[appendix/sdk-reference.md](appendix/sdk-reference.md)** - All SDK client methods

### Artifacts

- **[artifacts/research.md](../artifacts/research.md)** - OpenCode vs Claude Code plugin parity analysis
- **[artifacts/plan.md](../artifacts/plan.md)** - Execution plan with subagent prompts
- **[artifacts/commands/](../artifacts/commands/)** - `/research-codebase` command definition
- **[artifacts/skills/](../artifacts/skills/)** - `researching-codebases` and `writing-clearly-and-concisely` skills
- **[artifacts/agents/](../artifacts/agents/)** - `code-locator`, `code-analyzer`, `code-pattern-finder` agent definitions
