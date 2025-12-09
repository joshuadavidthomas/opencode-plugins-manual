# Hooks Reference

A plugin implements hooks to receive lifecycle events and modify OpenCode behavior.

## Quick Reference

| Hook                         | Purpose                              | Can Modify                 |
| ---------------------------- | ------------------------------------ | -------------------------- |
| `config`                     | Inject commands, agents, MCP servers | Config object              |
| `tool`                       | Register custom tools                | N/A (returns tools)        |
| `auth`                       | Register auth providers              | N/A (returns provider)     |
| `event`                      | Subscribe to all system events       | N/A (observer)             |
| `chat.message`               | Process incoming messages            | Message and parts          |
| `chat.params`                | Modify LLM parameters                | Temperature, topP, options |
| `permission.ask`             | Handle permission requests           | Status (ask/deny/allow)    |
| `tool.execute.before`        | Pre-execution hook                   | Tool arguments             |
| `tool.execute.after`         | Post-execution hook                  | Tool output                |
| `experimental.text.complete` | Post-generation modification         | Generated text             |

## config

Modify the configuration at runtime. Use this to inject commands, agents, or MCP servers into the user's environment.

> As of commit [3efc95b](https://github.com/sst/opencode/tree/3efc95b157e05bc3c42554be1b5778f8f1b64cfe)

**Type signature:**

```typescript
config?: (input: Config) => Promise<void>
```

**Parameters:**

- `input.command` — Command definitions (see [05-config-hook.md](05-config-hook.md))
- `input.agent` — Agent definitions (see [05-config-hook.md](05-config-hook.md))
- `input.mcp` — MCP server definitions (see [05-config-hook.md](05-config-hook.md))

**Mutates:** The `input` object directly.

**Triggered:** Once during plugin initialization.

**Example:**

```typescript
export const MyPlugin: Plugin = async (ctx) => ({
  config: async (config) => {
    config.command = config.command || {}
    config.command["greet"] = {
      template: "Say hello to $ARGUMENTS",
      description: "Greet someone",
    }
  },
})
```

**When to use:**

- Bundle commands with your plugin
- Bundle agents with your plugin
- Bundle MCP servers with your plugin
- Modify existing configuration programmatically

## tool

Register custom tools that the model can invoke.

> As of commit [3efc95b](https://github.com/sst/opencode/tree/3efc95b157e05bc3c42554be1b5778f8f1b64cfe)

**Type signature:**

```typescript
tool?: {
  [key: string]: ToolDefinition
}
```

**Returns:** An object mapping tool names to tool definitions.

**Triggered:** Once during plugin initialization.

**Example:**

```typescript
import { tool } from "@opencode-ai/plugin"

export const MyPlugin: Plugin = async (ctx) => ({
  tool: {
    timestamp: tool({
      description: "Get current timestamp",
      args: {},
      async execute(args, context) {
        return new Date().toISOString()
      },
    }),
  },
})
```

**When to use:**

- Add functionality unavailable through existing tools
- Integrate with external APIs or services
- Provide domain-specific operations
- Replace the need for a separate MCP server

See [06-custom-tools.md](06-custom-tools.md) for complete documentation.

## auth

Register authentication providers for LLM services.

> As of commit [3efc95b](https://github.com/sst/opencode/tree/3efc95b157e05bc3c42554be1b5778f8f1b64cfe)

**Type signature:**

```typescript
auth?: AuthHook
```

**Returns:** An auth hook configuration with provider name, methods, and loader.

**Triggered:** During provider initialization when authentication is needed.

**Example:**

```typescript
export const MyPlugin: Plugin = async (ctx) => ({
  auth: {
    provider: "my-llm-service",
    methods: [
      {
        type: "api",
        label: "API Key",
        prompts: [
          {
            type: "text",
            key: "apiKey",
            message: "Enter your API key",
          },
        ],
        async authorize(inputs) {
          return {
            type: "success",
            key: inputs.apiKey,
          }
        },
      },
    ],
  },
})
```

**When to use:**

- Add support for new LLM providers
- Implement custom OAuth flows
- Handle API key management

## event

Subscribe to all system events.

> As of commit [3efc95b](https://github.com/sst/opencode/tree/3efc95b157e05bc3c42554be1b5778f8f1b64cfe)

**Type signature:**

```typescript
event?: (input: { event: Event }) => Promise<void>
```

**Parameters:**

- `input.event` — The event object with `type` and event-specific data

**Triggered:** Whenever any subscribed event occurs.

**Available events:**

- **Command:** `command.executed`
- **File:** `file.edited`, `file.watcher.updated`
- **LSP:** `lsp.client.diagnostics`, `lsp.updated`
- **Message:** `message.part.removed`, `message.part.updated`, `message.removed`, `message.updated`
- **Permission:** `permission.replied`, `permission.updated`
- **Session:** `session.created`, `session.compacted`, `session.deleted`, `session.diff`, `session.error`, `session.idle`, `session.status`, `session.updated`
- **Tool:** `tool.execute.after`, `tool.execute.before`
- **TUI:** `tui.prompt.append`, `tui.command.execute`, `tui.toast.show`

**Example:**

```typescript
export const MyPlugin: Plugin = async (ctx) => ({
  event: async ({ event }) => {
    if (event.type === "session.idle") {
      console.log("Session completed")
    }
    if (event.type === "file.edited") {
      console.log("File edited:", event.path)
    }
  },
})
```

**When to use:**

- Monitor session activity
- Track file changes
- Trigger actions on specific events
- Log system activity

## chat.message

Process incoming user messages before they're sent to the model.

> As of commit [3efc95b](https://github.com/sst/opencode/tree/3efc95b157e05bc3c42554be1b5778f8f1b64cfe)

**Type signature:**

```typescript
"chat.message"?: (
  input: {
    sessionID: string
    agent?: string
    model?: { providerID: string; modelID: string }
    messageID?: string
  },
  output: {
    message: UserMessage
    parts: Part[]
  }
) => Promise<void>
```

**Parameters:**

- `input.sessionID` — Current session ID
- `input.agent` — Agent name (if any)
- `input.model` — Provider and model IDs
- `input.messageID` — Message ID
- `output.message` — The message object
- `output.parts` — Message parts array

**Mutates:** The `output` object directly.

**Triggered:** When user submits a message ([`packages/opencode/src/session/prompt.ts:1046`](https://github.com/sst/opencode/blob/3efc95b/packages/opencode/src/session/prompt.ts#L1046)).

**Example:**

```typescript
export const MyPlugin: Plugin = async (ctx) => ({
  "chat.message": async (input, output) => {
    // Add a system note to every message
    output.parts.push({
      type: "text",
      text: "\n[Plugin note: Injected context]",
    })
  },
})
```

**When to use:**

- Inject context into messages
- Modify message content
- Add system instructions dynamically
- Track message patterns

## chat.params

Modify parameters sent to the LLM.

> As of commit [3efc95b](https://github.com/sst/opencode/tree/3efc95b157e05bc3c42554be1b5778f8f1b64cfe)

**Type signature:**

```typescript
"chat.params"?: (
  input: {
    sessionID: string
    agent: string
    model: Model
    provider: ProviderContext
    message: UserMessage
  },
  output: {
    temperature: number
    topP: number
    options: Record<string, any>
  }
) => Promise<void>
```

**Parameters:**

- `input.sessionID` — Current session ID
- `input.agent` — Agent name
- `input.model` — Model information
- `input.provider` — Provider context
- `input.message` — The user message
- `output.temperature` — Temperature value (0-2)
- `output.topP` — Top-p sampling value (0-1)
- `output.options` — Provider-specific options

**Mutates:** The `output` object directly.

**Triggered:** Before each LLM request ([`packages/opencode/src/session/prompt.ts:920`](https://github.com/sst/opencode/blob/3efc95b/packages/opencode/src/session/prompt.ts#L920)).

**Example:**

```typescript
export const MyPlugin: Plugin = async (ctx) => ({
  "chat.params": async (input, output) => {
    // Increase temperature for creative tasks
    if (input.message.text?.includes("creative")) {
      output.temperature = 1.5
    }
  },
})
```

**When to use:**

- Adjust model parameters dynamically
- Implement agent-specific settings
- Override defaults based on context

## permission.ask

Handle permission requests before they're shown to the user.

> As of commit [3efc95b](https://github.com/sst/opencode/tree/3efc95b157e05bc3c42554be1b5778f8f1b64cfe)

**Type signature:**

```typescript
"permission.ask"?: (
  input: Permission,
  output: { status: "ask" | "deny" | "allow" }
) => Promise<void>
```

**Parameters:**

- `input` — Permission request details
- `output.status` — Set to "ask" (prompt user), "deny" (reject), or "allow" (approve)

**Mutates:** The `output` object directly.

**Triggered:** When a tool requires permission ([`packages/opencode/src/permission/index.ts:81`](https://github.com/sst/opencode/blob/3efc95b/packages/opencode/src/permission/index.ts#L81)).

**Example:**

```typescript
export const MyPlugin: Plugin = async (ctx) => ({
  "permission.ask": async (input, output) => {
    // Auto-approve read operations
    if (input.tool === "read") {
      output.status = "allow"
    }
    // Auto-deny dangerous commands
    if (input.tool === "bash" && input.args?.includes("rm -rf")) {
      output.status = "deny"
    }
  },
})
```

**When to use:**

- Auto-approve safe operations
- Deny dangerous operations
- Implement custom permission policies

## tool.execute.before

Intercept tool calls before execution.

> As of commit [3efc95b](https://github.com/sst/opencode/tree/3efc95b157e05bc3c42554be1b5778f8f1b64cfe)

**Type signature:**

```typescript
"tool.execute.before"?: (
  input: {
    tool: string
    sessionID: string
    callID: string
  },
  output: {
    args: any
  }
) => Promise<void>
```

**Parameters:**

- `input.tool` — Tool name
- `input.sessionID` — Current session ID
- `input.callID` — Unique call ID
- `output.args` — Tool arguments (can be modified)

**Mutates:** The `output` object directly.

**Triggered:** Before every tool execution ([`packages/opencode/src/session/prompt.ts:983`](https://github.com/sst/opencode/blob/3efc95b/packages/opencode/src/session/prompt.ts#L983) and [`:1007`](https://github.com/sst/opencode/blob/3efc95b/packages/opencode/src/session/prompt.ts#L1007)).

**Example:**

```typescript
export const MyPlugin: Plugin = async (ctx) => ({
  "tool.execute.before": async (input, output) => {
    if (input.tool === "bash") {
      console.log("Running command:", output.args.command)
    }
    // Modify arguments
    if (input.tool === "write" && !output.args.filePath.endsWith(".txt")) {
      output.args.filePath += ".txt"
    }
  },
})
```

**When to use:**

- Log tool usage
- Validate arguments
- Modify arguments before execution
- Block certain operations

## tool.execute.after

Intercept tool results after execution.

> As of commit [3efc95b](https://github.com/sst/opencode/tree/3efc95b157e05bc3c42554be1b5778f8f1b64cfe)

**Type signature:**

```typescript
"tool.execute.after"?: (
  input: {
    tool: string
    sessionID: string
    callID: string
  },
  output: {
    title: string
    output: string
    metadata: any
  }
) => Promise<void>
```

**Parameters:**

- `input.tool` — Tool name
- `input.sessionID` — Current session ID
- `input.callID` — Unique call ID
- `output.title` — Result title (can be modified)
- `output.output` — Result content (can be modified)
- `output.metadata` — Additional metadata (can be modified)

**Mutates:** The `output` object directly.

**Triggered:** After every tool execution ([`packages/opencode/src/session/prompt.ts:995`](https://github.com/sst/opencode/blob/3efc95b/packages/opencode/src/session/prompt.ts#L995) and [`:1020`](https://github.com/sst/opencode/blob/3efc95b/packages/opencode/src/session/prompt.ts#L1020)).

**Example:**

```typescript
export const MyPlugin: Plugin = async (ctx) => ({
  "tool.execute.after": async (input, output) => {
    // Run formatter after file edits
    if (input.tool === "write" || input.tool === "edit") {
      await ctx.$`prettier --write ${output.metadata.path}`
    }
    // Redact sensitive data
    if (output.output.includes("API_KEY")) {
      output.output = output.output.replace(/API_KEY=\w+/g, "API_KEY=***")
    }
  },
})
```

**When to use:**

- Post-process tool results
- Run additional commands
- Format or transform output
- Redact sensitive information

## experimental.text.complete

Modify text after the model generates it.

> As of commit [3efc95b](https://github.com/sst/opencode/tree/3efc95b157e05bc3c42554be1b5778f8f1b64cfe)

**Type signature:**

```typescript
"experimental.text.complete"?: (
  input: {
    sessionID: string
    messageID: string
    partID: string
  },
  output: {
    text: string
  }
) => Promise<void>
```

**Parameters:**

- `input.sessionID` — Current session ID
- `input.messageID` — Message ID
- `input.partID` — Part ID
- `output.text` — Generated text (can be modified)

**Mutates:** The `output` object directly.

**Triggered:** After text generation completes ([`packages/opencode/src/session/processor.ts:73`](https://github.com/sst/opencode/blob/3efc95b/packages/opencode/src/session/processor.ts#L73)).

**Example:**

```typescript
export const MyPlugin: Plugin = async (ctx) => ({
  "experimental.text.complete": async (input, output) => {
    // Add signature to responses
    output.text += "\n\n---\nGenerated by OpenCode"
  },
})
```

**When to use:**

- Post-process generated text
- Add branding or signatures
- Apply text transformations

**Warning:** Marked experimental. API may change.

## Summary

Hooks let plugins observe and modify OpenCode behavior at specific points in the lifecycle. The most powerful hooks are:

- **`config`** — Bundle commands, agents, and MCP servers
- **`tool`** — Add custom functionality
- **`tool.execute.after`** — Post-process tool results
- **`event`** — Monitor system activity

Start with these four. Add others as needed.
