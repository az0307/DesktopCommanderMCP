# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

Desktop Commander (`@wonderwhy-er/desktop-commander`) is an MCP (Model Context Protocol) server that gives AI assistants (Claude Desktop, Claude Code, etc.) terminal command execution, process management, and filesystem/text-editing tools. It's built on top of the MCP Filesystem Server pattern with added search-and-replace file editing. Distributed as an npm package, a standalone `dist/index.js` MCP server, a Docker image, and an MCPB bundle.

## Commands

```bash
npm install                 # install dependencies
npm run build                # tsc compile + copy setup/uninstall/tracking scripts and data into dist/
npm run watch                 # tsc --watch
npm run clean                 # remove dist/

npm test                       # npm run build && node test/run-all-tests.js  (runs the full test suite)
node test/<name>.js            # run a single test file directly (after npm run build)
npm run test:debug            # run test runner with --inspect

npm start                      # node dist/index.js — run the built server directly (stdio)
npm run start:debug           # run with --inspect-brk=9229 for Node debugger

npm run setup                  # install + build + node setup-claude-server.js (configures Claude Desktop)
npm run setup:debug           # same, but registers the server in debug mode
npm run remove                 # install + build + node uninstall-claude-server.js

npm run link:local            # build + npm link, for testing a local checkout as the global package
npm run inspector               # npx @modelcontextprotocol/inspector dist/index.js — MCP Inspector UI

npm run build:mcpb            # build the .mcpb bundle (scripts/build-mcpb.cjs)
npm run sync-version           # sync version across package.json/manifest/server files
npm run bump / bump:minor / bump:major   # bump version (scripts/sync-version.js --bump)
npm run release / release:minor / release:major / release:dry / release:mcp  # full automated release (scripts/publish-release.cjs) — bumps version, builds, tests, tags, publishes to npm + MCP Registry

npm run logs:view / logs:analyze / logs:export / logs:clear   # inspect the fuzzy-search debug logs (each runs a build first)
```

There is no lint script and no linter configured. `npm test` requires a build first (it's baked into the script); there's no separate "type-check only" command besides running `tsc` via `npm run build`.

Node.js >= 18 is required (`engines` in package.json). Module type is ESM (`"type": "module"`) — all local imports use `.js` extensions even though the source is `.ts` (TS `Node16` module resolution).

### Tests

`test/run-all-tests.js` auto-discovers tests: every file in `test/` whose name starts with `test` and ends with `.js` (excluding itself), with `test.js` always run first if present, in a spawned subprocess with a pass/fail summary. There is no test framework (no Jest/Mocha) — each `test/test-*.js` file is a standalone Node script with its own assertions that exits non-zero on failure. Tests run against the **built** `dist/` output, not the TypeScript source directly, so always `npm run build` (or just `npm test`, which does it for you) before running tests.

## Architecture

### Entry point and request flow

```
src/index.ts        → process boots, loads config, wires transport, calls server.connect()
src/server.ts        → MCP Server instance: declares the full tool list (ListToolsRequestSchema)
                          and dispatches CallToolRequestSchema via a big switch/case to handlers
src/handlers/*.ts    → one function per tool (or a thin re-export from src/tools/*.ts)
src/tools/*.ts       → the actual tool implementations (filesystem, edit, process, search, config...)
```

- `src/server.ts` is the largest file — it holds the human/LLM-facing description text for every tool (the `description` fields feed prompt engineering for the calling model) and the `CallToolRequestSchema` handler that dispatches by `name` to the right handler. New tools are added in three places: a Zod schema in `src/tools/schemas.ts`, a tool object (with description + `inputSchema: zodToJsonSchema(...)`) in the `allTools` array in `server.ts`, and a `case` in the dispatch switch calling into `src/handlers/`.
- `src/handlers/index.ts` re-exports everything from `filesystem-handlers.ts`, `terminal-handlers.ts`, `process-handlers.ts`, `edit-search-handlers.ts`, `search-handlers.ts`, `history-handlers.ts`. Handlers are thin — most just validate args against a Zod schema and delegate to `src/tools/*.ts` or the manager singletons.
- Tool call results are recorded via `toolHistory` (in-memory, last 1000 calls — backs the `get_recent_tool_calls` tool) and `usageTracker` (drives onboarding/feedback message injection appended to tool responses).

### Core singletons (manager classes)

- **`configManager`** (`src/config-manager.ts`) — persists `ServerConfig` (blockedCommands, defaultShell, allowedDirectories, fileReadLineLimit, fileWriteLineLimit, telemetryEnabled, clientId, currentClient) to `~/.claude-server-commander/config.json` (path defined in `src/config.ts`). Lazily initializes on first `getConfig()`/`getValue()`/`setValue()` call; falls back to an in-memory default config if disk I/O fails.
- **`commandManager`** (`src/command-manager.ts`) — parses a shell command string into its constituent sub-commands (splitting on `;`, `&&`, `||`, `|`, `&`, recursing into `(...)` subshells, stripping env-var prefixes) and checks each against `config.blockedCommands` before a command is allowed to run.
- **`terminalManager`** (`src/terminal-manager.ts`) — spawns processes via `child_process.spawn(command, [], { shell })`, tracks them in a `sessions` map keyed by PID, and detects "process is waiting for input" via regex prompt-pattern matching (`analyzeProcessState` in `src/utils/process-detection.ts`) plus a 100ms periodic check — this is what lets `start_process`/`interact_with_process`/`read_process_output` treat REPLs (python3 -i, node -i, bash, ssh, etc.) as interactive sessions instead of blocking until exit. Completed sessions are cached (last 100) so output can still be read after the process exits.
- **`search-manager.ts`** — backs the streaming `start_search`/`get_more_search_results`/`stop_search`/`list_searches` tools (ripgrep-based, via `@vscode/ripgrep`), returning a session ID immediately and paginating results rather than blocking on the whole search.

### Filesystem safety model

All filesystem tools funnel through `validatePath()` in `src/tools/filesystem.ts`, which resolves the path against `config.allowedDirectories` (empty array = full filesystem access). This restriction applies **only to the filesystem tools** (`read_file`, `write_file`, `list_directory`, etc.) — it does **not** restrict `start_process`/terminal commands, which can access any path the OS user can. `commandManager.validateCommand()` similarly only blocks commands by exact base-command name; both restrictions are explicitly documented (SECURITY.md, README "Important Security Warnings") as bypassable guardrails for the AI, not hardened sandboxing — the only real isolation boundary is the Docker installation option.

### Editing: `edit_block`

`src/tools/edit.ts` implements the search/replace tool (`edit_block`, wired through `src/handlers/edit-search-handlers.ts`). On an exact-match miss it falls back to fuzzy matching (`src/tools/fuzzySearch.ts`, using `fastest-levenshtein`) and logs the attempt (see `src/utils/fuzzySearchLogger.ts`) to `~/.claude-server-commander-logs/fuzzy-search.log` for later analysis via the `npm run logs:*` scripts (`scripts/analyze-fuzzy-logs.js`, `view-fuzzy-logs.js`, `export-fuzzy-logs.js`, `clear-fuzzy-logs.js`). `write_file`/`edit_block` enforce `fileWriteLineLimit` (default 50) to push the calling model toward small, chunked writes.

### Cross-cutting utilities

- `src/utils/system-info.ts` — detects OS/shell and generates the `OS_GUIDANCE`/`DEV_TOOL_GUIDANCE`/`PATH_GUIDANCE` strings injected into tool descriptions in `server.ts`.
- `src/utils/capture.ts` — telemetry (`capture()`, `capture_call_tool()`); respects `config.telemetryEnabled` (default on) and records an opt-out event the moment a user flips it off.
- `src/custom-stdio.ts` — `FilteredStdioServerTransport`, a wrapper around the MCP SDK's stdio transport that defers log/notification sends until after the MCP `initialized` handshake completes (see the `deferLog`/`flushDeferredMessages` dance in both `index.ts` and `server.ts`).
- `src/npm-scripts/setup.ts` / `uninstall.ts` — invoked when the CLI is run as `desktop-commander setup` / `desktop-commander remove` (also exposed as top-level `setup-claude-server.js` / `uninstall-claude-server.js`, copied into `dist/` at build time), used to register/unregister the server in `claude_desktop_config.json`.
- `track-installation.js` (root, copied to `dist/`) runs as the npm `postinstall` hook.

### Config file vs. repo-root `config.json`

Don't confuse the root-level `config.json` in this repo (a minimal example/default blocklist, not read by the running server) with the actual runtime config the server reads/writes at `~/.claude-server-commander/config.json` (path constant: `CONFIG_FILE` in `src/config.ts`). Runtime config is managed exclusively through the `get_config`/`set_config_value` MCP tools or by editing that file directly — never hardcode config values elsewhere.

## Security-relevant conventions

- `allowedDirectories: []` (the default when unset) means **no filesystem restriction** — the same is true if `/` is in the list. Treat this as a known footgun when working on config-related code.
- Directory and command restrictions are best-effort guardrails, not hardened sandboxing (symlinks, command substitution, and terminal access can bypass them) — see `SECURITY.md`. Don't "fix" this by adding false claims of strong sandboxing to docs/descriptions; the documented recommendation for real isolation is the Docker install path.
- Config changes (`set_config_value`) are explicitly meant to happen in a separate chat/session from active file/command work, per the tool's own description and the README's "Best Practices" section — keep that guidance intact if touching the tool description.
