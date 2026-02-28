# AGENTS.md

Guidance for AI coding agents (Claude Code, Cursor, Copilot, etc.) working in this repository.

## Project Overview

**Audapolis** is an open-source desktop app for editing spoken-word media with automatic transcription. All processing is offline — data stays local. Users include podcasters, journalists, and radio producers.

**Core concept:** a word-processor-like UX where users edit text and the underlying audio follows.

**Architecture:** monorepo with two sub-projects that communicate over a local REST API:

```
audapolis/
├── app/        # Electron desktop app (TypeScript, React, Redux Toolkit)
├── server/     # Python transcription backend (FastAPI, Vosk)
└── AGENTS.md   # ← you are here
```

The Electron app spawns the Python server as a child process on `127.0.0.1` with a random port and Bearer-token auth.

## Setup Commands

```bash
# 1. Prerequisites: Node.js 16, Python 3.8–3.10 (not 3.9.0, not >=3.11), Poetry

# 2. Install git hooks (runs lint, format, type-check, and fast tests on every commit)
pre-commit install

# 3. Install dependencies
cd app && npm install
cd server && poetry install

# 4. Run in development (from app/ — Electron spawns the Python server automatically)
cd app && npm start
```

In dev mode the main process runs `poetry run python run.py` from `../server`. In production it uses a bundled binary from `process.resourcesPath`.

## Development Workflow

### Frontend (`app/`)

| Command | What it does |
|---|---|
| `npm start` | Launch Electron + Python server in dev mode |
| `npm run build` | Compile TypeScript + Vite bundle |
| `npm run check` | Run tsc + ESLint (**zero warnings required**) |
| `npm run check:tsc` | TypeScript type-check only |
| `npm run check:eslint` | ESLint only |
| `npm run fmt` | Auto-format with Prettier |

### Backend (`server/`)

| Command | What it does |
|---|---|
| `poetry run uvicorn app.main:app --reload` | Run server with hot reload |
| `poetry run python run.py` | Run via entry script (picks random port) |
| `./build_server.sh` | Build standalone binary (requires Rust + PyOxidizer) |

### Working Across the Monorepo

- **Frontend changes only:** work in `app/`, run `npm run check` and `npm run test:fast`.
- **Backend changes only:** work in `server/`, format with `black .` and lint with `flake8`.
- **Cross-cutting changes:** run `pre-commit run --all-files` from the repo root to validate everything.
- The `app/src/server_api/api.ts` file contains typed wrappers for every server endpoint — update it whenever you add/change a FastAPI route.

## Testing

### How to run tests

```bash
# Fast unit tests (Jest) — REQUIRED to pass in CI and pre-commit
cd app && npm run test:fast

# E2E tests (Puppeteer) — allowed to be flaky in CI
cd app && npm run test:puppeteer

# All tests together
cd app && npm run test

# Run a single test file
cd app && npx jest path/to/file.spec.ts
```

### Test conventions

- **Unit tests:** `*.spec.ts` files colocated with source code.
- **E2E tests:** `*.pup.spec.ts` files, run with Puppeteer (uses `xvfb-run` on Linux).
- Pre-commit hooks run fast tests on every commit.
- When adding features, always add Jest unit tests. E2E tests are a bonus.

### Existing test coverage

Tests exist for: document model (`core/document.spec.ts`), WebVTT export (`core/webvtt.spec.ts`), editor state (edit, play, selection, selectors, transcript correction), and utilities.

## Code Style

### TypeScript / React (`app/`)

- **Strict TypeScript**: `strictNullChecks` and `noImplicitAny` enabled.
- **`no-explicit-any` is OFF**: using `any` is allowed, but prefer proper types.
- **Unused imports are errors**: enforced by `eslint-plugin-unused-imports`. Prefix unused vars with `_`.
- **Exported functions need return types**: `explicit-module-boundary-types` is a warning.
- **Prettier config** (`.prettierrc`): 100-char line width, single quotes, `es5` trailing commas.
- **Import style**: use ES module imports. The `eslint-plugin-import` enforces resolution rules.
- **Styling**: use Styled Components for component-scoped CSS. Use Evergreen UI primitives where possible.
- **State**: all state lives in Redux Toolkit slices under `app/src/state/`. Use `createSlice` + `createAsyncThunk`. Never mutate state outside Immer drafts.
- **IPC**: never call `ipcRenderer` / `ipcMain` directly in components. Use the abstractions in `app/ipc/ipc_main.ts` (main process) and `app/ipc/ipc_renderer.ts` (renderer).

### Python (`server/`)

- **Black** with default line length (88). No custom config — just run `black .`.
- **isort** with `profile = "black"` (in `pyproject.toml`).
- **Flake8** with `max-line-length = 100`, `E203` ignored (in `.flake8` at repo root).
- **Pydantic v1** models for request/response schemas.
- Use async/await for FastAPI route handlers.
- Long-running work (transcription, model downloads) must use `BackgroundTasks`, not block the request.

## Build and Deployment

### CI/CD (GitHub Actions)

**`check.yml`** — runs on every PR and push to `main`:
1. **Lint job**: `pre-commit run --all-files` (isort, Black, Flake8, ESLint `--max-warnings 0`, tsc, fast tests).
2. **Test job**: `npm run test:fast` (required) + `npm run test:puppeteer` (allowed to fail).

**`build.yml`** — runs on push to `main` and PRs to `main`:
- Skips if only docs/specs changed.
- Builds cross-platform (macOS, Ubuntu, Windows): Python 3.10 + Rust → PyOxidizer server binary → Node 16 → Electron app.
- Auto-releases to GitHub Releases on `main`.

### Before pushing

```bash
# Run all the same checks CI will run
pre-commit run --all-files
```

This validates: trailing whitespace, end-of-file fixup, isort, Black, Flake8, ESLint (`--max-warnings 0`), tsc, and Jest fast tests.

## Pull Request Guidelines

- Reference the related issue in the PR description.
- Ensure `pre-commit run --all-files` passes locally before pushing.
- Ensure the branch merges cleanly into `main`.
- Open PRs as draft if work is still in progress.
- All `check.yml` jobs must pass (lint + fast tests). Puppeteer flakes are OK.

## Directory Layout

```
app/
├── main_process/           # Electron main process
│   ├── index.ts            # Window setup, app lifecycle
│   ├── server.ts           # Spawns & manages the Python server
│   ├── menu.ts             # Menu bar
│   └── types.ts            # Shared types (ServerInfo, etc.)
├── ipc/                    # IPC layer (NOTE: sibling to src/, not inside it)
│   ├── ipc_main.ts         # Main process handlers
│   └── ipc_renderer.ts     # Renderer wrappers
├── src/                    # React renderer process
│   ├── components/         # Reusable UI components
│   ├── pages/              # Page-level components (Editor, Landing, etc.)
│   ├── state/              # Redux slices
│   │   ├── editor/         # Document editing (largest slice, uses redux-undo)
│   │   ├── nav.ts          # Navigation / UI state
│   │   ├── models.ts       # ML model availability
│   │   ├── server.ts       # Server connection config
│   │   ├── transcribe.ts   # Transcription task tracking
│   │   └── index.ts        # Store setup
│   ├── core/               # Core logic (document model, player, ffmpeg, exports)
│   ├── server_api/api.ts   # Typed fetch wrappers for every server endpoint
│   ├── util/               # Helpers (including logging: util/log.ts)
│   ├── types/              # TypeScript declarations
│   └── tour/               # Onboarding tour
├── .eslintrc.json
├── .prettierrc
├── tsconfig.json
└── package.json

server/
├── app/
│   ├── main.py             # FastAPI app + all route definitions
│   ├── transcribe.py       # Vosk speech-to-text logic
│   ├── models.py           # Model download/list/delete
│   ├── tasks.py            # Background task queue
│   ├── otio.py             # OpenTimelineIO export
│   ├── config.py           # Server config
│   └── models.yml          # Available transcription models
├── run.py                  # Entry point (port selection, uvicorn launch)
├── pyproject.toml
└── poetry.lock
```

## App–Server Communication

Startup handshake (two sequential JSON messages on stdout):

1. `run.py` picks a random port (40000–60000), prints `{"msg": "server_starting", "port": <port>}`.
2. FastAPI's `startup` event prints `{"msg": "server_started", "token": "<token>"}` once uvicorn is ready.
3. `app/main_process/server.ts` parses both and publishes to the renderer via IPC.
4. Redux `server` slice stores the connection config.
5. Components call `app/src/server_api/api.ts` helpers, which inject `Authorization: Bearer <token>`.

**Auth:** base64-encoded 64 random bytes, regenerated each launch.

### Server API Endpoints

| Method | Path | Purpose |
|---|---|---|
| POST | `/tasks/start_transcription/` | Start async transcription |
| POST | `/tasks/download_model/` | Download a model |
| GET | `/tasks/list/` | List all tasks |
| GET | `/tasks/{uuid}/` | Get task status |
| DELETE | `/tasks/{uuid}/` | Remove a task |
| GET | `/models/available` | List available models |
| GET | `/models/downloaded` | List downloaded models |
| POST | `/models/delete` | Delete a downloaded model |
| POST | `/util/otio/convert` | Export to OpenTimelineIO |

All endpoints require Bearer token auth. Note: some paths have trailing slashes, some don't — match exactly or you'll get 307 redirects.

## Debugging and Troubleshooting

### Electron / frontend

- **DevTools**: `Cmd+Alt+I` (macOS) / `Ctrl+Alt+I` opens Chromium DevTools in the renderer. DevTools also auto-open in development mode.
- **Log files**: the main process writes structured JSON logs to `app.getPath('logs')/<timestamp>.log`. Old logs (>5 days) are auto-deleted.
- **Export logs**: the app has a built-in "export debug logs" feature that zips the log file (`app/src/util/log.ts`).
- **Server stderr**: the main process captures Python server stderr and forwards it via IPC channel `local-server-stderr`.

### Python server

- Run standalone for easier debugging: `cd server && poetry run uvicorn app.main:app --reload --log-level debug`
- FastAPI auto-generates interactive docs at `http://127.0.0.1:<port>/docs` (but you'll need the Bearer token).

### Common issues

- **Python version**: `^3.8, !=3.9.0, <3.11`. Using 3.11+ will fail dependency resolution.
- **Server port**: never hardcode a port. Always read from Redux `server` slice or IPC messages.
- **Trailing slashes**: some endpoints have them (`/tasks/start_transcription/`), some don't (`/models/available`). Mismatches cause 307 redirects.
- **Lint is strict**: ESLint with `--max-warnings 0`. Any warning is a CI failure.
- **Black vs Flake8 line length**: Black uses 88 (default), Flake8 allows 100. Lines between 88–100 chars will pass Flake8 but get reformatted by Black.
- **Models at runtime**: transcription models download to the user's app data directory, not the repo.
- **IPC directory**: `app/ipc/` is a sibling of `app/src/`, not inside it. Renderer imports use `../../ipc/...`.
- **Context isolation disabled**: Node.js APIs are accessible in the renderer. Intentional for this Electron version, but be careful with security-sensitive code.
- **Vosk version**: `!=0.3.43` is excluded in `pyproject.toml` — don't pin that version.
- **CORS**: the server allows all origins (`"*"`) for dev convenience. Do not add restrictive CORS rules without understanding the Electron context.

## Security Considerations

- **Auth token**: generated fresh each launch, never persisted. All endpoints require it.
- **CORS wildcard**: the server sets `allow_origins=["*"]` — acceptable because it only binds to `127.0.0.1` and the token provides access control.
- **Context isolation OFF**: the Electron renderer has full Node.js access. Don't introduce code that evaluates untrusted content in the renderer.
- **No remote code**: models are downloaded from known URLs defined in `server/app/models.yml`. Don't add arbitrary download sources.
