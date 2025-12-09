---
date: 2025-12-09T11:15:30-06:00
query: "What does OpenCode support for custom plugins compared to Claude Code? What parity exists and where are the gaps?"
repository: https://github.com/sst/opencode
branch: dev
commit: 3efc95b15
cwd: /home/josh/projects/sst/opencode
tags: [plugins, claude-code, parity, hooks, agents, commands, mcp]
---

# Research: OpenCode vs Claude Code Plugin Parity

## Summary

OpenCode has a **fundamentally different plugin architecture** than Claude Code. While Claude Code plugins are **declarative bundles** containing various components (commands, agents, skills, hooks, MCP), OpenCode plugins are **programmatic hooks** - TypeScript/JavaScript modules that hook into specific lifecycle events.

**Key Finding**: Although commands and agents are typically configured via separate files, **plugins CAN bundle commands and agents** by injecting them via the `config` hook. This was verified experimentally on 2025-12-09.

## Detailed Findings

### Comparison Table

| Component          | Claude Code                                       | OpenCode                                    | Parity        |
| ------------------ | ------------------------------------------------- | ------------------------------------------- | ------------- |
| **Commands**       | `commands/` dir in plugin with markdown files     | `config` hook injection (verified)          | Achievable    |
| **Agents**         | `agents/` dir in plugin with markdown files       | `config` hook injection (verified)          | Achievable    |
| **Skills**         | `skills/` dir with `SKILL.md` format              | **No equivalent** - use `tool` hook instead | Gap           |
| **Hooks**          | `hooks.json` with event matchers + shell commands | TypeScript hook functions                   | Different     |
| **MCP Servers**    | `.mcp.json` bundled in plugin                     | `config` hook injection                     | Achievable    |
| **Custom Tools**   | N/A (via MCP)                                     | `tool` hook with Zod schemas                | OpenCode-only |
| **Auth Providers** | N/A                                               | `auth` hook                                 | OpenCode-only |
| **Manifest**       | `plugin.json` with metadata                       | No manifest - module exports                | Different     |

### Commands - Supported via Config Hook Injection

**Claude Code**: Plugin bundles commands in `commands/` directory as markdown files.

**OpenCode**: Commands can be injected via the `config` hook. **Verified working.**

```typescript
export const MyPlugin: Plugin = async (ctx) => ({
  config: async (config) => {
    config.command = config.command || {}
    config.command["my-command"] = {
      template: "Your prompt template with $ARGUMENTS placeholder",
      description: "What this command does",
    }
  },
})
```

**Command Schema** (`packages/opencode/src/config/config.ts:375-382`):

- `template`: string (required) - the prompt template
- `description`: string (optional)
- `agent`: string (optional) - which agent to use
- `model`: string (optional) - override model
- `subtask`: boolean (optional) - force subagent invocation

**Alternative**: File-based commands in `.opencode/command/` (project) or `~/.config/opencode/command/` (global).

**Migration Path**: Convert markdown command files to programmatic injection via `config` hook.

### Agents - Supported via Config Hook Injection

**Claude Code**: Plugin bundles agents in `agents/` directory with markdown files defining capabilities.

**OpenCode**: Agents can be injected via the `config` hook. **Verified working.**

```typescript
export const MyPlugin: Plugin = async (ctx) => ({
  config: async (config) => {
    config.agent = config.agent || {}
    config.agent["my-agent"] = {
      prompt: "You are a specialized agent that...",
      description: "When to use this agent",
      mode: "subagent", // or "primary" or "all"
    }
  },
})
```

**Agent Schema** (`packages/opencode/src/config/config.ts:384-419`):

- `prompt`: string (optional) - system prompt for the agent
- `description`: string (optional) - when to use the agent
- `mode`: "subagent" | "primary" | "all" (optional)
- `model`: string (optional) - e.g., "anthropic/claude-sonnet-4-20250514"
- `temperature`: number (optional)
- `top_p`: number (optional)
- `tools`: Record<string, boolean> (optional) - enable/disable specific tools
- `maxSteps`: number (optional)
- `color`: hex string (optional) - e.g., "#FF5733"
- `permission`: object (optional) - edit, bash, webfetch, doom_loop, external_directory

**Alternative**: File-based agents in `.opencode/agent/` (project) or `~/.config/opencode/agent/` (global).

**Migration Path**: Convert markdown agent files to programmatic injection via `config` hook.

### Skills - No Equivalent

**Claude Code**: Skills use `SKILL.md` format with frontmatter and supporting files. Model-invoked based on task context.

**OpenCode**: No skill system. The closest equivalents are:

- **Custom Tools** via the `tool` hook
- **Custom Agents** in `.opencode/agent/`

**Migration Path**: Convert skills to either custom tools via `tool` hook (for tool-like behavior) or custom agents (for task delegation).

### Hooks - Different Architecture

**Claude Code**: JSON config with event matchers and shell commands

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "hooks": [{ "type": "command", "command": "scripts/format.sh" }]
      }
    ]
  }
}
```

**OpenCode**: TypeScript hook functions (`packages/plugin/src/index.ts:144-182`)

```typescript
{
  "tool.execute.after": async (input, output) => {
    if (input.tool === "write" || input.tool === "edit") {
      await $`./scripts/format.sh`
    }
  }
}
```

**Event Mapping**:

| Claude Code Event   | OpenCode Equivalent                                    |
| ------------------- | ------------------------------------------------------ |
| `PreToolUse`        | `tool.execute.before`                                  |
| `PostToolUse`       | `tool.execute.after`                                   |
| `PermissionRequest` | `permission.ask`                                       |
| `UserPromptSubmit`  | `chat.message`                                         |
| `SessionStart/End`  | `event` hook + `session.created`/`session.idle` events |
| `Stop/SubagentStop` | No direct equivalent                                   |
| `PreCompact`        | `event` hook + `session.compacted`                     |
| `Notification`      | `event` hook + `tui.toast.show`                        |

**Migration Path**: Convert JSON hook configs to TypeScript functions.

### MCP Servers - Partial Support

**Claude Code**: MCP servers bundled in plugin with `.mcp.json`

**OpenCode**: MCP servers are configured in `opencode.json`, **not bundled with plugins**. However, plugins can modify MCP config at runtime via the `config` hook:

```typescript
export const MyPlugin: Plugin = async (ctx) => ({
  config: async (config) => {
    config.mcp = config.mcp || {}
    config.mcp["my-server"] = {
      type: "remote",
      url: "http://localhost:3000",
      enabled: true,
    }
  },
})
```

MCP schema definition: `packages/opencode/src/config/config.ts:305-370`

**Migration Path**: Either add MCP config to `opencode.json` or use the `config` hook to inject it.

### Custom Tools - OpenCode Advantage

OpenCode plugins can define **custom tools** directly - something Claude Code requires MCP servers for:

```typescript
import { type Plugin, tool } from "@opencode-ai/plugin"

export const MyPlugin: Plugin = async (ctx) => ({
  tool: {
    mytool: tool({
      description: "My custom tool",
      args: { input: tool.schema.string() },
      async execute(args) {
        return `Result: ${args.input}`
      },
    }),
  },
})
```

Tool definition helper: `packages/plugin/src/tool.ts:10-19`
Tool registry integration: `packages/opencode/src/tool/registry.ts:45-50`

### OpenCode Hooks Interface

Full hooks interface (`packages/plugin/src/index.ts:144-182`):

| Hook                         | Purpose                                            |
| ---------------------------- | -------------------------------------------------- |
| `event`                      | Subscribe to all system events                     |
| `config`                     | Modify config at runtime                           |
| `tool`                       | Register custom tools                              |
| `auth`                       | Register auth providers                            |
| `chat.message`               | Process incoming messages                          |
| `chat.params`                | Modify LLM parameters (temperature, topP, options) |
| `permission.ask`             | Handle permission requests                         |
| `tool.execute.before`        | Pre-execution hook (can modify args)               |
| `tool.execute.after`         | Post-execution hook (can modify output)            |
| `experimental.text.complete` | Post-generation text modification                  |

### Available Events for `event` Hook

From `packages/web/src/content/docs/plugins.mdx:68-127`:

- Command: `command.executed`
- File: `file.edited`, `file.watcher.updated`
- LSP: `lsp.client.diagnostics`, `lsp.updated`
- Message: `message.part.removed`, `message.part.updated`, `message.removed`, `message.updated`
- Permission: `permission.replied`, `permission.updated`
- Session: `session.created`, `session.compacted`, `session.deleted`, `session.diff`, `session.error`, `session.idle`, `session.status`, `session.updated`
- Tool: `tool.execute.after`, `tool.execute.before`
- TUI: `tui.prompt.append`, `tui.command.execute`, `tui.toast.show`

## Code References

Key locations for future reference:

- `packages/plugin/src/index.ts:144-182` - Hooks interface definition
- `packages/plugin/src/tool.ts:10-19` - Tool definition helper
- `packages/opencode/src/plugin/index.ts:14-53` - Plugin loading and discovery
- `packages/opencode/src/plugin/index.ts:55-74` - Hook triggering mechanism
- `packages/opencode/src/config/config.ts:180-216` - Command loading from markdown
- `packages/opencode/src/config/config.ts:218-259` - Agent loading from markdown
- `packages/opencode/src/config/config.ts:290-303` - Plugin file discovery
- `packages/opencode/src/agent/agent.ts:10-41` - Agent schema
- `packages/opencode/src/tool/registry.ts:45-50` - Plugin tool collection
- `packages/web/src/content/docs/plugins.mdx` - User-facing plugin documentation

## Translation Guide

If translating a Claude Code plugin to OpenCode:

| Claude Code Component  | OpenCode Approach                                 |
| ---------------------- | ------------------------------------------------- |
| `commands/` directory  | Inject via `config` hook into `config.command`    |
| `agents/` directory    | Inject via `config` hook into `config.agent`      |
| `skills/` directory    | Convert to custom tools via `tool` hook or agents |
| `hooks/hooks.json`     | Convert to TypeScript hook functions              |
| `.mcp.json`            | Inject via `config` hook into `config.mcp`        |
| `plugin.json` manifest | Not needed - plugin is just exports               |

### Complete Plugin Example

A plugin that bundles commands, agents, tools, and MCP servers:

```typescript
import { type Plugin, tool } from "@opencode-ai/plugin"

export const MyPlugin: Plugin = async (ctx) => ({
  // Inject commands and agents via config hook
  config: async (config) => {
    // Bundle commands
    config.command = config.command || {}
    config.command["my-command"] = {
      template: "Do something with $ARGUMENTS",
      description: "My bundled command",
    }

    // Bundle agents
    config.agent = config.agent || {}
    config.agent["my-agent"] = {
      prompt: "You are a specialized agent...",
      description: "Use for specific tasks",
      mode: "subagent",
    }

    // Bundle MCP servers
    config.mcp = config.mcp || {}
    config.mcp["my-server"] = {
      type: "remote",
      url: "https://my-mcp-server.com/mcp",
    }
  },

  // Register custom tools
  tool: {
    mytool: tool({
      description: "My custom tool",
      args: { input: tool.schema.string() },
      async execute(args) {
        return `Result: ${args.input}`
      },
    }),
  },

  // Handle events
  event: async ({ event }) => {
    if (event.type === "session.idle") {
      console.log("Session completed!")
    }
  },

  // Intercept tool execution
  "tool.execute.before": async (input, output) => {
    console.log(`Tool ${input.tool} called with`, output.args)
  },
})
```

## Remaining Gaps

1. **No Skills system** - No `SKILL.md` format or model-invoked skill selection. Workaround: use custom tools or agents.

2. **No matcher patterns** - Hook filtering must be done in code, not via regex matchers. Workaround: implement filtering logic in TypeScript.

3. **No `${CLAUDE_PLUGIN_ROOT}`** - Plugins must use their own path resolution. Workaround: use `ctx.directory` or `import.meta.url`.

4. **No declarative manifests** - No `plugin.json` with metadata, versioning, keywords. Plugins are just TypeScript modules.

5. **Programmatic vs Declarative** - OpenCode requires code to bundle components; Claude Code uses file-based declarations. This is a design difference, not a capability gap.

## Verification

Tested on 2025-12-09 in the OpenCode repository:

1. Created `.opencode/plugin/test-injection.ts` with `config` hook injection
2. Restarted OpenCode
3. `/plugin-test` command appeared and executed successfully
4. `plugin-test-agent` agent was available via Task tool and responded correctly

**Conclusion**: Config hook injection is a viable approach for plugin authors to bundle commands, agents, and MCP servers with their plugins.

## Open Questions

- Would a `SKILL.md` equivalent be valuable for model-invoked capabilities?
- Should there be a plugin manifest format for metadata and discovery?
- Should OpenCode document the `config` hook injection pattern officially?
