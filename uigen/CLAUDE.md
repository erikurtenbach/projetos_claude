# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
npm run setup
npm run dev
npm run build
npm run lint
npm run test
npx vitest run <file>
npm run db:reset
```

## Architecture

Next.js 15 App Router with React 19, TypeScript, Tailwind CSS v4, Prisma (SQLite). Path alias `@/*` maps to `./src/*`.

### AI Chat Flow

1. `ChatProvider` (`src/lib/contexts/chat-context.tsx`) uses Vercel AI SDK's `useAIChat` hook to send messages to `POST /api/chat`
2. Request body includes: messages, serialized files (from VirtualFileSystem), and projectId
3. `src/app/api/chat/route.ts` prepends the system prompt (`src/lib/prompts/generation.tsx`), then calls `streamText` with two tools:
   - **str_replace_editor** (`src/lib/tools/str-replace.ts`) — view, create, str_replace, insert, undo_edit
   - **file_manager** (`src/lib/tools/file-manager.ts`) — rename, delete
4. Tool calls are received client-side via `handleToolCall` in ChatProvider, which applies them to the VirtualFileSystem
5. If the user is authenticated, chat history and file state are saved to the database after each response

Without `ANTHROPIC_API_KEY`, `src/lib/provider.ts` returns a `MockLanguageModel` that simulates a 4-step component creation flow with realistic tool calls (max 4 steps instead of 40).

### Virtual File System

`src/lib/file-system.ts` — In-memory file tree, never writes to disk. Core abstraction used throughout:
- Stores files as `Map<string, FileNode>` where FileNode has type (file/directory), content, and optional children
- Serializes to/from JSON for database persistence (Project.data column)
- Provides text editor commands: `viewFile`, `replaceInFile`, `insertInFile` (used by AI tools)
- `FileSystemProvider` context (`src/lib/contexts/file-system-context.tsx`) wraps it for React components

### Live Preview System

`src/lib/transform/jsx-transformer.ts` is the core of the preview pipeline:
- `transformJSX()` — Babel-powered JSX/TSX to plain JS transformation (runs in browser)
- `createImportMap()` — Builds ES module import map: local files become blob URLs, third-party packages resolve to esm.sh CDN, `@/` alias is handled, missing imports get placeholder modules, CSS files are collected into a single stylesheet
- `createPreviewHTML()` — Generates full HTML for iframe srcdoc: import map, Tailwind CDN, error boundary, dynamic module loading

`src/components/preview/PreviewFrame.tsx` renders the iframe with this generated HTML.

### Layout

`src/app/main-content.tsx` — Resizable 2-panel layout:
- Left (35%): Chat interface (`ChatInterface.tsx`)
- Right (65%): Preview tab (default) or Code tab. Code tab has nested resizable panels: file tree (30%) + Monaco editor (70%)

### Authentication

- Server actions in `src/actions/` handle signUp, signIn, signOut
- `src/lib/auth.ts` — JWT sessions via `jose`, stored in httpOnly cookies, 7-day expiration. `JWT_SECRET` is auto-generated in dev mode
- `src/middleware.ts` — Protects `/api/projects` and `/api/filesystem` routes
- Anonymous users: work tracked in sessionStorage via `src/lib/anon-work-tracker.ts`, no database persistence

### Data Model (Prisma)

Schema at `prisma/schema.prisma` — always reference it to understand the database structure. Prisma client output goes to `src/generated/prisma/` (not the default location).

### System Prompt

`src/lib/prompts/generation.tsx` instructs the AI to:
- Create React components with Tailwind CSS
- Always use `/App.jsx` as entrypoint with default export
- Use `@/` import alias for non-library files
- Keep responses brief, no summaries unless asked

## Testing

Vitest + React Testing Library + jsdom. Test files live alongside source in `__tests__/` directories. The `vitest.config.mts` configures jsdom environment and React plugin.

## Environment Variables

- `ANTHROPIC_API_KEY` — Optional. Without it, the mock provider is used instead of Claude.
- `JWT_SECRET` — Optional in dev (auto-generated), should be set in production.
