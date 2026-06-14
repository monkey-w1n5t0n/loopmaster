# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What is Loopmaster

Loopmaster is a browser-based music coding app (loopmaster.xyz). Users write code to generate music using a custom DSP language that compiles to WebAssembly. The app features a code editor with inline audio widgets (knobs, waveforms, filters, ADSR envelopes, etc.), project sharing, and real-time audio synthesis.

## Development Commands

```bash
bun i                  # Install frontend dependencies
bun run install:all    # Install both frontend (bun) and backend (deno) deps
bun dev                # Run both frontend and API server concurrently (portless loopmaster bun dev)
bun run dev:web        # Frontend only (Vite dev server with HTTPS)
bun run dev:api        # API server only (Deno)
bun run build          # Production build
```

Requires SSL certs at `~/.ssl-certs/localhost-key.pem` and `~/.ssl-certs/localhost.pem` for the dev server.

Copy `.env.example` to `.env` and configure `KV_PATH`, `RESEND_API_KEY`, `OPENAI_API_KEY`, and `VITE_MESPEAK_URL`.

## Architecture

### Frontend (`src/`)
- **Framework**: Preact with `@preact/signals` for reactive state (aliased as React in Vite config)
- **Styling**: Twind (Tailwind-in-JS) loaded from CDN in `index.html`, plus utility `cn()` in `src/lib/cn.ts`
- **Bundler**: rolldown-vite (Vite replacement via `npm:rolldown-vite`) with `@preact/preset-vite`
- **Entry**: `index.html` -> `src/client.tsx` -> `App.tsx`
- **Routing**: Custom signal-based router in `src/router.tsx` (no library — uses `history.pushState`)
- **State**: `src/state.ts` — large central module with all app signals (session, editor, theme, DSP context, projects, playback state). Most components import directly from here.
- **API client**: `src/api.ts` — `API` class wrapping fetch for all backend calls
- **DSP pipeline**: `src/dsp.ts` — creates and manages DSP contexts that bridge the editor to the `engine` package's WebAssembly audio worklet

### Key external packages (GitHub dependencies)
- **`engine`** (`github:loopmaster-xyz/engine`) — DSP compiler, WebAssembly runtime, audio worklet. WASM files at `node_modules/engine/as/build/`. Vite aliases `/as` to this path.
- **`editor`** (`github:loopmaster-xyz/editor`) — Code editor component with widget support
- **`utils`** (`github:stagas/utils`) — Shared utilities (debounce, deferred, RGB manipulation, etc.)

### Backend (`deno/`)
- **Runtime**: Deno with `--unstable-kv`
- **Framework**: Hono (`@hono/hono`)
- **Database**: Deno KV (stored at path from `KV_PATH` env var, defaults to `./storage/kv`)
- **Schema validation**: Zod v4 — all API types defined with Zod schemas in `deno/types.ts` (shared with frontend)
- **Auth**: Cookie-based sessions (`deno/auth.ts`)
- **AI features**: OpenAI integration for track generation (`deno/openai.ts`)
- **API proxy**: Vite proxies `/api` to `http://127.0.0.1:8787`

### Widgets (`src/widgets/`)
Inline editor widgets for audio parameters. Each widget file exports a `create*Widget` factory. These are assembled in `src/dsp.ts` and rendered inside the code editor. Examples: `wave.ts`, `filter.ts`, `knob.ts`, `adsr.ts`, `piano.ts`, `reverb.ts`, `compressor.ts`, `sampler.ts`, `timeline.ts`.

### Vendor
- `vendor/WaveFFT/` — FFT library, aliased as `wavefft` in Vite config

### Themes
`src/themes/` contains 200+ terminal color scheme JSON files. Theme selection is managed via signals in `state.ts`.

## Important Patterns

- TypeScript with `"strict": true`, JSX via Preact (`jsxImportSource: "preact"`)
- `loader.mjs` registers Node.js module hooks to strip TypeScript types at import time (used by Vite via `NODE_OPTIONS="--import ./loader.mjs"`)
- COOP/COEP headers enabled via `vite-plugin-coop-coep` (required for `SharedArrayBuffer` used by audio worklets)
- The frontend imports types directly from `../deno/types.ts` — types are shared across the monorepo boundary
- Vite `optimizeDeps.exclude` includes `engine` and `utils/mouse-buttons` to avoid pre-bundling issues
