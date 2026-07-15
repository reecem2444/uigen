# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
npm run setup        # First-time setup: install deps, generate Prisma client, run migrations
npm run dev          # Start dev server with Turbopack at http://localhost:3000
npm run build        # Production build
npm run test         # Run Vitest test suite
npm run db:reset     # Reset SQLite database (drops and recreates)
npx prisma migrate dev --name <name>  # Create and apply a new migration
```

To run a single test file: `npx vitest run src/path/to/__tests__/file.test.ts`

## Environment

Copy `.env.txt` to `.env` and add `ANTHROPIC_API_KEY`. Without a key, the app runs in demo mode using a `MockLanguageModel` that generates placeholder components.

## Architecture

UIGen is an AI-powered React component generator with a three-panel interface (chat, code editor, live preview). Users describe components in natural language; Claude generates code that updates a virtual file system displayed in real time.

### Key architectural decisions

**Virtual File System** — No files are written to disk. `src/lib/file-system.ts` provides a Map-based in-memory tree. Files are serialized to JSON and stored in the `Project.data` column in SQLite. The AI tools (`str_replace_editor`, `file_manager`) operate on this virtual FS.

**Live Preview** — `src/lib/transform/jsx-transformer.ts` uses Babel Standalone in the browser to compile JSX, creates blob URLs for each module, and builds an import map pointing package imports to `esm.sh` CDN. `src/components/preview/PreviewFrame.tsx` injects this into a sandboxed iframe via `srcdoc`.

**AI Streaming with Tool Calls** — `src/app/api/chat/route.ts` calls Claude via Vercel AI SDK's `streamText()`. Claude may respond with tool calls (`str_replace_editor` / `file_manager`) that mutate the virtual FS. The file system state is serialized into each request so Claude always has current context. `onFinish` persists the project to the database.

**State Management** — Two React contexts handle shared state:
- `src/lib/contexts/chat-context.tsx` — wraps Vercel AI's `useAIChat()`, routes tool call results to the file system
- `src/lib/contexts/file-system-context.tsx` — holds the `VirtualFileSystem` instance, processes AI tool calls, triggers re-renders

**Authentication** — JWT sessions stored in httpOnly cookies (`src/lib/auth.ts`). Anonymous sessions are allowed; persistence requires sign-up. The `userId` on `Project` is optional.

**Provider abstraction** — `src/lib/provider.ts` returns a real Claude model when `ANTHROPIC_API_KEY` is set, or a `MockLanguageModel` for demo/test use.

### Data flow for a chat message

1. User submits prompt → `ChatContext` calls Vercel AI's `useAIChat` → POST to `/api/chat`
2. Server builds request with serialized file system + system prompt (Anthropic prompt caching via `experimental_providerMetadata`)
3. Claude streams response; tool calls stream back as delta events
4. Client receives tool calls → `FileSystemContext.handleToolCall()` applies mutations
5. Changed files trigger preview refresh (new blob URLs + iframe reload)
6. On stream completion, server saves updated project to DB

### Entry point and routing

- `/` (`src/app/page.tsx`) — authenticates user, redirects to their latest project or creates one
- `/[projectId]` (`src/app/[projectId]/page.tsx`) — loads project from DB, renders `MainContent`
- `src/app/main-content.tsx` — three-panel resizable layout, wraps both context providers

### AI system prompt

`src/lib/prompts/generation.tsx` instructs Claude to:
- Always create `/App.jsx` as the entry point
- Use Tailwind CSS for all styling (no inline styles)
- Use `@/` alias for local file imports
- Keep prose responses brief; focus on generating code

### Database schema

```
User    { id, email, password, createdAt, updatedAt }
Project { id, name, userId?, messages (JSON), data (JSON), createdAt, updatedAt }
```

Prisma client is generated to `src/generated/prisma/`. Run `npx prisma generate` after schema changes.
