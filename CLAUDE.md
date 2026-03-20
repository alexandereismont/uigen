# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Data Structure Reference

When you need to understand the data models, refer to `prisma/schema.prisma` as the source of truth.

## Code Style

Only add comments for complex or non-obvious logic. Self-explanatory code should not have comments.

## Commands

```bash
# First-time setup (install deps + generate Prisma client + run migrations)
npm run setup

# Development server (Turbopack, http://localhost:3000)
npm run dev

# Run all tests
npm test

# Run a single test file
npx vitest run src/lib/__tests__/file-system.test.ts

# Lint
npm run lint

# Build
npm run build

# Reset database
npm run db:reset

# Regenerate Prisma client after schema changes
npx prisma generate

# Run migrations after schema changes
npx prisma migrate dev
```

Set `ANTHROPIC_API_KEY` in `.env` to use real Claude generation. Without it, the app runs with a mock provider that returns static components.

## Architecture

UIGen is a Next.js 15 app (App Router) that lets users describe React components in chat and see them rendered live. The key architectural insight is a **virtual file system** that lives entirely in memory—no files are written to disk. Claude generates code by calling file-manipulation tools, the client applies those tool calls to the in-memory FS, and the preview iframe renders the result.

### Data Flow

1. **User sends chat message** → `ChatProvider` (`src/lib/contexts/chat-context.tsx`) calls `/api/chat` with messages + serialized virtual FS
2. **API route** (`src/app/api/chat/route.ts`) reconstructs a server-side `VirtualFileSystem`, calls Claude with two tools: `str_replace_editor` and `file_manager`, and streams back the response
3. **Tool calls stream to client** → `onToolCall` in `ChatProvider` → `handleToolCall` in `FileSystemProvider` updates the client-side `VirtualFileSystem`
4. **`refreshTrigger`** increments in `FileSystemContext` → `PreviewFrame` re-renders: serializes all files → Babel transforms JSX → creates blob URLs + import map → sets `iframe.srcdoc`
5. **On finish**, if authenticated + `projectId` present, the API route persists messages and serialized FS to SQLite via Prisma

### Virtual File System (`src/lib/file-system.ts`)

`VirtualFileSystem` is a `Map<string, FileNode>`-backed tree with a root at `/`. Files and directories are stored by absolute path. Key operations used by Claude's tools:
- `createFileWithParents` / `replaceInFile` / `insertInFile` — used by `str_replace_editor`
- `rename` / `deleteFile` — used by `file_manager`
- `serialize()` / `deserializeFromNodes()` — JSON-safe round-trip (Maps are converted to plain objects) used to pass FS state between client and server

### AI Tools (`src/lib/tools/`)

Two tools are registered with Claude at `/api/chat`:
- **`str_replace_editor`**: `view`, `create`, `str_replace`, `insert` commands operating on the server-side VFS. Tool results feed back into Claude's context for multi-step generation.
- **`file_manager`**: `rename`, `delete` commands.

Tool calls are executed both server-side (for Claude's feedback loop) and client-side (via `onToolCall` to update the client VFS).

### JSX Transformation & Preview (`src/lib/transform/jsx-transformer.ts`)

`createImportMap()` takes the VFS file map, Babel-transforms each `.jsx/.tsx` file into a blob URL, and builds an ES module import map. Third-party packages are mapped to `https://esm.sh/<package>`. The `@/` import alias is mapped to root `/`. Missing local imports get placeholder stub modules. `createPreviewHTML()` assembles the final iframe HTML with the import map, Tailwind CDN, and a React error boundary.

The preview entry point priority: `/App.jsx` → `/App.tsx` → `/index.jsx` → `/index.tsx` → `/src/App.jsx` → first `.jsx/.tsx` found.

### Contexts

- **`FileSystemContext`** (`src/lib/contexts/file-system-context.tsx`): Owns the singleton client-side `VirtualFileSystem` instance and a `refreshTrigger` counter. `handleToolCall` translates AI SDK tool call payloads into VFS mutations.
- **`ChatContext`** (`src/lib/contexts/chat-context.tsx`): Wraps `useChat` from `@ai-sdk/react`, passes serialized FS in the request body, and routes tool calls to `FileSystemContext`.

### Auth & Persistence

JWT sessions via `jose` stored in cookies (`src/lib/auth.ts`). Passwords bcrypt-hashed. Server actions in `src/actions/` handle sign-up/in/out and project CRUD. Projects store `messages` (JSON) and `data` (serialized VFS JSON) as strings in SQLite. `userId` on `Project` is optional—anonymous sessions don't persist.

### Generation Prompt (`src/lib/prompts/generation.tsx`)

Claude is instructed to always create `/App.jsx` as the entry point, use Tailwind (not inline styles), use the `@/` alias for local imports, and operate on the virtual FS root `/`.

### Provider Fallback (`src/lib/provider.ts`)

If `ANTHROPIC_API_KEY` is not set, a `MockLanguageModel` is used that returns static component templates—useful for local development without API access.
