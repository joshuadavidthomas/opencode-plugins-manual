# Custom Tools

Plugins register custom tools that the model can invoke. Custom tools extend OpenCode capabilities without requiring a separate MCP server.

> As of commit [3efc95b](https://github.com/sst/opencode/tree/3efc95b157e05bc3c42554be1b5778f8f1b64cfe)

## The tool() Helper

The `tool()` helper creates tool definitions with type safety.

**Location:** packages/plugin/src/tool.ts:10-19

**Type signature:**

```typescript
function tool<Args extends z.ZodRawShape>(input: {
  description: string
  args: Args
  execute(args: z.infer<z.ZodObject<Args>>, context: ToolContext): Promise<string>
}): ToolDefinition
```

**Returns:** A `ToolDefinition` object that OpenCode registers.

## Basic Example

```typescript
import { type Plugin, tool } from "@opencode-ai/plugin"

export const MyPlugin: Plugin = async (ctx) => ({
  tool: {
    timestamp: tool({
      description: "Get the current timestamp in ISO format",
      args: {},
      async execute(args, context) {
        return new Date().toISOString()
      },
    }),
  },
})
```

The model can now invoke:

```typescript
timestamp() // Returns: "2025-12-09T18:30:00.000Z"
```

## Tool Schema with Arguments

Use `tool.schema` (Zod) to define typed arguments.

**Available via:** `tool.schema.*` (packages/plugin/src/tool.ts:17)

The `tool.schema` object is Zod (`z` from `"zod"`). All Zod types are available.

**Common types:**

```typescript
tool.schema.string() // String
tool.schema.number() // Number
tool.schema.boolean() // Boolean
tool.schema.array(z.string()) // Array of strings
tool.schema.object({
  /* ... */
}) // Nested object
tool.schema.enum(["a", "b"]) // Enum
tool.schema.string().optional() // Optional field
```

**Example with arguments:**

```typescript
export const MyPlugin: Plugin = async (ctx) => ({
  tool: {
    greet: tool({
      description: "Greet someone by name",
      args: {
        name: tool.schema.string().describe("The person's name"),
        enthusiastic: tool.schema.boolean().optional().describe("Use excited tone"),
      },
      async execute(args, context) {
        const greeting = `Hello, ${args.name}!`
        return args.enthusiastic ? greeting.toUpperCase() : greeting
      },
    }),
  },
})
```

The model invokes:

```typescript
greet({ name: "Alice", enthusiastic: true })
// Returns: "HELLO, ALICE!"
```

## Zod Patterns

**String with constraints:**

```typescript
args: {
  email: tool.schema.string().email(),
  url: tool.schema.string().url(),
  pattern: tool.schema.string().regex(/^\d{3}-\d{3}-\d{4}$/),
  limited: tool.schema.string().min(1).max(100)
}
```

**Numbers:**

```typescript
args: {
  age: tool.schema.number().int().positive(),
  score: tool.schema.number().min(0).max(100),
  price: tool.schema.number().nonnegative()
}
```

**Arrays:**

```typescript
args: {
  tags: tool.schema.array(tool.schema.string()),
  coords: tool.schema.array(tool.schema.number()).length(2),
  items: tool.schema.array(tool.schema.object({
    id: tool.schema.string(),
    quantity: tool.schema.number()
  }))
}
```

**Objects:**

```typescript
args: {
  config: tool.schema.object({
    host: tool.schema.string(),
    port: tool.schema.number(),
    secure: tool.schema.boolean().optional(),
  })
}
```

**Enums and unions:**

```typescript
args: {
  format: tool.schema.enum(["json", "yaml", "toml"]),
  mode: tool.schema.union([
    tool.schema.literal("auto"),
    tool.schema.literal("manual")
  ])
}
```

**Defaults and transforms:**

```typescript
args: {
  count: tool.schema.number().default(10),
  name: tool.schema.string().transform(s => s.toLowerCase())
}
```

## Execution Context

The `execute` function receives a context object with session information.

**Type signature:**

```typescript
type ToolContext = {
  sessionID: string // Current session ID
  messageID: string // Current message ID
  agent: string // Agent name
  abort: AbortSignal // Abort signal for cancellation
}
```

**Example:**

```typescript
export const MyPlugin: Plugin = async (ctx) => ({
  tool: {
    log_session: tool({
      description: "Log the current session ID",
      args: {},
      async execute(args, context) {
        console.log("Session:", context.sessionID)
        console.log("Agent:", context.agent)
        return `Logged session ${context.sessionID}`
      },
    }),
  },
})
```

**Abort signal:**

Use `context.abort` to check for cancellation:

```typescript
async execute(args, context) {
  for (let i = 0; i < 100; i++) {
    if (context.abort.aborted) {
      return "Operation cancelled"
    }
    await doWork(i)
  }
  return "Completed"
}
```

## Return Values

Tools return strings. OpenCode shows the returned string to the model.

**Simple return:**

```typescript
async execute(args, context) {
  return "Operation successful"
}
```

**Formatted return:**

```typescript
async execute(args, context) {
  return `Result: ${value}\nStatus: ${status}\nTime: ${elapsed}ms`
}
```

**JSON return:**

```typescript
async execute(args, context) {
  return JSON.stringify({
    success: true,
    data: results,
    count: results.length
  }, null, 2)
}
```

**Markdown return:**

```typescript
async execute(args, context) {
  return `## Results\n\n- Item 1\n- Item 2\n\n**Total:** ${count}`
}
```

The model interprets the returned string. Use clear formatting.

## Error Handling

Throw errors for failures. OpenCode catches them and shows the error to the model.

**Example:**

```typescript
export const MyPlugin: Plugin = async (ctx) => ({
  tool: {
    fetch_data: tool({
      description: "Fetch data from an API",
      args: {
        url: tool.schema.string().url(),
      },
      async execute(args, context) {
        const response = await fetch(args.url)

        if (!response.ok) {
          throw new Error(`HTTP ${response.status}: ${response.statusText}`)
        }

        return await response.text()
      },
    }),
  },
})
```

**Error format:**

Thrown errors appear in the tool output:

```
Error: HTTP 404: Not Found
```

The model sees this and can respond accordingly.

**Validation errors:**

Zod validates arguments automatically. Invalid arguments produce descriptive errors:

```typescript
args: {
  count: tool.schema.number().int().positive()
}
// If model passes -5, Zod throws: "Number must be greater than 0"
```

## Complete Example: File Analyzer

This tool analyzes files and returns statistics.

```typescript
import { type Plugin, tool } from "@opencode-ai/plugin"
import { readFile } from "node:fs/promises"

export const FileToolsPlugin: Plugin = async (ctx) => ({
  tool: {
    analyze_file: tool({
      description: "Analyze a file and return statistics (lines, words, size)",
      args: {
        path: tool.schema.string().describe("Path to the file"),
        include_preview: tool.schema.boolean().optional().describe("Include content preview"),
      },
      async execute(args, context) {
        try {
          const content = await readFile(args.path, "utf-8")
          const lines = content.split("\n")
          const words = content.split(/\s+/).filter((w) => w.length > 0)
          const bytes = Buffer.byteLength(content, "utf-8")

          let result = `File: ${args.path}\n`
          result += `Lines: ${lines.length}\n`
          result += `Words: ${words.length}\n`
          result += `Size: ${bytes} bytes\n`

          if (args.include_preview) {
            const preview = lines.slice(0, 10).join("\n")
            result += `\nPreview (first 10 lines):\n${preview}`
          }

          return result
        } catch (error) {
          throw new Error(`Failed to analyze file: ${error.message}`)
        }
      },
    }),
  },
})
```

The model can invoke:

```typescript
analyze_file({
  path: "/path/to/file.txt",
  include_preview: true,
})
```

## Plugin Context Access

Tools have access to the plugin context passed during initialization.

**Example:**

```typescript
export const MyPlugin: Plugin = async (ctx) => {
  // Plugin context available here
  const { client, project, directory, worktree, $ } = ctx

  return {
    tool: {
      project_info: tool({
        description: "Get information about the current project",
        args: {},
        async execute(args, context) {
          return `Project: ${project.name}\nDirectory: ${directory}`
        },
      }),

      run_command: tool({
        description: "Run a shell command in the project directory",
        args: {
          command: tool.schema.string(),
        },
        async execute(args, context) {
          // Use the Bun shell from ctx
          const result = await ctx.$`${args.command}`.text()
          return result
        },
      }),
    },
  }
}
```

**Plugin context fields:**

- `ctx.client` — OpenCode API client
- `ctx.project` — Current project info
- `ctx.directory` — Plugin directory path
- `ctx.worktree` — Git worktree path
- `ctx.$` — Bun shell for running commands

## Tool Registration

Tools register during plugin initialization.

**Location:** packages/opencode/src/tool/registry.ts:45-50

**Process:**

1. OpenCode loads plugins
2. Collects `tool` objects from all plugins
3. Registers each tool by name
4. Makes tools available to the model

**Namespacing:**

Tool names must be unique across all plugins. Use prefixes:

```typescript
tool: {
  "myplugin-analyze": tool({ /* ... */ }),
  "myplugin-transform": tool({ /* ... */ })
}
```

**Verification:**

Check available tools in a session:

```
List all available tools
```

The model responds with registered tools, including yours.

## Tools vs. MCP Servers

**Custom tools** (via plugin) and **MCP servers** both add functionality. When to use each?

**Use custom tools when:**

- Simple, single-purpose operations
- Logic fits in one function
- No external dependencies
- Want fastest execution
- Bundling with a plugin

**Use MCP servers when:**

- Complex multi-tool systems
- External service integration
- Requires separate process
- Shared across multiple projects
- Implements standard protocol

**Example decision:**

- **Custom tool:** "Get current timestamp" — simple, no dependencies
- **MCP server:** "Database operations" — complex, maintains connections

## Performance Notes

**Tool execution is synchronous from the model's perspective.** The model waits for your tool to complete.

**Keep tools fast:**

```typescript
// Good: Quick operation
async execute(args, context) {
  return new Date().toISOString()
}

// Bad: Blocks for 30 seconds
async execute(args, context) {
  await sleep(30000)
  return "Done"
}
```

**For long operations:**

Return early with a status:

```typescript
async execute(args, context) {
  // Start background job
  startJob(args).catch(console.error)

  // Return immediately
  return "Job started. Check status with job_status tool."
}
```

## Common Patterns

**HTTP requests:**

```typescript
fetch_api: tool({
  description: "Fetch data from an API",
  args: {
    endpoint: tool.schema.string().url(),
  },
  async execute(args, context) {
    const response = await fetch(args.endpoint)
    if (!response.ok) throw new Error(`HTTP ${response.status}`)
    return await response.text()
  },
})
```

**File operations:**

```typescript
write_json: tool({
  description: "Write data to a JSON file",
  args: {
    path: tool.schema.string(),
    data: tool.schema.record(tool.schema.any()),
  },
  async execute(args, context) {
    await writeFile(args.path, JSON.stringify(args.data, null, 2))
    return `Wrote data to ${args.path}`
  },
})
```

**Shell commands:**

```typescript
git_status: tool({
  description: "Get git status for the project",
  args: {},
  async execute(args, context) {
    return await ctx.$`git status --short`.text()
  },
})
```

**Data processing:**

```typescript
analyze_csv: tool({
  description: "Analyze a CSV file",
  args: {
    path: tool.schema.string(),
  },
  async execute(args, context) {
    const content = await readFile(args.path, "utf-8")
    const lines = content.trim().split("\n")
    const rows = lines.map((line) => line.split(","))

    return `Rows: ${rows.length}\nColumns: ${rows[0].length}`
  },
})
```

## Testing Custom Tools

Test your tools outside of OpenCode first:

```typescript
// test-tools.ts
import { tool } from "@opencode-ai/plugin"

const myTool = tool({
  description: "Test tool",
  args: {
    input: tool.schema.string(),
  },
  async execute(args, context) {
    return `Processed: ${args.input}`
  },
})

// Mock context
const mockContext = {
  sessionID: "test",
  messageID: "test",
  agent: "test",
  abort: new AbortController().signal,
}

// Test it
const result = await myTool.execute({ input: "hello" }, mockContext)
console.log(result) // "Processed: hello"
```

Run with:

```bash
bun test-tools.ts
```

Once verified, add to your plugin.

## Summary

Custom tools extend OpenCode with new functionality:

```typescript
import { type Plugin, tool } from "@opencode-ai/plugin"

export const MyPlugin: Plugin = async (ctx) => ({
  tool: {
    tool_name: tool({
      description: "What this tool does",
      args: {
        param: tool.schema.string().describe("Parameter description"),
      },
      async execute(args, context) {
        // Tool implementation
        return "Result string"
      },
    }),
  },
})
```

Key points:

- Use `tool()` helper for type safety
- Define arguments with `tool.schema` (Zod)
- Return strings (the model sees this output)
- Throw errors for failures
- Keep execution fast
- Test tools independently before integrating

Custom tools provide a simpler alternative to MCP servers for straightforward functionality.
