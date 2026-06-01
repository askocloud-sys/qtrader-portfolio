# DeeplySecure Chatbot

Embeddable LLM-powered chat widget for the DeeplySecure security awareness training platform. Ships as a `<ds-chatbot>` web component that can be dropped into any of the three DeeplySecure apps (admin, user, superadmin) with a single script tag.

## Architecture

```
Browser (DeeplySecure app)
  |
  |  1. User logs in -> Supabase issues JWT
  |  2. <ds-chatbot token="..."> receives token via attribute
  |
  v
Chatbot Frontend (this repo, web component)
  |
  |  3. Sends messages with Authorization: Bearer <token>
  |
  v
Chatbot Backend (this repo, Express)
  |
  |  4. Validates JWT against Supabase
  |  5. Resolves user identity + org context
  |  6. Embeds last user message (OpenRouter text-embedding-3-small)
  |  7. Retrieves relevant context via match_rag_documents (pgvector)
  |  8. Streams LLM response via OpenRouter with RAG context
  |
  v
Supabase (shared instance, pgvector) + OpenRouter (google/gemini-3.1-flash-lite-preview)
```

No second login required. The chatbot piggybacks on the session the user already has in the host app.

## Tech Stack

| Layer     | Technology                                                     |
| --------- | -------------------------------------------------------------- |
| Runtime   | Bun                                                            |
| Frontend  | React 19, Vite 7, Tailwind CSS v4, Framer Motion, Lucide React |
| Backend   | Express, TypeScript                                            |
| Auth      | Supabase JWT (shared with host app, validated server-side)     |
| LLM       | Vercel AI SDK v6 + OpenRouter (`google/gemini-3.1-flash-lite-preview`) |
| RAG       | pgvector (Supabase), OpenRouter embeddings (`openai/text-embedding-3-small`) |
| Widget    | Web Component with Shadow DOM (style-isolated)                 |

## Quick Start

### Prerequisites

- [Bun](https://bun.sh) installed
- OpenRouter API key
- Supabase project credentials (same as the DeeplySecure monorepo) -- only needed when running with auth enabled

### Setup

```bash
cp .env.example .env.development
# Fill in OPENROUTER_API_KEY (required)
# Fill in SUPABASE_URL, SUPABASE_ANON_KEY, VITE_SUPABASE_URL, VITE_SUPABASE_ANON_KEY (only if not using SKIP_AUTH)
```

For production runtime, create a separate `.env` file (for example `cp .env.example .env`) or provide the same variables through the host environment.

### Install & Run

```bash
bun install
bun run dev          # starts server + client on port 6001
```

Open http://localhost:6001, paste a Supabase JWT token (from the admin app's localStorage, key `sb-*-auth-token` -> `access_token`), and start chatting.

### Standalone Mode (no Supabase)

To run the chatbot without Supabase authentication (useful for local development and testing the LLM integration in isolation), add to your `.env` / `.env.development`:

```
SKIP_AUTH=true
VITE_SKIP_AUTH=true
```

- `SKIP_AUTH=true` -- server-side: bypasses JWT validation, injects a dummy dev user
- `VITE_SKIP_AUTH=true` -- client-side: skips the token input screen, goes straight to the chat bubble

In this mode the only required env var is `OPENROUTER_API_KEY`.

### Build the Widget

```bash
bun run build:widget   # produces dist/widget/chatbot.js
```

This outputs a single IIFE script that registers the `<ds-chatbot>` custom element. No npm install needed on the consumer side. The build replaces Node-style `process.env.NODE_ENV` (and Framer Motion’s `process.env?.NODE_ENV`) so the script runs in the browser without a `process` global.

### Full Production Build

```bash
bun run build          # builds client (Vite) + server (esbuild)
bun run start          # runs production server and explicitly loads .env when present
```

## Integration

### 1. Load the script

```html
<script src="https://chatbot.deeplysecure.com/widget/chatbot.js" defer></script>
```

### 2. Add the element

```html
<ds-chatbot
  token="<supabase-access-token>"
  api-url="https://chatbot.deeplysecure.com"
  lang="en"
  position="bottom-right"
></ds-chatbot>
```

Or in React (e.g., DashboardLayout):

```tsx
function ChatbotWidget() {
  const { session } = useAuth();
  const ref = useRef<HTMLElement>(null);

  useEffect(() => {
    if (ref.current && session?.access_token) {
      ref.current.setAttribute("token", session.access_token);
    }
  }, [session?.access_token]);

  return <ds-chatbot ref={ref} api-url="https://chatbot.deeplysecure.com" />;
}
```

### Attributes

| Attribute      | Default        | Description                                       |
| -------------- | -------------- | ------------------------------------------------- |
| `token`        | --             | Supabase JWT access token                         |
| `api-url`      | --             | Chatbot backend URL                               |
| `position`     | `bottom-right` | Widget position (`bottom-right` or `bottom-left`) |
| `lang`         | `en`           | UI language (`en` or `de`)                        |
| `module-id`    | --             | Scope RAG retrieval to a specific module           |
| `context-type` | --             | Filter RAG results by content type (`intro`, `glossary`, `quiz`) |

### Token Delivery

The widget accepts tokens three ways (checked in order):

1. **Attribute** -- set `token="..."` directly (simplest for React integration)
2. **postMessage** -- parent sends `{ type: "AUTH_TOKEN", token: "..." }` (for iframe or non-React hosts)
3. **URL hash** -- `#token=...` (for standalone/testing, cleared after reading)

### RAG Context Filtering

The chatbot uses retrieval-augmented generation (RAG) to ground its answers in training content stored in the `rag_documents` table (populated by the ingestion CLI in `chatbot/ingestion/`).

By default, the chatbot retrieves the most relevant content across all modules for the user's language. Use the `module-id` and `context-type` attributes to scope retrieval:

```html
<!-- General assistant -- retrieves from all content -->
<ds-chatbot token="..." api-url="https://chatbot.deeplysecure.com" />

<!-- Module companion -- scoped to a specific module -->
<ds-chatbot token="..." api-url="..." module-id="grundlagen-1" />

<!-- Quiz helper -- only quiz content for a module -->
<ds-chatbot token="..." api-url="..." module-id="grundlagen-1" context-type="quiz" />
```

These attributes are forwarded as `X-Module-Id` and `X-Context-Type` headers on every chat request. The server uses them to filter the `match_rag_documents` Supabase RPC call. The user's language (from their profile) is always applied as a filter automatically.

## Project Structure

```
chatbot/
  client/
    src/
      components/
        ChatWidget.tsx       # Floating bubble + expandable chat panel
        ChatMessage.tsx      # Message rendering (AI SDK v6 parts API)
      context/
        auth.tsx             # Token provider (prop / postMessage / hash)
      lib/
        utils.ts             # cn() utility
      web-component.tsx      # <ds-chatbot> custom element wrapper
      App.tsx                # Standalone dev testing page
      main.tsx               # SPA entry point
      index.css              # Tailwind + DeeplySecure theme
    index.html
  server/
    index.ts                 # Express server with CORS
    env.ts                   # Dotenv preload
    vite.ts                  # Vite dev middleware
    static.ts                # Production static serving
    routes.ts                # Route registration
    routes/
      chat.ts                # POST /api/chat -- streaming LLM endpoint with RAG
      __tests__/
        chat.test.ts         # Tests for filter extraction and prompt building
    middleware/
      auth.ts                # JWT validation (admin_users + managed_users)
    lib/
      supabase.ts            # Supabase client factory
      embeddings.ts          # Query embedding via OpenRouter
  script/
    build.ts                 # Client + server build
  specs/
    chatbot-concept.md       # Web component architecture design
    deeply-secure-chatbot-specs.md  # Feature requirements
    jwt-sharing.md           # Auth integration spec
  vite.config.ts             # SPA dev/build config
  vite.widget.config.ts      # Widget IIFE build config
```

## Scripts

| Command             | Description                                    |
| ------------------- | ---------------------------------------------- |
| `bun run dev`       | Start dev server (Express + Vite HMR) on :6001 |
| `bun run dev:client`| Start Vite client only on :6001                |
| `bun run build`     | Production build (client + server)             |
| `bun run build:widget` | Build embeddable widget to `dist/widget/chatbot.js` |
| `bun run start`     | Run production server                          |
| `bun run check`     | TypeScript type check                          |
| `bun run test`      | Run tests                                      |

## Environment Variables

### Required

| Variable            | Description                       |
| ------------------- | --------------------------------- |
| `SUPABASE_URL`      | Supabase project URL              |
| `SUPABASE_ANON_KEY` | Supabase public anon key          |
| `OPENROUTER_API_KEY` | OpenRouter API key               |
| `VITE_SUPABASE_URL` | Same as SUPABASE_URL (client-side)|
| `VITE_SUPABASE_ANON_KEY` | Same as SUPABASE_ANON_KEY (client-side) |

### Optional

| Variable             | Default | Description                        |
| -------------------- | ------- | ---------------------------------- |
| `PORT`               | `6001`  | Server port                        |
| `HOST`               | `0.0.0.0` | Server bind address             |
| `ALLOWED_ORIGINS`    | --      | Comma-separated CORS origins       |
| `SKIP_AUTH`            | --      | Set to `true` to bypass JWT auth (server) |
| `VITE_SKIP_AUTH`       | --      | Set to `true` to skip token input (client) |
| `VITE_CHATBOT_API_URL` | --   | Chatbot API URL for standalone dev |
