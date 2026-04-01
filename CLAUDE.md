# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

This is a monorepo containing multiple independent web projects, all in Portuguese (pt-BR):

- **Root `index.html`** — Landing page linking to all projects
- **`buraco/`** — "ATP 2000" card game score tracker (single-file HTML app using Supabase for backend)
- **`seguros/`** — Insurance dashboard (single-file HTML app using Chart.js)
- **`uigen/`** — AI-powered React component generator (Next.js full-stack app)

## Project: buraco/

Single `index.html` file (~2000+ lines). Uses Supabase JS client for data persistence. All CSS, JS, and HTML are inline. No build step — open directly in browser.

## Project: seguros/

Single `index.html` file. Uses Chart.js for data visualization. No build step.

## Project: uigen/

### Commands

```bash
cd uigen
npm run setup          # Install deps + generate Prisma client + run migrations
npm run dev            # Dev server with Turbopack (localhost:3000)
npm run build          # Production build
npm run lint           # ESLint
npm run test           # Vitest (all tests)
npx vitest run <file>  # Run a single test file
npm run db:reset       # Reset SQLite database
```

### Architecture

Next.js 15 App Router with React 19, TypeScript, Tailwind CSS v4, Prisma (SQLite).

**AI chat flow:** Client (`ChatProvider` context) sends messages to `POST /api/chat/route.ts`, which uses Vercel AI SDK's `streamText` with Claude (via `@ai-sdk/anthropic`). The AI has two tools: `str_replace_editor` (create/edit files) and `file_manager` (rename/delete). All file operations happen on a `VirtualFileSystem` (in-memory, no disk writes). Without an `ANTHROPIC_API_KEY` env var, a `MockLanguageModel` returns static responses.

**Key layers:**
- `src/lib/file-system.ts` — `VirtualFileSystem` class: in-memory file tree with serialize/deserialize for persistence
- `src/lib/contexts/file-system-context.tsx` — React context that wraps VirtualFileSystem for client components
- `src/lib/contexts/chat-context.tsx` — React context managing chat state via Vercel AI SDK's `useChat`
- `src/lib/transform/jsx-transformer.ts` — Babel-based JSX/TSX transformer for live preview (runs in browser)
- `src/lib/provider.ts` — LLM provider selection: real Anthropic API or mock fallback
- `src/lib/prompts/generation.tsx` — System prompt for AI component generation
- `src/lib/tools/` — AI tool definitions (`str-replace.ts`, `file-manager.ts`) operating on VirtualFileSystem
- `src/lib/auth.ts` — JWT-based auth using `jose`
- `src/components/preview/PreviewFrame.tsx` — iframe-based live preview with Babel transform

**Data model (Prisma/SQLite):** `User` has many `Project`. Project stores `messages` (JSON string of chat history) and `data` (JSON string of serialized VirtualFileSystem). `userId` is optional (anonymous users supported).

**Prisma client output** is generated to `src/generated/prisma/` (not the default location).

### Testing

Tests use Vitest + Testing Library + jsdom. Test files live alongside source in `__tests__/` directories.
