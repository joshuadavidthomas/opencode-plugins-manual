# OpenCode Plugins: The Missing Manual - Design Document

**Date:** 2025-12-09
**Status:** Ready for execution
**Commit Reference:** 3efc95b15

## Overview

Create comprehensive documentation for OpenCode plugin authors that fills gaps in official docs. Serves both new plugin authors and those migrating from Claude Code.

### Background

OpenCode's plugin documentation covers basics but lacks:

- Config hook injection (commands, agents, MCP servers)
- Full hooks interface reference
- Event system details
- SDK client documentation
- Auth hook implementation
- Claude Code migration guide

We verified that config hook injection works for bundling commands and agents with plugins - this is undocumented but functional.

## Deliverables

**Location:** `.research/opencode-plugins-manual/`

| File                          | Purpose                                                         | Depth     |
| ----------------------------- | --------------------------------------------------------------- | --------- |
| `00-overview.md`              | What plugins are, capabilities summary, quick example           | Overview  |
| `01-quick-start.md`           | Locations, basic structure, minimal working plugin              | Tutorial  |
| `02-plugin-context.md`        | PluginInput deep dive (client, project, $, directory, worktree) | Deep dive |
| `03-hooks-reference.md`       | All 10 hooks with signatures, when to use each                  | Reference |
| `04-config-hook.md`           | Injecting commands, agents, MCP - the undocumented power        | Deep dive |
| `05-custom-tools.md`          | Tool helper, Zod patterns, execution context                    | Deep dive |
| `06-events.md`                | Event system, common events, patterns                           | Deep dive |
| `07-sdk-client.md`            | Client API overview with pointers to source                     | Overview  |
| `08-auth-hook.md`             | Auth providers overview with pointers to source                 | Overview  |
| `09-claude-code-migration.md` | Feature mapping, migration patterns                             | Reference |
| `appendix/event-schemas.md`   | Full event payload schemas                                      | Appendix  |
| `appendix/sdk-reference.md`   | Full SDK client methods                                         | Appendix  |
| `appendix/config-schemas.md`  | Full command/agent/mcp schemas                                  | Appendix  |

**Total: 13 files**

## Style Guidelines

- **Evergreen core + dated details**: Concepts first, then implementation with commit refs
- **Technical but approachable**: No fluff, clear examples
- **File:line references**: Point to source code for verification
- **Summary tables**: Quick scanning for busy developers
- **Working code examples**: Copy-paste ready

## Execution Plan

### Pre-Execution Setup

Create directories:

```
.scratch/phase-01/
.scratch/phase-02/
.scratch/phase-03/
.scratch/phase-04/
.research/opencode-plugins-manual/
.research/opencode-plugins-manual/appendix/
```

### Phase Structure

```
┌─────────────────────────────────────────────────────────┐
│  Setup: Create directories                               │
└─────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│  Parallel Execution:                                     │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐        │
│  │  Phase 1    │ │  Phase 2    │ │  Phase 3    │        │
│  │ Foundations │ │    Core     │ │  Advanced   │        │
│  │  3 files    │ │   3 files   │   3 files   │        │
│  └─────────────┘ └─────────────┘ └─────────────┘        │
└─────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│  Phase 4: Reference & Migration (depends on 1-3)         │
│  4 files (migration doc + 3 appendices)                  │
└─────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│  Cleanup: Remove .scratch/ directories                   │
└─────────────────────────────────────────────────────────┘
```

### Subagent Constraints

All subagents must follow:

- Only create/edit files within the current working directory
- Use `.scratch/phase-XX/` for any temporary work
- Do NOT use /tmp or any directory outside CWD
- Only use read-only tools for code analysis (read, glob, grep)
- Write output files to `.research/opencode-plugins-manual/`
- Do not run bash commands that modify system state
- Do not require any permission approvals

---

## Subagent Prompts

### Shared Preamble

All subagents receive this context:

```
PROJECT: OpenCode Plugins: The Missing Manual
OUTPUT LOCATION: .research/opencode-plugins-manual/
COMMIT REFERENCE: 3efc95b15 (use for file:line citations)

CONSTRAINTS:
- Only create/edit files within the current working directory
- Use .scratch/phase-XX/ for any temporary work
- Do NOT use /tmp or any directory outside CWD
- Only use read-only tools for code analysis (read, glob, grep)
- Write output files to .research/opencode-plugins-manual/
- Do not run bash commands that modify system state
- Do not require any permission approvals

CODE EXPLORATION:
When you need to find or analyze code, delegate to specialized agents:
- Use `code-locator` to find files, directories, and components
- Use `code-analyzer` to analyze implementation details
- Use `code-pattern-finder` to find similar patterns or examples

When delegating, instruct sub-agents:
- Read-only operations only (read, glob, grep)
- No bash commands that modify state
- No operations requiring permission confirmations
- Return findings with file:line references

WRITING STYLE:
Use the `writing-clearly-and-concisely` skill. Apply Strunk's principles:
active voice, positive form, concrete language, omit needless words.
Avoid AI writing patterns (puffery, empty phrases, promotional adjectives).

DOCUMENT STYLE:
- Evergreen concepts first, then dated implementation details
- Mark implementation details with: "> As of commit 3efc95b15"
- Include working code examples (copy-paste ready)
- Use summary tables for quick scanning
- File:line references to source code

REFERENCE DOC (for context, verify against source):
.research/2025-12-09_opencode-vs-claude-code-plugin-parity.md
```

### Phase 1 Prompt: Foundations

```
[SHARED PREAMBLE]

SCRATCH DIRECTORY: .scratch/phase-01/

YOUR FILES:
1. 00-overview.md - What plugins are, capabilities summary, quick example
2. 01-quick-start.md - Plugin locations, basic structure, minimal working plugin
3. 02-plugin-context.md - Deep dive on PluginInput (client, project, $, directory, worktree)

SOURCE FILES TO CONSULT:
- packages/plugin/src/index.ts (PluginInput type, lines 25-31)
- packages/plugin/src/shell.ts (BunShell types)
- packages/opencode/src/plugin/index.ts (plugin loading, lines 14-53)
- packages/web/src/content/docs/plugins.mdx (existing docs/examples)
- packages/sdk/js/src/gen/sdk.gen.ts (client methods overview)

INSTRUCTIONS:

00-overview.md:
- What a plugin IS (TypeScript module exporting Plugin functions)
- All capabilities (tools, hooks, events, auth, config injection)
- One minimal example
- Capabilities comparison table

01-quick-start.md:
- Where to put plugins (.opencode/plugin/, ~/.config/opencode/plugin/)
- Basic plugin structure with types
- Step-by-step: create your first plugin
- How plugins are discovered and loaded
- npm package vs file:// local plugins

02-plugin-context.md (deep dive):
- All 5 PluginInput fields with practical examples
- client: SDK client overview
- project: Project type fields
- directory: Current working directory
- worktree: Git root / project root
- $: Bun shell - .text(), .json(), .lines(), .quiet(), .nothrow(), .cwd(), .env()

Create all 3 files.
```

### Phase 2 Prompt: Core Capabilities

```
[SHARED PREAMBLE]

SCRATCH DIRECTORY: .scratch/phase-02/

YOUR FILES:
1. 03-hooks-reference.md - All 10 hooks with signatures, when to use each
2. 04-config-hook.md - Deep dive on injecting commands, agents, MCP servers
3. 05-custom-tools.md - Tool helper, Zod patterns, execution context

SOURCE FILES TO CONSULT:
- packages/plugin/src/index.ts:144-182 (Hooks interface)
- packages/plugin/src/tool.ts (tool helper)
- packages/opencode/src/config/config.ts:375-419 (Command/Agent schemas)
- packages/opencode/src/config/config.ts:305-370 (MCP schemas)
- packages/opencode/src/session/prompt.ts (hook trigger points - search for Plugin.trigger)
- packages/opencode/src/tool/registry.ts:45-50 (tool registration)

VERIFIED WORKING EXAMPLE:
.opencode/plugin/test-injection.ts (config hook injection works!)

INSTRUCTIONS:

03-hooks-reference.md:
- All 10 hooks with type signatures
- When to use each hook
- Quick example for each
- Summary table

04-config-hook.md (KEY INSIGHT - this is undocumented but verified):
- Injecting commands into config.command
- Injecting agents into config.agent
- Injecting MCP servers into config.mcp
- Full schemas for each
- Working examples
- Reference the verified test we created

05-custom-tools.md:
- The tool() helper function
- Zod schema patterns (tool.schema.string(), etc.)
- Execution context
- Return values
- Error handling
- Complete working examples

Create all 3 files.
```

### Phase 3 Prompt: Advanced Topics

```
[SHARED PREAMBLE]

SCRATCH DIRECTORY: .scratch/phase-03/

YOUR FILES:
1. 06-events.md - Event system deep dive, common events, patterns
2. 07-sdk-client.md - Client API overview with pointers to source
3. 08-auth-hook.md - Auth providers overview with pointers to source

SOURCE FILES TO CONSULT:
- packages/opencode/src/bus/index.ts (event system core)
- packages/opencode/src/session/index.ts:87-120 (session events)
- packages/opencode/src/session/message-v2.ts:377-406 (message events)
- packages/opencode/src/session/status.ts:26-41 (status events)
- packages/opencode/src/file/index.ts:113-120 (file events)
- packages/opencode/src/permission/index.ts:40-50 (permission events)
- packages/opencode/src/cli/cmd/tui/event.ts:4-39 (TUI events)
- packages/sdk/js/src/gen/sdk.gen.ts (client methods)
- packages/plugin/src/index.ts:35-142 (AuthHook type)

INSTRUCTIONS:

06-events.md (deep dive):
- How the event system works (Bus.event, Bus.publish, Bus.subscribe)
- Common events with payload schemas: session.idle, file.edited, tool.execute.*, permission.*
- Event subscription patterns
- Summary table of all events with pointers to definitions

07-sdk-client.md (overview + pointers):
- What the client provides
- Overview of sub-clients (session, file, config, find, etc.)
- Common operations with examples
- Pointers to full reference in appendix

08-auth-hook.md (overview + pointers):
- What auth hooks are for
- OAuth vs API key authentication
- The loader function
- Structure overview with pointers to source
- Note: full deep dive can be added later

Create all 3 files.
```

### Phase 4 Prompt: Reference & Migration

```
[SHARED PREAMBLE]

SCRATCH DIRECTORY: .scratch/phase-04/

YOUR FILES:
1. 09-claude-code-migration.md - Feature mapping for Claude Code plugin authors
2. appendix/event-schemas.md - Full event payload schemas
3. appendix/sdk-reference.md - Full SDK client methods
4. appendix/config-schemas.md - Full command/agent/mcp schemas

SOURCE FILES TO CONSULT:
- packages/opencode/src/config/config.ts (full schemas)
- packages/sdk/js/src/gen/types.gen.ts (SDK types)
- packages/sdk/js/src/gen/sdk.gen.ts (SDK methods)
- All event definition files from Phase 3 sources

PRIMARY REFERENCE:
.research/2025-12-09_opencode-vs-claude-code-plugin-parity.md

CLAUDE CODE FEATURES TO MAP:
- Commands (commands/ directory) → config.command injection
- Agents (agents/ directory) → config.agent injection
- Skills (skills/ with SKILL.md) → custom tools or agents
- Hooks (hooks.json with matchers) → TypeScript hook functions
- MCP servers (.mcp.json) → config.mcp injection
- Manifest (plugin.json) → not needed
- ${CLAUDE_PLUGIN_ROOT} → ctx.directory or import.meta.url

INSTRUCTIONS:

09-claude-code-migration.md:
- Side-by-side feature comparison table
- Migration patterns for each component
- What's different, what's better in OpenCode
- Complete plugin example showing all translations

appendix/event-schemas.md:
- Full payload schemas for all event types
- Organized by category (session, message, file, permission, etc.)
- Minimal prose, maximum information density

appendix/sdk-reference.md:
- All client sub-clients and their methods
- Method signatures with parameter types
- Organized by sub-client

appendix/config-schemas.md:
- Full Command schema
- Full Agent schema
- Full MCP schemas (local, remote)
- All optional fields documented

Create all 4 files.
```

---

## Phase 1: Foundations

**Files:** `00-overview.md`, `01-quick-start.md`, `02-plugin-context.md`

**Purpose:** Get a new plugin author from zero to "I understand the basics and have a working plugin"

**Source files to consult:**

- `packages/plugin/src/index.ts` (PluginInput type, lines 25-31)
- `packages/plugin/src/shell.ts` (BunShell types)
- `packages/opencode/src/plugin/index.ts` (plugin loading)
- `packages/web/src/content/docs/plugins.mdx` (existing docs/examples)
- `packages/sdk/js/src/gen/sdk.gen.ts` (client methods)

**Reference doc:** `.research/2025-12-09_opencode-vs-claude-code-plugin-parity.md`

**Dependencies:** None (can start immediately)

---

## Phase 2: Core Capabilities

**Files:** `03-hooks-reference.md`, `04-config-hook.md`, `05-custom-tools.md`

**Purpose:** The meat of plugin development - what hooks exist, how to use them

**Source files to consult:**

- `packages/plugin/src/index.ts:144-182` (Hooks interface)
- `packages/plugin/src/tool.ts` (tool helper)
- `packages/opencode/src/config/config.ts:375-419` (Command/Agent schemas)
- `packages/opencode/src/session/prompt.ts` (hook trigger points)
- `packages/opencode/src/tool/registry.ts` (tool registration)

**Verified working example:** `.opencode/plugin/test-injection.ts`

**Reference doc:** `.research/2025-12-09_opencode-vs-claude-code-plugin-parity.md`

**Key insight:** Config hook injection for commands/agents/MCP is undocumented but verified working.

**Dependencies:** None (can run parallel with Phase 1)

---

## Phase 3: Advanced Topics

**Files:** `06-events.md`, `07-sdk-client.md`, `08-auth-hook.md`

**Purpose:** Deeper capabilities for more complex plugins

**Source files to consult:**

- `packages/opencode/src/bus/index.ts` (event system core)
- `packages/opencode/src/session/index.ts` (session events)
- `packages/opencode/src/session/message-v2.ts` (message events)
- `packages/opencode/src/file/index.ts` (file events)
- `packages/opencode/src/permission/index.ts` (permission events)
- `packages/opencode/src/cli/cmd/tui/event.ts` (TUI events)
- `packages/sdk/js/src/gen/sdk.gen.ts` (client methods)
- `packages/plugin/src/index.ts:35-142` (AuthHook type)

**Reference doc:** `.research/2025-12-09_opencode-vs-claude-code-plugin-parity.md`

**Depth:** Events get deep dive; SDK and auth get overview + pointers (room to expand later)

**Dependencies:** None (can run parallel with Phase 1 & 2)

---

## Phase 4: Reference & Migration

**Files:** `09-claude-code-migration.md`, `appendix/event-schemas.md`, `appendix/sdk-reference.md`, `appendix/config-schemas.md`

**Purpose:** Claude Code migration guide and full reference schemas

**Source files to consult:**

- `packages/opencode/src/config/config.ts` (full schemas)
- `packages/sdk/js/src/gen/types.gen.ts` (SDK types)
- All event definition files from Phase 3

**Primary reference:** `.research/2025-12-09_opencode-vs-claude-code-plugin-parity.md`

**Claude Code features to map:**

- Commands (`commands/` directory)
- Agents (`agents/` directory)
- Skills (`skills/` with SKILL.md)
- Hooks (`hooks.json` with matchers)
- MCP servers (`.mcp.json`)
- Manifest (`plugin.json`)
- `${CLAUDE_PLUGIN_ROOT}` variable

**Dependencies:** Phases 1-3 should complete first (migration doc references other sections)

---

## Success Criteria

- [ ] All 13 files created in `.research/opencode-plugins-manual/`
- [ ] Each file has working code examples
- [ ] File:line references point to real locations (commit `3efc95b15`)
- [ ] Config hook injection is documented as verified working
- [ ] A new plugin author could go from zero to working plugin
- [ ] A Claude Code author could find OpenCode equivalents

## Design Decisions

### Audience

New plugin authors first, with Claude Code migration as later reference. Both audiences served but emphasis on teaching the OpenCode way.

### Depth Strategy

"Deep dive core, reference appendix for rest" - main docs cover the 80% case deeply, appendices have full schemas for the 20%.

### File Organization

`.research/opencode-plugins-manual/` - keeps it with research for now, can be moved to final destination later.

### Consistency Approach

Grouped subagent phases (not one-per-file) to maintain consistent voice within related topics.

## References

- Research doc: `.research/2025-12-09_opencode-vs-claude-code-plugin-parity.md`
- Verified test: `.opencode/plugin/test-injection.ts`
- Existing docs: `packages/web/src/content/docs/plugins.mdx`
- Plugin types: `packages/plugin/src/index.ts`
