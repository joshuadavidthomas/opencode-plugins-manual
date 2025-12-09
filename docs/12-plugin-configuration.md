# Plugin Configuration

Complex plugins often need user-customizable settings:

- Enable/disable features
- Override default values (models, timeouts, thresholds)
- Provide API keys or credentials
- Customize agent behavior

Rather than hardcoding values, expose them through a configuration file that users can modify.

## The Configuration Pattern

The recommended pattern, demonstrated by [oh-my-opencode](https://github.com/code-yeongyu/oh-my-opencode):

1. Define a configuration schema with [Zod](https://zod.dev)
2. Load config from a JSON file in the project or global directory
3. Provide a JSON Schema for IDE autocomplete
4. Use sensible defaults when no config exists

## Defining a Configuration Schema

Use Zod to define your configuration structure with types and validation:

```typescript
// src/config/schema.ts
import { z } from "zod"

export const MyPluginConfigSchema = z.object({
  // Optional JSON Schema reference for IDE autocomplete
  $schema: z.string().optional(),
  
  // Feature flags
  enabled_features: z.array(z.string()).optional(),
  disabled_features: z.array(z.string()).optional(),
  
  // Numeric settings with constraints
  context_threshold: z.number().min(0).max(1).optional(),
  timeout_ms: z.number().min(1000).max(60000).optional(),
  
  // Nested configuration
  agents: z.record(z.string(), z.object({
    model: z.string().optional(),
    temperature: z.number().min(0).max(2).optional(),
    disabled: z.boolean().optional(),
  })).optional(),
})

// Infer TypeScript type from schema
export type MyPluginConfig = z.infer<typeof MyPluginConfigSchema>
```

**Real-world example:** [oh-my-opencode/src/config/schema.ts](https://github.com/code-yeongyu/oh-my-opencode/blob/master/src/config/schema.ts)

## Loading Configuration

Load configuration from the project directory with fallback to defaults:

```typescript
// src/config/loader.ts
import * as fs from "fs"
import * as path from "path"
import { MyPluginConfigSchema, type MyPluginConfig } from "./schema"

export function loadConfig(directory: string): MyPluginConfig {
  // Check multiple possible locations
  const configPaths = [
    path.join(directory, "my-plugin.json"),           // Project root
    path.join(directory, ".my-plugin.json"),          // Hidden file
    path.join(directory, ".opencode", "my-plugin.json"), // .opencode directory
  ]

  for (const configPath of configPaths) {
    try {
      if (fs.existsSync(configPath)) {
        const content = fs.readFileSync(configPath, "utf-8")
        const rawConfig = JSON.parse(content)
        
        // Validate with Zod
        const result = MyPluginConfigSchema.safeParse(rawConfig)
        
        if (!result.success) {
          console.warn(`Config validation error in ${configPath}:`, result.error.issues)
          return {} // Return empty config on validation error
        }
        
        return result.data
      }
    } catch (error) {
      // Ignore parse errors, continue to next location
    }
  }

  // No config found, return empty (use defaults)
  return {}
}
```

**Real-world example:** [oh-my-opencode/src/index.ts:43-69](https://github.com/code-yeongyu/oh-my-opencode/blob/master/src/index.ts#L43-L69)

## Using Configuration in Your Plugin

Load configuration during plugin initialization:

```typescript
// src/index.ts
import type { Plugin } from "@opencode-ai/plugin"
import { loadConfig } from "./config/loader"
import { createFeatureA } from "./features/feature-a"
import { createFeatureB } from "./features/feature-b"

export const MyPlugin: Plugin = async (ctx) => {
  // Load configuration from project directory
  const config = loadConfig(ctx.directory)
  
  // Use config to conditionally enable features
  const featureA = config.disabled_features?.includes("feature-a")
    ? null
    : createFeatureA(ctx, config)
  
  const featureB = config.disabled_features?.includes("feature-b")
    ? null
    : createFeatureB(ctx, config)

  return {
    event: async (input) => {
      await featureA?.event?.(input)
      await featureB?.event?.(input)
    },
    
    "tool.execute.after": async (input, output) => {
      // Use config values
      if (config.context_threshold) {
        // Apply threshold logic
      }
    },
  }
}

export default MyPlugin
```

## Global vs Project Configuration

Support both global (user-wide) and project-specific configuration:

```typescript
import * as os from "os"
import * as path from "path"

export function loadConfig(projectDirectory: string): MyPluginConfig {
  // Global config location
  const globalConfigPath = path.join(
    os.homedir(),
    ".config",
    "opencode",
    "my-plugin.json"
  )
  
  // Project config location
  const projectConfigPath = path.join(projectDirectory, "my-plugin.json")
  
  // Load global config first
  let config: MyPluginConfig = {}
  if (fs.existsSync(globalConfigPath)) {
    const globalConfig = loadConfigFile(globalConfigPath)
    if (globalConfig) config = globalConfig
  }
  
  // Merge project config (overrides global)
  if (fs.existsSync(projectConfigPath)) {
    const projectConfig = loadConfigFile(projectConfigPath)
    if (projectConfig) {
      config = { ...config, ...projectConfig }
    }
  }
  
  return config
}
```

**Precedence:** Project config overrides global config, which overrides defaults.

## JSON Schema for IDE Autocomplete

Provide a JSON Schema so users get autocomplete and validation in their editor:

**1. Generate schema from Zod:**

```typescript
// script/build-schema.ts
import { zodToJsonSchema } from "zod-to-json-schema"
import { MyPluginConfigSchema } from "../src/config/schema"
import * as fs from "fs"

const jsonSchema = zodToJsonSchema(MyPluginConfigSchema, "MyPluginConfig")

fs.writeFileSync(
  "assets/my-plugin.schema.json",
  JSON.stringify(jsonSchema, null, 2)
)
```

**2. Host the schema** (options):
- Include in your npm package and reference via unpkg
- Host on GitHub raw content
- Include in your repository's assets

**3. Users reference the schema:**

```json
{
  "$schema": "https://unpkg.com/my-opencode-plugin/assets/my-plugin.schema.json",
  "context_threshold": 0.7,
  "disabled_features": ["feature-a"]
}
```

**Real-world example:** 
- Schema generation: [oh-my-opencode/script/build-schema.ts](https://github.com/code-yeongyu/oh-my-opencode/blob/master/script/build-schema.ts)
- Generated schema: [oh-my-opencode/assets/oh-my-opencode.schema.json](https://github.com/code-yeongyu/oh-my-opencode/blob/master/assets/oh-my-opencode.schema.json)

## Exporting Configuration Types

Export your configuration types so users can reference them in TypeScript:

```typescript
// src/index.ts
export default MyPlugin

// Export types for users who want type-safe config
export type { MyPluginConfig } from "./config/schema"
export { MyPluginConfigSchema } from "./config/schema"
```

Type exports are safe—they don't get called as plugins. See [Packaging and Distribution](11-packaging-and-distribution.md#type-exports-are-safe).

## Configuration File Locations

Recommended locations (in order of precedence):

| Location | Scope | Use Case |
|----------|-------|----------|
| `./my-plugin.json` | Project | Project-specific settings |
| `./.opencode/my-plugin.json` | Project | Keep config with other OpenCode files |
| `~/.config/opencode/my-plugin.json` | Global | User-wide defaults |

## Best Practices

1. **Validate early.** Parse and validate config during plugin initialization, not during hook execution.

2. **Fail gracefully.** Invalid config should log a warning and fall back to defaults, not crash.

3. **Version your schema.** When adding breaking changes, consider versioning your config format.

**Reference implementation:** [oh-my-opencode](https://github.com/code-yeongyu/oh-my-opencode)
