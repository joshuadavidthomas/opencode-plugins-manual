# OpenCode Plugins: Quick Start

## Plugin Locations

Plugins load from two locations:

1. **Project plugins**: `.opencode/plugin/` directory in your project
2. **Global plugins**: `~/.config/opencode/plugin/` directory

> As of commit 3efc95b15

OpenCode discovers plugin files and registers them from the `plugin` array in `opencode.json`. File-based plugins use the `file://` protocol.

## Basic Plugin Structure

A plugin is a TypeScript or JavaScript module that exports one or more plugin functions:

```typescript
import type { Plugin } from "@opencode-ai/plugin"

export const MyPlugin: Plugin = async (ctx) => {
  // Initialization code runs once at startup
  console.log("Plugin loaded for project:", ctx.project.name)

  // Return hooks object
  return {
    event: async ({ event }) => {
      // Handle events
    },
    tool: {
      // Define custom tools
    },
    config: async (config) => {
      // Modify configuration
    },
    // ... other hooks
  }
}
```

### Type Definition

```typescript
type PluginInput = {
  client: ReturnType<typeof createOpencodeClient>
  project: Project
  directory: string
  worktree: string
  $: BunShell
}

type Plugin = (input: PluginInput) => Promise<Hooks>
```

See packages/plugin/src/index.ts:25-33

### Available Hooks

All hooks are optional. Implement only what you need:

| Hook                         | Type                                         | Purpose                           |
| ---------------------------- | -------------------------------------------- | --------------------------------- |
| `event`                      | `(input: { event: Event }) => Promise<void>` | Subscribe to all system events    |
| `config`                     | `(input: Config) => Promise<void>`           | Modify runtime configuration      |
| `tool`                       | `Record<string, ToolDefinition>`             | Register custom tools             |
| `auth`                       | `AuthHook`                                   | Register auth providers           |
| `chat.message`               | Function                                     | Process incoming messages         |
| `chat.params`                | Function                                     | Modify LLM parameters             |
| `permission.ask`             | Function                                     | Handle permission requests        |
| `tool.execute.before`        | Function                                     | Pre-execution hook                |
| `tool.execute.after`         | Function                                     | Post-execution hook               |
| `experimental.text.complete` | Function                                     | Post-generation text modification |

See packages/plugin/src/index.ts:144-182 for complete type definitions.

## Step-by-Step: Your First Plugin

### 1. Create Plugin File

Choose a location:

- Project-specific: `.opencode/plugin/notify.ts`
- Global: `~/.config/opencode/plugin/notify.ts`

Create the file:

```typescript
// .opencode/plugin/notify.ts
import type { Plugin } from "@opencode-ai/plugin"

export const NotifyPlugin: Plugin = async (ctx) => {
  return {
    event: async ({ event }) => {
      if (event.type === "session.idle") {
        await ctx.$`osascript -e 'display notification "Session completed!" with title "OpenCode"'`.nothrow()
      }
    },
  }
}
```

This plugin sends a macOS notification when sessions complete.

### 2. Register Plugin

Add to `opencode.json` in your project root:

```json
{
  "plugin": ["file://.opencode/plugin/notify.ts"]
}
```

Or for global plugins:

```json
{
  "plugin": ["file://~/.config/opencode/plugin/notify.ts"]
}
```

### 3. Restart OpenCode

Plugins load at startup. Restart OpenCode to load your plugin:

```bash
# If running in terminal
# Exit and restart

# Check logs to verify plugin loaded
# Plugins log "loading plugin" with their path
```

See packages/opencode/src/plugin/index.ts:35 for loading logs.

### 4. Test Your Plugin

Start a session and send a message. When the session completes (idle state), you should see a notification.

## Common Plugin Patterns

### Event-Driven Actions

```typescript
export const EventPlugin: Plugin = async (ctx) => ({
  event: async ({ event }) => {
    switch (event.type) {
      case "file.edited":
        console.log("File edited:", event.properties.path)
        break
      case "session.created":
        console.log("New session started")
        break
      case "tool.execute.after":
        console.log("Tool completed:", event.properties.tool)
        break
    }
  },
})
```

Available events: command.executed, file.edited, file.watcher.updated, lsp.client.diagnostics, message.updated, session.created, session.idle, tool.execute.after, and more. See packages/web/src/content/docs/plugins.mdx:68-127 for complete list.

### Custom Tool

```typescript
import { type Plugin, tool } from "@opencode-ai/plugin"

export const TimestampPlugin: Plugin = async (ctx) => ({
  tool: {
    get_timestamp: tool({
      description: "Get current Unix timestamp in milliseconds",
      args: {},
      async execute(args) {
        return Date.now().toString()
      },
    }),
    format_date: tool({
      description: "Format timestamp as ISO 8601",
      args: {
        timestamp: tool.schema.number().describe("Unix timestamp in milliseconds"),
      },
      async execute(args) {
        return new Date(args.timestamp).toISOString()
      },
    }),
  },
})
```

Tools use Zod schemas via `tool.schema`. See packages/plugin/src/tool.ts:10-19.

### Command Injection

```typescript
export const CommandPlugin: Plugin = async (ctx) => ({
  config: async (config) => {
    config.command = config.command || {}
    config.command["check-deps"] = {
      template: "Check package.json for outdated dependencies and suggest updates",
      description: "Analyze dependencies",
      agent: "general",
    }
  },
})
```

Command schema supports: `template` (required), `description`, `agent`, `model`, `subtask`. See packages/opencode/src/config/config.ts:375-382.

### Agent Injection

```typescript
export const AgentPlugin: Plugin = async (ctx) => ({
  config: async (config) => {
    config.agent = config.agent || {}
    config.agent["sql-expert"] = {
      prompt: "You are an expert in SQL query optimization and database design. Help users write efficient queries.",
      description: "SQL query optimization and database design",
      mode: "subagent",
      temperature: 0.1,
    }
  },
})
```

Agent schema supports: `prompt`, `description`, `mode` (subagent|primary|all), `model`, `temperature`, `top_p`, `tools`, `maxSteps`, `color`, `permission`. See packages/opencode/src/config/config.ts:384-419.

### Tool Interception

```typescript
export const GuardPlugin: Plugin = async (ctx) => ({
  "tool.execute.before": async (input, output) => {
    // Block reading sensitive files
    if (input.tool === "read" && output.args.filePath.includes(".env")) {
      throw new Error("Cannot read .env files")
    }

    // Log all bash commands
    if (input.tool === "bash") {
      console.log("Executing:", output.args.command)
    }
  },

  "tool.execute.after": async (input, output) => {
    // Auto-format written files
    if (input.tool === "write" || input.tool === "edit") {
      const file = output.metadata?.filePath
      if (file?.endsWith(".ts") || file?.endsWith(".js")) {
        await ctx.$`prettier --write ${file}`.nothrow()
      }
    }
  },
})
```

Before hooks can modify `output.args`. After hooks can modify `output.output` and `output.metadata`. Throw to abort execution.

## npm Package Plugins

Publish reusable plugins as npm packages:

### Package Structure

```
my-opencode-plugin/
├── package.json
├── tsconfig.json
└── src/
    └── index.ts
```

### package.json

```json
{
  "name": "my-opencode-plugin",
  "version": "1.0.0",
  "type": "module",
  "main": "dist/index.js",
  "types": "dist/index.d.ts",
  "peerDependencies": {
    "@opencode-ai/plugin": "*"
  }
}
```

### src/index.ts

```typescript
import type { Plugin } from "@opencode-ai/plugin"

export const MyPlugin: Plugin = async (ctx) => ({
  // ... hooks
})
```

### Usage

Install and register:

```bash
bun add my-opencode-plugin
```

```json
{
  "plugin": ["my-opencode-plugin@1.0.0"]
}
```

OpenCode resolves npm packages and installs them if missing. See packages/opencode/src/plugin/index.ts:36-41.

## Local Development with file://

Use `file://` paths during development:

```json
{
  "plugin": ["file://.opencode/plugin/dev-plugin.ts", "file:///absolute/path/to/plugin.ts"]
}
```

Relative paths resolve from the worktree root. Absolute paths work as-is.

## Debugging

### Check Plugin Loading

Plugins log when they load:

```
[plugin] loading plugin { path: 'file://.opencode/plugin/example.ts' }
```

See packages/opencode/src/plugin/index.ts:35.

### Console Logging

Use `console.log()` in your plugin. Output appears in OpenCode's logs.

### TypeScript Errors

OpenCode compiles TypeScript plugins at runtime using Bun. Type errors will prevent the plugin from loading.

Install types for better IDE support:

```bash
bun add -D @opencode-ai/plugin @opencode-ai/sdk
```

## Next Steps

- **[Plugin Context](03-plugin-context.md)**: Learn about the context object (client, project, $, directory, worktree)
