# Config Schemas

> As of commit [3efc95b](https://github.com/sst/opencode/tree/3efc95b157e05bc3c42554be1b5778f8f1b64cfe)

Complete schemas for command, agent, and MCP configuration.

Source: [`packages/opencode/src/config/config.ts`](https://github.com/sst/opencode/blob/3efc95b/packages/opencode/src/config/config.ts)

## Command Schema

Commands define reusable prompts with placeholders.

```typescript
{
  template: string          // Required - Prompt template with $ARGUMENTS
  description?: string      // Description shown in command list
  agent?: string           // Agent to use (default: current agent)
  model?: string           // Model override (provider/model format)
  subtask?: boolean        // Force subagent invocation
}
```

Source: [`config.ts:375-382`](https://github.com/sst/opencode/blob/3efc95b/packages/opencode/src/config/config.ts#L375-L382)

### Example

```typescript
config.command = config.command || {}
config.command["format-code"] = {
  template: "Format the code in $ARGUMENTS using the project's conventions",
  description: "Format code files",
  agent: "build",
  model: "anthropic/claude-sonnet-4-20250514",
  subtask: false,
}
```

### Field Details

**template** (required):

- Prompt sent to LLM
- Use `$ARGUMENTS` placeholder for user input
- Example: `"Fix the tests in $ARGUMENTS"`

**description** (optional):

- Shown in command picker
- Keep under 80 characters

**agent** (optional):

- Agent name (must exist in config)
- Default: current active agent

**model** (optional):

- Override model for this command
- Format: `"provider/model"`
- Example: `"anthropic/claude-sonnet-4-20250514"`

**subtask** (optional):

- `true`: Always run in new subagent session
- `false` or omitted: Run in current session
- Useful for isolation

## Agent Schema

Agents are specialized configurations with system prompts and tool access.

```typescript
{
  prompt?: string                   // System prompt
  description?: string              // When to use this agent
  mode?: "subagent" | "primary" | "all"
  model?: string                    // Model override
  temperature?: number              // Temperature (0-1)
  top_p?: number                   // Top-p sampling
  color?: string                   // Hex color (e.g., "#FF5733")
  maxSteps?: number                // Max agentic iterations
  tools?: Record<string, boolean>  // Enable/disable tools
  disable?: boolean                // Disable agent
  permission?: {
    edit?: "ask" | "allow" | "deny"
    bash?: "ask" | "allow" | "deny" | Record<string, "ask" | "allow" | "deny">
    webfetch?: "ask" | "allow" | "deny"
    doom_loop?: "ask" | "allow" | "deny"
    external_directory?: "ask" | "allow" | "deny"
  }
}
```

Source: [`config.ts:384-419`](https://github.com/sst/opencode/blob/3efc95b/packages/opencode/src/config/config.ts#L384-L419)

### Example

```typescript
config.agent = config.agent || {}
config.agent["code-reviewer"] = {
  prompt: `You are a code reviewer. Focus on:
- Bugs and logic errors
- Performance issues
- Security vulnerabilities`,
  description: "Reviews code for issues",
  mode: "subagent",
  model: "anthropic/claude-sonnet-4-20250514",
  temperature: 0.3,
  top_p: 0.9,
  color: "#FF5733",
  maxSteps: 10,
  tools: {
    bash: false, // Disable bash
    read: true, // Enable read
    edit: true, // Enable edit
  },
  permission: {
    edit: "ask", // Ask before edits
    bash: "deny", // Never allow bash
    webfetch: "allow", // Always allow webfetch
  },
}
```

### Field Details

**prompt** (optional):

- System prompt for agent
- Define role and capabilities
- Can include examples and guidelines

**description** (optional):

- Shown when selecting agents
- Describe when to use this agent

**mode** (optional):

- `"subagent"`: Available for Task tool invocation
- `"primary"`: Available as main agent
- `"all"`: Available in both contexts
- Default: `"all"`

**model** (optional):

- Override default model
- Format: `"provider/model"`

**temperature** (optional):

- Randomness (0-1)
- Lower = more deterministic
- Higher = more creative

**top_p** (optional):

- Nucleus sampling (0-1)
- Alternative to temperature

**color** (optional):

- Hex color for UI
- Format: `"#RRGGBB"`
- Example: `"#FF5733"`

**maxSteps** (optional):

- Max agentic iterations before text-only response
- Prevents infinite loops
- Default: system default

**tools** (optional):

- Enable/disable specific tools
- Format: `{ toolName: boolean }`
- Example: `{ bash: false, read: true }`
- Only affects this agent

**disable** (optional):

- Disable agent without removing config
- Useful for debugging

**permission** (optional):

- Control permission prompts per action type
- `"ask"`: Prompt user each time
- `"allow"`: Auto-approve
- `"deny"`: Auto-deny
- `bash` can be string or object for per-command rules

**permission.bash object format**:

```typescript
{
  bash: {
    "npm install": "allow",  // Always allow npm install
    "rm -rf": "deny",        // Never allow rm -rf
    "*": "ask",              // Ask for everything else
  }
}
```

## MCP Server Schemas

### Local MCP Server

Run MCP server as local process.

```typescript
{
  type: "local"
  command: string[]                  // Command and args
  environment?: Record<string, string>  // Environment variables
  enabled?: boolean                  // Enable on startup (default: true)
  timeout?: number                   // Timeout in ms (default: 5000)
}
```

Source: [`config.ts:305-326`](https://github.com/sst/opencode/blob/3efc95b/packages/opencode/src/config/config.ts#L305-L326)

**Example**:

```typescript
config.mcp = config.mcp || {}
config.mcp["my-server"] = {
  type: "local",
  command: ["node", "/path/to/server.js"],
  environment: {
    API_KEY: "{env:MY_API_KEY}",
    DEBUG: "true",
  },
  enabled: true,
  timeout: 10000,
}
```

**Field Details**:

**type**: Always `"local"`

**command** (required):

- Array of command and arguments
- First element: executable
- Rest: arguments
- Example: `["node", "server.js", "--port", "3000"]`

**environment** (optional):

- Environment variables for process
- Can use `{env:VAR}` to reference system env vars
- Example: `{ API_KEY: "{env:API_KEY}" }`

**enabled** (optional):

- `true`: Start on OpenCode startup (default)
- `false`: Require manual connection

**timeout** (optional):

- Milliseconds to wait for server tools
- Default: 5000 (5 seconds)

### Remote MCP Server

Connect to MCP server over HTTP.

```typescript
{
  type: "remote"
  url: string                        // Server URL
  enabled?: boolean                  // Enable on startup (default: true)
  headers?: Record<string, string>   // Request headers
  oauth?: McpOAuthConfig | false     // OAuth config or disable
  timeout?: number                   // Timeout in ms (default: 5000)
}
```

Source: [`config.ts:343-367`](https://github.com/sst/opencode/blob/3efc95b/packages/opencode/src/config/config.ts#L343-L367)

**Example**:

```typescript
config.mcp = config.mcp || {}
config.mcp["remote-server"] = {
  type: "remote",
  url: "https://mcp.example.com",
  headers: {
    Authorization: "Bearer {env:MCP_TOKEN}",
    "X-Custom-Header": "value",
  },
  oauth: {
    clientId: "my-client-id",
    clientSecret: "{env:OAUTH_SECRET}",
    scope: "read write",
  },
  enabled: true,
  timeout: 10000,
}
```

**Field Details**:

**type**: Always `"remote"`

**url** (required):

- Full URL to MCP server
- Must be HTTP or HTTPS
- Example: `"https://mcp.example.com/api"`

**enabled** (optional):

- Same as local MCP

**headers** (optional):

- HTTP headers for requests
- Can use `{env:VAR}` for secrets
- Example: `{ "Authorization": "Bearer {env:TOKEN}" }`

**oauth** (optional):

- OAuth configuration (see below)
- Set to `false` to disable auto-detection
- Omit to enable auto-detection

**timeout** (optional):

- Same as local MCP

### MCP OAuth Config

OAuth authentication for remote MCP servers.

```typescript
{
  clientId?: string      // OAuth client ID
  clientSecret?: string  // OAuth client secret
  scope?: string         // OAuth scopes
}
```

Source: [`config.ts:328-340`](https://github.com/sst/opencode/blob/3efc95b/packages/opencode/src/config/config.ts#L328-L340)

**Example**:

```typescript
oauth: {
  clientId: "my-app",
  clientSecret: "{env:OAUTH_SECRET}",
  scope: "read write admin",
}
```

**Field Details**:

**clientId** (optional):

- OAuth client identifier
- If omitted, attempts dynamic client registration (RFC 7591)

**clientSecret** (optional):

- OAuth client secret
- Use `{env:VAR}` for secrets
- Required by some servers

**scope** (optional):

- Space-separated OAuth scopes
- Example: `"read write"`
- Depends on server requirements

## Environment Variable Substitution

All config fields support environment variable substitution:

```typescript
{
  command: ["node", "{env:SERVER_PATH}"],
  environment: {
    API_KEY: "{env:MY_API_KEY}",
    PORT: "{env:PORT}",
  },
  headers: {
    "Authorization": "Bearer {env:TOKEN}",
  },
}
```

Variables are replaced at runtime from `process.env`.

## File Reference Substitution

Config files support file references:

```typescript
{
  prompt: "{file:./prompts/reviewer.md}",
  template: "{file:./templates/format.txt}",
}
```

Relative paths resolved from config file location.

Source: [`config.ts:757-797`](https://github.com/sst/opencode/blob/3efc95b/packages/opencode/src/config/config.ts#L757-L797)

## Permission Values

Three permission levels:

```typescript
type Permission = "ask" | "allow" | "deny"
```

- `"ask"`: Prompt user for confirmation (default)
- `"allow"`: Auto-approve without prompt
- `"deny"`: Auto-deny without prompt

Source: [`config.ts:372-373`](https://github.com/sst/opencode/blob/3efc95b/packages/opencode/src/config/config.ts#L372-L373)

## Model Format

Model strings use `provider/model` format:

```
anthropic/claude-sonnet-4-20250514
openai/gpt-4-turbo
google/gemini-pro
```

Get available models:

```typescript
const providers = await ctx.client.config.providers()
for (const provider of providers) {
  console.log(provider.id, Object.keys(provider.models))
}
```

## Complete Config Example

```typescript
export const MyPlugin: Plugin = async (ctx) => ({
  config: async (config) => {
    // Commands
    config.command = config.command || {}
    config.command["review"] = {
      template: "Review the code in $ARGUMENTS for issues",
      description: "Code review",
      agent: "reviewer",
    }
    config.command["test"] = {
      template: "Write tests for $ARGUMENTS",
      description: "Generate tests",
      agent: "build",
      model: "anthropic/claude-sonnet-4-20250514",
    }

    // Agents
    config.agent = config.agent || {}
    config.agent["reviewer"] = {
      prompt: "You are a code reviewer...",
      description: "Reviews code",
      mode: "subagent",
      temperature: 0.3,
      color: "#FF5733",
      tools: {
        bash: false,
        edit: false,
      },
      permission: {
        edit: "deny",
      },
    }
    config.agent["tester"] = {
      prompt: "You write comprehensive tests...",
      description: "Writes tests",
      mode: "subagent",
      maxSteps: 15,
      tools: {
        bash: true,
        write: true,
      },
      permission: {
        bash: {
          "npm test": "allow",
          "npm install": "allow",
          "*": "ask",
        },
      },
    }

    // MCP Servers
    config.mcp = config.mcp || {}
    config.mcp["local-analyzer"] = {
      type: "local",
      command: ["node", path.join(ctx.directory, "analyzer.js")],
      environment: {
        API_KEY: "{env:ANALYZER_KEY}",
      },
    }
    config.mcp["remote-api"] = {
      type: "remote",
      url: "https://api.example.com/mcp",
      headers: {
        Authorization: "Bearer {env:API_TOKEN}",
      },
      oauth: {
        clientId: "my-plugin",
        scope: "read write",
      },
    }
  },
})
```

## See Also

- `04-config-hook.md` - Config injection patterns and best practices
- `09-claude-code-migration.md` - Migrating from Claude Code
- `07-sdk-client.md` - Reading config via SDK
