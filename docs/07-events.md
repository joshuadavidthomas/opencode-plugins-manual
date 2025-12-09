# Event System

## Overview

OpenCode plugins subscribe to events through the `event` hook. The event system uses a central bus where components publish events and plugins consume them.

## How It Works

> As of commit 3efc95b15

The event bus lives in `packages/opencode/src/bus/index.ts:7-142`. Three functions define the pattern:

- `Bus.event(type, schema)` - Define an event with Zod schema
- `Bus.publish(event, properties)` - Emit an event
- `Bus.subscribe(event, callback)` - Listen for an event

### Defining Events

Components define events with a type string and Zod schema for the payload:

```typescript
export const Event = {
  Created: Bus.event(
    "session.created",
    z.object({
      info: SessionInfo,
    }),
  ),
}
```

Source: `packages/opencode/src/session/index.ts:88-93`

### Publishing Events

Components publish events by calling `Bus.publish`:

```typescript
await Bus.publish(Session.Event.Created, { info: sessionInfo })
```

The bus delivers the event to all subscribers listening for that type plus wildcard subscribers.

Source: `packages/opencode/src/bus/index.ts:78-101`

### Subscribing in Plugins

Plugins subscribe via the `event` hook:

```typescript
import { type Plugin } from "@opencode-ai/plugin"

export const MyPlugin: Plugin = async (ctx) => ({
  event: async ({ event }) => {
    if (event.type === "session.idle") {
      console.log("Session completed:", event.properties.sessionID)
    }
  },
})
```

The `event` hook receives all events. Filter by `event.type`.

## Common Events

### Session Events

> Defined in `packages/opencode/src/session/index.ts:87-120`

| Event             | Payload                                   | When                     |
| ----------------- | ----------------------------------------- | ------------------------ |
| `session.created` | `{ info: SessionInfo }`                   | New session starts       |
| `session.updated` | `{ info: SessionInfo }`                   | Session metadata changes |
| `session.deleted` | `{ info: SessionInfo }`                   | Session removed          |
| `session.diff`    | `{ sessionID: string, diff: FileDiff[] }` | File changes detected    |
| `session.error`   | `{ sessionID?: string, error: object }`   | Error occurs             |

### Session Status Events

> Defined in `packages/opencode/src/session/status.ts:26-41`

| Event            | Payload                               | When                              |
| ---------------- | ------------------------------------- | --------------------------------- |
| `session.status` | `{ sessionID: string, status: Info }` | Status changes                    |
| `session.idle`   | `{ sessionID: string }`               | Session becomes idle (deprecated) |

Use `session.status` to detect when the agent finishes work. The `session.idle` event is deprecated but still emitted for backward compatibility.

### Session Compaction Events

> Defined in `packages/opencode/src/session/compaction.ts:24-30`

| Event               | Payload                 | When                      |
| ------------------- | ----------------------- | ------------------------- |
| `session.compacted` | `{ sessionID: string }` | Context window compressed |

### Message Events

> Defined in `packages/opencode/src/session/message-v2.ts:377-406`

| Event                  | Payload                                                    | When                 |
| ---------------------- | ---------------------------------------------------------- | -------------------- |
| `message.updated`      | `{ info: MessageInfo }`                                    | Message changes      |
| `message.removed`      | `{ sessionID: string, messageID: string }`                 | Message deleted      |
| `message.part.updated` | `{ part: Part, delta?: string }`                           | Message part streams |
| `message.part.removed` | `{ sessionID: string, messageID: string, partID: string }` | Message part deleted |

### File Events

> Defined in `packages/opencode/src/file/index.ts:113-120`

| Event         | Payload            | When                              |
| ------------- | ------------------ | --------------------------------- |
| `file.edited` | `{ file: string }` | File modified via write/edit tool |

> Defined in `packages/opencode/src/file/watcher.ts:18-26`

| Event                  | Payload                                              | When                        |
| ---------------------- | ---------------------------------------------------- | --------------------------- |
| `file.watcher.updated` | `{ file: string, event: "add"\|"change"\|"unlink" }` | File system change detected |

The `file.edited` event fires when the agent modifies a file. The `file.watcher.updated` event fires when any process modifies a file.

### Permission Events

> Defined in `packages/opencode/src/permission/index.ts:40-50`

| Event                | Payload                                                         | When                               |
| -------------------- | --------------------------------------------------------------- | ---------------------------------- |
| `permission.updated` | Full permission info                                            | Permission request created/updated |
| `permission.replied` | `{ sessionID: string, permissionID: string, response: string }` | User responds to permission        |

### Command Events

> Defined in `packages/opencode/src/command/index.ts:11-19`

| Event              | Payload                                                                     | When               |
| ------------------ | --------------------------------------------------------------------------- | ------------------ |
| `command.executed` | `{ name: string, sessionID: string, arguments: string, messageID: string }` | Slash command runs |

### Todo Events

> Defined in `packages/opencode/src/session/todo.ts:17-24`

| Event          | Payload                                | When              |
| -------------- | -------------------------------------- | ----------------- |
| `todo.updated` | `{ sessionID: string, todos: Todo[] }` | Todo list changes |

### TUI Events

> Defined in `packages/opencode/src/cli/cmd/tui/event.ts:4-39`

| Event                 | Payload                                                                   | When                    |
| --------------------- | ------------------------------------------------------------------------- | ----------------------- |
| `tui.prompt.append`   | `{ text: string }`                                                        | Text appended to prompt |
| `tui.command.execute` | `{ command: string }`                                                     | TUI command triggered   |
| `tui.toast.show`      | `{ title?: string, message: string, variant: string, duration?: number }` | Notification shown      |

### LSP Events

> Defined in `packages/opencode/src/lsp/index.ts:15-17`

| Event         | Payload | When              |
| ------------- | ------- | ----------------- |
| `lsp.updated` | `{}`    | LSP state changes |

> Defined in `packages/opencode/src/lsp/client.ts:28-37`

| Event                    | Payload                                                         | When                 |
| ------------------------ | --------------------------------------------------------------- | -------------------- |
| `lsp.client.diagnostics` | `{ serverID: string, path: string, diagnostics: Diagnostic[] }` | Diagnostics received |

### PTY Events

> Defined in `packages/opencode/src/pty/index.ts:76-79`

| Event         | Payload                            | When                |
| ------------- | ---------------------------------- | ------------------- |
| `pty.created` | `{ info: PtyInfo }`                | PTY session starts  |
| `pty.updated` | `{ info: PtyInfo }`                | PTY session changes |
| `pty.exited`  | `{ id: string, exitCode: number }` | PTY process exits   |
| `pty.deleted` | `{ id: string }`                   | PTY session removed |

### VCS Events

> Defined in `packages/opencode/src/project/vcs.ts:13-19`

| Event                | Payload               | When               |
| -------------------- | --------------------- | ------------------ |
| `vcs.branch.updated` | `{ branch?: string }` | Git branch changes |

### Installation Events

> Defined in `packages/opencode/src/installation/index.ts:20-32`

| Event                           | Payload               | When                      |
| ------------------------------- | --------------------- | ------------------------- |
| `installation.updated`          | `{ version: string }` | OpenCode version detected |
| `installation.update.available` | `{ version: string }` | Newer version available   |

### IDE Events

> Defined in `packages/opencode/src/ide/index.ts:19-25`

| Event           | Payload           | When                   |
| --------------- | ----------------- | ---------------------- |
| `ide.installed` | `{ ide: string }` | IDE extension detected |

## Subscription Patterns

### Listen for Specific Events

```typescript
export const MyPlugin: Plugin = async (ctx) => ({
  event: async ({ event }) => {
    if (event.type === "session.idle") {
      // Handle session completion
    }

    if (event.type === "file.edited") {
      // Handle file changes
      console.log("File changed:", event.properties.file)
    }
  },
})
```

### Listen for Multiple Events

```typescript
event: async ({ event }) => {
  switch (event.type) {
    case "session.created":
      await handleSessionStart(event.properties)
      break
    case "session.idle":
      await handleSessionEnd(event.properties)
      break
    case "message.updated":
      await handleMessageChange(event.properties)
      break
  }
}
```

### Pattern Match on Type

```typescript
event: async ({ event }) => {
  // All tool execution events
  if (event.type.startsWith("tool.execute")) {
    console.log("Tool event:", event.type)
  }

  // All session events
  if (event.type.startsWith("session.")) {
    console.log("Session event:", event.type)
  }
}
```

### Access Event Context

The event hook receives a `PluginInput` context with useful properties:

```typescript
export const MyPlugin: Plugin = async (ctx) => ({
  event: async ({ event }) => {
    // ctx available in plugin function
    console.log("Directory:", ctx.directory)
    console.log("Provider:", ctx.provider)

    if (event.type === "session.idle") {
      // Write logs to plugin directory
      await Bun.write(`${ctx.directory}/session-log.txt`, `Session ${event.properties.sessionID} completed\n`)
    }
  },
})
```

Source: `packages/plugin/src/index.ts:18-32`

## Wildcard Subscriptions

The bus supports wildcard subscriptions that receive all events:

```typescript
Bus.subscribeAll((event) => {
  console.log("Any event:", event.type, event.properties)
})
```

Source: `packages/opencode/src/bus/index.ts:122-124`

Plugins use the `event` hook, which is a wildcard subscription. Filter events in your handler.

## Instance Scoping

Events are scoped to the current OpenCode instance (project directory). Each project runs a separate event bus.

When an instance disposes, the bus publishes a `server.instance.disposed` event to cleanup subscriptions.

Source: `packages/opencode/src/bus/index.ts:16-37`

## Direct Subscription (Advanced)

Plugins can also subscribe directly using `Bus.subscribe` for type-safe callbacks:

```typescript
import { Session } from "@opencode-ai/opencode"

Bus.subscribe(Session.Event.Created, (event) => {
  // event.type is "session.created"
  // event.properties is typed as { info: SessionInfo }
  console.log("New session:", event.properties.info.id)
})
```

This requires importing types from the OpenCode core package, which plugin authors may not want to depend on. The `event` hook is the standard approach.

## Summary Table

| Category         | Events                                                          | Source                                                                                  |
| ---------------- | --------------------------------------------------------------- | --------------------------------------------------------------------------------------- |
| **Session**      | created, updated, deleted, diff, error, status, idle, compacted | `session/index.ts:87-120`<br>`session/status.ts:26-41`<br>`session/compaction.ts:24-30` |
| **Message**      | updated, removed, part.updated, part.removed                    | `session/message-v2.ts:377-406`                                                         |
| **File**         | edited, watcher.updated                                         | `file/index.ts:113-120`<br>`file/watcher.ts:18-26`                                      |
| **Permission**   | updated, replied                                                | `permission/index.ts:40-50`                                                             |
| **Command**      | executed                                                        | `command/index.ts:11-19`                                                                |
| **Todo**         | updated                                                         | `session/todo.ts:17-24`                                                                 |
| **TUI**          | prompt.append, command.execute, toast.show                      | `cli/cmd/tui/event.ts:4-39`                                                             |
| **LSP**          | updated, client.diagnostics                                     | `lsp/index.ts:15-17`<br>`lsp/client.ts:28-37`                                           |
| **PTY**          | created, updated, exited, deleted                               | `pty/index.ts:76-79`                                                                    |
| **VCS**          | branch.updated                                                  | `project/vcs.ts:13-19`                                                                  |
| **Installation** | updated, update.available                                       | `installation/index.ts:20-32`                                                           |
| **IDE**          | installed                                                       | `ide/index.ts:19-25`                                                                    |

All event definitions include Zod schemas for validation and type inference.
