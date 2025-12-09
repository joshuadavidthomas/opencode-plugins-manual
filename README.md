# OpenCode Plugins: The Missing Manual

Documentation for OpenCode plugins, including undocumented features and Claude Code migration patterns.

**Source commit**: [3efc95b](https://github.com/sst/opencode/tree/3efc95b157e05bc3c42554be1b5778f8f1b64cfe)<br />
**Last refreshed**: 2025-12-09

## What This Is

The official OpenCode plugin documentation covers basics, but requires examining source code or navigating TypeScript types to understand:

- Config hook injection (commands, agents, MCP servers)
- Full hooks interface reference
- Event system details
- SDK client documentation
- Claude Code migration patterns

This manual fills those gaps with working examples and source references.

I created this because I wanted to write plugins but grew tired of thin documentation and constant source diving, whether searching manually or using an LLM to research the codebase.

I've tried to establish guardrails around the documentation process by providing direct source access, using structured prompts, and coordinating specialized agents to verify examples. However, these docs are entirely LLM-generated and may contain hallucinations or inaccuracies. Always verify against the OpenCode source code, and please open an issue if you find errors.

## Contents

- **[00-about.md](docs/00-about.md)** - How this manual was generated
- **[01-overview.md](docs/01-overview.md)** - What plugins are, capabilities, minimal examples
- **[02-quick-start.md](docs/02-quick-start.md)** - Plugin locations, basic structure, tutorial
- **[03-plugin-context.md](docs/03-plugin-context.md)** - PluginInput deep dive
- **[04-hooks-reference.md](docs/04-hooks-reference.md)** - All 10 hooks with signatures
- **[05-config-hook.md](docs/05-config-hook.md)** - **Undocumented**: Injecting commands/agents/MCP
- **[06-custom-tools.md](docs/06-custom-tools.md)** - Tool helper, Zod patterns
- **[07-events.md](docs/07-events.md)** - Event system with 33+ events
- **[08-sdk-client.md](docs/08-sdk-client.md)** - Client API overview
- **[09-auth-hook.md](docs/09-auth-hook.md)** - Auth providers (OAuth/API key)
- **[10-claude-code-migration.md](docs/10-claude-code-migration.md)** - Feature mapping for Claude Code authors
- **[11-packaging-and-distribution.md](11-packaging-and-distribution.md)** - Build requirements, plugin discovery, npm publishing

### Appendices

- [config-schemas.md](docs/appendix/config-schemas.md) - Full schemas
- [event-schemas.md](docs/appendix/event-schemas.md) - Event payloads
- [sdk-reference.md](docs/appendix/sdk-reference.md) - All SDK methods

### Artifacts

For those curious about how this manual was created, the artifacts, commands, and agents are contained in the [artifacts/](artifacts/) directory, including:

- [Research](artifacts/research.md) - OpenCode vs Claude Code analysis
- [Plan](artifacts/plan.md) - Execution plan with subagent prompts
- [Commands](artifacts/commands/), [Skills](artifacts/skills/), [Agents](artifacts/agents/) - Tools used

## Contributing

Found an error or outdated info? Open an issue with the file, section, and suggested correction.

## License

opencode-plugins-manual is licensed under the MIT license. See the [`LICENSE`](LICENSE) file for more information.

---

opencode-plugins-manual is not built by, or affiliated with, the OpenCode team.

OpenCode is ©2025 Anomaly.
