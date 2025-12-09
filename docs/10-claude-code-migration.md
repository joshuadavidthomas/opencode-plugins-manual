# Migrating from Claude Code Plugins

> As of commit [3efc95b](https://github.com/sst/opencode/tree/3efc95b157e05bc3c42554be1b5778f8f1b64cfe)

For Claude Code plugin authors moving to OpenCode.

## Architecture Differences

**Claude Code**: Declarative bundles with file-based components
**OpenCode**: Programmatic hooks with TypeScript functions

| Concept           | Claude Code            | OpenCode                  |
| ----------------- | ---------------------- | ------------------------- |
| Plugin definition | `plugin.json` manifest | TypeScript module exports |
| Discovery         | Directory scanning     | Module imports            |
| Extension         | File addition          | Code modification         |
| Lifecycle         | Load-time only         | Event-driven hooks        |

## Feature Mapping

### Commands

**Claude Code** bundles commands in `commands/` directory:

```markdown
## <!-- commands/format-code.md -->

## description: Format code using project conventions

Format the code in $ARGUMENTS following the project's style guide.
```

**OpenCode** injects commands via `config` hook:

```typescript
import { type Plugin } from "@opencode-ai/plugin"

export const MyPlugin: Plugin = async (ctx) => ({
  config: async (config) => {
    config.command = config.command || {}
    config.command["format-code"] = {
      template: "Format the code in $ARGUMENTS following the project's style guide.",
      description: "Format code using project conventions",
    }
  },
})
```

**Key differences**:

- OpenCode commands are TypeScript objects, not markdown files
- Commands registered at runtime, not load-time
- Can use `ctx.directory` for plugin-relative paths

### Agents

**Claude Code** bundles agents in `agents/` directory:

```markdown
## <!-- agents/code-reviewer.md -->

description: Reviews code for bugs and style issues
mode: subagent

---

You are a code reviewer. Analyze code for:

- Bugs and logic errors
- Style violations
- Performance issues
```

**OpenCode** injects agents via `config` hook:

```typescript
export const MyPlugin: Plugin = async (ctx) => ({
  config: async (config) => {
    config.agent = config.agent || {}
    config.agent["code-reviewer"] = {
      prompt: `You are a code reviewer. Analyze code for:
- Bugs and logic errors
- Style violations
- Performance issues`,
      description: "Reviews code for bugs and style issues",
      mode: "subagent",
    }
  },
})
```

**Key differences**:

- Frontmatter fields become object properties
- Markdown content becomes `prompt` field
- More configuration options (model, temperature, tools)

See `appendix/config-schemas.md` for full agent schema.

### Skills

**Claude Code** has `SKILL.md` format:

```markdown
## <!-- skills/my-skill/SKILL.md -->

name: my-skill
description: Does something useful

---

# My Skill

Instructions for the model...
```

**OpenCode** has no skill system. Convert skills to:

**Option 1: Custom tool**

```typescript
import { type Plugin, tool } from "@opencode-ai/plugin"

export const MyPlugin: Plugin = async (ctx) => ({
  tool: {
    "my-skill": tool({
      description: "Does something useful",
      args: { input: tool.schema.string() },
      async execute(args) {
        // Skill logic here
        return "Result"
      },
    }),
  },
})
```

**Option 2: Custom agent**

```typescript
export const MyPlugin: Plugin = async (ctx) => ({
  config: async (config) => {
    config.agent = config.agent || {}
    config.agent["my-skill"] = {
      prompt: "Instructions from SKILL.md...",
      description: "Does something useful",
      mode: "subagent",
    }
  },
})
```

**Trade-offs**:

- Tools: Explicit invocation, structured I/O
- Agents: Model decides when to use, natural language I/O

### Hooks

**Claude Code** uses JSON with matchers:

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "hooks": [
          {
            "type": "command",
            "command": ["prettier", "--write", "${FILE}"]
          }
        ]
      }
    ]
  }
}
```

**OpenCode** uses TypeScript functions:

```typescript
export const MyPlugin: Plugin = async (ctx) => ({
  "tool.execute.after": async (input, output) => {
    if (input.tool === "write" || input.tool === "edit") {
      const file = output.args.filePath
      await Bun.$`prettier --write ${file}`
    }
  },
})
```

**Event mapping**:

| Claude Code         | OpenCode                           |
| ------------------- | ---------------------------------- |
| `PreToolUse`        | `tool.execute.before`              |
| `PostToolUse`       | `tool.execute.after`               |
| `PermissionRequest` | `permission.ask`                   |
| `UserPromptSubmit`  | `chat.message`                     |
| `SessionStart`      | `event` hook + `session.created`   |
| `SessionEnd`        | `event` hook + `session.idle`      |
| `PreCompact`        | `event` hook + `session.compacted` |
| `Notification`      | `event` hook + `tui.toast.show`    |
| `Stop`              | No direct equivalent               |
| `SubagentStop`      | No direct equivalent               |

**Key differences**:

- No regex matchers—use TypeScript conditionals
- Full JavaScript/TypeScript instead of shell commands
- Access to full event payload and context

See `06-events.md` for all events and `appendix/event-schemas.md` for payloads.

### MCP Servers

**Claude Code** bundles MCP config:

```json
<!-- .mcp.json -->
{
  "my-server": {
    "type": "local",
    "command": ["node", "server.js"]
  }
}
```

**OpenCode** injects via `config` hook:

```typescript
export const MyPlugin: Plugin = async (ctx) => ({
  config: async (config) => {
    config.mcp = config.mcp || {}
    config.mcp["my-server"] = {
      type: "local",
      command: ["node", path.join(ctx.directory, "server.js")],
    }
  },
})
```

**Key differences**:

- Use `ctx.directory` for plugin-relative paths
- Runtime configuration instead of static files
- Can use async logic (check existence, load configs, etc.)

See `appendix/config-schemas.md` for MCP schemas.

### Environment Variables

**Claude Code**: `${CLAUDE_PLUGIN_ROOT}` for plugin directory

**OpenCode**: `ctx.directory` or `import.meta.url`

```typescript
// ctx.directory
const configPath = path.join(ctx.directory, "config.json")

// import.meta.url
const pluginDir = path.dirname(fileURLToPath(import.meta.url))
const configPath = path.join(pluginDir, "config.json")
```

## Complete Migration Example

**Claude Code plugin**:

```
my-plugin/
├── plugin.json
├── commands/
│   └── analyze.md
├── agents/
│   └── analyzer.md
├── skills/
│   └── analysis-skill/
│       └── SKILL.md
├── hooks.json
└── .mcp.json
```

**OpenCode plugin** (`index.ts`):

```typescript
import { type Plugin, tool } from "@opencode-ai/plugin"
import path from "path"

export const MyPlugin: Plugin = async (ctx) => ({
  // Bundle commands
  config: async (config) => {
    config.command = config.command || {}
    config.command["analyze"] = {
      template: "Analyze $ARGUMENTS for code quality issues",
      description: "Analyze code for quality issues",
      agent: "analyzer", // Use our custom agent
    }

    // Bundle agents
    config.agent = config.agent || {}
    config.agent["analyzer"] = {
      prompt: `You are a code analyzer...`,
      description: "Analyzes code for quality issues",
      mode: "subagent",
      color: "#FF5733",
    }

    // Bundle MCP servers
    config.mcp = config.mcp || {}
    config.mcp["analysis-server"] = {
      type: "local",
      command: ["node", path.join(ctx.directory, "mcp-server.js")],
    }
  },

  // Convert skill to custom tool
  tool: {
    "analyze-code": tool({
      description: "Performs deep code analysis",
      args: {
        code: tool.schema.string(),
        language: tool.schema.string().optional(),
      },
      async execute(args) {
        // Skill logic here
        return `Analysis of ${args.language || "unknown"} code complete`
      },
    }),
  },

  // Convert hooks
  "tool.execute.after": async (input, output) => {
    if (input.tool === "write" || input.tool === "edit") {
      const file = output.args.filePath
      // Run formatter
      await Bun.$`prettier --write ${file}`.catch(() => {})
    }
  },

  // Listen to events
  event: async ({ event }) => {
    if (event.type === "session.idle") {
      console.log("Session completed")
    }
  },
})
```

**What changed**:

- Single TypeScript file replaces directory structure
- Commands, agents, MCP injected via `config` hook
- Skill converted to custom tool
- Hooks converted to TypeScript functions
- Can use full Node.js/Bun APIs

## What OpenCode Adds

Beyond Claude Code parity, OpenCode provides:

### Custom Tools

Define tools directly without MCP:

```typescript
tool: {
  "my-tool": tool({
    description: "Does something",
    args: { input: tool.schema.string() },
    async execute(args) {
      return "Result"
    },
  }),
}
```

Source: `packages/plugin/src/tool.ts:10-19`

### Auth Providers

Register custom auth providers:

```typescript
auth: {
  "my-provider": {
    type: "oauth",
    async authorize(url) {
      // Handle OAuth flow
    },
  },
}
```

See `08-auth-hook.md` for details.

### LLM Parameter Modification

Modify temperature, top_p, options at runtime:

```typescript
"chat.params": async (params) => {
  params.temperature = 0.7
  params.topP = 0.9
  return params
}
```

### Text Post-Processing

Modify LLM output before display:

```typescript
"experimental.text.complete": async (text) => {
  // Transform generated text
  return text.replace(/old/g, "new")
}
```

### SDK Client Access

Use `ctx.client` for programmatic API access:

```typescript
const sessions = await ctx.client.session.list()
const config = await ctx.client.config.get()
```

See `07-sdk-client.md` and `appendix/sdk-reference.md`.

## Summary Table

| Feature         | Claude Code             | OpenCode             | Notes              |
| --------------- | ----------------------- | -------------------- | ------------------ |
| Commands        | `commands/*.md`         | `config` hook        | Runtime injection  |
| Agents          | `agents/*.md`           | `config` hook        | Runtime injection  |
| Skills          | `skills/*/SKILL.md`     | Custom tools/agents  | No skill system    |
| Hooks           | `hooks.json`            | TypeScript functions | More powerful      |
| MCP             | `.mcp.json`             | `config` hook        | Runtime injection  |
| Custom tools    | Via MCP only            | `tool` hook          | Direct definition  |
| Auth            | N/A                     | `auth` hook          | OpenCode only      |
| Events          | Limited                 | Full event bus       | OpenCode only      |
| SDK             | N/A                     | `ctx.client`         | OpenCode only      |
| Path resolution | `${CLAUDE_PLUGIN_ROOT}` | `ctx.directory`      | Runtime value      |
| Format          | Declarative             | Programmatic         | Different paradigm |

## Migration Checklist

- [ ] Convert `plugin.json` metadata to package.json
- [ ] Convert `commands/*.md` to `config.command` injection
- [ ] Convert `agents/*.md` to `config.agent` injection
- [ ] Convert `skills/` to custom tools or agents
- [ ] Convert `hooks.json` to TypeScript hook functions
- [ ] Convert `.mcp.json` to `config.mcp` injection
- [ ] Replace `${CLAUDE_PLUGIN_ROOT}` with `ctx.directory`
- [ ] Test hook functions with actual events
- [ ] Add TypeScript types for type safety
- [ ] Update documentation for new format

## See Also

- `02-plugin-context.md` - Context object and utilities
- `03-hooks-reference.md` - All available hooks
- `04-config-hook.md` - Config injection patterns
- `05-custom-tools.md` - Tool definition guide
- `06-events.md` - Event system overview
- `appendix/config-schemas.md` - Full schemas for commands, agents, MCP
- `appendix/event-schemas.md` - Full event payload schemas
