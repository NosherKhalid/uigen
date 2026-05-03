# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# Initial setup
npm run setup          # install deps + prisma generate + migrate

# Development
npm run dev            # Next.js dev server with Turbopack

# Build & start
npm run build
npm run start

# Tests
npm test               # Run all tests with Vitest
npm test -- src/lib/__tests__/file-system.test.ts  # Run single test file
```

Requires `ANTHROPIC_API_KEY` in `.env` for real AI responses. Without it, a mock provider returns static component templates.

## Architecture

UIGen is an AI-powered React component generator. Users describe components in chat; Claude generates and edits files; the result renders live in an iframe.

### Request Flow

```
User chat message
  → ChatContext (useChat via Vercel AI SDK)
  → POST /api/chat
      → getLanguageModel() → Claude API (or MockLanguageModel fallback)
      → streamText() with two tools: str_replace_editor, file_manager
      → on finish: serialize & save to Prisma (authenticated users only)
  → Tool calls handled in ChatContext → FileSystemContext
  → VirtualFileSystem updated in memory
  → PreviewFrame re-renders via jsx-transformer (Babel + iframe)
```

### Key Abstractions

**VirtualFileSystem** (`src/lib/file-system.ts`): In-memory file tree (no disk writes). Supports create/read/update/delete/rename/replace/insert. Serialized as JSON for persistence.

**FileSystemContext** (`src/lib/contexts/file-system-context.tsx`): Wraps VirtualFileSystem in React context. Applies AI tool calls (str_replace_editor, file_manager) to the file tree. Auto-selects entry point (prefers `App.jsx`).

**ChatContext** (`src/lib/contexts/chat-context.tsx`): Wraps Vercel AI SDK's `useChat`. Passes current file system state to `/api/chat`. Bridges tool call results back to FileSystemContext.

**jsx-transformer** (`src/lib/transform/jsx-transformer.ts`): Compiles JSX with Babel Standalone, builds import maps, injects Tailwind CSS, renders into an iframe.

**Provider** (`src/lib/provider.ts`): Returns real `anthropic('claude-haiku-4-5')` if `ANTHROPIC_API_KEY` is set; otherwise returns `MockLanguageModel` cycling through Counter/Form/Card templates.

### Routing

- `/` — Home. Unauthenticated users see MainContent; authenticated users redirect to their latest project or a new one.
- `/[projectId]` — Protected. Loads project from Prisma, deserializes messages + file system, renders full editor UI.
- `/api/chat` — POST endpoint for AI streaming.

### Auth

JWT stored in HTTP-only cookies (7-day expiry, signed with `JWT_SECRET`). `src/middleware.ts` protects `/[projectId]` routes. Server actions in `src/actions/index.ts` handle signUp/signIn/signOut.

### Database

Prisma + SQLite. `Project` stores `messages` and `data` (file system) as JSON strings. `userId` is nullable for anonymous projects that are later claimed.

### UI Layout

Split-panel layout (resizable): left panel = chat, right panel = tabs (Preview | Code). Preview uses iframe; Code uses Monaco editor with a file tree sidebar.

### Testing

Vitest + jsdom + React Testing Library. Tests live in `__tests__/` directories adjacent to the code they test.

## Code Style

Add comments sparingly — only for complex or non-obvious logic.
