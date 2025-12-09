# OpenCode Plugins: Plugin Context

The plugin function receives a context object (`PluginInput`) with five fields:

```typescript
type PluginInput = {
  client: ReturnType<typeof createOpencodeClient>
  project: Project
  directory: string
  worktree: string
  $: BunShell
}
```

See [`packages/plugin/src/index.ts:25-31`](https://github.com/sst/opencode/blob/3efc95b/packages/plugin/src/index.ts#L25-L31)

## client: SDK Client

The `client` field provides a typed SDK for the OpenCode API. Use it to interact with sessions, files, configuration, and system state.

### Creating Sessions

```typescript
export const MyPlugin: Plugin = async (ctx) => ({
  event: async ({ event }) => {
    if (event.type === "file.edited" && event.properties.path.endsWith(".test.ts")) {
      // Create a session to run tests
      const session = await ctx.client.session.create({
        body: { agent: "general" },
      })

      await ctx.client.session.prompt({
        path: { id: session.data.id },
        body: {
          message: `Run tests for ${event.properties.path}`,
        },
      })
    }
  },
})
```

### Reading Configuration

```typescript
export const ConfigReader: Plugin = async (ctx) => {
  const config = await ctx.client.config.get()
  console.log("Available commands:", Object.keys(config.data.command || {}))

  return {}
}
```

### Listing Files

```typescript
export const FileScanner: Plugin = async (ctx) => ({
  tool: {
    scan_ts_files: tool({
      description: "Count TypeScript files in project",
      args: {},
      async execute() {
        const files = await ctx.client.file.list({
          query: { path: ctx.directory },
        })

        const tsFiles = files.data.filter((f) => f.name.endsWith(".ts") && !f.name.endsWith(".d.ts"))

        return `Found ${tsFiles.length} TypeScript files`
      },
    }),
  },
})
```

### Subscribing to Events

```typescript
export const EventStream: Plugin = async (ctx) => {
  // Subscribe to events via SSE
  const events = await ctx.client.event.subscribe()

  // Process events (in background)
  ;(async () => {
    for await (const event of events) {
      if (event.type === "session.error") {
        console.error("Session error:", event.properties)
      }
    }
  })()

  return {}
}
```

### Available Client Methods

The SDK client mirrors the OpenCode API. Major namespaces:

| Namespace         | Methods                                                | Purpose                    |
| ----------------- | ------------------------------------------------------ | -------------------------- |
| `client.session`  | create, list, get, update, delete, prompt, abort, fork | Manage chat sessions       |
| `client.file`     | list, read, status                                     | Read files and directories |
| `client.config`   | get, update, providers                                 | Read/modify configuration  |
| `client.command`  | list                                                   | List available commands    |
| `client.project`  | list, current                                          | Project information        |
| `client.mcp`      | status, add, connect, disconnect                       | MCP server management      |
| `client.tool`     | ids, list                                              | Tool introspection         |
| `client.provider` | list, auth                                             | Authentication providers   |
| `client.tui`      | appendPrompt, showToast, executeCommand                | TUI interactions           |

See [`packages/sdk/js/src/gen/sdk.gen.ts`](https://github.com/sst/opencode/blob/3efc95b/packages/sdk/js/src/gen/sdk.gen.ts) for complete API surface.

## project: Project Information

The `project` field contains metadata about the current project:

```typescript
type Project = {
  id: string
  name: string
  path: string
  // ... additional fields
}
```

### Usage

```typescript
export const ProjectPlugin: Plugin = async (ctx) => {
  console.log("Plugin loaded for project:", ctx.project.name)
  console.log("Project path:", ctx.project.path)
  console.log("Project ID:", ctx.project.id)

  return {
    event: async ({ event }) => {
      if (event.type === "session.created") {
        console.log(`New session in ${ctx.project.name}`)
      }
    },
  }
}
```

### Example: Project-Specific Behavior

```typescript
export const ProjectBehavior: Plugin = async (ctx) => ({
  config: async (config) => {
    // Only inject command for specific projects
    if (ctx.project.name === "my-monorepo") {
      config.command = config.command || {}
      config.command["deploy"] = {
        template: "Deploy the application to staging environment",
        description: "Deploy to staging",
      }
    }
  },
})
```

## directory: Current Working Directory

The `directory` field is the current working directory where OpenCode was started.

### Usage

```typescript
export const DirectoryPlugin: Plugin = async (ctx) => ({
  tool: {
    list_files_here: tool({
      description: "List files in current directory",
      args: {},
      async execute() {
        const result = await ctx.$`ls -la`.cwd(ctx.directory).text()
        return result
      },
    }),
  },
})
```

### Example: Directory-Relative Operations

```typescript
export const RelativeOps: Plugin = async (ctx) => ({
  "tool.execute.after": async (input, output) => {
    if (input.tool === "write") {
      const filePath = output.metadata?.filePath
      if (filePath && filePath.startsWith(ctx.directory)) {
        console.log("File written in current directory:", filePath)
      }
    }
  },
})
```

## worktree: Git Root / Project Root

The `worktree` field is the git worktree root. If not in a git repository, it falls back to the directory.

### Usage

```typescript
export const WorktreePlugin: Plugin = async (ctx) => ({
  tool: {
    project_stats: tool({
      description: "Show project statistics from git root",
      args: {},
      async execute() {
        const loc = await ctx.$`find ${ctx.worktree} -name "*.ts" | xargs wc -l`.text()
        const commits = await ctx.$`git -C ${ctx.worktree} rev-list --count HEAD`.text()

        return `Lines of TypeScript: ${loc}\nCommits: ${commits.trim()}`
      },
    }),
  },
})
```

### Example: Worktree-Relative Paths

```typescript
export const RelativePath: Plugin = async (ctx) => {
  const relPath = ctx.directory.replace(ctx.worktree + "/", "")
  console.log("Current directory relative to worktree:", relPath)

  return {}
}
```

### directory vs. worktree

- **directory**: Where OpenCode was started (cwd)
- **worktree**: Git root or fallback to directory

Use `worktree` for project-wide operations. Use `directory` for operations relative to where the user invoked OpenCode.

## $: Bun Shell

The `$` field is Bun's shell API for executing commands. It provides a tagged template for running shell commands with pipes, redirects, and environment variables.

See [`packages/plugin/src/shell.ts`](https://github.com/sst/opencode/blob/3efc95b/packages/plugin/src/shell.ts) for type definitions.

### Basic Usage

```typescript
export const ShellPlugin: Plugin = async (ctx) => ({
  event: async ({ event }) => {
    if (event.type === "session.idle") {
      await ctx.$`echo "Session completed at $(date)"`
    }
  },
})
```

### Reading Output

#### .text()

Read stdout as string:

```typescript
const result = await ctx.$`git status --short`.text()
console.log("Git status:", result)
```

#### .json()

Parse stdout as JSON:

```typescript
const pkg = await ctx.$`cat package.json`.json()
console.log("Package name:", pkg.name)
```

#### .lines()

Iterate stdout line by line:

```typescript
for await (const line of ctx.$`git log --oneline -10`.lines()) {
  console.log("Commit:", line)
}
```

### Execution Options

#### .quiet()

Suppress output to terminal (only buffer):

```typescript
const output = await ctx.$`npm install`.quiet().text()
```

#### .nothrow()

Don't throw on non-zero exit codes:

```typescript
const result = await ctx.$`git diff --exit-code`.nothrow()
if (result.exitCode !== 0) {
  console.log("Repository has changes")
}
```

#### .cwd(path)

Set working directory:

```typescript
await ctx.$`npm test`.cwd(`${ctx.worktree}/packages/core`)
```

#### .env(vars)

Set environment variables:

```typescript
await ctx.$`node script.js`.env({ NODE_ENV: "production" })
```

### Combining Options

Chain multiple options:

```typescript
const output = await ctx.$`npm test`.cwd(`${ctx.worktree}/packages/core`).env({ CI: "true" }).quiet().nothrow().text()
```

### Variable Interpolation

Use `${}` to safely interpolate variables:

```typescript
const file = "package.json"
const content = await ctx.$`cat ${file}`.text()
```

Bun automatically escapes variables to prevent injection attacks.

### Example: File Processing

```typescript
export const FileProcessor: Plugin = async (ctx) => ({
  tool: {
    format_all: tool({
      description: "Format all TypeScript files in project",
      args: {},
      async execute() {
        const files = await ctx.$`find ${ctx.worktree} -name "*.ts" -not -path "*/node_modules/*"`.quiet().text()

        const fileList = files.trim().split("\n").filter(Boolean)

        for (const file of fileList) {
          await ctx.$`prettier --write ${file}`.quiet().nothrow()
        }

        return `Formatted ${fileList.length} files`
      },
    }),
  },
})
```

### Example: Conditional Execution

```typescript
export const ConditionalExec: Plugin = async (ctx) => ({
  event: async ({ event }) => {
    if (event.type === "file.edited") {
      const path = event.properties.path

      // Check if file is staged
      const status = await ctx.$`git status --short ${path}`.quiet().nothrow().text()

      if (status.trim().startsWith("M")) {
        console.log("File modified but not staged:", path)
      }
    }
  },
})
```

### Example: Background Processes

```typescript
export const BackgroundTask: Plugin = async (ctx) => {
  // Start background process (don't await)
  ctx.$`npm run dev`.cwd(ctx.worktree).nothrow()

  return {
    event: async ({ event }) => {
      if (event.type === "session.idle") {
        // Clean up or log
        console.log("Session idle, background process still running")
      }
    },
  }
}
```

### Shell Error Handling

By default, non-zero exit codes throw. Access error details:

```typescript
try {
  await ctx.$`exit 1`
} catch (error) {
  console.log("Exit code:", error.exitCode)
  console.log("Stdout:", error.stdout.toString())
  console.log("Stderr:", error.stderr.toString())
}
```

Or use `.nothrow()` and check `exitCode`:

```typescript
const result = await ctx.$`test -f package.json`.nothrow()
if (result.exitCode === 0) {
  console.log("package.json exists")
} else {
  console.log("package.json not found")
}
```

## Complete Context Example

```typescript
import { type Plugin, tool } from "@opencode-ai/plugin"

export const ComprehensivePlugin: Plugin = async (ctx) => {
  // Access project info at init
  console.log(`Plugin loaded for ${ctx.project.name}`)
  console.log(`Working directory: ${ctx.directory}`)
  console.log(`Git root: ${ctx.worktree}`)

  // Check git status using shell
  const status = await ctx.$`git -C ${ctx.worktree} status --short`.quiet().nothrow().text()

  if (status.trim()) {
    console.log("Repository has uncommitted changes")
  }

  // Get configuration using client
  const config = await ctx.client.config.get()
  const hasTests = "test" in (config.data.command || {})

  return {
    tool: {
      project_info: tool({
        description: "Show comprehensive project information",
        args: {},
        async execute() {
          const info = {
            name: ctx.project.name,
            path: ctx.project.path,
            directory: ctx.directory,
            worktree: ctx.worktree,
            hasTests,
            gitStatus: status.trim() || "clean",
          }

          return JSON.stringify(info, null, 2)
        },
      }),
    },

    event: async ({ event }) => {
      if (event.type === "file.edited") {
        const file = event.properties.path

        // Run linter using shell
        await ctx.$`eslint ${file}`.quiet().nothrow()

        // Update file status using client
        const fileStatus = await ctx.client.file.status()
        console.log("Files changed:", fileStatus.data.length)
      }
    },

    config: async (config) => {
      // Inject project-specific command
      config.command = config.command || {}
      config.command["project-status"] = {
        template: `Show status for ${ctx.project.name}`,
        description: "Project status check",
      }
    },
  }
}
```

## Summary

The plugin context provides:

1. **client**: Type-safe SDK for OpenCode API
   - Manage sessions, files, configuration
   - Subscribe to events
   - Interact with TUI

2. **project**: Current project metadata
   - name, path, id
   - Use for project-specific behavior

3. **directory**: Current working directory
   - Where OpenCode was invoked
   - Use for relative operations

4. **worktree**: Git root or fallback
   - Project-wide operations
   - Repository-relative paths

5. **$**: Bun shell for command execution
   - .text(), .json(), .lines() for output
   - .quiet(), .nothrow(), .cwd(), .env() for control
   - Safe variable interpolation
   - Promise-based with type-safe errors

Use these together to build plugins that integrate deeply with OpenCode and the project environment.
