# OpenCode Plugins: Overview

## What Is a Plugin?

A plugin is a **TypeScript or JavaScript module** that exports one or more functions. Each function receives a context object and returns hooks that extend OpenCode's behavior.

Plugins run inside OpenCode's runtime. They can:

- Add custom tools
- Register authentication providers
- Intercept tool execution
- Modify LLM parameters
- Subscribe to system events
- Inject commands, agents, and MCP servers

## Minimal Example

```typescript
// .opencode/plugin/example.ts
import type { Plugin } from "@opencode-ai/plugin"

export const ExamplePlugin: Plugin = async (ctx) => {
  console.log("Plugin initialized for:", ctx.project.name)

  return {
    event: async ({ event }) => {
      if (event.type === "session.idle") {
        console.log("Session completed")
      }
    },
  }
}
```

This plugin logs when sessions complete. The plugin function receives context (project info, SDK client, shell access) and returns an object with hook implementations.

## Plugin Capabilities

### Custom Tools

Define tools the model can call. Tools use Zod schemas for type-safe arguments.

```typescript
import { type Plugin, tool } from "@opencode-ai/plugin"

export const ToolPlugin: Plugin = async (ctx) => ({
  tool: {
    timestamp: tool({
      description: "Get current Unix timestamp",
      args: {},
      async execute() {
        return Date.now().toString()
      },
    }),
  },
})
```

### Event Subscriptions

React to system events: file edits, session state changes, tool executions, LSP diagnostics, and more.

```typescript
export const EventPlugin: Plugin = async (ctx) => ({
  event: async ({ event }) => {
    if (event.type === "file.edited") {
      await ctx.$`echo "File edited: ${event.properties.path}"`
    }
  },
})
```

### Tool Execution Hooks

Run code before or after any tool executes. Modify arguments or output.

```typescript
export const HookPlugin: Plugin = async (ctx) => ({
  "tool.execute.before": async (input, output) => {
    if (input.tool === "read" && output.args.filePath.includes(".env")) {
      throw new Error("Cannot read .env files")
    }
  },
})
```

### Configuration Injection

Inject commands, agents, and MCP servers at runtime. Commands and agents appear alongside file-based ones.

```typescript
export const ConfigPlugin: Plugin = async (ctx) => ({
  config: async (config) => {
    config.command = config.command || {}
    config.command["timestamp"] = {
      template: "What is the current timestamp?",
      description: "Get current time",
    }
  },
})
```

### Authentication Providers

Register custom OAuth or API key authentication flows.

```typescript
export const AuthPlugin: Plugin = async (ctx) => ({
  auth: {
    provider: "my-service",
    methods: [
      {
        type: "api",
        label: "API Key",
        prompts: [
          {
            type: "text",
            key: "apiKey",
            message: "Enter API key",
          },
        ],
        async authorize(inputs) {
          return { type: "success", key: inputs.apiKey }
        },
      },
    ],
  },
})
```

### LLM Parameter Modification

Change temperature, top_p, or provider-specific options before each LLM call.

```typescript
export const ParamsPlugin: Plugin = async (ctx) => ({
  "chat.params": async (input, output) => {
    if (input.agent === "my-agent") {
      output.temperature = 0.2
    }
  },
})
```

### Permission Handling

Override permission prompts or enforce policies.

```typescript
export const PermissionPlugin: Plugin = async (ctx) => ({
  "permission.ask": async (input, output) => {
    if (input.type === "bash" && input.value.includes("rm -rf")) {
      output.status = "deny"
    }
  },
})
```

## Capability Summary

| Capability        | Hook                                           | Use Case                                       |
| ----------------- | ---------------------------------------------- | ---------------------------------------------- |
| Custom tools      | `tool`                                         | Add functionality without MCP                  |
| Event handling    | `event`                                        | React to file edits, sessions, diagnostics     |
| Tool interception | `tool.execute.before`<br/>`tool.execute.after` | Validate inputs, transform outputs             |
| Config injection  | `config`                                       | Bundle commands, agents, MCP servers           |
| Auth providers    | `auth`                                         | Custom OAuth or API key flows                  |
| LLM parameters    | `chat.params`                                  | Adjust temperature, top_p per agent            |
| Permissions       | `permission.ask`                               | Enforce policies, auto-approve safe operations |
| Message handling  | `chat.message`                                 | Process incoming user messages                 |
| Text completion   | `experimental.text.complete`                   | Post-process generated text                    |

## Plugin Loading

> As of commit [3efc95b](https://github.com/sst/opencode/tree/3efc95b157e05bc3c42554be1b5778f8f1b64cfe)

OpenCode loads plugins from the `plugin` array in `opencode.json`:

```json
{
  "plugin": ["file://.opencode/plugin/example.ts", "my-plugin-package@1.0.0"]
}
```

Plugins can be:

- **Local files**: `file://` paths (relative to worktree or absolute)
- **npm packages**: Package name with optional `@version`

OpenCode also auto-loads default plugins unless `OPENCODE_DISABLE_DEFAULT_PLUGINS` is set:

- `opencode-copilot-auth` - GitHub Copilot OAuth authentication
- `opencode-anthropic-auth` - Anthropic OAuth (Claude Pro/Max) and API key creation

See [`packages/opencode/src/plugin/index.ts:29-33`](https://github.com/sst/opencode/blob/3efc95b/packages/opencode/src/plugin/index.ts#L29-L33) and [Auth Hook](09-auth-hook.md#official-auth-plugins) for details.

## Plugin Structure

Each exported function is a separate plugin. A single module can export multiple plugins:

```typescript
export const PluginOne: Plugin = async (ctx) => ({
  /* ... */
})
export const PluginTwo: Plugin = async (ctx) => ({
  /* ... */
})
```

The plugin function:

1. Receives `PluginInput` (context object)
2. Runs once at startup
3. Returns `Hooks` object (implements desired hooks)

All hooks are optional. Return only the hooks you need.

## Next Steps

- **[Quick Start](02-quick-start.md)**: Create your first plugin
- **[Plugin Context](03-plugin-context.md)**: Deep dive on the context object
