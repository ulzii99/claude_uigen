# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

UIGen is an AI-powered React component generator with live preview. It uses the Anthropic Claude AI API to generate React components based on user descriptions, with a virtual file system that renders components in real-time without writing to disk.

## Essential Commands

### Setup
```bash
npm run setup
```
Installs dependencies, generates Prisma client, and runs database migrations. Run this after cloning.

### Development
```bash
npm run dev           # Start development server with Turbopack
npm run dev:daemon    # Start in background, logs to logs.txt
```
Open http://localhost:3000

### Testing
```bash
npm test              # Run all Vitest tests
```

### Database
```bash
npx prisma migrate dev          # Create/apply new migration
npx prisma generate             # Regenerate Prisma client
npm run db:reset                # Reset database (force)
npx prisma studio               # Open database GUI
```

### Build
```bash
npm run build         # Build for production
npm run start         # Start production server
npm run lint          # Run ESLint
```

## Architecture

### Core Systems

**Virtual File System (VFS)**
- Location: `src/lib/file-system.ts`
- Class: `VirtualFileSystem` - in-memory file system with CRUD operations
- Context: `src/lib/contexts/file-system-context.tsx` - React context wrapping VFS
- Files are stored in memory only; nothing written to disk during component generation
- Supports file/directory operations: create, read, update, delete, rename
- Serializable for persistence to database

**AI Component Generation Flow**
1. User sends prompt via `ChatInterface` component
2. Request to `/api/chat/route.ts` endpoint
3. Vercel AI SDK streams response from Claude (or mock provider)
4. Claude uses two tools to manipulate VFS:
   - `str_replace_editor` (src/lib/tools/str-replace.ts) - create/edit files
   - `file_manager` (src/lib/tools/file-manager.ts) - rename/delete files
5. Tool calls update VFS via `FileSystemContext.handleToolCall()`
6. Frontend automatically re-renders preview when files change

**Live Preview System**
- Location: `src/lib/transform/jsx-transformer.ts`
- Transforms JSX/TSX files using Babel to browser-executable code
- Creates ES Module import maps with blob URLs for each file
- Supports `@/` path alias (maps to root `/`)
- Loads external dependencies from esm.sh CDN
- Renders in iframe with Tailwind CSS CDN
- Error handling: syntax errors shown in preview, runtime errors caught by ErrorBoundary

**Authentication & Projects**
- JWT-based auth: `src/lib/auth.ts` (server-only)
- Anonymous users: can use app but don't persist projects
- Registered users: projects saved to SQLite database via Prisma
- Middleware: `src/middleware.ts` - session verification
- Actions: `src/actions/` - server actions for CRUD on projects

### Data Models (Prisma)

The database schema is defined in the `prisma/schema.prisma` file. Reference it anytime you need to understand the structure of data stored in the database.

```prisma
User {
  id, email, password (bcrypt hashed)
  projects: Project[]
}

Project {
  id, name, userId (nullable for anonymous)
  messages: String (JSON-serialized chat history)
  data: String (JSON-serialized VFS state)
}
```

Prisma client generated to `src/generated/prisma/` (not `node_modules`).

### Key Files & Directories

- `src/app/` - Next.js 15 App Router pages
  - `page.tsx` - Landing/login page
  - `[projectId]/page.tsx` - Main editor page for a project
  - `api/chat/route.ts` - Streaming AI chat endpoint
- `src/components/`
  - `chat/` - Chat UI components (MessageList, MessageInput, ChatInterface)
  - `ui/` - shadcn/ui components
- `src/lib/`
  - `file-system.ts` - VFS implementation
  - `transform/jsx-transformer.ts` - JSX to browser code transformation
  - `provider.ts` - Mock vs real Claude API provider selection
  - `prompts/generation.tsx` - System prompt for Claude
  - `contexts/` - React contexts (FileSystemContext, ChatContext)
- `src/actions/` - Next.js server actions for database operations

### Mock Provider (No API Key)

When `ANTHROPIC_API_KEY` is not set in `.env`:
- `MockLanguageModel` class (src/lib/provider.ts) returns static component code
- Simulates multi-step tool calling with delays to demo the UX
- Supports counter, form, and card component types
- Limits to 4 steps to prevent repetition

## Testing

Tests use Vitest with React Testing Library:
- `src/components/chat/__tests__/` - Chat component tests
- `src/lib/__tests__/` - VFS and transformer tests
- `src/lib/contexts/__tests__/` - Context tests
- Config: `vitest.config.mts`

## Path Aliases

`@/` maps to `src/` directory (configured in `tsconfig.json`).

Example: `import { VirtualFileSystem } from "@/lib/file-system"`

## Code Style

- Use comments sparingly. Only comment complex code where the logic isn't self-evident.

## Important Notes

- **No file writes during generation**: The VFS is entirely in-memory until explicitly saved to database
- **Anonymous user tracking**: Uses localStorage to track work count (see `src/lib/anon-work-tracker.ts`)
- **API key optional**: App works without Claude API key using mock provider
- **Prisma location**: Client generates to `src/generated/prisma/`, not default location
- **Preview updates**: Preview re-renders when VFS `refreshTrigger` state changes (any file operation)
- **Streaming**: Chat uses Vercel AI SDK streaming for real-time updates
- **Prompt caching**: System prompt uses Anthropic prompt caching (`cacheControl: { type: "ephemeral" }`)
