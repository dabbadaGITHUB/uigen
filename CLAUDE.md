# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# Initial setup (install deps + generate Prisma client + run migrations)
npm run setup

# Development server (Turbopack)
npm run dev

# Build for production
npm run build

# Run all tests
npm run test

# Lint
npm run lint

# Reset and re-run database migrations
npm run db:reset
```

All scripts wrap with `NODE_OPTIONS='--require ./node-compat.cjs'` for Node.js compatibility — this is handled automatically via `package.json`.

The app requires `ANTHROPIC_API_KEY` in `.env`. If absent, it falls back to a `MockLanguageModel` that generates a static component without calling the API.

## Architecture

UIGen is a Next.js 15 (App Router) full-stack app that lets users describe React components in natural language and see them generated and previewed live.

### Request flow

1. User types a prompt in **ChatInterface** → sent to `POST /api/chat`
2. The API route calls `streamText()` (Vercel AI SDK) with Claude Haiku 4.5
3. Claude uses two tools to manipulate a **VirtualFileSystem** passed in the request body:
   - `str_replace_editor` — line-based string replacement in files
   - `file_manager` — create/delete/rename files
4. Tool results stream back to the client; **FileSystemContext** applies changes in memory
5. **PreviewFrame** re-renders the iframe whenever the virtual FS changes

### Virtual file system

`VirtualFileSystem` (`src/lib/file-system.ts`) is an in-memory tree. It is never written to disk during a session — it travels as JSON in the API request body and is stored as `Project.data` (JSON string) in SQLite when saved. `App.jsx` is always the required root export.

### State management

Two React contexts carry the app state:

- **FileSystemContext** (`src/lib/contexts/file-system-context.tsx`) — owns the virtual FS and exposes file CRUD helpers
- **ChatContext** (`src/lib/contexts/chat-context.tsx`) — wraps `useChat` from `@ai-sdk/react`; passes the serialized FS with every message

### Authentication

JWT-based (jose), stored in an HttpOnly cookie with 7-day expiry. Anonymous sessions are supported — `userId` is optional on the `Project` model. `src/middleware.ts` protects `/api/projects/*` and `/api/filesystem/*`. Server actions in `src/actions/` handle user and project CRUD.

### Database

Prisma with SQLite (`prisma/dev.db`). Two models:

- **User** — email + bcrypt password
- **Project** — `messages` (JSON string), `data` (serialized VirtualFileSystem JSON), optional `userId`

Prisma client is generated into `src/generated/prisma/`.

### Key directories

| Path | Purpose |
|------|---------|
| `src/app/api/chat/route.ts` | Streaming AI endpoint; tool definitions live here |
| `src/lib/file-system.ts` | VirtualFileSystem class |
| `src/lib/provider.ts` | AI model factory (real vs. mock) |
| `src/lib/prompts/generation.tsx` | System prompt sent to Claude |
| `src/lib/tools/` | `str_replace_editor` and `file_manager` tool implementations |
| `src/lib/contexts/` | FileSystemContext and ChatContext |
| `src/components/chat/` | ChatInterface, MessageList, MessageInput |
| `src/components/editor/` | Monaco CodeEditor + FileTree |
| `src/components/preview/` | PreviewFrame (iframe-based live preview) |
| `src/actions/` | Next.js server actions for projects and auth |

### Path alias

`@/*` maps to `./src/*` (configured in `tsconfig.json`).

### Testing

Vitest with jsdom. Tests live alongside source in `__tests__/` subdirectories. Run a single test file:

```bash
npx vitest run src/components/chat/__tests__/ChatInterface.test.tsx
```

### MCP servers

Four MCP servers are configured for this project (all pre-approved in `.claude/settings.local.json`):

| Server | Command | Use case |
|--------|---------|----------|
| `playwright` | `npx @playwright/mcp@latest` | Browser automation — navigate, click, screenshot for E2E/UI verification |
| `sqlite` | `npx @modelcontextprotocol/server-sqlite prisma/dev.db` | Query `prisma/dev.db` directly to inspect users, projects, and serialized FS data |
| `filesystem` | `npx @modelcontextprotocol/server-filesystem .` | Scoped read/write access to project files |
| `fetch` | `npx @modelcontextprotocol/server-fetch` | Make HTTP requests to test `/api/chat` and other endpoints |

MCP servers load on session start — restart Claude Code after adding new servers.
