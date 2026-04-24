# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# First-time setup (install deps + generate Prisma client + run migrations)
npm run setup

# Development server (Turbopack)
npm run dev

# Build for production
npm run build

# Run all tests
npm test

# Run a single test file
npx vitest run src/components/chat/__tests__/ChatInterface.test.tsx

# Lint
npm run lint

# Reset database (drops all data and re-runs migrations)
npm run db:reset
```

The dev script requires `NODE_OPTIONS='--require ./node-compat.cjs'` (already in the npm script) to suppress Node 25+ Web Storage SSR errors.

## Architecture

### Request flow

1. User types a message → `ChatProvider` (wraps `useAIChat` from `@ai-sdk/react`) → POST `/api/chat`
2. The API route streams back via Vercel AI SDK `streamText` using the Claude model (or a `MockLanguageModel` when `ANTHROPIC_API_KEY` is absent)
3. Claude calls one of two tools during generation: `str_replace_editor` or `file_manager`
4. Tool calls are intercepted client-side via `onToolCall` and forwarded to `FileSystemContext.handleToolCall`, which mutates the in-memory `VirtualFileSystem`
5. `FileSystemContext` increments `refreshTrigger`, which causes `PreviewFrame` to rebuild the preview

### Virtual file system

`src/lib/file-system.ts` — `VirtualFileSystem` is a fully in-memory tree (no disk I/O). It is serialized to JSON and sent with every chat request as `files`, then deserialized server-side to give the model an accurate view of current state. Persistence for authenticated users is handled by saving the serialized output to the `Project.data` column in SQLite.

### Preview pipeline

`src/lib/transform/jsx-transformer.ts` — On every `refreshTrigger`:
- All `.jsx`/`.tsx`/`.js`/`.ts` files are transformed via `@babel/standalone` (runs in the browser)
- Each transformed file becomes a blob URL
- An ES module import map is constructed mapping package names and file paths to blob URLs; third-party packages without local files resolve to `https://esm.sh/<package>`
- The import map + a `<script type="module">` bootstrapping React are injected into `iframe.srcdoc`
- Tailwind CSS is loaded from CDN in the iframe

### AI tools exposed to Claude

- `str_replace_editor` (`src/lib/tools/str-replace.ts`): commands `view`, `create`, `str_replace`, `insert` — operates on the server-side `VirtualFileSystem` instance and returns result strings
- `file_manager` (`src/lib/tools/file-manager.ts`): commands `rename`, `delete`

### Context providers

- `FileSystemProvider` (`src/lib/contexts/file-system-context.tsx`) — owns the singleton `VirtualFileSystem` and exposes file CRUD + `handleToolCall`
- `ChatProvider` (`src/lib/contexts/chat-context.tsx`) — wraps `useAIChat`; nests inside `FileSystemProvider` so it can access `fileSystem.serialize()` to include current files in every request body

### Auth

`src/lib/auth.ts` — JWT stored in an httpOnly cookie (`auth-token`), signed with `JWT_SECRET` env var. Sessions expire after 7 days. `src/middleware.ts` protects routes. Anonymous users can work without signing in; their session state is tracked via `src/lib/anon-work-tracker.ts`.

### Database

Prisma with SQLite (`prisma/dev.db`). Two models: `User` and `Project`. `Project.messages` and `Project.data` are JSON stored as strings. Prisma client is generated to `src/generated/prisma`.

### Provider / model selection

`src/lib/provider.ts` — returns `anthropic("claude-haiku-4-5")` when `ANTHROPIC_API_KEY` is set; otherwise returns `MockLanguageModel`, which replays hardcoded counter/form/card component sequences without hitting the API.

### Testing

Vitest with jsdom + React Testing Library. Config in `vitest.config.mts`. Tests live alongside source in `__tests__` subdirectories.
