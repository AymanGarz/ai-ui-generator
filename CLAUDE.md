# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

UIGen is an AI-powered React component generator with live preview. Users describe components in a chat interface, and the LLM generates React/JSX code that renders in a sandboxed iframe preview. The app supports both anonymous usage and authenticated users with persistent projects.

## Commands

- `npm run setup` — Install deps, generate Prisma client, run migrations (first-time setup)
- `npm run dev` — Start dev server with Turbopack (port 3000)
- `npm run build` — Production build (requires `NODE_OPTIONS='--require ./node-compat.cjs'`)
- `npm run lint` — ESLint
- `npm run test` — Run all tests with Vitest
- `npx vitest run src/lib/__tests__/file-system.test.ts` — Run a single test file
- `npx prisma migrate dev` — Run database migrations
- `npx prisma generate` — Regenerate Prisma client
- `npm run db:reset` — Reset database (destructive)

## Architecture

### Core Flow

1. User sends a chat message describing a component
2. `POST /api/chat` (route handler) streams a response from Anthropic (or a mock provider if no API key)
3. The LLM uses two AI SDK tools (`str_replace_editor`, `file_manager`) to create/edit files in a `VirtualFileSystem`
4. Tool calls are reflected on both server (for persistence) and client (for live preview) via the `FileSystemContext`
5. The `PreviewFrame` transforms JSX files using Babel standalone, creates an import map with blob URLs, and renders in a sandboxed iframe

### Key Abstractions

- **VirtualFileSystem** (`src/lib/file-system.ts`) — In-memory file system with tree structure. Serialized as JSON for transport between client/server. No files are written to disk. The generated code lives entirely in this virtual FS with paths like `/App.jsx`, `/components/Counter.jsx`.

- **AI Tools** — Two tools exposed to the LLM:
  - `str_replace_editor` (`src/lib/tools/str-replace.ts`) — create, view, str_replace, insert operations on virtual files
  - `file_manager` (`src/lib/tools/file-manager.ts`) — rename and delete operations

- **JSX Transformer** (`src/lib/transform/jsx-transformer.ts`) — Client-side Babel transformation that converts JSX/TSX to browser-runnable JS. Creates blob URLs for each file and builds an import map. Third-party imports resolve to `esm.sh`. Handles `@/` path aliases.

- **Provider** (`src/lib/provider.ts`) — Supports Anthropic (`ANTHROPIC_API_KEY`) with model `claude-opus-4-6`, or a `MockLanguageModel` fallback.

### Context Providers

The app is wrapped in two nested React contexts (in `src/app/main-content.tsx`):
- `FileSystemProvider` — manages the virtual file system state, handles tool call side effects
- `ChatProvider` — wraps Vercel AI SDK's `useChat` with `DefaultChatTransport` (dynamic `body` function for current FS state), manages input state locally, uses `sendMessage` for submissions

### Routing & Auth

- `/` — Anonymous mode or redirect to most recent project for authenticated users
- `/[projectId]` — Authenticated project page (redirects to `/` if not authenticated)
- Auth uses JWT tokens in httpOnly cookies (`jose` library), bcrypt for passwords
- Middleware protects `/api/projects` and `/api/filesystem` routes
- Server Actions in `src/actions/` handle sign-up, sign-in, sign-out, project CRUD

### Database

SQLite via Prisma. Two models: `User` and `Project`. Projects store serialized messages and virtual FS data as JSON strings. Prisma client output goes to `src/generated/prisma/`.

## Tech Stack

- Next.js 15 (App Router, Turbopack), React 19, TypeScript
- Tailwind CSS v4, shadcn/ui (new-york style)
- Vercel AI SDK (`ai`, `@ai-sdk/react`, `@ai-sdk/anthropic`)
- Prisma + SQLite
- Vitest + Testing Library + jsdom for tests
- Monaco Editor for code editing
- `@babel/standalone` for client-side JSX transformation

## Conventions

- Path alias: `@/*` maps to `./src/*`
- UI components from shadcn live in `src/components/ui/`
- Tests live in `__tests__/` directories adjacent to the code they test
- The generated virtual FS always uses `/App.jsx` as the entry point, with `@/` import aliases for local files

## Gotchas

- **Run `npm run setup` before `npm run dev`** — The Prisma client (`@/generated/prisma`) is not checked into the repo. Running `npm run dev` without generating it first will fail with "Module not found".
- **Use `--legacy-peer-deps` for dependency installs** — There are peer dependency conflicts between `ai`, `@ai-sdk/react`, and `zod`. Plain `npm install` may fail with ERESOLVE errors.
- **AI SDK v6 API** — `useChat` from `@ai-sdk/react@3.x` does NOT return `handleSubmit`/`input`/`setInput`/`handleInputChange`. It returns `sendMessage`/`messages`/`status`. Input state must be managed manually. The server must use `toUIMessageStreamResponse()` (not `toDataStreamResponse()`), and messages are `UIMessage` (with `parts`), not the old `Message` (with `content`).
- **Provider model spec** — `ai@6.x` requires `LanguageModelV2` (spec version `"v2"`). Old v1 providers will throw `AI_UnsupportedModelVersionError`. Mock providers must implement the v2 interface.
- **AI SDK v6 `tool()` uses `inputSchema`, NOT `parameters`** — `tool()` is an identity function. `prepareToolsAndToolChoice` reads `tool.inputSchema`. Using the old v5 key `parameters` silently produces empty schemas (`properties: {}`) sent to the API. Always use `inputSchema`.
- **`streamText` API renames** — `maxSteps` is removed; use `stopWhen: stepCountIs(N)` (import from `"ai"`). `maxTokens` is removed; use `maxOutputTokens`.
- **Anthropic prompt caching** — The system prompt in `POST /api/chat` uses `providerOptions: { anthropic: { cacheControl: { type: "ephemeral" } } }` to enable Anthropic's prompt caching. This reduces latency and cost for the system prompt across multi-turn conversations. See `src/app/api/chat/route.ts`.
- **Anthropic model naming** — Model is set in `src/lib/provider.ts` as `claude-haiku-4-5`. The `@ai-sdk/anthropic` provider uses `anthropic("claude-haiku-4-5")` — no custom configuration or fetch wrappers needed.
- **`ANTHROPIC_API_KEY` env var** — Set in `.env`. When missing or empty, the app falls back to `MockLanguageModel` which returns canned responses. The mock implements `LanguageModelV1` from `@ai-sdk/provider`.
- **`onToolCall` never fires for server-executed tools** — `providerExecuted: true` gates it out. Use a `useEffect` watching `messages` and filter parts where `state === "output-available"` instead.

## Dev Server

- **Always run on port 3000.** Before starting the dev server, kill any process occupying port 3000 (and any stale dev server terminals). If the server is already running, kill it first, then relaunch.

## Self-Checking Process

After making changes or fixing errors, always self-verify by checking the running dev server:

1. Kill any existing process on port 3000
2. Run the dev server in the background (`npm run dev`)
3. `curl` localhost:3000 to trigger a page compile and check the HTTP status
4. Read the dev server output log (`tail`) to inspect any build/runtime errors
5. Fix the error, then repeat from step 3 until the page returns 200 with no errors in the logs
6. Do not rely on the user to report errors — proactively catch them yourself
