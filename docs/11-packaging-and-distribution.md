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

**Discovery details:**

- **Extensions**: Only `.ts` and `.js` files—not `.mjs`, `.cjs`, or `.mts`
- **Top-level only**: Nested directories are not scanned (`plugin/utils/helper.ts` won't be discovered)
- **Symlinks**: Followed (you can symlink plugins from elsewhere)
- **Dot files**: Included (`.my-plugin.ts` works)

If you accidentally create a `plugins/` directory (plural), OpenCode throws a `ConfigDirectoryTypoError` to help you catch the mistake.

See [`packages/opencode/src/config/config.ts:290`](https://github.com/sst/opencode/blob/3efc95b/packages/opencode/src/config/config.ts#L290)

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

### Versioning & Updates

**Pin your versions.** OpenCode uses Bun's lockfile (`~/.cache/opencode/bun.lock`) which pins resolved versions. If you specify a plugin without a version:

```json
{
  "plugin": ["my-opencode-plugin"]
}
```

Bun resolves `latest` **once** and caches that version. Even when newer versions are published, you'll stay on the old one.

**Recommended:**

```json
{
  "plugin": ["my-opencode-plugin@1.0.0"]
}
```

#### Upgrading Plugins

Change the version in your config and restart OpenCode:

```json
// Before
"plugin": ["my-opencode-plugin@1.0.0"]

// After  
"plugin": ["my-opencode-plugin@2.0.0"]
```

OpenCode detects the version mismatch and installs the new version automatically.

#### Stuck on an Old Version?

If you previously used an unpinned version, clear the cache:

```bash
rm -rf ~/.cache/opencode/node_modules ~/.cache/opencode/bun.lock
```

Then restart OpenCode with a pinned version in your config.

## Plugin Discovery

OpenCode discovers plugins from three sources:

1. **Config files**: The `plugin` array in `opencode.json`
2. **Local directories**: Files matching `plugin/*.{ts,js}` in `.opencode/` and `~/.config/opencode/`
3. **Default plugins**: `opencode-copilot-auth` and `opencode-anthropic-auth` (unless disabled)

Config arrays from multiple sources merge and deduplicate. See [`packages/opencode/src/config/config.ts:25-32`](https://github.com/sst/opencode/blob/3efc95b/packages/opencode/src/config/config.ts#L25-L32)

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

See [`packages/opencode/src/config/config.ts:830-836`](https://github.com/sst/opencode/blob/3efc95b/packages/opencode/src/config/config.ts#L830-L836)

### Disabling Default Plugins

Set the environment variable to skip built-in auth plugins:

```bash
OPENCODE_DISABLE_DEFAULT_PLUGINS=1 opencode
```

See [`packages/opencode/src/plugin/index.ts:30-33`](https://github.com/sst/opencode/blob/3efc95b/packages/opencode/src/plugin/index.ts#L30-L33)

## Dependency Management

### npm Plugin Dependencies

OpenCode installs npm plugins to a shared cache at `~/.cache/opencode/node_modules/`. The installation:

1. Checks if the package and version already exist in cache
2. Runs `bun add --force --exact --cwd <cache> <pkg>@<version>`
3. Retries up to 3 times with linear backoff (500ms, 1000ms, 1500ms)
4. Records the resolved version in the cache's `package.json`

See [`packages/opencode/src/bun/index.ts:63-144`](https://github.com/sst/opencode/blob/3efc95b/packages/opencode/src/bun/index.ts#L63-L144)

### Local Plugin Dependencies

For local plugin directories, OpenCode automatically installs the `@opencode-ai/plugin` SDK:

```bash
bun add @opencode-ai/plugin@<version> --exact
```

This runs when OpenCode loads the plugin directory. A `.gitignore` is created to exclude `node_modules`, `package.json`, and `bun.lock`.

See [`packages/opencode/src/config/config.ts:159-178`](https://github.com/sst/opencode/blob/3efc95b/packages/opencode/src/config/config.ts#L159-L178)

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

See [`packages/opencode/src/plugin/index.ts:14-53`](https://github.com/sst/opencode/blob/3efc95b/packages/opencode/src/plugin/index.ts#L14-L53)

## Plugin Behavior & Isolation

Understanding how plugins interact helps you write robust plugins and debug issues.

### All Exports Are Loaded

OpenCode calls **every exported function** in your module, not just a default export:

```typescript
// Both of these become separate plugin instances
export const PluginA: Plugin = async (ctx) => ({ /* hooks */ })
export const PluginB: Plugin = async (ctx) => ({ /* hooks */ })
```

If you export helper functions, they'll be called too (and likely error). Keep non-plugin exports internal or in separate files.

See [`packages/opencode/src/plugin/index.ts:43-46`](https://github.com/sst/opencode/blob/3efc95b/packages/opencode/src/plugin/index.ts#L43-L46)

**Type exports are safe.** Only function exports are called. Type-only exports and re-exports work fine:

```typescript
// Default plugin export
export default MyPlugin

// Safe: type exports are not called
export type { MyPluginConfig }
export type { SomeType } from "./types"

// Safe: re-exporting types from another module
export { ConfigSchema } from "./config"  // If ConfigSchema is a type/const

// NOT safe: function exports will be called
export function helperFunction() {}      // Called as plugin, will error
export const utility = () => {}          // Called as plugin, will error
```

This pattern is used by [oh-my-opencode](https://github.com/code-yeongyu/oh-my-opencode/blob/master/src/index.ts#L273-L281) to export configuration types alongside the plugin.

### Error Handling: Loading vs Hooks

**Loading errors are NOT caught.** If your plugin throws during `import()` or initialization, it breaks all plugins loaded after yours:

```typescript
// This breaks the plugin chain
export const BadPlugin: Plugin = async (ctx) => {
  throw new Error("oops") // Stops all subsequent plugins from loading
}
```

**Hook errors ARE caught.** Once loaded, if your hook throws during execution, it's logged but doesn't affect other plugins:

```typescript
export const SafePlugin: Plugin = async (ctx) => ({
  event: async ({ event }) => {
    throw new Error("oops") // Logged, but other plugins continue
  },
})
```

See [`packages/opencode/src/plugin/index.ts:64-71`](https://github.com/sst/opencode/blob/3efc95b/packages/opencode/src/plugin/index.ts#L64-L71)

### No Sandboxing

Plugins run in the same process with full access to:

- **File system** via `Bun.$` shell and `Bun.file`
- **Network** (no restrictions)
- **Environment variables** (`process.env`)
- **SDK client** (shared instance)

All plugins receive the same `PluginInput` object. There's no isolation between plugins.

### Tool ID Conflicts

If two plugins register tools with the same ID, **the last one wins**:

```typescript
// Plugin A (loaded first)
Tool.register({ id: "my-tool", ... }) // Registered

// Plugin B (loaded second)  
Tool.register({ id: "my-tool", ... }) // Replaces Plugin A's tool
```

Use unique, namespaced IDs to avoid conflicts (e.g., `myplugin-toolname`).

See [`packages/opencode/src/tool/registry.ts:73-80`](https://github.com/sst/opencode/blob/3efc95b/packages/opencode/src/tool/registry.ts#L73-L80)

### Hook Output Mutation

All plugins receive the **same `output` object** in hook callbacks. Mutations are visible to subsequent plugins:

```typescript
export const PluginA: Plugin = async (ctx) => ({
  config: async (input, output) => {
    output.config.someValue = "modified" // Visible to Plugin B
  },
})
```

This enables plugin composition but requires care—don't assume the output object is pristine.

### No Timeouts

OpenCode does **not** set timeouts on:

- Plugin installation (`bun add`)
- Module import (`import()`)
- Plugin initialization (`fn(input)`)

If your plugin hangs during any of these, OpenCode hangs. Keep initialization fast and defer heavy work to hook callbacks.

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

### Native Dependencies

If your plugin uses native Node modules (packages with compiled binaries like `@ast-grep/napi`, `better-sqlite3`, or `sharp`), additional configuration is required.

**1. Mark native dependencies as external in your build:**

```bash
bun build src/index.ts --outdir dist --external @ast-grep/napi
```

This tells Bun not to bundle the native module—it will be installed separately at runtime.

**2. Add `trustedDependencies` to package.json:**

```json
{
  "name": "my-opencode-plugin",
  "trustedDependencies": [
    "@ast-grep/napi",
    "@ast-grep/cli"
  ],
  "dependencies": {
    "@ast-grep/napi": "^0.40.0"
  }
}
```

The `trustedDependencies` field allows Bun to run post-install scripts for these packages, which is required for native binary installation.

**Real-world example:** [oh-my-opencode](https://github.com/code-yeongyu/oh-my-opencode) uses this pattern for AST-grep integration:
- [`package.json:60-64`](https://github.com/code-yeongyu/oh-my-opencode/blob/master/package.json#L60-L64) — `trustedDependencies`
- [`package.json:19`](https://github.com/code-yeongyu/oh-my-opencode/blob/master/package.json#L19) — build script with `--external`

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
rm -rf ~/.cache/opencode/node_modules ~/.cache/opencode/bun.lock
```

OpenCode reinstalls plugins on next launch. See [Versioning & Updates](#versioning--updates) for version-specific issues.

### Plugin Hangs on Startup

OpenCode has no timeouts for plugin installation or initialization. If startup hangs:

1. **Identify the culprit**: Remove plugins one at a time from your config
2. **Check for blocking calls**: Ensure your plugin's initialization doesn't make slow network requests or wait on external resources
3. **Defer heavy work**: Move expensive operations to hook callbacks rather than the plugin factory function

### One Plugin Breaks Others

Plugin loading errors are not isolated—one bad plugin breaks all plugins loaded after it.

1. **Check load order**: Plugins load in config order, then auto-discovered plugins, then defaults
2. **Look for the first error**: The plugin that throws stops the chain; subsequent plugins won't load
3. **Test in isolation**: Remove other plugins to confirm which one is failing

### Tool Not Appearing

If your registered tool doesn't show up:

1. **Check the ID**: Tool IDs must be unique. Another plugin may have registered the same ID (last one wins)
2. **Use namespaced IDs**: Prefix your tool IDs with your plugin name (e.g., `myplugin-mytool`)
3. **Verify registration**: Ensure `Tool.register()` is called during initialization, not in a hook

### Local Plugin Import Errors

If local plugins fail with confusing import errors:

1. **Check SDK installation**: The `@opencode-ai/plugin` SDK installs silently—if it fails, you get cryptic errors
2. **Reinstall manually**: Run `bun add @opencode-ai/plugin` in your `.opencode/` directory
3. **Check `.gitignore`**: OpenCode creates a `.gitignore` for `node_modules`, `package.json`, and `bun.lock`—don't delete it

## Summary

| Plugin Type | Build Step | Registration | Dependencies |
|-------------|------------|--------------|--------------|
| Local `.ts` | None | `file://path/to/plugin.ts` | Auto-installed SDK |
| Local `.js` | None | `file://path/to/plugin.js` | Auto-installed SDK |
| npm package | None (optional) | `package-name@version` | Installed to cache |
