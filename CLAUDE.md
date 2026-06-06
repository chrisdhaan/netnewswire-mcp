# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

MCP server that bridges Claude to [NetNewsWire](https://netnewswire.com/) via AppleScript on macOS. All communication with NetNewsWire happens through `osascript`; there is no HTTP API or native addon.

## Commands

```bash
npm run build          # Compile TypeScript → dist/
npm run dev            # Watch mode (recompiles on save)
npm start              # Run the compiled server (requires dist/)
npm run lint           # ESLint (typescript-eslint recommended rules)
npm test               # Vitest unit tests (parsers only, no NNW required)
npx tsx scripts/e2e-test.ts   # Full E2E test — requires NetNewsWire running
```

Run a single test file or pattern:
```bash
npx vitest run src/parsers.test.ts
```

## Architecture

### Data Flow

```
Claude (MCP client)
  ↓ JSON-RPC over stdio
src/index.ts          — entry point, wires McpServer to StdioServerTransport
src/server.ts         — registers all tools, calls ensureRunning() before each
src/applescript/bridge.ts     — executes osascript, handles -600/not-running errors
src/applescript/scripts.ts    — AppleScript template strings (one function per tool)
src/parsers.ts        — converts AppleScript text output → typed TypeScript objects
```

### AppleScript Output Protocol

Scripts emit structured text that parsers convert to objects:

- **`listFeeds`** — `ACCOUNT:name|active`, `FOLDER:name`, `FEED:name|url|homepage` (pipe-delimited, one per line)
- **`getArticles` / `searchArticles`** — `ARTICLE:` lines using ASCII unit separator `\x1f` (char 31) as delimiter to avoid collisions with pipe characters in titles/URLs/summaries
- **`readArticle`** — `KEY:value` lines (colon-prefix), with multi-line values accumulated until the next key
- **`markArticles`** — returns `MARKED:N` count string
- **`markAllUnread`** — uses UI scripting (clicks row 3 "All Unread" in sidebar, triggers `Article > Mark All as Read` menu item); instant regardless of library size, no article iteration

### Key Constraints

**Performance on large iCloud libraries**: Full library traversal via AppleScript times out. Mitigations already in place:
- `readArticle` and `markArticles` accept an optional `folderName` hint to scope the search to a single folder before falling back to full traversal.
- `getArticles` uses AppleScript `where` clauses for unread/starred filtering.
- The bridge uses a 60-second timeout and a 10 MB stdout buffer.
- `markAllUnread` intentionally avoids iteration by using UI scripting instead.

**iCloud account feed duplication**: `listFeeds` uses `every feed of acct` (top-level feeds only), not `allFeeds`, because `allFeeds` on iCloud accounts includes folder feeds, causing duplicates.

**AppleScript injection**: All user-supplied strings are sanitized by `escapeForAppleScript()` in `scripts.ts` before being interpolated into script templates.

### NetNewsWire Scripting Dictionary

Key object properties (from NNW's sdef):
- **Article**: `id`, `title`, `url`, `external url`, `contents` (plain text), `html`, `summary`, `published date`, `arrived date`, `read` (r/w), `starred` (r/w), `feed`
- **Feed**: `url`, `name`, `homepage url`, `icon url`, `favicon url`, `articles`, `authors`
- **Folder**: `name`, `id`, `feeds`, `articles`
- **Account**: `name`, `id`, `active`, `allFeeds`, `folders`, `opml representation`

### Adding a New Tool

1. Add a script template function to `src/applescript/scripts.ts`
2. Add a parser (if the output format is new) to `src/parsers.ts` with tests in `src/parsers.test.ts`
3. Register the tool in `src/server.ts` using `server.tool(name, description, zodSchema, handler)`
4. Add the tool entry to `manifest.json`

### Distribution

- `npm publish` — publishes to npm as `@jellllly/netnewswire-mcp`
- `.mcpb` bundle — created for Claude Desktop one-click install; uses `.mcpbignore` to exclude dev files
- CI: `.github/workflows/ci.yml` (lint + test), `.github/workflows/release.yml` (npm publish on tag)
