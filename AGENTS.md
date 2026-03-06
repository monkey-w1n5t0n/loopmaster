# Repository Guidelines

## Project Structure & Module Organization
`src/` contains the Preact + TypeScript client. UI lives in `src/components/`, shared hooks in `src/hooks/`, audio/editor logic in `src/widgets/`, and app wiring in files such as `src/client.tsx`, `src/router.tsx`, and `src/state.ts`. `deno/` contains the Hono API, auth, KV access, and shared schemas for the backend. Static assets and tutorials live in `public/`. `vendor/WaveFFT/` is a vendored wasm dependency; treat it as external unless a task explicitly targets it. `dist/` is generated output and should not be edited by hand.

## Build, Test, and Development Commands
Run `bun run install:all` once to install Bun and Deno dependencies. Use `bun run dev` to start the web app and API together. Use `bun run dev:web` for the Vite client only and `bun run dev:api` for the Deno server only. Run `bun run build` before opening a PR; it produces the production bundle in `dist/` and catches TypeScript or bundling regressions.

## Coding Style & Naming Conventions
Follow the existing TypeScript style: 2-space indentation, single quotes, trailing commas where valid, and no semicolons. Prefer named exports for shared helpers and PascalCase for component filenames such as `src/components/ShareModal.tsx`. Use camelCase for functions and variables, and keep backend modules small and purpose-specific, similar to `deno/auth.ts` and `deno/kv.ts`. Keep generated files, lockfiles, and vendored code changes isolated in separate commits when possible.

## Testing Guidelines
There is no dedicated automated test suite checked in yet. For now, treat `bun run build` as the minimum validation step and manually smoke-test the affected flow in `bun run dev`. If you add tests, keep frontend tests alongside the feature as `*.test.ts` or `*.test.tsx`, and place Deno tests under `deno/` using `*.test.ts`.

## Commit & Pull Request Guidelines
Recent history uses short, prefixed commit subjects such as `build: update`, `docs: sound synthesis tutorial`, and `feature: change 1st example`. Keep that format: `<type>: <brief imperative summary>`. PRs should explain user-visible changes, note config or migration impacts, link related issues, and include screenshots or short recordings for UI work.

## Configuration & Environment
Copy `.env.example` to `.env` and keep secrets out of git. The Vite dev server expects local HTTPS certs in `~/.ssl-certs/localhost.pem` and `~/.ssl-certs/localhost-key.pem`. Deno KV data is stored under `storage/` by default.
