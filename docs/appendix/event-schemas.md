# Event Payload Schemas

> As of commit [3efc95b](https://github.com/sst/opencode/tree/3efc95b157e05bc3c42554be1b5778f8f1b64cfe)

Complete event type definitions and payload schemas.

Source: [`packages/sdk/js/src/gen/types.gen.ts`](https://github.com/sst/opencode/blob/3efc95b/packages/sdk/js/src/gen/types.gen.ts)

## Session Events

### session.created

```typescript
{
  type: "session.created"
  properties: {
    info: {
      id: string
      projectID: string
      directory: string
      parentID?: string
      summary?: {
        additions: number
        deletions: number
        files: number
        diffs?: Array<FileDiff>
      }
      share?: { url: string }
      title: string
      version: string
      time: {
        created: number
        updated: number
        compacting?: number
      }
      revert?: {
        messageID: string
        partID?: string
        snapshot?: string
        diff?: string
      }
    }
  }
}
```

Source: [`types.gen.ts:562-567`](https://github.com/sst/opencode/blob/3efc95b/packages/sdk/js/src/gen/types.gen.ts#L562-L567)

### session.updated

```typescript
{
  type: "session.updated"
  properties: {
    info: Session // Same as session.created
  }
}
```

Source: [`types.gen.ts:569-574`](https://github.com/sst/opencode/blob/3efc95b/packages/sdk/js/src/gen/types.gen.ts#L569-L574)

### session.deleted

```typescript
{
  type: "session.deleted"
  properties: {
    info: Session // Same as session.created
  }
}
```

Source: [`types.gen.ts:576-581`](https://github.com/sst/opencode/blob/3efc95b/packages/sdk/js/src/gen/types.gen.ts#L576-L581)

### session.status

```typescript
{
  type: "session.status"
  properties: {
    sessionID: string
    status:
      | { type: "idle" }
      | { type: "retry"; attempt: number; message: string; next: number }
      | { type: "busy" }
  }
}
```

Source: [`types.gen.ts:467-473`](https://github.com/sst/opencode/blob/3efc95b/packages/sdk/js/src/gen/types.gen.ts#L467-L473)

### session.idle

```typescript
{
  type: "session.idle"
  properties: {
    sessionID: string
  }
}
```

Source: [`types.gen.ts:475-480`](https://github.com/sst/opencode/blob/3efc95b/packages/sdk/js/src/gen/types.gen.ts#L475-L480)

### session.compacted

```typescript
{
  type: "session.compacted"
  properties: {
    sessionID: string
  }
}
```

Source: [`types.gen.ts:482-487`](https://github.com/sst/opencode/blob/3efc95b/packages/sdk/js/src/gen/types.gen.ts#L482-L487)

### session.diff

```typescript
{
  type: "session.diff"
  properties: {
    sessionID: string
    diff: Array<{
      file: string
      before: string
      after: string
      additions: number
      deletions: number
    }>
  }
}
```

Source: [`types.gen.ts:583-589`](https://github.com/sst/opencode/blob/3efc95b/packages/sdk/js/src/gen/types.gen.ts#L583-L589)

### session.error

```typescript
{
  type: "session.error"
  properties: {
    sessionID?: string
    error?:
      | { name: "ProviderAuthError"; data: { providerID: string; message: string } }
      | { name: "UnknownError"; data: { message: string } }
      | { name: "MessageOutputLengthError"; data: {} }
      | { name: "MessageAbortedError"; data: { message: string } }
      | { name: "APIError"; data: { message: string; statusCode?: number; isRetryable: boolean; responseHeaders?: Record<string, string>; responseBody?: string } }
  }
}
```

Source: [`types.gen.ts:591-597`](https://github.com/sst/opencode/blob/3efc95b/packages/sdk/js/src/gen/types.gen.ts#L591-L597)

## Message Events

### message.updated

```typescript
{
  type: "message.updated"
  properties: {
    info: UserMessage | AssistantMessage
  }
}
```

**UserMessage**:

```typescript
{
  id: string
  sessionID: string
  role: "user"
  time: { created: number }
  summary?: {
    title?: string
    body?: string
    diffs: Array<FileDiff>
  }
  agent: string
  model: { providerID: string; modelID: string }
  system?: string
  tools?: Record<string, boolean>
}
```

**AssistantMessage**:

```typescript
{
  id: string
  sessionID: string
  role: "assistant"
  time: { created: number; completed?: number }
  error?: ProviderAuthError | UnknownError | MessageOutputLengthError | MessageAbortedError | ApiError
  parentID: string
  modelID: string
  providerID: string
  mode: string
  path: { cwd: string; root: string }
  summary?: boolean
  cost: number
  tokens: {
    input: number
    output: number
    reasoning: number
    cache: { read: number; write: number }
  }
  finish?: string
}
```

Source: [`types.gen.ts:145-150`](https://github.com/sst/opencode/blob/3efc95b/packages/sdk/js/src/gen/types.gen.ts#L145-L150)

### message.removed

```typescript
{
  type: "message.removed"
  properties: {
    sessionID: string
    messageID: string
  }
}
```

Source: [`types.gen.ts:152-158`](https://github.com/sst/opencode/blob/3efc95b/packages/sdk/js/src/gen/types.gen.ts#L152-L158)

### message.part.updated

```typescript
{
  type: "message.part.updated"
  properties: {
    part: Part  // See Part types below
    delta?: string
  }
}
```

**Part types**:

- `TextPart`: `{ id, sessionID, messageID, type: "text", text, synthetic?, ignored?, time?, metadata? }`
- `ReasoningPart`: `{ id, sessionID, messageID, type: "reasoning", text, metadata?, time }`
- `FilePart`: `{ id, sessionID, messageID, type: "file", mime, filename?, url, source? }`
- `ToolPart`: `{ id, sessionID, messageID, type: "tool", callID, tool, state, metadata? }`
- `StepStartPart`: `{ id, sessionID, messageID, type: "step-start", snapshot? }`
- `StepFinishPart`: `{ id, sessionID, messageID, type: "step-finish", reason, snapshot?, cost, tokens }`
- `SnapshotPart`: `{ id, sessionID, messageID, type: "snapshot", snapshot }`
- `PatchPart`: `{ id, sessionID, messageID, type: "patch", hash, files }`
- `AgentPart`: `{ id, sessionID, messageID, type: "agent", name, source? }`
- `RetryPart`: `{ id, sessionID, messageID, type: "retry", attempt, error, time }`
- `CompactionPart`: `{ id, sessionID, messageID, type: "compaction", auto }`
- `SubtaskPart`: `{ id, sessionID, messageID, type: "subtask", prompt, description, agent }`

Source: [`types.gen.ts:160-412`](https://github.com/sst/opencode/blob/3efc95b/packages/sdk/js/src/gen/types.gen.ts#L160-L412)

### message.part.removed

```typescript
{
  type: "message.part.removed"
  properties: {
    sessionID: string
    messageID: string
    partID: string
  }
}
```

Source: [`types.gen.ts:414-421`](https://github.com/sst/opencode/blob/3efc95b/packages/sdk/js/src/gen/types.gen.ts#L414-L421)

## File Events

### file.edited

```typescript
{
  type: "file.edited"
  properties: {
    file: string // Absolute path
  }
}
```

Source: [`types.gen.ts:489-494`](https://github.com/sst/opencode/blob/3efc95b/packages/sdk/js/src/gen/types.gen.ts#L489-L494)

### file.watcher.updated

```typescript
{
  type: "file.watcher.updated"
  properties: {
    file: string
    event: "add" | "change" | "unlink"
  }
}
```

Source: [`types.gen.ts:599-605`](https://github.com/sst/opencode/blob/3efc95b/packages/sdk/js/src/gen/types.gen.ts#L599-L605)

## Permission Events

### permission.updated

```typescript
{
  type: "permission.updated"
  properties: {
    id: string
    type: string
    pattern?: string | Array<string>
    sessionID: string
    messageID: string
    callID?: string
    title: string
    metadata: Record<string, unknown>
    time: { created: number }
  }
}
```

Source: [`types.gen.ts:439-442`](https://github.com/sst/opencode/blob/3efc95b/packages/sdk/js/src/gen/types.gen.ts#L439-L442)

### permission.replied

```typescript
{
  type: "permission.replied"
  properties: {
    sessionID: string
    permissionID: string
    response: string
  }
}
```

Source: [`types.gen.ts:444-451`](https://github.com/sst/opencode/blob/3efc95b/packages/sdk/js/src/gen/types.gen.ts#L444-L451)

## Command Events

### command.executed

```typescript
{
  type: "command.executed"
  properties: {
    name: string
    sessionID: string
    arguments: string
    messageID: string
  }
}
```

Source: [`types.gen.ts:523-531`](https://github.com/sst/opencode/blob/3efc95b/packages/sdk/js/src/gen/types.gen.ts#L523-L531)

## Todo Events

### todo.updated

```typescript
{
  type: "todo.updated"
  properties: {
    sessionID: string
    todos: Array<{
      content: string // Brief description
      status: string // pending, in_progress, completed, cancelled
      priority: string // high, medium, low
      id: string // Unique identifier
    }>
  }
}
```

Source: [`types.gen.ts:515-521`](https://github.com/sst/opencode/blob/3efc95b/packages/sdk/js/src/gen/types.gen.ts#L515-L521)

## TUI Events

### tui.prompt.append

```typescript
{
  type: "tui.prompt.append"
  properties: {
    text: string
  }
}
```

Source: [`types.gen.ts:614-619`](https://github.com/sst/opencode/blob/3efc95b/packages/sdk/js/src/gen/types.gen.ts#L614-L619)

### tui.command.execute

```typescript
{
  type: "tui.command.execute"
  properties: {
    command:
      | "session.list"
      | "session.new"
      | "session.share"
      | "session.interrupt"
      | "session.compact"
      | "session.page.up"
      | "session.page.down"
      | "session.half.page.up"
      | "session.half.page.down"
      | "session.first"
      | "session.last"
      | "prompt.clear"
      | "prompt.submit"
      | "agent.cycle"
      | string  // Extensible
  }
}
```

Source: [`types.gen.ts:621-644`](https://github.com/sst/opencode/blob/3efc95b/packages/sdk/js/src/gen/types.gen.ts#L621-L644)

### tui.toast.show

```typescript
{
  type: "tui.toast.show"
  properties: {
    title?: string
    message: string
    variant: "info" | "success" | "warning" | "error"
    duration?: number  // Milliseconds
  }
}
```

Source: [`types.gen.ts:645-656`](https://github.com/sst/opencode/blob/3efc95b/packages/sdk/js/src/gen/types.gen.ts#L645-L656)

## PTY Events

### pty.created

```typescript
{
  type: "pty.created"
  properties: {
    info: {
      id: string
      title: string
      command: string
      args: Array<string>
      cwd: string
      status: "running" | "exited"
      pid: number
    }
  }
}
```

Source: [`types.gen.ts:668-673`](https://github.com/sst/opencode/blob/3efc95b/packages/sdk/js/src/gen/types.gen.ts#L668-L673)

### pty.updated

```typescript
{
  type: "pty.updated"
  properties: {
    info: Pty // Same as pty.created
  }
}
```

Source: [`types.gen.ts:675-680`](https://github.com/sst/opencode/blob/3efc95b/packages/sdk/js/src/gen/types.gen.ts#L675-L680)

### pty.exited

```typescript
{
  type: "pty.exited"
  properties: {
    id: string
    exitCode: number
  }
}
```

Source: [`types.gen.ts:682-688`](https://github.com/sst/opencode/blob/3efc95b/packages/sdk/js/src/gen/types.gen.ts#L682-L688)

### pty.deleted

```typescript
{
  type: "pty.deleted"
  properties: {
    id: string
  }
}
```

Source: [`types.gen.ts:690-695`](https://github.com/sst/opencode/blob/3efc95b/packages/sdk/js/src/gen/types.gen.ts#L690-L695)

## LSP Events

### lsp.updated

```typescript
{
  type: "lsp.updated"
  properties: Record<string, unknown>
}
```

Source: [`types.gen.ts:32-37`](https://github.com/sst/opencode/blob/3efc95b/packages/sdk/js/src/gen/types.gen.ts#L32-L37)

### lsp.client.diagnostics

```typescript
{
  type: "lsp.client.diagnostics"
  properties: {
    serverID: string
    path: string
  }
}
```

Source: [`types.gen.ts:24-30`](https://github.com/sst/opencode/blob/3efc95b/packages/sdk/js/src/gen/types.gen.ts#L24-L30)

## VCS Events

### vcs.branch.updated

```typescript
{
  type: "vcs.branch.updated"
  properties: {
    branch?: string
  }
}
```

Source: [`types.gen.ts:607-612`](https://github.com/sst/opencode/blob/3efc95b/packages/sdk/js/src/gen/types.gen.ts#L607-L612)

## Installation Events

### installation.updated

```typescript
{
  type: "installation.updated"
  properties: {
    version: string
  }
}
```

Source: [`types.gen.ts:10-15`](https://github.com/sst/opencode/blob/3efc95b/packages/sdk/js/src/gen/types.gen.ts#L10-L15)

### installation.update-available

```typescript
{
  type: "installation.update-available"
  properties: {
    version: string
  }
}
```

Source: [`types.gen.ts:17-22`](https://github.com/sst/opencode/blob/3efc95b/packages/sdk/js/src/gen/types.gen.ts#L17-L22)

## Server Events

### server.connected

```typescript
{
  type: "server.connected"
  properties: Record<string, unknown>
}
```

Source: [`types.gen.ts:697-702`](https://github.com/sst/opencode/blob/3efc95b/packages/sdk/js/src/gen/types.gen.ts#L697-L702)

### server.instance.disposed

```typescript
{
  type: "server.instance.disposed"
  properties: {
    directory: string
  }
}
```

Source: [`types.gen.ts:3-8`](https://github.com/sst/opencode/blob/3efc95b/packages/sdk/js/src/gen/types.gen.ts#L3-L8)

## Global Event Wrapper

All events are wrapped in a `GlobalEvent` when received via the global event stream:

```typescript
{
  directory: string // Instance directory
  payload: Event // One of the events above
}
```

Source: [`types.gen.ts:738-741`](https://github.com/sst/opencode/blob/3efc95b/packages/sdk/js/src/gen/types.gen.ts#L738-L741)

## See Also

- `06-events.md` - Event system overview and subscription
- `03-hooks-reference.md` - Event-related hooks
- `07-sdk-client.md` - Subscribing to events via SDK
