# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

- **Setup**: `npm run setup` (installs deps, generates Prisma client, runs migrations)
- **Dev server**: `npm run dev` (Next.js with Turbopack)
- **Dev daemon**: `npm run dev:daemon` (background, logs to logs.txt)
- **Build**: `npm run build`
- **Lint**: `npm run lint`
- **Tests**: `npm run test` (vitest, runs in watch mode)
- **Single test**: `npx vitest run path/to/test.ts`
- **DB reset**: `npm run db:reset`
- **Prisma generate**: `npx prisma generate` (after schema changes)
- **Prisma migrate**: `npx prisma migrate dev` (after schema changes)

## Architecture

UIGen is an AI-powered React component generator with live preview. Users describe components via chat, Claude generates code into a virtual file system, and a live preview renders the result in an iframe.

### Core Flow

1. **Chat API** (`src/app/api/chat/route.ts`): POST endpoint using Vercel AI SDK's `streamText`. Sends messages to Claude (or a mock provider) with two tools: `str_replace_editor` and `file_manager`. The AI modifies files in a `VirtualFileSystem` instance server-side, and results are streamed back.

2. **Virtual File System** (`src/lib/file-system.ts`): In-memory tree-based filesystem (`VirtualFileSystem` class). No files are written to disk. Supports create, read, update, delete, rename, and text editor operations (view, str_replace, insert). Serialized as JSON and sent with each chat request.

3. **Client-side tool replay** (`src/lib/contexts/file-system-context.tsx`): The `FileSystemProvider` maintains client-side VFS state. Tool calls from the AI stream are replayed on the client via `handleToolCall` to keep the UI in sync with server-side changes.

4. **JSX Transform & Preview** (`src/lib/transform/jsx-transformer.ts`): Transforms JSX/TSX files using `@babel/standalone` in the browser, creates blob URLs, builds an import map, and generates a self-contained HTML document rendered in a sandboxed iframe (`src/components/preview/PreviewFrame.tsx`). Third-party imports resolve via `esm.sh`. CSS files are injected as `<style>` tags.

5. **Provider** (`src/lib/provider.ts`): Returns either the Anthropic Claude model (claude-haiku-4-5) or a `MockLanguageModel` if no `ANTHROPIC_API_KEY` is set. The mock provider returns static component code for testing without an API key.

### Key Conventions

- Entry point for generated components is always `/App.jsx` in the virtual filesystem
- Local file imports in generated code use `@/` alias (maps to VFS root)
- All generated components use React + Tailwind CSS for styling
- The system prompt for generation is in `src/lib/prompts/generation.tsx`

### Data Layer

- **Database**: SQLite via Prisma (`prisma/schema.prisma`). Prisma client output goes to `src/generated/prisma/`.
- **Models**: `User` (email/password auth) and `Project` (stores serialized messages and VFS data as JSON strings)
- **Auth**: JWT-based via `jose` (`src/lib/auth.ts`), with middleware protecting API routes (`src/middleware.ts`)
- **Server actions**: `src/actions/` contains `create-project.ts`, `get-project.ts`, `get-projects.ts`
- Anonymous users can use the app without auth; work is tracked in localStorage (`src/lib/anon-work-tracker.ts`)

### Layout

- Two-panel resizable layout (`src/app/main-content.tsx`): left panel is chat, right panel toggles between Preview (iframe) and Code (file tree + Monaco editor)
- UI components from shadcn/ui in `src/components/ui/`

### AI Tools (given to Claude during generation)

- **str_replace_editor** (`src/lib/tools/str-replace.ts`): view, create, str_replace, insert commands on VFS files
- **file_manager** (`src/lib/tools/file-manager.ts`): rename and delete operations

### Testing

- Vitest with jsdom environment and React Testing Library
- Tests live alongside source in `__tests__/` directories
- Path aliases (`@/`) work in tests via `vite-tsconfig-paths`
