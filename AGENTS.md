# Repository Guidelines

## Project Structure & Module Organization
- `src/` holds the React 19 + TypeScript UI: `components/` for screens, `API/` for fetch helpers, `assets/` for static SVGs, and `types.ts` for shared contracts.
- `worker/index.ts` implements the Cloudflare Worker router, while `worker-configuration.d.ts` and `wrangler.template.jsonc` describe bindings; copy/rename the template to `wrangler.jsonc` before deploying.
- `public/` contains static assets served by Vite; `docs/` (e.g., `PRETTIER.md`) centralizes process docs; `script/chromeToJSON.py` converts Chrome bookmarks for import.
- Database bootstrap SQL lives in `init_table.sql`; update it alongside any schema change before hitting D1.

## Build, Test & Development Commands
- `pnpm dev`: start the Vite dev server on localhost with hot reload.
- `pnpm build`: type-check (`tsc -b`) and emit the production bundle in `dist/`.
- `pnpm preview`: run the built bundle to smoke-test production assets.
- `pnpm lint`: execute ESLint with the shared config; run before every commit.
- `pnpm format` / `pnpm format:check`: apply or verify Prettier formatting for tracked files.
- `pnpm deploy`: build and push the latest worker + assets via Wrangler; requires configured Cloudflare credentials.
- `pnpm cf-typegen`: refresh Worker runtime types after editing bindings or env vars.

## Coding Style & Naming Conventions
- Follow `.prettierrc.cjs` (2-space indent, single quotes, 100-char width) and the guidance in `docs/PRETTIER.md`; never hand-edit formatting in PRs.
- Prefer functional React components with `PascalCase` names, hooks/utilities in `camelCase`, and D1 binding/environment constants as `SCREAMING_SNAKE_CASE`.
- Keep UI state colocated with the component; cross-cutting logic belongs in `src/API` or dedicated hooks.

## Testing Guidelines
- Automated tests are not yet wired up; when adding features, create component or hook tests under `src/__tests__` using Vitest + React Testing Library (align filenames as `ComponentName.test.tsx`).
- Until tests exist, rely on `pnpm dev` for manual verification and exercise authentication, drag-and-drop, and D1-backed CRUD paths after every change.

## Commit & Pull Request Guidelines
- Follow Conventional Commits (`feat:`, `fix:`, `docs:`) as seen in `git log`; scope optional but encouraged (`feat(ui): ...`).
- One logical change per commit; include DB or schema diffs alongside the code that relies on them.
- PRs must describe motivation, list testing commands, link related issues, and attach UI screenshots/gifs for visible changes; mention any required Wrangler/D1 config updates.
- Ensure no `wrangler.template.jsonc` secrets leak—use `.dev.vars` for local overrides and document new env keys in the PR body.
