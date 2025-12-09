# SDK Client

## Overview

The SDK client provides programmatic access to OpenCode's API. Use it to control sessions, read files, manage configuration, and interact with the running instance.

> As of commit 3efc95b15

The client is auto-generated from OpenAPI specs. Source: `packages/sdk/js/src/gen/sdk.gen.ts`

## When to Use

Use the SDK client when your plugin needs to:

- Query or manipulate sessions programmatically
- Read project files without using tools
- Modify configuration at runtime
- Control the TUI (append to prompt, show notifications)
- Manage MCP servers dynamically
- Integrate with external systems that need OpenCode state

## Basic Usage

```typescript
import { type Plugin } from "@opencode-ai/plugin"

export const MyPlugin: Plugin = async (ctx) => {
  const { client } = ctx

  // Get current configuration
  const config = await client.config.get()

  // List all sessions
  const sessions = await client.session.list()

  return {
    event: async ({ event }) => {
      if (event.type === "session.idle") {
        // Read session messages when complete
        const messages = await client.session.messages({
          path: { id: event.properties.sessionID },
        })
        console.log("Session had", messages.data.length, "messages")
      }
    },
  }
}
```

The `ctx.client` object is available in the plugin initialization function and provides access to all sub-clients.

## Sub-Clients

The SDK client is organized into sub-clients by domain:

| Client        | Purpose              | Methods                                                                                                                                                  |
| ------------- | -------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **global**    | Cross-project events | `event()` - SSE stream of all events                                                                                                                     |
| **project**   | Project management   | `list()`, `current()`                                                                                                                                    |
| **session**   | Session operations   | `list()`, `create()`, `get()`, `delete()`, `update()`, `messages()`, `prompt()`, `status()`, `diff()`, `abort()`, `fork()`, `share()`, `unshare()`, etc. |
| **command**   | Slash commands       | `list()`                                                                                                                                                 |
| **file**      | File operations      | `list()`, `read()`, `status()`                                                                                                                           |
| **find**      | Search operations    | `text()`, `files()`, `symbols()`                                                                                                                         |
| **config**    | Configuration        | `get()`, `update()`, `providers()`                                                                                                                       |
| **tool**      | Tool introspection   | `ids()`, `list()`                                                                                                                                        |
| **mcp**       | MCP server control   | `status()`, `add()`, `connect()`, `disconnect()`                                                                                                         |
| **lsp**       | LSP server status    | `status()`                                                                                                                                               |
| **formatter** | Formatter status     | `status()`                                                                                                                                               |
| **tui**       | TUI control          | `appendPrompt()`, `submitPrompt()`, `clearPrompt()`, `showToast()`, `executeCommand()`, `openHelp()`, `openSessions()`, `openThemes()`, `openModels()`   |
| **pty**       | Terminal sessions    | `list()`, `create()`, `get()`, `update()`, `remove()`, `connect()`                                                                                       |
| **provider**  | Auth providers       | `list()`, `auth()`                                                                                                                                       |
| **auth**      | Authentication       | `set()`                                                                                                                                                  |
| **vcs**       | Version control      | `get()`                                                                                                                                                  |
| **path**      | Path utilities       | `get()`                                                                                                                                                  |
| **instance**  | Instance control     | `dispose()`                                                                                                                                              |
| **app**       | Application info     | `log()`, `agents()`                                                                                                                                      |

All clients defined in `packages/sdk/js/src/gen/sdk.gen.ts:233-1050`

## Common Operations

### Session Management

```typescript
// List all sessions
const sessions = await client.session.list()

// Get session details
const session = await client.session.get({
  path: { id: "ses_..." },
})

// Create a new session
const newSession = await client.session.create({
  body: {
    title: "New task",
  },
})

// Send a prompt
await client.session.prompt({
  path: { id: session.id },
  body: {
    message: "Fix the bug in auth.ts",
  },
})

// Get session status
const status = await client.session.status({
  path: { id: session.id },
})

// Abort running session
await client.session.abort({
  path: { id: session.id },
})
```

### File Operations

```typescript
// List files in project
const files = await client.file.list()

// Read a file
const content = await client.file.read({
  query: { path: "src/index.ts" },
})

// Get git status for files
const status = await client.file.status()
```

### Search Operations

```typescript
// Search text in files
const results = await client.find.text({
  body: {
    query: "async function",
    include: "*.ts",
  },
})

// Find files by name pattern
const files = await client.find.files({
  body: {
    pattern: "**/*.test.ts",
  },
})

// Find symbols (requires LSP)
const symbols = await client.find.symbols({
  body: {
    query: "MyComponent",
  },
})
```

### Configuration

```typescript
// Get current config
const config = await client.config.get()

// Update config
await client.config.update({
  body: {
    model: "anthropic/claude-sonnet-4-20250514",
    agent: {
      "my-agent": {
        prompt: "Custom agent prompt",
        mode: "subagent",
      },
    },
  },
})
```

### TUI Control

```typescript
// Show notification
await client.tui.showToast({
  body: {
    message: "Task completed successfully",
    variant: "success",
    duration: 3000,
  },
})

// Append text to prompt
await client.tui.appendPrompt({
  body: {
    text: "Review the changes in src/",
  },
})

// Submit prompt programmatically
await client.tui.submitPrompt()

// Execute TUI command
await client.tui.executeCommand({
  body: {
    command: "session.new",
  },
})
```

### MCP Management

```typescript
// Get MCP server status
const status = await client.mcp.status()

// Add MCP server dynamically
await client.mcp.add({
  body: {
    name: "my-server",
    type: "remote",
    url: "http://localhost:3000",
  },
})

// Connect to MCP server
await client.mcp.connect({
  path: { name: "my-server" },
})

// Disconnect MCP server
await client.mcp.disconnect({
  path: { name: "my-server" },
})
```

### Tool Introspection

```typescript
// Get all tool IDs
const toolIds = await client.tool.ids()

// Get tool details
const tools = await client.tool.list({
  body: {
    ids: ["read", "write", "bash"],
  },
})
```

## Path Parameters vs Query Parameters

The SDK uses TypeScript to enforce correct parameter placement:

```typescript
// Path parameters go in path object
await client.session.get({
  path: { id: "ses_..." },
})

// Query parameters go in query object
await client.file.read({
  query: { path: "src/index.ts" },
})

// Body parameters go in body object
await client.session.prompt({
  path: { id: "ses_..." },
  body: {
    message: "Do something",
  },
})
```

## Error Handling

All methods throw on HTTP errors by default. Catch them with try/catch:

```typescript
try {
  const session = await client.session.get({
    path: { id: "invalid" },
  })
} catch (error) {
  console.error("Failed to get session:", error)
}
```

## Type Safety

The SDK is fully typed. TypeScript will catch incorrect parameter names or types:

```typescript
// ✓ Correct
await client.session.list()

// ✗ TypeScript error: Property 'listAll' does not exist
await client.session.listAll()

// ✓ Correct
await client.session.prompt({
  path: { id: sessionId },
  body: { message: "Hello" },
})

// ✗ TypeScript error: Property 'text' does not exist in type
await client.session.prompt({
  path: { id: sessionId },
  body: { text: "Hello" },
})
```

## SSE Streams

Some endpoints return Server-Sent Events streams:

```typescript
// Global event stream (all projects)
const stream = await client.global.event()

// Read events from stream
for await (const event of stream) {
  console.log("Event:", event)
}
```

## Plugin Context

The client is available in the plugin context:

```typescript
export const MyPlugin: Plugin = async (ctx) => {
  // ctx.client is the SDK client
  // ctx.directory is the project directory
  // ctx.provider is the auth provider (if any)

  const config = await ctx.client.config.get()

  return {
    // Plugin hooks
  }
}
```

Source: `packages/plugin/src/index.ts:18-32`

## Full Reference

The complete API is auto-generated from OpenAPI specs. For detailed method signatures, parameters, and return types:

- **Type definitions**: `packages/sdk/js/src/gen/types.gen.ts`
- **Client implementation**: `packages/sdk/js/src/gen/sdk.gen.ts`
- **OpenAPI spec**: `packages/sdk/openapi.json`

The SDK is generated by `@hey-api/openapi-ts` and follows OpenAPI conventions.
