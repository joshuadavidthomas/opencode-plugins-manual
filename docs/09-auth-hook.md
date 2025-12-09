# Auth Hook

## Overview

The `auth` hook lets plugins register authentication providers. These providers handle OAuth flows or API key authentication for services that need credentials.

Use cases:

- Authenticate with external APIs (GitHub, Linear, Slack)
- Store and refresh OAuth tokens
- Prompt users for API keys
- Provide credentials to MCP servers

## Basic Structure

```typescript
import { type Plugin, type AuthHook } from "@opencode-ai/plugin"

export const MyPlugin: Plugin = async (ctx) => ({
  auth: {
    provider: "myservice",
    methods: [
      {
        type: "oauth",
        label: "OAuth (recommended)",
        async authorize() {
          // Return OAuth flow details
        },
      },
      {
        type: "api",
        label: "API Key",
        async authorize(inputs) {
          // Handle API key input
        },
      },
    ],
  } satisfies AuthHook,
})
```

The `auth` hook returns an object with:

- `provider` - Unique identifier for this auth provider
- `loader` - Optional function to load credentials into memory
- `methods` - Array of authentication methods (OAuth or API key)

## OAuth Authentication

OAuth is the standard for third-party authentication. The flow:

1. Plugin generates authorization URL
2. OpenCode opens browser to that URL
3. User approves access
4. Plugin receives callback with code or tokens
5. Plugin exchanges code for access/refresh tokens (if using "code" method)

### Auto Method (Recommended)

The `auto` method handles the callback automatically via a local server:

```typescript
{
  type: "oauth",
  label: "OAuth (recommended)",
  async authorize(inputs) {
    const state = generateRandomState()
    const codeVerifier = generatePKCEVerifier()
    const codeChallenge = await generatePKCEChallenge(codeVerifier)

    return {
      url: `https://api.myservice.com/oauth/authorize?` +
        `client_id=${CLIENT_ID}&` +
        `redirect_uri=http://localhost:3838/callback&` +
        `response_type=code&` +
        `state=${state}&` +
        `code_challenge=${codeChallenge}&` +
        `code_challenge_method=S256`,
      instructions: "Authorize OpenCode to access your MyService account",
      method: "auto",
      async callback() {
        // OpenCode handles the redirect and extracts the code
        // This function receives the code and exchanges it for tokens
        const response = await fetch("https://api.myservice.com/oauth/token", {
          method: "POST",
          body: JSON.stringify({
            grant_type: "authorization_code",
            code: receivedCode,
            code_verifier: codeVerifier,
            redirect_uri: "http://localhost:3838/callback",
          }),
        })

        const tokens = await response.json()

        return {
          type: "success",
          provider: "myservice",
          access: tokens.access_token,
          refresh: tokens.refresh_token,
          expires: Date.now() + tokens.expires_in * 1000,
        }
      },
    }
  },
}
```

### Code Method

The `code` method requires the user to paste the authorization code:

```typescript
{
  type: "oauth",
  label: "OAuth (manual code)",
  async authorize(inputs) {
    return {
      url: "https://api.myservice.com/oauth/authorize?...",
      instructions: "Authorize and paste the code back into OpenCode",
      method: "code",
      async callback(code) {
        // Exchange code for tokens
        const tokens = await exchangeCode(code)

        return {
          type: "success",
          access: tokens.access_token,
          refresh: tokens.refresh_token,
          expires: Date.now() + tokens.expires_in * 1000,
        }
      },
    }
  },
}
```

## API Key Authentication

For services that use API keys instead of OAuth:

```typescript
{
  type: "api",
  label: "API Key",
  prompts: [
    {
      type: "text",
      key: "api_key",
      message: "Enter your MyService API key:",
      placeholder: "ms_...",
      validate: (value) => {
        if (!value.startsWith("ms_")) {
          return "API key must start with ms_"
        }
      },
    },
  ],
  async authorize(inputs) {
    // Validate the API key
    const response = await fetch("https://api.myservice.com/user", {
      headers: {
        Authorization: `Bearer ${inputs.api_key}`,
      },
    })

    if (!response.ok) {
      return { type: "failed" }
    }

    return {
      type: "success",
      key: inputs.api_key,
    }
  },
}
```

### Conditional Prompts

Prompts can be conditional based on previous inputs:

```typescript
prompts: [
  {
    type: "select",
    key: "auth_type",
    message: "Select authentication type:",
    options: [
      { label: "Production", value: "prod" },
      { label: "Development", value: "dev" },
    ],
  },
  {
    type: "text",
    key: "api_key",
    message: "Enter API key:",
    condition: (inputs) => inputs.auth_type === "prod",
  },
  {
    type: "text",
    key: "dev_token",
    message: "Enter dev token:",
    condition: (inputs) => inputs.auth_type === "dev",
  },
]
```

## Loader Function

The optional `loader` function runs when OpenCode starts. Use it to load credentials into memory or refresh expired tokens:

```typescript
export const MyPlugin: Plugin = async (ctx) => ({
  auth: {
    provider: "myservice",
    async loader(auth, provider) {
      // auth() returns stored credentials
      const credentials = await auth()

      if (!credentials) {
        return {}
      }

      // Check if token expired
      if (credentials.expires && credentials.expires < Date.now()) {
        // Refresh the token
        const response = await fetch("https://api.myservice.com/oauth/refresh", {
          method: "POST",
          body: JSON.stringify({
            refresh_token: credentials.refresh,
          }),
        })

        const tokens = await response.json()

        // Update stored credentials
        await provider.update({
          access: tokens.access_token,
          refresh: tokens.refresh_token,
          expires: Date.now() + tokens.expires_in * 1000,
        })

        return {
          access_token: tokens.access_token,
        }
      }

      // Return credentials for use
      return {
        access_token: credentials.access,
      }
    },
    methods: [
      // ... auth methods
    ],
  },
})
```

The loader returns an object that becomes available to other parts of your plugin via `ctx.provider`.

## TypeScript Definitions

> As of commit [3efc95b](https://github.com/sst/opencode/tree/3efc95b157e05bc3c42554be1b5778f8f1b64cfe)

Full type definitions in `packages/plugin/src/index.ts:35-142`:

```typescript
export type AuthHook = {
  provider: string
  loader?: (auth: () => Promise<Auth>, provider: Provider) => Promise<Record<string, any>>
  methods: (
    | {
        type: "oauth"
        label: string
        prompts?: Array<TextPrompt | SelectPrompt>
        authorize(inputs?: Record<string, string>): Promise<AuthOuathResult>
      }
    | {
        type: "api"
        label: string
        prompts?: Array<TextPrompt | SelectPrompt>
        authorize?(inputs?: Record<string, string>): Promise<AuthResult>
      }
  )[]
}

export type AuthOuathResult = { url: string; instructions: string } & (
  | {
      method: "auto"
      callback(): Promise<AuthResult>
    }
  | {
      method: "code"
      callback(code: string): Promise<AuthResult>
    }
)

type AuthResult =
  | ({
      type: "success"
      provider?: string
    } & (
      | {
          refresh: string
          access: string
          expires: number
        }
      | { key: string }
    ))
  | {
      type: "failed"
    }
```

### Prompt Types

```typescript
type TextPrompt = {
  type: "text"
  key: string
  message: string
  placeholder?: string
  validate?: (value: string) => string | undefined
  condition?: (inputs: Record<string, string>) => boolean
}

type SelectPrompt = {
  type: "select"
  key: string
  message: string
  options: Array<{
    label: string
    value: string
    hint?: string
  }>
  condition?: (inputs: Record<string, string>) => boolean
}
```

## Provider Context

When a user authenticates, your plugin receives provider info in the context:

```typescript
export const MyPlugin: Plugin = async (ctx) => {
  // ctx.provider is set after authentication
  if (ctx.provider) {
    console.log("Provider:", ctx.provider.provider)
    console.log("Credentials:", ctx.provider.value)
  }

  return {
    // Plugin hooks
  }
}
```

Source: `packages/plugin/src/index.ts:18-32`

## Official Auth Plugins

OpenCode ships with two auth plugins loaded by default. These serve as working examples of auth hook implementations.

**opencode-copilot-auth** ([npm](https://www.npmjs.com/package/opencode-copilot-auth))

- OAuth device code flow for GitHub Copilot
- Supports GitHub.com and GitHub Enterprise
- Token refresh via Copilot API
- Zero-cost billing for Copilot subscribers

**opencode-anthropic-auth** ([npm](https://www.npmjs.com/package/opencode-anthropic-auth))

- PKCE OAuth for Claude Pro/Max subscriptions
- API key creation via Anthropic Console OAuth
- Manual API key entry fallback
- Zero-cost billing for Pro/Max subscribers

Both are written in JavaScript with JSDoc type annotations. View the source on unpkg:

- [opencode-copilot-auth/index.mjs](https://unpkg.com/opencode-copilot-auth/index.mjs)
- [opencode-anthropic-auth/index.mjs](https://unpkg.com/opencode-anthropic-auth/index.mjs)

To disable these default plugins:

```bash
OPENCODE_DISABLE_DEFAULT_PLUGINS=1 opencode
```

## Further Reading

For MCP server authentication patterns, see the MCP documentation:

- MCP servers can request auth through the plugin's auth hook
- The plugin becomes an auth provider for the MCP server
- Credentials flow from user → plugin → MCP server

This deep dive can be added in a future appendix on MCP integration patterns.
