# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# Development
npm run dev          # Start dev server with Turbopack (required node-compat.cjs shim)
npm run build        # Production build
npm run lint         # ESLint

# Testing
npm test             # Run Vitest (watch mode)

# Database
npm run setup        # Install deps + generate Prisma client + run migrations
npm run db:reset     # Reset database (destructive)
npx prisma migrate dev   # Run migrations after schema changes
npx prisma generate      # Regenerate client after schema changes
```

## Architecture

UIGen is an AI-powered React component generator. Users describe components in a chat interface; Claude creates/edits files in a virtual file system; a live preview renders the result in a sandboxed iframe.

### Core Flow

1. **Chat** → `src/app/api/chat/route.ts` streams Claude responses via Vercel AI SDK
2. **Tool use** → Claude calls `str_replace_editor` or `file_manager` tools (`src/lib/tools/`) to manipulate files
3. **Virtual FS** → `src/lib/file-system.ts` (`VirtualFileSystem` class) stores all files in memory (nothing written to disk)
4. **Preview** → `src/lib/transform/` transpiles JSX via Babel standalone; rendered in sandboxed iframe with dynamic import maps

### Key Architectural Decisions

- **VirtualFileSystem** (`src/lib/file-system.ts`): In-memory file system using a `Map`. All AI-generated files live here. Supports serialize/deserialize for persistence to Prisma.
- **Streaming AI**: `src/app/api/chat/route.ts` uses `streamText()` from Vercel AI SDK. Tool calls are processed mid-stream (up to 40 steps). Project saved to DB on `onFinish`.
- **Mock provider**: When `ANTHROPIC_API_KEY` is absent, `src/lib/provider.ts` returns a mock that generates static components (4 max steps).
- **Contexts**: `ChatContext` (`src/lib/contexts/chat-context.tsx`) manages messages and the AI streaming loop. `FileSystemContext` (`src/lib/contexts/file-system-context.tsx`) manages VFS state with a `refreshTrigger` counter to force re-renders.
- **Auth**: JWT in HTTP-only cookies (jose + bcrypt). `src/middleware.ts` protects routes. Anonymous usage is supported — projects can have a null `userId`.

### Layout

`src/app/main-content.tsx` renders a 3-panel resizable layout:
- Left (35%): Chat interface
- Right (65%): Tabs — Preview (iframe) | Code (file tree + Monaco editor)

### Database (Prisma + SQLite)

Schema in `prisma/schema.prisma`. Two models:
- `User`: email/password auth
- `Project`: stores serialized `messages` (JSON array) and `data` (serialized VFS JSON)

### Path Alias

`@/*` maps to `src/*` (configured in `tsconfig.json` and `vitest.config.mts`).

### Test Structure

Tests use Vitest + Testing Library. Co-located in `__tests__/` folders next to source:
- `src/lib/__tests__/file-system.test.ts` — VirtualFileSystem unit tests (most comprehensive)
- `src/components/chat/__tests__/` — Chat UI component tests
- `src/lib/contexts/__tests__/` — Context tests
- `src/lib/transform/__tests__/` — JSX transformer tests
