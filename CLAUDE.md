# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Is

UIGen is an AI-powered React component generator with live preview. Users describe components in natural language, Claude generates code via tool calls, and the output renders instantly in an iframe sandbox.

## Commands

```bash
npm run setup        # First-time setup: install deps, generate Prisma client, run migrations
npm run dev          # Dev server with Turbopack
npm run build        # Production build
npm run lint         # ESLint
npm run test         # Run all Vitest tests
npm run db:reset     # Wipe and recreate the database
```

**Single test file:**
```bash
npx vitest src/components/chat/__tests__/ChatInterface.test.tsx
```

**Environment:** Copy `.env.example` if it exists, or set `ANTHROPIC_API_KEY` and `JWT_SECRET`. Without `ANTHROPIC_API_KEY`, a mock provider returns a static component.

## Architecture

### Data Flow

1. User types prompt. `ChatInterface` calls `/api/chat` via Vercel AI SDK.
2. The API route streams Claude's response with two tools: `str_replace_editor` and `file_manager`.
3. Tool calls mutate the `VirtualFileSystem` (in-memory tree, not disk).
4. `PreviewFrame` detects filesystem changes, re-transforms JSX via Babel standalone in the browser, and re-renders in an isolated iframe.

### Key Files

| Path | Purpose |
|------|---------|
| `src/app/main-content.tsx` | Three-panel layout (chat / preview / code editor) |
| `src/app/api/chat/route.ts` | Streaming AI endpoint, tool definitions |
| `src/lib/contexts/file-system-context.tsx` | VirtualFileSystem React context, single source of truth for generated files |
| `src/lib/contexts/chat-context.tsx` | Wraps Vercel AI SDK `useChat` hook |
| `src/lib/transform/jsx-transformer.ts` | Babel + import map transforms JSX to runnable code in browser |
| `src/lib/prompts/generation.ts` | System prompt sent to Claude |
| `src/lib/tools/` | `str-replace.ts` and `file-manager.ts` tool implementations |
| `src/actions/` | Server actions for auth and project CRUD |
| `src/lib/auth.ts` | JWT sessions in httpOnly cookies |
| `src/lib/provider.ts` | Selects Anthropic vs. mock model based on env |
| `prisma/schema.prisma` | `User` and `Project` models (SQLite) |

### State and Persistence

- **Anonymous users:** Work is tracked in localStorage via `anon-work-tracker.ts`.
- **Authenticated users:** Projects serialise the VirtualFileSystem to JSON and persist to Prisma/SQLite.
- On load, project data deserialises back into the VirtualFileSystem context.

### AI Integration

- Model: `claude-haiku-4-5` (set in `src/lib/provider.ts`).
- Uses `streamText` from Vercel AI SDK with system prompt caching (`ephemeral` cache control).
- Two tools: `str_replace_editor` (targeted edits) and `file_manager` (create/delete).
- Mock provider returns a static component when `ANTHROPIC_API_KEY` is absent.

### Auth

JWT tokens are signed with `JWT_SECRET`, stored as httpOnly cookies, expire in 7 days. Middleware at `src/middleware.ts` protects `/api/*` routes. Server actions in `src/actions/index.ts` handle sign-up/in/out.

### Conventions

- Path alias `@/*` maps to `src/*`.
- Tailwind CSS v4 with CSS variables. Use `cn()` (from `src/lib/utils.ts`) for class composition.
- UI primitives come from Radix UI / shadcn, configured in `components.json`.
- Tests live in `__tests__/` subdirectories beside the code they test.
