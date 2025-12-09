# Advanced Plugin Patterns

This document covers advanced patterns for building production-quality plugins, using [oh-my-opencode](https://github.com/code-yeongyu/oh-my-opencode) as a reference implementation.

## Stateful Hook Factories

Simple plugins define hooks inline. Complex plugins need **stateful hooks**—hooks that track information across invocations. The factory pattern makes this manageable.

### The Problem

Hooks are called repeatedly, but sometimes you need to:
- Track which sessions have been processed
- Accumulate data across tool calls
- Avoid duplicate actions

### The Factory Pattern

Create a factory function that returns hook handlers with access to shared state:

```typescript
// src/hooks/context-window-monitor.ts
import type { PluginInput } from "@opencode-ai/plugin"

export function createContextWindowMonitorHook(ctx: PluginInput) {
  // State persists across hook invocations
  const remindedSessions = new Set<string>()

  return {
    "tool.execute.after": async (input, output) => {
      const { sessionID } = input
      
      // Check if we've already reminded this session
      if (remindedSessions.has(sessionID)) return
      
      // Check context usage via SDK
      const messages = await ctx.client.session.messages({
        path: { id: sessionID }
      })
      
      // Calculate token usage...
      if (tokenUsage > threshold) {
        remindedSessions.add(sessionID)  // Track that we've reminded
        output.output += "\n\n[Context window reminder...]"
      }
    },
    
    event: async ({ event }) => {
      // Clean up state when session is deleted
      if (event.type === "session.deleted") {
        const sessionInfo = event.properties?.info
        if (sessionInfo?.id) {
          remindedSessions.delete(sessionInfo.id)
        }
      }
    },
  }
}
```

**Real-world example:** [oh-my-opencode/src/hooks/context-window-monitor.ts](https://github.com/code-yeongyu/oh-my-opencode/blob/master/src/hooks/context-window-monitor.ts)

### Benefits

1. **Encapsulation** — State is hidden inside the factory closure
2. **Testability** — Create fresh instances for testing
3. **Reusability** — Same factory can create multiple independent instances
4. **Type safety** — TypeScript infers types from the factory function

## Hook Composition

Large plugins need multiple independent hook handlers. Compose them in your main plugin:

```typescript
// src/index.ts
import type { Plugin } from "@opencode-ai/plugin"
import { createContextWindowMonitor } from "./hooks/context-window-monitor"
import { createSessionRecovery } from "./hooks/session-recovery"
import { createCommentChecker } from "./hooks/comment-checker"
import { loadConfig } from "./config"

export const MyPlugin: Plugin = async (ctx) => {
  const config = loadConfig(ctx.directory)
  
  // Create hook handlers
  const contextMonitor = createContextWindowMonitor(ctx)
  const sessionRecovery = createSessionRecovery(ctx)
  const commentChecker = createCommentChecker()

  return {
    event: async (input) => {
      // Call each handler in sequence
      await contextMonitor.event?.(input)
      await sessionRecovery.event?.(input)
    },

    "tool.execute.before": async (input, output) => {
      await commentChecker["tool.execute.before"]?.(input, output)
    },

    "tool.execute.after": async (input, output) => {
      await contextMonitor["tool.execute.after"]?.(input, output)
      await commentChecker["tool.execute.after"]?.(input, output)
    },
  }
}

export default MyPlugin
```

**Real-world example:** [oh-my-opencode/src/index.ts:71-270](https://github.com/code-yeongyu/oh-my-opencode/blob/master/src/index.ts#L71-L270)

### Composition Patterns

**Sequential execution:**
```typescript
"tool.execute.after": async (input, output) => {
  await handlerA["tool.execute.after"]?.(input, output)
  await handlerB["tool.execute.after"]?.(input, output)
  await handlerC["tool.execute.after"]?.(input, output)
}
```

**Conditional execution:**
```typescript
"tool.execute.after": async (input, output) => {
  if (!config.disabled_features?.includes("feature-a")) {
    await featureA["tool.execute.after"]?.(input, output)
  }
}
```

**Early exit:**
```typescript
"tool.execute.before": async (input, output) => {
  const result = await validator["tool.execute.before"]?.(input, output)
  if (result?.blocked) {
    throw new Error(result.reason)  // Stops execution
  }
}
```

## State Lifecycle Management

Plugins that track per-session state must clean up to avoid memory leaks.

### The Problem

```typescript
const sessionData = new Map<string, SessionState>()

// This grows forever if you don't clean up!
sessionData.set(sessionID, { ... })
```

### The Solution: Clean Up on Session Events

```typescript
export function createStatefulHook(ctx: PluginInput) {
  const sessionState = new Map<string, { 
    startTime: number
    tokenCount: number 
  }>()
  
  return {
    "tool.execute.after": async (input, output) => {
      const state = sessionState.get(input.sessionID) || { 
        startTime: Date.now(), 
        tokenCount: 0 
      }
      state.tokenCount++
      sessionState.set(input.sessionID, state)
    },
    
    event: async ({ event }) => {
      // Clean up when session ends
      if (event.type === "session.deleted") {
        const sessionID = event.properties?.info?.id
        if (sessionID) {
          sessionState.delete(sessionID)
        }
      }
    },
  }
}
```

### Common Session Events for Cleanup

| Event | When to Use |
|-------|-------------|
| `session.deleted` | Session permanently removed |
| `session.idle` | Session finished processing (may resume) |
| `session.error` | Session encountered an error |

**Real-world example:** [oh-my-opencode/src/hooks/claude-code-hooks/index.ts:264-273](https://github.com/code-yeongyu/oh-my-opencode/blob/master/src/hooks/claude-code-hooks/index.ts#L264-L273)

## SDK Client Advanced Usage

The `ctx.client` SDK provides powerful capabilities beyond basic hooks.

### Sending Prompts Programmatically

Inject messages into a session:

```typescript
await ctx.client.session.prompt({
  path: { id: sessionID },
  body: { 
    parts: [{ type: "text", text: "continue" }] 
  },
  query: { directory: ctx.directory },
})
```

**Use cases:**
- Auto-continue after recoverable errors
- Inject follow-up prompts from hooks
- Trigger actions based on session state

**Real-world example:** [oh-my-opencode/src/index.ts:205-211](https://github.com/code-yeongyu/oh-my-opencode/blob/master/src/index.ts#L205-L211)

### Aborting Sessions

Stop a running session:

```typescript
await ctx.client.session.abort({ 
  path: { id: sessionID } 
}).catch(() => {})  // Ignore errors if already stopped
```

**Real-world example:** [oh-my-opencode/src/hooks/session-recovery/index.ts:215](https://github.com/code-yeongyu/oh-my-opencode/blob/master/src/hooks/session-recovery/index.ts#L215)

### Fetching Session Messages

Analyze conversation history:

```typescript
const response = await ctx.client.session.messages({
  path: { id: sessionID },
})

const messages = response.data ?? response
const assistantMessages = messages.filter(m => m.info.role === "assistant")
```

**Use cases:**
- Calculate token usage
- Analyze conversation patterns
- Find specific message types

**Real-world example:** [oh-my-opencode/src/hooks/context-window-monitor.ts:48-67](https://github.com/code-yeongyu/oh-my-opencode/blob/master/src/hooks/context-window-monitor.ts#L48-L67)

### Toast Notifications

Show feedback to users:

```typescript
await ctx.client.tui.showToast({
  body: {
    title: "Recovery Complete",
    message: "Session recovered from error",
    variant: "success",  // "success" | "warning" | "error"
    duration: 3000,      // milliseconds
  },
})
```

**Real-world example:** [oh-my-opencode/src/hooks/session-recovery/index.ts:245-254](https://github.com/code-yeongyu/oh-my-opencode/blob/master/src/hooks/session-recovery/index.ts#L245-L254)

### Getting Session Info

Check session properties:

```typescript
const sessionInfo = await ctx.client.session.get({
  path: { id: sessionID },
})

const parentSessionId = sessionInfo.data?.parentID
const isSubsession = !!parentSessionId
```

**Real-world example:** [oh-my-opencode/src/hooks/claude-code-hooks/index.ts:72-77](https://github.com/code-yeongyu/oh-my-opencode/blob/master/src/hooks/claude-code-hooks/index.ts#L72-L77)

## Error Recovery Patterns

Robust plugins handle errors gracefully.

### Detecting Recoverable Errors

```typescript
function isRecoverableError(error: unknown): boolean {
  const message = getErrorMessage(error).toLowerCase()
  
  return (
    message.includes("tool_result") ||
    message.includes("thinking") ||
    message.includes("non-empty content")
  )
}
```

**Real-world example:** [oh-my-opencode/src/hooks/session-recovery/index.ts:57-80](https://github.com/code-yeongyu/oh-my-opencode/blob/master/src/hooks/session-recovery/index.ts#L57-L80)

### Graceful Degradation

Always catch errors in hooks to avoid breaking the plugin chain:

```typescript
"tool.execute.after": async (input, output) => {
  try {
    await riskyOperation()
  } catch (error) {
    // Log but don't throw
    console.error("Operation failed:", error)
    // Optionally notify user
    await ctx.client.tui.showToast({
      body: {
        title: "Warning",
        message: "Feature X encountered an error",
        variant: "warning",
        duration: 3000,
      },
    }).catch(() => {})
  }
}
```

### Preventing Duplicate Processing

Use Sets or Maps to track processed items:

```typescript
const processingErrors = new Set<string>()

async function handleError(messageID: string) {
  // Prevent duplicate handling
  if (processingErrors.has(messageID)) return
  processingErrors.add(messageID)
  
  try {
    await recoverFromError(messageID)
  } finally {
    processingErrors.delete(messageID)
  }
}
```

**Real-world example:** [oh-my-opencode/src/hooks/session-recovery/index.ts:190-273](https://github.com/code-yeongyu/oh-my-opencode/blob/master/src/hooks/session-recovery/index.ts#L190-L273)

## Output Mutation Patterns

The `tool.execute.after` hook can append information to tool output.

### Appending Context

```typescript
"tool.execute.after": async (input, output) => {
  if (shouldAddReminder()) {
    output.output += `\n\n[SYSTEM REMINDER]
Your context window is ${usage}% full. Consider:
- Breaking into smaller tasks
- Being more concise
- Compacting the session`
  }
}
```

### Truncating Large Output

```typescript
"tool.execute.after": async (input, output) => {
  if (input.tool === "grep" && output.output.length > MAX_OUTPUT) {
    const truncated = output.output.slice(0, MAX_OUTPUT)
    output.output = truncated + `\n\n[Output truncated. ${output.output.length - MAX_OUTPUT} characters omitted.]`
  }
}
```

### Adding Warnings

```typescript
"tool.execute.after": async (input, output) => {
  const warnings = await checkForIssues(input, output)
  if (warnings.length > 0) {
    output.output += "\n\n**Warnings:**\n" + warnings.join("\n")
  }
}
```

## Real-World Examples from oh-my-opencode

| Feature | Pattern Used | Source |
|---------|--------------|--------|
| Context Window Monitor | Stateful factory + output mutation | [context-window-monitor.ts](https://github.com/code-yeongyu/oh-my-opencode/blob/master/src/hooks/context-window-monitor.ts) |
| Session Recovery | Error detection + SDK client | [session-recovery/index.ts](https://github.com/code-yeongyu/oh-my-opencode/blob/master/src/hooks/session-recovery/index.ts) |
| Todo Continuation | Event handling + prompt injection | [todo-continuation-enforcer.ts](https://github.com/code-yeongyu/oh-my-opencode/blob/master/src/hooks/todo-continuation-enforcer.ts) |
| Grep Output Truncator | Output mutation | [grep-output-truncator.ts](https://github.com/code-yeongyu/oh-my-opencode/blob/master/src/hooks/grep-output-truncator.ts) |
| Think Mode Switcher | Event detection + state tracking | [think-mode/index.ts](https://github.com/code-yeongyu/oh-my-opencode/blob/master/src/hooks/think-mode/index.ts) |
| Terminal Title Updates | Event handling + external effects | [index.ts:147-235](https://github.com/code-yeongyu/oh-my-opencode/blob/master/src/index.ts#L147-L235) |

## Summary

Advanced plugin development uses these patterns:

1. **Hook Factories** — Create stateful hooks with closures
2. **Hook Composition** — Combine multiple handlers in one plugin
3. **State Lifecycle** — Clean up on session events to prevent leaks
4. **SDK Client** — Use advanced methods for complex interactions
5. **Error Recovery** — Detect, handle, and recover from errors gracefully
6. **Output Mutation** — Append context, truncate, or add warnings

**Reference implementation:** [oh-my-opencode](https://github.com/code-yeongyu/oh-my-opencode) demonstrates all these patterns in production.
