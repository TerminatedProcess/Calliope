# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

Calliope is a local-first story-to-video studio: an LLM drafts story → script → shot clips, ComfyUI renders them over HTTP, ffmpeg stitches the film. Upstream targets Windows (`*.bat` scripts, `.venv\Scripts\...`); this checkout runs on Linux, so use the POSIX equivalents below. No venv or `node_modules` is committed — set them up first. Locally, the fish `.salias` provides `mkenv` (root `.venv`, Python 3.13, direnv-activated), `install` (backend editable + `npm install`), and `rmenv`; with that root venv active, run the Python commands below directly instead of via `uv run`.

## Commands

Backend (`calliope-backend/`, Python ≥3.11, `uv.lock` present):

```bash
cd calliope-backend
uv sync --extra dev                      # or: python -m venv .venv && .venv/bin/pip install -e ".[dev]"
uv run python -m calliope.main --host 127.0.0.1 --port 8247
uv run pytest                            # asyncio_mode=auto
uv run pytest tests/test_cli_plan.py::test_name -q
uv run ruff check src tests              # line-length 100, rules E,F,I,N,W
uv run python -m calliope.cli.main --help   # calliope-cli (calliope-cli.bat wraps this)
uv run python scripts/export_contracts.py   # regenerate contracts.json (see below)
```

Frontend (`calliope-web/`, SvelteKit 2 + Svelte 5, adapter-static SPA):

```bash
cd calliope-web
npm install
npm run dev        # :5173, proxies /api -> 127.0.0.1:8247
npm run check      # svelte-check (type check)
npm test           # node scripts: SSR-storage guard + carry-values guard
npm run lint       # prettier --check + eslint
npm run build      # -> build/, which PyInstaller (calliope.spec) bundles as calliope/static
```

Config: `calliope-backend/calliope_config.json` (copy from `.example`; holds API keys, never commit). Data defaults to `calliope-backend/data/`.

## Architecture

**Backend** (`src/calliope/`): FastAPI app built in `main.py:create_app`; routers under `/api/*`; serves the built SPA from `calliope/static` if present. SQLite via plain `sqlite3`; schema + additive `ALTER TABLE` migrations live in `db.py:migrate_db` (add new columns there, guarded). `config.py` holds a mutable `settings` singleton persisted to `calliope_config.json`.

- **Job queue** (`queue/`): SQLite-backed queue + background worker started in the app lifespan. All ComfyUI renders go through it. The worker talks to ComfyUI only via HTTP (`/upload/image`, `/prompt`, `/history`) — never touch ComfyUI's local input/output folders.
- **ComfyUI integration** (`comfyui/`): no hardcoded node IDs. Editable nodes are discovered from node-title role tags like `Name (Input:prompt)` / `(Output:video)` in API-format workflow JSON (`roles.py` alias tables → `parser.py` → `patcher.py`). `profiles.py` selects prompt format (`prose` vs `minimax_h3_ref`, a deterministic six-section compile; the LLM rewrite only runs on explicit Regenerate). Generic `(Input:image)` slots fill in node-id order, which defines `<Subject N>` numbering. `dry_run.py` produces placeholder results (tests run with `dry_run=True`).
- **Events** (`events/bus.py`): in-process SSE bus; frontend subscribes via `/api/events`.
- **Export** (`export/runner.py`): ffmpeg normalize → 0.5s crossfades → loudness normalize, at the clips' majority frame rate. Runs locally, no ComfyUI.
- **Agent harness** (`agent/harness/`): plugin-composed tool registry. `__init__.py` is the composition root; `plugins/*` each contribute tools/prompt sections/hooks (tool names unique across plugins); `registry.py` runs every tool through a pre-execute → execute → post-execute pipeline where policy lives (`policy.py`: the user's own words are the permission; negation is clause-scoped). `project_id` is injected via `ToolContext`, never an LLM argument. `log.py` is an append-only session event log (`agent_events`) — LLM history and UI chat rows are *derived* from it, never stored separately. `loop.py` = turn/step driver; `orchestrator.py` = planner + role-scoped sub-agents (`ROLE_TOOLS`); `runner.py` = asyncio task lifecycle (agent work stays off the render queue). `tools.py` is a compat shim — new code uses `get_registry()` / `get_prompts()`. `plugins/comfy_mcp.py` is intentionally not loaded.
- **Skills**: builtins in `skills_builtin/` are seeded once into `<data_dir>/skills/` without overwriting user edits (`agent/skills_store.py`).
- **Authoring + CLI** (`authoring/`, `cli/`): `calliope-cli` writes project *text* only. `authoring/` is the only SQL the CLI uses: functions take a caller-owned connection and never commit; `cli.context.cli_txn` owns the transaction so the `cli_audit` row and data land together. Concurrency guard is content hashing (`authoring/hash.py`, `--expect-hash`), not a revision counter. Hard rules enforced by tests (`test_cli_boundary.py`, `test_cli_physical.py`): no delete-like verbs (`cli/policy.py` FORBIDDEN_VERBS/OPS), no model calls, no image/video writes, single filesystem write (audit JSONL mirror). Exit codes in `cli/io.py` are a contract with `skills_opencode/calliope/SKILL.md`: 2 = bad payload, 3 = hash drift.

**Frontend** (`calliope-web/src/`): SPA (SSR still evaluates modules — use `lib/storage.ts` helpers, never `typeof localStorage`; `npm test` enforces this). `lib/api.ts` is the REST client, `lib/events.ts` the SSE client. UI strings go through `lib/i18n.svelte.ts` with seven dictionaries in `lib/i18n/` (en is the type source — add keys to all). Main routes: `project/[id]` (Story/Assets/Script/Video stages), `canvas/[id]` (AI Canvas agent chat, @xyflow), `build-scene` (Three.js shot composer driven by agent `shot_*` tools, no ComfyUI), `playground`, `library`, `settings`.

## Cross-layer contracts

Backend and frontend share stringly-typed names (agent tool names, harness event types, `event_bus.publish("…")` names, ComfyUI roles, prompt profiles, guard codes). After changing any of these, rerun `scripts/export_contracts.py` and commit `contracts.json`; `tests/test_contract_vocabulary.py` fails if the web bundle references a name that no longer exists.

## Gotchas

- Version is single-sourced in `src/calliope/__init__.py` (`__version__`); `calliope-web/package.json` version is bumped separately.
- Tests must never write the real config: `tests/conftest.py` sets `CALLIOPE_CONFIG_FILE` before importing `calliope`. Keep that ordering in any new test entry point.
- Release notes go in `docs/changelogs/<date>-<slug>.md` and get indexed in `docs/changelogs/README.md` (newest first).
