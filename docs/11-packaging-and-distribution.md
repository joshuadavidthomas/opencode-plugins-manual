# Packaging and Distribution

This document covers how OpenCode loads plugins and how to publish them as npm packages.

## Build Requirements

OpenCode runs on Bun, which compiles TypeScript at import time. **No build step is required** for any plugin type—local or npm.

### Local Plugins (file://)

```typescript
// .opencode/plugin/my-plugin.ts — runs directly, no compilation needed
import type { Plugin } from "@opencode-ai/plugin"

export const MyPlugin: Plugin = async (ctx) => ({
  event: async ({ event }) => {
    console.log(event.type)
  },
})
```

Register with a `file://` path:

```json
{
  "plugin": ["file://.opencode/plugin/my-plugin.ts"]
}
```

OpenCode discovers local plugins via the glob pattern `plugin/*.{ts,js}` in:
- `.opencode/` (project directory)
- `~/.config/opencode/` (global directory)

See packages/opencode/src/config/config.ts:290

### npm Plugins

npm plugins work the same way—Bun compiles TypeScript when it imports the package. Point your `main` field directly at your TypeScript source:

```json
{
  "name": "my-opencode-plugin",
  "version": "1.0.0",
  "type": "module",
  "main": "./src/index.ts",
  "files": ["src"]
}
```

Register by package name with optional version:

```json
{
  "plugin": ["my-opencode-plugin@1.0.0"]
}
```

If you omit the version, OpenCode installs `latest`.

## Plugin Discovery

OpenCode discovers plugins from three sources:

1. **Config files**: The `plugin` array in `opencode.json`
2. **Local directories**: Files matching `plugin/*.{ts,js}` in `.opencode/` and `~/.config/opencode/`
3. **Default plugins**: `opencode-copilot-auth` and `opencode-anthropic-auth` (unless disabled)

Config arrays from multiple sources merge and deduplicate. See packages/opencode/src/config/config.ts:25-32

### Path Resolution

Relative `file://` paths resolve from the config file's directory via `import.meta.resolve()`. Absolute paths work as-is.

```json
{
  "plugin": [
    "file://.opencode/plugin/local.ts",
    "file:///absolute/path/to/plugin.ts"
  ]
}
```

See packages/opencode/src/config/config.ts:830-836

### Disabling Default Plugins

Set the environment variable to skip built-in auth plugins:

```bash
OPENCODE_DISABLE_DEFAULT_PLUGINS=1 opencode
```

See packages/opencode/src/plugin/index.ts:30-33

## Dependency Management

### npm Plugin Dependencies

OpenCode installs npm plugins to a shared cache at `~/.cache/opencode/node_modules/`. The installation:

1. Checks if the package and version already exist in cache
2. Runs `bun add --force --exact --cwd <cache> <pkg>@<version>`
3. Retries up to 3 times with exponential backoff (500ms, 1000ms, 1500ms)
4. Records the resolved version in the cache's `package.json`

See packages/opencode/src/bun/index.ts:63-144

### Local Plugin Dependencies

For local plugin directories, OpenCode automatically installs the `@opencode-ai/plugin` SDK:

```bash
bun add @opencode-ai/plugin@<version> --exact
```

This runs when OpenCode loads the plugin directory. A `.gitignore` is created to exclude `node_modules`, `package.json`, and `bun.lock`.

See packages/opencode/src/config/config.ts:159-178

## Loading Sequence

OpenCode loads plugins once at startup in this order:

```
1. Config.get()
   ├── Load opencode.json from project and global dirs
   ├── Merge plugin arrays (deduplicate)
   ├── Resolve file:// paths via import.meta.resolve()
   └── Discover local plugins via glob

2. Plugin.state()
   ├── Build final plugin list
   ├── Add default plugins (if not disabled)
   │
   └── For each plugin:
       ├── If file:// → import() directly (Bun compiles TS)
       └── If npm → BunProc.install() → import() module path

3. For each exported Plugin function:
   └── Call fn(PluginInput) → get Hooks object

4. Plugin.init()
   ├── Call hook.config?(config) for each plugin
   └── Subscribe hook.event?() to event bus
```

See packages/opencode/src/plugin/index.ts:14-53

## Publishing to npm

### Project Structure

```
my-opencode-plugin/
├── src/
│   └── index.ts
├── package.json
└── tsconfig.json    # Optional, for editor support
```

### package.json

```json
{
  "name": "my-opencode-plugin",
  "version": "1.0.0",
  "type": "module",
  "main": "./src/index.ts",
  "files": ["src"],
  "peerDependencies": {
    "@opencode-ai/plugin": "*"
  },
  "devDependencies": {
    "@opencode-ai/plugin": "latest"
  }
}
```

### tsconfig.json (optional, for editor support)

```json
{
  "compilerOptions": {
    "target": "ESNext",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "strict": true,
    "noEmit": true
  },
  "include": ["src"]
}
```

### src/index.ts

```typescript
import type { Plugin } from "@opencode-ai/plugin"

export const MyPlugin: Plugin = async (ctx) => ({
  // hooks
})
```

### Publish

```bash
# Test locally first
# In your project's opencode.json:
# "plugin": ["file:///path/to/my-opencode-plugin/src/index.ts"]

# Publish
npm publish
```

### Testing Before Publish

Test your plugin locally via `file://` path:

```json
{
  "plugin": ["file:///absolute/path/to/my-opencode-plugin/src/index.ts"]
}
```

Restart OpenCode and verify your hooks work. Check logs for loading errors.

### Optional: Pre-compiled Build

If you prefer to compile TypeScript before publishing (for faster cold starts, IDE tooling, or personal preference), add a build step:

**package.json** (compiled version):

```json
{
  "name": "my-opencode-plugin",
  "version": "1.0.0",
  "type": "module",
  "main": "./dist/index.js",
  "types": "./dist/index.d.ts",
  "files": ["dist"],
  "scripts": {
    "build": "tsc",
    "prepublishOnly": "npm run build"
  },
  "peerDependencies": {
    "@opencode-ai/plugin": "*"
  },
  "devDependencies": {
    "@opencode-ai/plugin": "latest",
    "typescript": "^5.0.0"
  }
}
```

**tsconfig.json**:

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "declaration": true,
    "outDir": "dist",
    "strict": true
  },
  "include": ["src"]
}
```

The official OpenCode plugins (`opencode-copilot-auth`, `opencode-anthropic-auth`) use pre-compiled JavaScript (`.mjs`), but this is a choice rather than a requirement.

### Examples

The official OpenCode auth plugins demonstrate published plugin patterns:

- **[opencode-copilot-auth](https://unpkg.com/opencode-copilot-auth/)** - Single JavaScript file with JSDoc types
- **[opencode-anthropic-auth](https://unpkg.com/opencode-anthropic-auth/)** - Single JavaScript file with JSDoc types

Both use `.mjs` with minimal `package.json` configurations. View the source to see working plugin patterns.

## Troubleshooting

### Plugin Not Loading

1. **Check the path**: `file://` paths must point to an existing file
2. **Check exports**: The module must export `Plugin` functions
3. **Check logs**: OpenCode logs `loading plugin { path: '...' }` for each plugin

### npm Install Fails

OpenCode retries installations 3 times. If all attempts fail, check:

- Network connectivity
- Package name spelling
- Version exists on npm registry

The error `BunInstallFailedError` includes the package name and version.

### TypeScript Errors

Bun reports syntax errors at import time for both local and npm plugins. Fix the error and restart OpenCode.

If you want to catch errors before publishing an npm plugin, run `tsc --noEmit` to type-check without compiling.

### Cache Issues

Clear the plugin cache:

```bash
rm -rf ~/.cache/opencode/node_modules
```

OpenCode reinstalls plugins on next launch.

## Summary

| Plugin Type | Build Step | Registration | Dependencies |
|-------------|------------|--------------|--------------|
| Local `.ts` | None | `file://path/to/plugin.ts` | Auto-installed SDK |
| Local `.js` | None | `file://path/to/plugin.js` | Auto-installed SDK |
| npm package | None (optional) | `package-name@version` | Installed to cache |
