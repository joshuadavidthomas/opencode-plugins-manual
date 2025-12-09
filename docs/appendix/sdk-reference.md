# SDK Client Reference

> As of commit [3efc95b](https://github.com/sst/opencode/tree/3efc95b157e05bc3c42554be1b5778f8f1b64cfe)

Complete reference for `ctx.client` methods.

Source: `packages/sdk/js/src/gen/sdk.gen.ts`

## Client Structure

```typescript
const client = new OpencodeClient()

client.global // Global events
client.project // Project management
client.session // Session lifecycle and prompts
client.config // Configuration
client.command // Commands
client.agent // Agent information  (via app.agents)
client.tool // Tool management
client.provider // Provider and model management
client.mcp // MCP server management
client.lsp // LSP server status
client.formatter // Formatter status
client.file // File operations
client.find // Search operations
client.path // Path information
client.vcs // Version control info
client.pty // PTY sessions
client.tui // TUI control
client.auth // Authentication
client.instance // Instance management
client.event // Event subscription
```

## global

### global.event()

Subscribe to all global events.

```typescript
const stream = await client.global.event()
for await (const event of stream) {
  console.log(event.directory, event.payload)
}
```

**Returns**: Server-Sent Events stream of `GlobalEvent`

Source: `sdk.gen.ts:236-242`

## project

### project.list(options?)

List all projects.

```typescript
const projects = await client.project.list()
const projects = await client.project.list({ query: { directory: "/path" } })
```

**Parameters**:

- `query.directory?: string` - Filter by directory

**Returns**: `Array<Project>`

```typescript
Project = {
  id: string
  worktree: string
  vcsDir?: string
  vcs?: "git"
  time: { created: number; initialized?: number }
}
```

Source: `sdk.gen.ts:248-253`

### project.current(options?)

Get current project.

```typescript
const project = await client.project.current()
```

**Parameters**:

- `query.directory?: string` - Instance directory

**Returns**: `Project`

Source: `sdk.gen.ts:258-263`

## session

### session.list(options?)

List all sessions.

```typescript
const sessions = await client.session.list()
const sessions = await client.session.list({ query: { directory: "/path" } })
```

**Returns**: `Array<Session>`

Source: `sdk.gen.ts:435-440`

### session.create(options?)

Create new session.

```typescript
const session = await client.session.create()
const session = await client.session.create({
  body: { parentID: "parent-id" },
})
```

**Returns**: `Session`

Source: `sdk.gen.ts:445-454`

### session.get(options)

Get session info.

```typescript
const session = await client.session.get({ path: { id: "session-id" } })
```

**Parameters**:

- `path.id: string` - Session ID

**Returns**: `Session`

Source: `sdk.gen.ts:479-484`

### session.update(options)

Update session properties.

```typescript
await client.session.update({
  path: { id: "session-id" },
  body: { title: "New Title" },
})
```

**Parameters**:

- `path.id: string` - Session ID
- `body.title?: string` - New title

**Returns**: `Session`

Source: `sdk.gen.ts:489-498`

### session.delete(options)

Delete session.

```typescript
await client.session.delete({ path: { id: "session-id" } })
```

**Returns**: `boolean`

Source: `sdk.gen.ts:469-474`

### session.status(options?)

Get session status.

```typescript
const status = await client.session.status()
```

**Returns**: `SessionStatus` (idle | busy | retry)

Source: `sdk.gen.ts:459-464`

### session.children(options)

Get session's child sessions.

```typescript
const children = await client.session.children({ path: { id: "session-id" } })
```

**Returns**: `Array<Session>`

Source: `sdk.gen.ts:503-508`

### session.todo(options)

Get session's todo list.

```typescript
const todos = await client.session.todo({ path: { id: "session-id" } })
```

**Returns**: `Array<Todo>`

```typescript
Todo = {
  content: string
  status: "pending" | "in_progress" | "completed" | "cancelled"
  priority: "high" | "medium" | "low"
  id: string
}
```

Source: `sdk.gen.ts:513-518`

### session.messages(options)

List messages in session.

```typescript
const messages = await client.session.messages({ path: { id: "session-id" } })
```

**Returns**: `Array<Message>` (UserMessage | AssistantMessage)

Source: `sdk.gen.ts:605-610`

### session.message(options)

Get specific message.

```typescript
const message = await client.session.message({
  path: { id: "session-id", messageID: "msg-id" },
})
```

**Returns**: `Message`

Source: `sdk.gen.ts:629-634`

### session.prompt(options)

Send message to session (synchronous).

```typescript
const result = await client.session.prompt({
  path: { id: "session-id" },
  body: {
    text: "Hello",
    agent: "general",
    parts: [{ type: "text", text: "Hello" }],
  },
})
```

**Parameters**:

- `path.id: string` - Session ID
- `body.text?: string` - Message text
- `body.agent?: string` - Agent name
- `body.parts?: Array<TextPartInput | FilePartInput | AgentPartInput | SubtaskPartInput>`
- `body.summarize?: boolean` - Generate summary

**Returns**: `Message`

Source: `sdk.gen.ts:615-624`

### session.promptAsync(options)

Send message to session (asynchronous).

```typescript
await client.session.promptAsync({
  path: { id: "session-id" },
  body: { text: "Hello" },
})
```

Same parameters as `session.prompt()` but returns immediately.

**Returns**: `boolean`

Source: `sdk.gen.ts:639-648`

### session.command(options)

Execute command in session.

```typescript
await client.session.command({
  path: { id: "session-id" },
  body: { name: "format-code", arguments: "src/" },
})
```

**Parameters**:

- `path.id: string` - Session ID
- `body.name: string` - Command name
- `body.arguments?: string` - Command arguments

**Returns**: `boolean`

Source: `sdk.gen.ts:653-662`

### session.shell(options)

Run shell command.

```typescript
const output = await client.session.shell({
  path: { id: "session-id" },
  body: { command: "ls -la" },
})
```

**Parameters**:

- `path.id: string` - Session ID
- `body.command: string` - Shell command

**Returns**: `string`

Source: `sdk.gen.ts:667-676`

### session.fork(options)

Fork session at specific message.

```typescript
const newSession = await client.session.fork({
  path: { id: "session-id" },
  body: { messageID: "msg-id" },
})
```

**Returns**: `Session`

Source: `sdk.gen.ts:537-546`

### session.abort(options)

Abort running session.

```typescript
await client.session.abort({ path: { id: "session-id" } })
```

**Returns**: `boolean`

Source: `sdk.gen.ts:551-556`

### session.share(options)

Share session.

```typescript
const result = await client.session.share({ path: { id: "session-id" } })
```

**Returns**: `{ url: string }`

Source: `sdk.gen.ts:571-576`

### session.unshare(options)

Unshare session.

```typescript
await client.session.unshare({ path: { id: "session-id" } })
```

**Returns**: `boolean`

Source: `sdk.gen.ts:561-566`

### session.diff(options)

Get session diff.

```typescript
const diff = await client.session.diff({ path: { id: "session-id" } })
```

**Returns**: `Array<FileDiff>`

Source: `sdk.gen.ts:581-586`

### session.summarize(options)

Generate session summary.

```typescript
await client.session.summarize({ path: { id: "session-id" } })
```

**Returns**: `boolean`

Source: `sdk.gen.ts:591-600`

### session.revert(options)

Revert to specific message.

```typescript
await client.session.revert({
  path: { id: "session-id" },
  body: { messageID: "msg-id", partID?: "part-id" }
})
```

**Returns**: `boolean`

Source: `sdk.gen.ts:681-690`

### session.unrevert(options)

Restore all reverted messages.

```typescript
await client.session.unrevert({ path: { id: "session-id" } })
```

**Returns**: `boolean`

Source: `sdk.gen.ts:695-700`

### session.init(options)

Initialize session with AGENTS.md analysis.

```typescript
await client.session.init({
  path: { id: "session-id" },
  body: { overwrite: false },
})
```

**Returns**: `boolean`

Source: `sdk.gen.ts:523-532`

## config

### config.get(options?)

Get config.

```typescript
const config = await client.config.get()
```

**Returns**: `Config` (see `appendix/config-schemas.md`)

Source: `sdk.gen.ts:341-346`

### config.update(options?)

Update config.

```typescript
await client.config.update({
  body: {
    model: "anthropic/claude-sonnet-4-20250514",
    theme: "catppuccin-mocha",
  },
})
```

**Returns**: `Config`

Source: `sdk.gen.ts:351-360`

### config.providers(options?)

List all providers.

```typescript
const providers = await client.config.providers()
```

**Returns**: `Array<Provider>`

Source: `sdk.gen.ts:365-370`

## command

### command.list(options?)

List all commands.

```typescript
const commands = await client.command.list()
```

**Returns**: `Array<Command>`

```typescript
Command = {
  name: string
  description?: string
  agent?: string
  model?: string
  template: string
  subtask?: boolean
}
```

Source: `sdk.gen.ts:707-712`

## app

### app.agents(options?)

List all agents.

```typescript
const agents = await client.app.agents()
```

**Returns**: `Array<Agent>`

Source: `sdk.gen.ts:858-863`

### app.log(options?)

Write log entry.

```typescript
await client.app.log({
  body: { level: "info", message: "Log message" },
})
```

**Returns**: `boolean`

Source: `sdk.gen.ts:844-853`

## tool

### tool.ids(options?)

List all tool IDs.

```typescript
const toolIds = await client.tool.ids()
```

**Returns**: `Array<string>`

Source: `sdk.gen.ts:377-382`

### tool.list(options)

List tools for provider/model.

```typescript
const tools = await client.tool.list({
  query: { provider: "anthropic", model: "claude-sonnet-4-20250514" },
})
```

**Returns**: `Array<ToolListItem>`

```typescript
ToolListItem = {
  id: string
  description: string
  parameters: unknown  // JSON schema
}
```

Source: `sdk.gen.ts:387-392`

## provider

### provider.list(options?)

List all providers.

```typescript
const providers = await client.provider.list()
```

**Returns**: `Array<Provider>`

Source: `sdk.gen.ts:757-762`

### provider.auth(options?)

Get provider auth methods.

```typescript
const methods = await client.provider.auth()
```

**Returns**: `Array<{ id: string; methods: Array<ProviderAuthMethod> }>`

Source: `sdk.gen.ts:767-772`

### provider.oauth.authorize(options)

Start OAuth flow.

```typescript
const auth = await client.provider.oauth.authorize({
  path: { id: "github" },
  body: { enterpriseUrl: "https://github.mycompany.com" },
})
```

**Returns**: `ProviderAuthAuthorization`

Source: `sdk.gen.ts:719-732`

### provider.oauth.callback(options)

Complete OAuth flow.

```typescript
await client.provider.oauth.callback({
  path: { id: "github" },
  body: { code: "auth-code" },
})
```

**Returns**: `boolean`

Source: `sdk.gen.ts:737-750`

## mcp

### mcp.status(options?)

Get MCP server statuses.

```typescript
const statuses = await client.mcp.status()
```

**Returns**: `Record<string, McpStatus>`

Source: `sdk.gen.ts:932-937`

### mcp.add(options?)

Add MCP server dynamically.

```typescript
await client.mcp.add({
  body: {
    name: "my-server",
    config: {
      type: "local",
      command: ["node", "server.js"],
    },
  },
})
```

**Returns**: `boolean`

Source: `sdk.gen.ts:942-951`

### mcp.connect(options)

Connect MCP server.

```typescript
await client.mcp.connect({ path: { name: "my-server" } })
```

**Returns**: `boolean`

Source: `sdk.gen.ts:956-961`

### mcp.disconnect(options)

Disconnect MCP server.

```typescript
await client.mcp.disconnect({ path: { name: "my-server" } })
```

**Returns**: `boolean`

Source: `sdk.gen.ts:966-971`

### mcp.auth.remove(options)

Remove OAuth credentials.

```typescript
await client.mcp.auth.remove({ path: { name: "my-server" } })
```

**Returns**: `boolean`

Source: `sdk.gen.ts:870-875`

### mcp.auth.start(options)

Start OAuth flow.

```typescript
const url = await client.mcp.auth.start({ path: { name: "my-server" } })
```

**Returns**: `{ url: string }`

Source: `sdk.gen.ts:880-885`

### mcp.auth.callback(options)

Complete OAuth with code.

```typescript
await client.mcp.auth.callback({
  path: { name: "my-server" },
  body: { code: "auth-code" },
})
```

**Returns**: `boolean`

Source: `sdk.gen.ts:890-899`

### mcp.auth.authenticate(options)

Full OAuth flow (opens browser).

```typescript
await client.mcp.auth.authenticate({ path: { name: "my-server" } })
```

**Returns**: `boolean`

Source: `sdk.gen.ts:904-911`

## lsp

### lsp.status(options?)

Get LSP server statuses.

```typescript
const statuses = await client.lsp.status()
```

**Returns**: `Array<LspStatus>`

```typescript
LspStatus = {
  id: string
  name: string
  root: string
  status: "connected" | "error"
}
```

Source: `sdk.gen.ts:980-985`

## formatter

### formatter.status(options?)

Get formatter statuses.

```typescript
const statuses = await client.formatter.status()
```

**Returns**: `Array<FormatterStatus>`

```typescript
FormatterStatus = {
  name: string
  extensions: Array<string>
  enabled: boolean
}
```

Source: `sdk.gen.ts:992-997`

## file

### file.list(options)

List files and directories.

```typescript
const files = await client.file.list({
  query: { path: "src/", recursive: true },
})
```

**Returns**: `Array<FileNode>`

```typescript
FileNode = {
  name: string
  path: string
  absolute: string
  type: "file" | "directory"
  ignored: boolean
}
```

Source: `sdk.gen.ts:812-817`

### file.read(options)

Read file contents.

```typescript
const content = await client.file.read({
  query: { path: "package.json" },
})
```

**Returns**: `FileContent`

```typescript
FileContent = {
  type: "text"
  content: string
  diff?: string
  patch?: GitPatch
  encoding?: "base64"
  mimeType?: string
}
```

Source: `sdk.gen.ts:822-827`

### file.status(options?)

Get file statuses.

```typescript
const statuses = await client.file.status()
```

**Returns**: `Array<File>`

```typescript
File = {
  path: string
  added: number
  removed: number
  status: "added" | "deleted" | "modified"
}
```

Source: `sdk.gen.ts:832-837`

## find

### find.text(options)

Find text in files.

```typescript
const results = await client.find.text({
  query: { pattern: "TODO", path: "src/" },
})
```

**Returns**: `Array<string>` (file paths)

Source: `sdk.gen.ts:780-785`

### find.files(options)

Find files by pattern.

```typescript
const files = await client.find.files({
  query: { pattern: "*.ts" },
})
```

**Returns**: `Array<string>` (file paths)

Source: `sdk.gen.ts:790-795`

### find.symbols(options)

Find workspace symbols.

```typescript
const symbols = await client.find.symbols({
  query: { query: "MyClass" },
})
```

**Returns**: `Array<Symbol>`

```typescript
Symbol = {
  name: string
  kind: number
  location: { uri: string; range: Range }
}
```

Source: `sdk.gen.ts:800-805`

## path

### path.get(options?)

Get current paths.

```typescript
const paths = await client.path.get()
```

**Returns**: `Path`

```typescript
Path = {
  state: string
  config: string
  worktree: string
  directory: string
}
```

Source: `sdk.gen.ts:411-416`

## vcs

### vcs.get(options?)

Get VCS info.

```typescript
const vcs = await client.vcs.get()
```

**Returns**: `VcsInfo`

```typescript
VcsInfo = {
  branch: string,
}
```

Source: `sdk.gen.ts:423-428`

## pty

### pty.list(options?)

List PTY sessions.

```typescript
const sessions = await client.pty.list()
```

**Returns**: `Array<Pty>`

Source: `sdk.gen.ts:271-276`

### pty.create(options?)

Create PTY session.

```typescript
const pty = await client.pty.create({
  body: {
    command: "bash",
    args: [],
    cwd: "/home/user",
    title: "Shell",
  },
})
```

**Returns**: `Pty`

Source: `sdk.gen.ts:281-290`

### pty.get(options)

Get PTY info.

```typescript
const pty = await client.pty.get({ path: { id: "pty-id" } })
```

**Returns**: `Pty`

Source: `sdk.gen.ts:305-310`

### pty.update(options)

Update PTY properties.

```typescript
await client.pty.update({
  path: { id: "pty-id" },
  body: { title: "New Title" },
})
```

**Returns**: `Pty`

Source: `sdk.gen.ts:315-324`

### pty.remove(options)

Remove PTY session.

```typescript
await client.pty.remove({ path: { id: "pty-id" } })
```

**Returns**: `boolean`

Source: `sdk.gen.ts:295-300`

### pty.connect(options)

Connect to PTY (WebSocket).

```typescript
await client.pty.connect({ path: { id: "pty-id" } })
```

**Returns**: `boolean`

Source: `sdk.gen.ts:329-334`

## tui

### tui.appendPrompt(options?)

Append text to TUI prompt.

```typescript
await client.tui.appendPrompt({ body: { text: "Hello" } })
```

**Returns**: `boolean`

Source: `sdk.gen.ts:1030-1039`

### tui.openHelp(options?)

Open help dialog.

```typescript
await client.tui.openHelp()
```

**Returns**: `boolean`

Source: `sdk.gen.ts:1044-1049`

### tui.openSessions(options?)

Open sessions dialog.

```typescript
await client.tui.openSessions()
```

**Returns**: `boolean`

Source: `sdk.gen.ts:1054-1059`

### tui.openThemes(options?)

Open themes dialog.

```typescript
await client.tui.openThemes()
```

**Returns**: `boolean`

Source: `sdk.gen.ts:1064-1069`

### tui.openModels(options?)

Open models dialog.

```typescript
await client.tui.openModels()
```

**Returns**: `boolean`

Source: `sdk.gen.ts:1074-1079`

### tui.submitPrompt(options?)

Submit current prompt.

```typescript
await client.tui.submitPrompt()
```

**Returns**: `boolean`

Source: `sdk.gen.ts:1084-1089`

### tui.clearPrompt(options?)

Clear current prompt.

```typescript
await client.tui.clearPrompt()
```

**Returns**: `boolean`

Source: `sdk.gen.ts:1094-1099`

### tui.executeCommand(options?)

Execute TUI command.

```typescript
await client.tui.executeCommand({ body: { command: "agent.cycle" } })
```

**Returns**: `boolean`

Source: `sdk.gen.ts:1104-1113`

### tui.showToast(options?)

Show toast notification.

```typescript
await client.tui.showToast({
  body: {
    title: "Success",
    message: "Operation complete",
    variant: "success",
    duration: 3000,
  },
})
```

**Returns**: `boolean`

Source: `sdk.gen.ts:1118-1127`

### tui.publish(options?)

Publish TUI event.

```typescript
await client.tui.publish({ body: { event: "custom-event" } })
```

**Returns**: `boolean`

Source: `sdk.gen.ts:1132-1141`

### tui.control.next(options?)

Get next TUI control request.

```typescript
const request = await client.tui.control.next()
```

**Returns**: `unknown`

Source: `sdk.gen.ts:1004-1009`

### tui.control.response(options?)

Submit TUI control response.

```typescript
await client.tui.control.response({ body: { response: "value" } })
```

**Returns**: `boolean`

Source: `sdk.gen.ts:1014-1023`

## auth

### auth.set(options)

Set authentication credentials.

```typescript
await client.auth.set({
  path: { id: "anthropic" },
  body: {
    type: "api",
    key: "sk-ant-...",
  },
})
```

**Returns**: `boolean`

Source: `sdk.gen.ts:916-925`

## instance

### instance.dispose(options?)

Dispose current instance.

```typescript
await client.instance.dispose()
```

**Returns**: `boolean`

Source: `sdk.gen.ts:399-404`

## event

### event.subscribe(options?)

Subscribe to events for current instance.

```typescript
const stream = await client.event.subscribe()
for await (const event of stream) {
  console.log(event.type, event.properties)
}
```

**Returns**: Server-Sent Events stream of `Event`

Source: `sdk.gen.ts:1149-1154`

## See Also

- `07-sdk-client.md` - SDK usage guide
- `appendix/event-schemas.md` - Event payload schemas
- `appendix/config-schemas.md` - Config type schemas
