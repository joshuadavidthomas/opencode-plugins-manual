# Config Hook: Bundling Commands, Agents, and MCP Servers

The `config` hook modifies configuration at runtime. Use it to bundle commands, agents, and MCP servers with your plugin.

> As of commit 3efc95b15

## Why Use the Config Hook

Users typically define commands in `.opencode/command/`, agents in `.opencode/agent/`, and MCP servers in `opencode.json`. The config hook lets your plugin inject these components programmatically.

**Benefits:**

- Ship complete plugins with bundled functionality
- No separate configuration files required
- Modify existing configuration at runtime
- Distribute reusable components as packages

## Basic Pattern

```typescript
import type { Plugin } from "@opencode-ai/plugin"

export const MyPlugin: Plugin = async (ctx) => ({
  config: async (config) => {
    // Initialize objects if needed
    config.command = config.command || {}
    config.agent = config.agent || {}
    config.mcp = config.mcp || {}

    // Inject your components
    config.command["my-command"] = {
      /* ... */
    }
    config.agent["my-agent"] = {
      /* ... */
    }
    config.mcp["my-server"] = {
      /* ... */
    }
  },
})
```

The config object is mutable. Your hook modifies it directly.

## Injecting Commands

Commands provide slash-commands that users can invoke from the prompt.

**Schema** (packages/opencode/src/config/config.ts:375-382):

```typescript
{
  template: string              // Required: prompt template
  description?: string          // Optional: what this command does
  agent?: string                // Optional: which agent to use
  model?: string                // Optional: override model
  subtask?: boolean             // Optional: force subagent invocation
}
```

**Template variables:**

- `$ARGUMENTS` — Everything after the command name
- `$FILE` — Currently selected file (if any)

**Example:**

```typescript
export const MyPlugin: Plugin = async (ctx) => ({
  config: async (config) => {
    config.command = config.command || {}

    config.command["explain"] = {
      template: "Explain this code: $ARGUMENTS",
      description: "Explain code in simple terms",
    }

    config.command["refactor"] = {
      template: "Refactor the code in $FILE to be more maintainable",
      description: "Refactor selected file",
      agent: "code-expert", // Use specific agent
    }

    config.command["analyze"] = {
      template: "Analyze the architecture of $ARGUMENTS",
      description: "Deep architecture analysis",
      subtask: true, // Force subagent invocation
    }
  },
})
```

**Verification:**

Commands appear in the `/` command menu immediately after plugin load. Test with `/your-command`.

## Injecting Agents

Agents are specialized personas that handle specific types of tasks.

**Schema** (packages/opencode/src/config/config.ts:384-419):

```typescript
{
  prompt?: string                            // System prompt
  description?: string                       // When to use this agent
  mode?: "subagent" | "primary" | "all"     // Where agent is available
  model?: string                             // e.g., "anthropic/claude-sonnet-4"
  temperature?: number                       // 0-2
  top_p?: number                             // 0-1
  tools?: Record<string, boolean>            // Enable/disable tools
  maxSteps?: number                          // Max iterations
  color?: string                             // Hex color (e.g., "#FF5733")
  permission?: {
    edit?: "ask" | "allow" | "deny"
    bash?: "ask" | "allow" | "deny" | Record<string, "ask" | "allow" | "deny">
    webfetch?: "ask" | "allow" | "deny"
    doom_loop?: "ask" | "allow" | "deny"
    external_directory?: "ask" | "allow" | "deny"
  }
}
```

**Mode values:**

- `"subagent"` — Available only via Task tool (default)
- `"primary"` — Available as main chat agent
- `"all"` — Available everywhere

**Example:**

```typescript
export const MyPlugin: Plugin = async (ctx) => ({
  config: async (config) => {
    config.agent = config.agent || {}

    config.agent["code-reviewer"] = {
      prompt:
        "You are an expert code reviewer. Focus on:\n" +
        "- Code quality and maintainability\n" +
        "- Potential bugs and edge cases\n" +
        "- Performance implications\n" +
        "Always provide specific, actionable feedback.",
      description: "Use when reviewing code changes for quality and bugs",
      mode: "subagent",
      temperature: 0.3, // More focused
      color: "#FF5733",
      permission: {
        edit: "deny", // Reviewers only read
        bash: "deny",
        webfetch: "allow", // Can fetch docs
      },
    }

    config.agent["creative-writer"] = {
      prompt: "You are a creative writing assistant.",
      description: "Use for creative writing tasks",
      mode: "primary", // Available as main agent
      temperature: 1.5, // More creative
      tools: {
        bash: false, // Disable bash tool
        read: true, // Keep read tool
      },
    }
  },
})
```

**Verification:**

- Subagents: Invoke via Task tool — `task(subagent_type="your-agent", ...)`
- Primary agents: Select via `--agent your-agent` flag or in TUI

See `.opencode/plugin/test-injection.ts` for a verified working example.

## Injecting MCP Servers

MCP (Model Context Protocol) servers provide additional tools to the model.

**Local MCP Schema** (packages/opencode/src/config/config.ts:305-326):

```typescript
{
  type: "local"
  command: string[]                         // Command and args
  environment?: Record<string, string>      // Environment variables
  enabled?: boolean                         // Enable on startup (default: true)
  timeout?: number                          // Timeout in ms (default: 5000)
}
```

**Remote MCP Schema** (packages/opencode/src/config/config.ts:343-367):

```typescript
{
  type: "remote"
  url: string                               // Server URL
  enabled?: boolean                         // Enable on startup (default: true)
  headers?: Record<string, string>          // HTTP headers
  oauth?: {                                 // OAuth config
    clientId?: string
    clientSecret?: string
    scope?: string
  } | false                                 // Set false to disable auto-detection
  timeout?: number                          // Timeout in ms (default: 5000)
}
```

**Example:**

```typescript
export const MyPlugin: Plugin = async (ctx) => ({
  config: async (config) => {
    config.mcp = config.mcp || {}

    // Local MCP server
    config.mcp["my-local-server"] = {
      type: "local",
      command: ["node", "/path/to/mcp-server.js"],
      environment: {
        API_KEY: process.env.MY_API_KEY || "",
      },
      timeout: 10000,
    }

    // Remote MCP server
    config.mcp["my-remote-server"] = {
      type: "remote",
      url: "https://api.example.com/mcp",
      headers: {
        Authorization: `Bearer ${process.env.API_TOKEN}`,
      },
      oauth: {
        clientId: "your-client-id",
        scope: "read write",
      },
    }
  },
})
```

**MCP server paths:**

Use `ctx.directory` to reference files relative to your plugin:

```typescript
export const MyPlugin: Plugin = async (ctx) => ({
  config: async (config) => {
    config.mcp = config.mcp || {}
    config.mcp["bundled-server"] = {
      type: "local",
      command: ["node", `${ctx.directory}/mcp-server.js`],
    }
  },
})
```

**Verification:**

List available MCP servers:

```bash
opencode mcp list
```

Your injected servers appear in the output if enabled.

## Complete Example

This plugin bundles a command, agent, and MCP server:

```typescript
import type { Plugin } from "@opencode-ai/plugin"

export const DataAnalysisPlugin: Plugin = async (ctx) => ({
  config: async (config) => {
    // Initialize
    config.command = config.command || {}
    config.agent = config.agent || {}
    config.mcp = config.mcp || {}

    // Bundle command
    config.command["analyze-data"] = {
      template: "Analyze the dataset at $ARGUMENTS and provide insights",
      description: "Perform data analysis on a dataset",
      agent: "data-analyst",
    }

    // Bundle agent
    config.agent["data-analyst"] = {
      prompt:
        "You are a data analyst expert. When analyzing data:\n" +
        "1. Identify patterns and trends\n" +
        "2. Look for anomalies\n" +
        "3. Suggest visualizations\n" +
        "4. Provide statistical summaries",
      description: "Use for data analysis tasks",
      mode: "subagent",
      temperature: 0.3,
      color: "#4A90E2",
      tools: {
        bash: false, // Disable bash for safety
      },
    }

    // Bundle MCP server (provides data analysis tools)
    config.mcp["data-tools"] = {
      type: "local",
      command: ["python", `${ctx.directory}/data-mcp-server.py`],
      environment: {
        PYTHONPATH: ctx.directory,
      },
    }
  },
})
```

Users install this plugin and get:

- `/analyze-data` command
- `data-analyst` agent
- Custom data analysis tools via MCP

All from one package.

## Verified Working Example

The test plugin at `.opencode/plugin/test-injection.ts` demonstrates config hook injection:

```typescript
import type { Plugin } from "@opencode-ai/plugin"

export const TestInjectionPlugin: Plugin = async (ctx) => ({
  config: async (config) => {
    // Inject command
    config.command = config.command || {}
    config.command["plugin-test"] = {
      template: "Say 'Hello from a plugin-injected command!'",
      description: "Test command injected via plugin config hook",
    }

    // Inject agent
    config.agent = config.agent || {}
    config.agent["plugin-test-agent"] = {
      prompt: "You are a test agent injected via plugin.",
      description: "Test agent injected via plugin config hook",
      mode: "subagent",
    }
  },
})
```

Verified on 2025-12-09:

1. Plugin loaded successfully
2. `/plugin-test` command appeared in menu
3. Command executed correctly
4. Agent invoked via Task tool

## Implementation Notes

**Timing:**

The config hook runs once during plugin initialization, before the first session starts.

**Mutation:**

The hook receives a mutable config object. Changes persist for the entire OpenCode session.

**Conflicts:**

If multiple plugins inject the same key, the last plugin wins. Namespace your keys to avoid collisions:

```typescript
config.command["myplugin-analyze"] = {
  /* ... */
}
```

**Debugging:**

Add logging to verify your hook runs:

```typescript
config: async (config) => {
  console.log("[myplugin] Config hook called")
  config.command = config.command || {}
  config.command["test"] = {
    /* ... */
  }
  console.log("[myplugin] Injected command 'test'")
}
```

Check the console output when OpenCode starts.

## Alternative: File-Based Configuration

You can still use file-based configuration instead of config hooks:

- **Commands:** `.opencode/command/my-command.md` or `~/.config/opencode/command/my-command.md`
- **Agents:** `.opencode/agent/my-agent.md` or `~/.config/opencode/agent/my-agent.md`
- **MCP:** `opencode.json` in project root or `~/.config/opencode/opencode.json`

File-based configuration loads before plugins, so config hooks can modify or override file-based settings.

**Use config hooks when:**

- Building reusable plugin packages
- Components require programmatic logic
- Users want zero-config installation

**Use files when:**

- Users need to customize easily
- Configuration is project-specific
- No programmatic logic required

## Summary

The config hook injects commands, agents, and MCP servers at runtime:

```typescript
export const MyPlugin: Plugin = async (ctx) => ({
  config: async (config) => {
    config.command = config.command || {}
    config.command["my-cmd"] = { template: "...", description: "..." }

    config.agent = config.agent || {}
    config.agent["my-agent"] = { prompt: "...", mode: "subagent" }

    config.mcp = config.mcp || {}
    config.mcp["my-server"] = { type: "remote", url: "..." }
  },
})
```

This lets plugins bundle complete functionality for distribution as packages.
