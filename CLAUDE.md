# VibeApp Development Guide

Browser-based AI-driven desktop environment. Users interact with an AI Agent via natural language; the Agent operates desktop apps (music, chess, diary, email, etc.) through an Action system. All data stored locally — no backend required.

An iframe-based micro-frontend VibeApp system (React + Cloud NAS).

## Architecture

**Monorepo**: pnpm workspaces + Turborepo

- `apps/webuiapps/` — Main SPA (React 18 + TypeScript + Vite)
- `packages/vibe-container/` — iframe communication SDK (stubbed in open-source mode via Vite alias → `src/lib/vibeContainerMock.ts`)

**Core data flow**: Agent ↔ Action System ↔ App

- Agent triggers actions via `app_action` tool calls
- Apps listen via `useAgentActionListener` and execute
- Results returned via `sendResult`
- Action categories: Operation (direct), Data Mutation (refresh Repo), Refresh (reload data), System (SYNC_STATE)

**Storage**: All file ops go through `@/lib` → `diskStorage` (dev, persisted to `~/.openroom/sessions/`) or Cloud NAS (prod). Direct `@gui/vibe-container` calls are forbidden.

**LLM proxy**: API calls routed through Vite dev server `/api/llm-proxy` to avoid CORS and protect API keys from browser exposure.

**Vibe workflow**: AI-driven app generation pipeline in `.claude/` — 6 creation stages + 4 change stages, state persisted in `.claude/thinking/{AppName}/workflow.json`.

## Quick Start

Create a new app: `/vibe {AppName} {Description}`
Modify an existing app: `/vibe {AppName} {ChangeDescription}`
Resume an existing workflow: `/vibe {AppName}`
Re-run from a specific stage: `/vibe {AppName} --from=04-codegen`

Creation mode runs 6 stages: Requirement Analysis -> Architecture Design -> Task Planning -> Code Generation -> Asset Generation -> Project Integration.
Change mode runs 4 stages: Change Impact Analysis -> Change Task Planning -> Change Code Implementation -> Change Verification.
You can also enter a requirement description directly, and the system will automatically detect and trigger the corresponding mode.

## Tech Stack

- **Framework**: React 18 + TypeScript + Vite
- **Styling**: Tailwind CSS (H5 Pages), CSS Modules (Components)
- **Icons**: Lucide React (No Emoji)
- **Storage**: Cloud NAS via `@/lib` (Repository Pattern)
- **Communication**: `postMessage` via `@/lib`

## Build & Commands

| Command | Description |
|---------|-------------|
| `pnpm install` | Install dependencies |
| `pnpm dev` | Dev server at `http://localhost:3000` |
| `pnpm build` | Production build |
| `pnpm run lint` | ESLint check + auto-fix |
| `pnpm run pretty` | Prettier formatting |
| `pnpm clean` | Clean build artifacts |
| `cd apps/webuiapps && pnpm test` | Vitest single run |
| `cd apps/webuiapps && pnpm test:watch` | Vitest watch mode |
| `cd apps/webuiapps && pnpm test:coverage` | Vitest coverage report |
| `pnpm test:e2e` | Playwright E2E (headless Chromium) |
| `pnpm test:e2e:ui` | Playwright E2E (interactive UI) |

**Environment setup**:

```bash
cp apps/webuiapps/.env.example apps/webuiapps/.env
```

**Docker**: Multi-stage build (Node 20 Alpine → Nginx 1.21.6 Alpine), port 3000. Build args: `ENV`, `SENTRY_AUTH_TOKEN`, `CDN_SOURCE`, `APP_PATH`, `BIZ_PROJECT_NAME`, `APP_ENVIRONMENT`.

**CI** (`.github/workflows/ci.yml`): `pnpm install --frozen-lockfile` → `pnpm run lint` → `pnpm build` on push/PR to main.

## Code Style

**Prettier** (`.prettierrc`): trailing commas, 2-space indent, semicolons, single quotes, 100 char width, always parens on arrows.

**ESLint** (`.eslintrc`):
- `eqeqeq: error` — must use `===`
- `no-var: error` — no `var`
- `no-console: warn` — only `console.warn`/`console.error`
- `@typescript-eslint/no-unused-vars: error` — unused vars error (`_` prefix params exempt)
- `@typescript-eslint/no-explicit-any: warn`
- `no-throw-literal: error` — must throw Error objects

**TypeScript**: strict mode, ES2020 target, `noUnusedLocals` + `noUnusedParameters`, path alias `@/*` → `src/*`.

**Commitlint**: Conventional commits. Allowed types: `build`, `chore`, `ci`, `docs`, `feat`, `fix`, `perf`, `refactor`, `revert`, `style`, `test`, `translation`, `security`, `changeset`.

**Pre-commit hook** (Husky + lint-staged): runs `pnpm run pretty` + `pnpm run lint` on `*.{ts,tsx}`.

**App conventions**:
- Icons: Lucide React only, no emoji
- Styling: Tailwind CSS (H5 pages), CSS Modules (components)
- Design tokens: CSS variables only, no hardcoded colors/spacing (see `.claude/rules/design-tokens.md`)
- i18n: each app uses `useTranslation('appName')` with its own namespace
- Concurrency: use `batchConcurrent` (batchSize=6) for multi-file async ops, no serial `await` or unbounded `Promise.all` (see `.claude/rules/concurrent-execution.md`)

## File Structure

```text
src/pages/{AppName}/
├── components/    # UI Components
├── pages/         # Sub-pages (H5 Templates)
├── data/          # Seed Data (JSON only)
├── assets/        # Generated Image Assets
├── mock/          # Dev Mock Data (TypeScript)
├── store/         # Context/Reducer
├── actions/       # Action Definitions
├── styles/        # CSS Variables
├── i18n/          # en.ts + zh.ts
├── meta/
│   ├── meta_cn/   # Chinese: guide.md + meta.yaml
│   └── meta_en/   # English: guide.md + meta.yaml
├── index.tsx      # Entry (lifecycle reports here only)
├── index.module.scss
└── types.ts
```

## Workflow Structure

```text
.claude/
├── commands/vibe.md           # Single entry orchestrator
├── workflow/
│   ├── stages/                # Stage definitions (loaded on-demand)
│   │   ├── 01-analysis.md          # Create: Requirement Analysis
│   │   ├── 02-architecture.md      # Create: Architecture Design
│   │   ├── 03-planning.md          # Create: Task Planning
│   │   ├── 04-codegen.md           # Create: Code Generation
│   │   ├── 05-assets.md            # Create: Asset Generation
│   │   ├── 06-integration.md       # Create: Project Integration
│   │   ├── 01-change-analysis.md       # Change: Impact Analysis
│   │   ├── 02-change-planning.md       # Change: Task Planning
│   │   ├── 03-change-codegen.md        # Change: Code Implementation
│   │   └── 04-change-verification.md   # Change: Verification
│   └── rules/                 # Stage-specific rules (loaded on-demand)
│       ├── app-definition.md
│       ├── responsive-layout.md
│       ├── guide-md.md
│       └── meta-yaml.md
├── rules/                     # Global rules (always loaded)
│   ├── data-interaction.md
│   ├── design-tokens.md
│   ├── concurrent-execution.md
│   └── post-task-check.md
└── thinking/{AppName}/        # Per-app workflow state & artifacts
    ├── workflow.json
    ├── 01-requirement-analysis.md
    ├── 02-architecture-design.md
    ├── 03-task-planning.md
    ├── 04-code-generation.md
    └── outputs/
        ├── requirement-breakdown.json
        ├── solution-design.json
        └── workflow-todolist.json
```

## Rules

The following global rules in `.claude/rules/` are mandatory constraints across all stages:

| Rule File | Description |
|---|---|
| `data-interaction.md` | Data interaction & NAS storage specification |
| `design-tokens.md` | Design token system & usage guidelines |
| `concurrent-execution.md` | Concurrent execution & task scheduling rules |
| `post-task-check.md` | Post-task completion checklist |

Stage-specific rules are in `.claude/workflow/rules/`, loaded on-demand by `/vibe`.

## Auto Workflow Trigger

When the user enters a requirement description directly (instead of using the `/vibe` command), automatically execute the workflow defined in `.claude/commands/vibe.md`.

### Creating a New App

Detection rules:

- The user's message contains an explicit new app requirement (e.g., "build a XX app", "I need a XX", "help me develop XX")
- Automatically extract AppName (PascalCase format) from the requirement; Description is the user's full requirement text
- Equivalent to executing `/vibe {AppName} {Description}`

### Modifying an Existing App

Detection rules:

- The user's message explicitly targets an existing App with a change request (e.g., "add lyrics feature to MusicApp", "MusicApp needs to support XX")
- Also includes cases where the App name is not specified but can be inferred from context (e.g., there is only one App, and the user says "add a lyrics display feature")
- Automatically identify AppName; Description is the change requirement text
- Equivalent to executing `/vibe {AppName} {ChangeDescription}` (vibe.md's mode detection logic automatically enters change mode)

### Cases That Do NOT Trigger

- The user is clearly asking questions, chatting, or making fine-grained code modifications (e.g., "change this button color to red")
- The user used the `/vibe` command (already handled by the command mechanism)
- The user requests resuming or re-running from a specific stage (must explicitly use `/vibe {AppName}` or `/vibe {AppName} --from=XX`)

## Testing

### Unit Tests (Vitest)

Run from the webuiapps package:

```bash
cd apps/webuiapps && pnpm test        # single run
cd apps/webuiapps && pnpm test:watch  # watch mode
```

- Config: `apps/webuiapps/vitest.config.ts`, environment: `happy-dom`
- Test files: `src/**/*.{test,spec}.{ts,tsx}`
- Existing tests in `apps/webuiapps/src/lib/__tests__/`
- Coverage threshold config: lines 75%, functions 85%, branches 70%
- **Quality bar**: changed code must exceed 90% coverage regardless of config thresholds

### E2E Tests (Playwright)

Run from the repo root. The dev server starts automatically.

```bash
pnpm test:e2e          # headless, Chromium only
pnpm test:e2e:ui       # interactive UI mode
```

- Config: `playwright.config.ts` (root)
- Tests: `e2e/` directory
- The web server (`pnpm dev`) is auto-launched on port 3000 and reused if already running.
- Only Chromium is configured by default; add projects in `playwright.config.ts` for Firefox/WebKit.
- After completing code changes that affect UI or routing, run `pnpm test:e2e` and report pass/fail.
- Use `data-testid` selectors, not class names or text content.

## Task completion quality bar (mandatory)

Before declaring a task complete, agents must satisfy all of the following:

1. **Unit tests must pass** for the affected package(s).
   - For `apps/webuiapps`, run the relevant Vitest command(s), for example:
     ```bash
     cd apps/webuiapps && pnpm test
     cd apps/webuiapps && pnpm test:coverage
     ```
2. **Code coverage must be > 90%** for the code touched by the task.
   - If current config thresholds are lower, do not treat that as sufficient.
   - Add or improve tests until the changed area exceeds 90% coverage, or explicitly report why that is not yet achievable.
3. **E2E coverage must be complete for impacted user flows.**
   - Do not stop at smoke tests if the change affects real behavior.
   - Cover the primary user path, key state transitions, and at least one meaningful assertion of successful behavior.
   - If UI behavior changes, prefer stable selectors (`data-testid`) over fragile class-name/text-only selectors.
4. **Report exact validation commands and results** in the final handoff.
   - Include what passed, what failed, and any known gaps.

Minimum expectation: no task is "done" if unit tests are red, coverage on changed code is below 90%, or impacted E2E coverage is missing/incomplete.

## Security

- No user auth — pure frontend app, all data local
- LLM API keys stored in `localStorage` (key: `webuiapps-llm-config`) and `~/.openroom/config.json`
- LLM proxy strips sensitive headers (`host`, `connection`, `content-length`, `x-llm-target-url`); custom headers via `x-custom-*` prefix
- Path sanitization in session data API: removes `..` and non-alphanumeric chars
- No hardcoded secrets; all env vars optional
- Optional Sentry integration (sourcemap upload on production build)
- Vulnerability reporting: private email disclosure, 48h acknowledgment (see `SECURITY.md`)

## Configuration

**Environment variables** (`apps/webuiapps/.env.example`):

| Variable | Required | Description |
|----------|----------|-------------|
| `CDN_PREFIX` | No | Static asset CDN prefix |
| `VITE_RUM_SITE` | No | RUM monitoring endpoint |
| `VITE_RUM_CLIENT_TOKEN` | No | RUM client token |
| `SENTRY_AUTH_TOKEN` | No | Sentry auth token |
| `SENTRY_ORG` | No | Sentry org slug |
| `SENTRY_PROJECT` | No | Sentry project slug |

**Runtime config** (`~/.openroom/`):

| File | Description |
|------|-------------|
| `config.json` | LLM config (provider, apiKey, baseUrl, model) |
| `characters.json` | Character config |
| `mods.json` | Mod config |
| `sessions/` | Session data directory |

**Supported LLM providers** (`src/lib/llmModels.ts`): OpenAI, Anthropic, DeepSeek, llama.cpp (local), MiniMax, Z.ai, Kimi, OpenRouter.

**Vite dev server API routes** (provided as plugins in `vite.config.ts`):

| Route | Purpose |
|-------|---------|
| `/api/llm-config` | LLM config read/write |
| `/api/session-data` | Session data CRUD |
| `/api/session-reset` | Session reset |
| `/api/llm-proxy` | LLM API proxy |
| `/api/characters` | Character config |
| `/api/mods` | Mod config |
| `/api/log` | Debug logging |
