# CLAUDE.md

Guidance for AI assistants (Claude Code and others) working in this repository.

## What this project is

**Context7** (`@upstash/context7`) delivers up-to-date, version-specific library
documentation and code examples straight into an LLM's context. Instead of relying
on stale training data, an agent resolves a library to a Context7 ID and fetches
current docs on demand.

It works in two modes:

- **CLI + Skills** — the `ctx7` CLI installs a skill that guides agents to fetch
  docs via `ctx7 library` / `ctx7 docs` (no MCP required).
- **MCP** — a Context7 MCP server exposing `resolve-library-id` and `query-docs`
  tools (remote: `https://mcp.context7.com/mcp`).

This repo is a **pnpm + TypeScript monorepo** holding the public source: the MCP
server, CLI, SDKs, and per-agent plugins/skills. The API backend, parsing engine,
and crawling engine are **private and not in this repo**.

## Repository layout

```
packages/        workspace packages (see below)
skills/          source Agent Skills (context7-mcp, context7-cli, find-docs)
rules/           agent rule snippets (context7-cli.md, context7-mcp.md)
plugins/         per-agent plugin distributions (claude, codex, cursor, copilot, context7-power)
docs/            public documentation site source (.mdx, docs.json, openapi.json)
i18n/            translated READMEs (README.<locale>.md, ~15 locales)
.changeset/      Changesets release config + pending changesets
.claude-plugin/  Claude marketplace manifest
.agents/         Codex/agents marketplace manifest
public/          cover/icon images
server.json      MCP registry manifest (io.github.upstash/context7)
```

### Workspace packages (`packages/*`)

| Package | npm name | Role |
|---------|----------|------|
| `mcp` | `@upstash/context7-mcp` | MCP server (Express + `@modelcontextprotocol/sdk`); stdio and HTTP transports. Ships an `.mcpb` bundle, Dockerfile, smithery config. |
| `cli` | `ctx7` | CLI: `setup`/`remove`, plus `library` / `docs` doc fetching. Installs skills/MCP for Cursor, Claude, OpenCode. Built with `tsup`. |
| `sdk` | `@upstash/context7-sdk` | JS/TS SDK client (the doc-fetching primitive). Dual CJS/ESM. |
| `tools-ai-sdk` | `@upstash/context7-tools-ai-sdk` | Vercel AI SDK tools + agent (`.` and `./agent` exports). |
| `pi` | `@upstash/context7-pi` | Extension for the pi.dev coding agent (extensions/lib/skills/prompts). |

Dependency direction: `tools-ai-sdk` peer-depends on `sdk` (`workspace:*` in dev).
Other packages are independent.

## Build / dev / test commands

Run from the repo root (commands fan out with `pnpm -r`):

```bash
pnpm install              # install all workspaces
pnpm build                # build every package (pnpm -r run build)
pnpm typecheck            # tsc --noEmit across packages
pnpm test                 # run all package test suites (vitest)
pnpm lint                 # eslint --fix across packages
pnpm lint:check           # eslint (no fix) — CI uses this
pnpm format               # prettier --write
pnpm format:check         # prettier --check — CI uses this
pnpm clean                # clean dist + node_modules
```

Targeted builds/tests exist at the root too:
`pnpm build:sdk`, `pnpm build:mcp`, `pnpm build:ai-sdk`, `pnpm test:sdk`,
`pnpm test:tools-ai-sdk`.

Within a single package, the usual scripts are `build`, `dev` (watch),
`typecheck`, `test`, `test:watch`, `lint`, `format`. Note differences:

- `mcp` builds with raw `tsc` (`tsc && chmod 755 dist/index.js`); `dev` is `tsc --watch`.
  Start the HTTP server with `pnpm --filter @upstash/context7-mcp start`.
- `cli`, `sdk`, `tools-ai-sdk` build with **tsup**; `sdk`/`tools-ai-sdk` emit dual CJS+ESM.
- `pi` is `noEmit` (type-checked only, distributed as source).

Tests use **Vitest** throughout. CLI tests live in `src/__tests__/`; other
packages co-locate `*.test.ts` next to source (mcp uses a `test/` dir).

## Lint / format / TypeScript

- **ESLint** (flat config, `eslint.config.js`): `typescript-eslint` recommended +
  `eslint-plugin-prettier` (`prettier/prettier: error`). `no-explicit-any` is a
  warning; unused vars error unless prefixed `_`. Ignores `dist/`, `build/`,
  `node_modules/`. Some packages have their own `eslint.config.js`.
- **Prettier** (`prettier.config.mjs`): LF endings, double quotes, 2-space tabs,
  `trailingComma: "es5"`, `printWidth: 100`, `arrowParens: "always"`. `docs/` and
  lockfiles are in `.prettierignore`.
- **TypeScript**: root `tsconfig.json` is strict, `target/lib ES2022`. Packages
  extend or redefine it — most use `moduleResolution: "bundler"` with `noEmit`
  and path aliases (`@http`, `@tools`, etc.); `mcp` uses `NodeNext`. Keep aliases
  in sync between `tsconfig.json` and the matching `tsup.config.ts`.

## Releases (Changesets)

- Versioning is per-package via **Changesets** (`.changeset/`); base branch is
  `master`, access `public`, internal deps bumped as `patch`.
- Add a changeset for any user-facing package change (`pnpm changeset`).
- `pnpm release` runs `pnpm build && changeset publish`; `release:snapshot`
  publishes a `canary` tag. Workflows: `release.yml`, `canary-release.yml`,
  `mcp-registry.yml`, `ecr-deploy.yml`.
- `pnpm-workspace.yaml` sets `minimumReleaseAge: 10080` (minutes) on installs.

## CI

`.github/workflows/test.yml` runs on PRs touching `packages/**` / configs and on
pushes to `master`: install (`--frozen-lockfile`) → `lint:check` → `format:check`
→ `build` → `typecheck` → `test`. Node 20, pnpm 10. Run the same sequence locally
before pushing. Tests may need `CONTEXT7_API_KEY` / AWS Bedrock secrets (provided
in CI) for integration paths.

## Skills, rules, and plugins

- `skills/` holds the canonical Agent Skills (each a `SKILL.md`). These are the
  source distributed by the CLI `setup` flow and copied into `plugins/<agent>/`.
- `plugins/{claude,codex,cursor,copilot}/context7/` are per-agent plugin packages
  (each with its own manifest: `.claude-plugin/plugin.json`, `.codex-plugin/`,
  `.cursor/`, plus `skills/`, optional `agents/`, `commands/`, `.mcp.json`).
- `rules/` and `docs/` (`.mdx`) are documentation/rule snippets — no build step.

When changing skill content, update both the `skills/` source and the copies
shipped under the relevant `plugins/` directories so they stay consistent.

## Conventions & gotchas

- ESM-first: most packages are `"type": "module"`. Respect each package's module
  resolution mode when adding imports (extensions required under NodeNext in `mcp`).
- Env config: see `.env.example` (e.g. `CONTEXT7_API_KEY`, `CONTEXT7_API_URL`,
  `UPSTASH_REDIS_REST_*` for the MCP HTTP server, `CTX7_TELEMETRY_DISABLED`).
- `server.json` and `gemini-extension.json` carry their own version fields that
  reference published MCP versions — don't assume they track the workspace
  `package.json` versions automatically.
- Don't hand-edit `dist/`, `pnpm-lock.yaml`, or generated bundles
  (`packages/mcp/mcpb/context7.mcpb`).
- README is mirrored into ~15 locales under `i18n/`. Significant README changes
  should be flagged for translation rather than silently diverging.
