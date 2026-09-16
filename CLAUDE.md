# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

Odysseus is a self-hosted, single-binary-ish AI workspace: a FastAPI backend serving a
no-build-step vanilla-JS frontend, with chat, an agent loop, model serving (Cookbook),
deep research, documents, memory/skills, email, calendar, notes and tasks.

## Commands

```bash
# First run (creates data dirs, DB, admin user; safe to re-run)
python setup.py
python -m uvicorn app:app --host 127.0.0.1 --port 7000

# Docker (the recommended path for normal testing)
docker compose up -d --build
docker compose logs --tail=120 odysseus

# Tests — ./data must exist (sqlite DB lives at ./data/app.db)
python -m pytest -q
python -m pytest tests/test_agent_loop.py::test_name      # single test
python -m pytest -m area_security                          # taxonomy marker
python -m pytest -m "area_services and sub_cookbook"

# Focused runs (validates area/sub-area names, passes extras after --)
python tests/run_focus.py --area security
python tests/run_focus.py --fast                           # fast lane = `not slow`
python tests/run_focus.py --area services --durations 25
python tests/run_focus.py --area services -- --maxfail=1 -q

# Syntax gates (what CI runs)
python -m compileall -q app.py core routes src services scripts tests
node --check static/js/<file-you-changed>.js
docker compose config                                      # for compose changes
```

Tests are tagged automatically at collection time by `tests/conftest.py` /
`tests/_taxonomy.py` with an `area_*` and a `sub_*` marker derived from the filename —
no marker needs to be written by hand. `slow` is the one opt-in marker and requires
duration evidence (`--durations`), not a guess. See `tests/README.md` (helper reference)
and `tests/TESTING_STANDARD.md` (the rules).

### Local environment notes (Windows)

- Windows is not actively tested upstream; Docker or a Linux/macOS install is the
  reference environment.
- Without UTF-8 mode, collection dies with `UnicodeDecodeError: 'charmap'` on tests that
  read `static/js` sources. Run with `PYTHONUTF8=1` (verified fix).
- A partial dependency install shows up as import/collection errors that look like real
  failures (`markdown`, `nh3`, `pytest_asyncio`). Install the full `requirements.txt`.

## Architecture

### Request path and wiring

`app.py` (~1150 lines) is the composition root, not a place for business logic. It:

1. Builds the middleware stack, **outermost first**: `AuthMiddleware` → request timeout →
   `SecurityHeadersMiddleware` → GZip → CORS.
2. Calls `src/app_initializer.initialize_managers()`, which constructs every long-lived
   manager (session, memory, skills, presets, API keys, chat/research handlers,
   model discovery) and returns them in a dict.
3. Registers each router via a `setup_*_routes(<managers it needs>)` factory from
   `routes/` — dependencies are passed in explicitly, not imported globally.
4. Runs startup/shutdown in the `_lifespan` context manager (schedulers, email pollers,
   background jobs).

Layering: `routes/` (HTTP + pydantic models, ~54 modules) → `src/` and `services/`
(logic) → `core/` (auth, DB models, session manager, middleware). `services/` holds the
larger subsystems that own a directory (`search/`, `memory/`, `research/`, `hwfit/`,
`tts/`, `stt/`, `docs/`, `shell/`, `youtube/`).

### Auth and owner scoping

This is the invariant most tests exist to protect. Four ways a request authenticates, all
resolved in `AuthMiddleware` (`app.py`):

- **Cookie session** → `request.state.current_user` = username.
- **Bearer `ody_…` API token** → `current_user` is the sandboxed pseudo-user `"api"`;
  the real owner is on `request.state.api_token_owner`.
- **Internal-tool loopback** (`X-Odysseus-Internal-Token`, `core/middleware.py`) — how the
  agent tool layer reaches admin-gated routes; optional `X-Odysseus-Owner` attributes the
  call to a real user. Gated on `_is_trusted_loopback`, which rejects anything carrying
  proxy/tunnel headers so a Cloudflare tunnel can't inherit local trust.
- **`LOCALHOST_BYPASS`** — dev only, same trusted-loopback gate.

Routes then use `src/auth_helpers.py`: `require_user` (owner-scoped user data; rejects
bearer tokens), `require_authenticated_request` (auth but no owner-scoped data),
`require_privilege(request, key)` (per-user flags in `auth.json`; admins pass everything;
unknown keys fail open), `effective_user` (ownership/attribution — resolves a token to its
owner so a paired phone sees the same data as the desktop), and `owner_filter(query,
Model, user)`. `require_user` returning `""` is legitimate single-user/anonymous mode, not
an error. A `NULL` owner column means a legacy shared row and is included by default.

### Agent and LLM layers

- `src/agent_loop.py` — multi-round streaming loop: system-prompt assembly from tool sets,
  request classification, tool-block resolution, runaway-call detection, verifier subagent.
- Tools split across `src/tool_schemas.py` / `tool_parsing.py` / `tool_execution.py` /
  `tool_policy.py` / `tool_security.py` / `tool_index.py`, with implementations in
  `src/agent_tools/` (filesystem, subprocess, web, document). `src/builtin_actions.py`
  reaches app features by HTTP-looping back into the app's own API with the internal token.
- `src/llm_core.py` — one provider-agnostic async client. Provider is *detected from the
  URL* (Ollama native vs OpenAI-compatible, Anthropic, Copilot, ChatGPT subscription,
  self-hosted vLLM/llama.cpp), which drives payload shape, headers, thinking support and
  temperature quirks. Dead-host tracking and response caching live here too.
- `src/endpoint_resolver.py` — turns `ModelEndpoint` DB rows into a concrete
  (url, model, headers) per owner, including chat/utility/vision fallback chains.
- `mcp_servers/` exposes Odysseus subsystems (email, memory, RAG, image gen) over MCP;
  `src/builtin_mcp.py` and `routes/mcp_routes.py` are the client side.

### Persistence and config

- `core/database.py` — SQLAlchemy models plus hand-rolled `_migrate_add_*` functions
  (no Alembic). `init_db()` runs at **module import**, so importing `core.database`
  creates and migrates `data/app.db` as a side effect. New columns need a matching
  `_migrate_*` function added to `init_db()`.
- `src/constants.py` is the single source of truth for every path and shared limit, and the
  only place that reads `ODYSSEUS_DATA_DIR`. `core/constants.py` only re-exports it.
- `src/settings.py` — runtime settings (`data/settings.json`), per-user settings and
  feature flags. `.env` is for deployment-level knobs only (`APP_BIND`, `APP_PORT`,
  `AUTH_ENABLED`, `DATABASE_URL`, `LOCALHOST_BYPASS`, upload size caps).
- Optional subsystems degrade instead of crashing: ChromaDB/fastembed missing → keyword
  fallback; that pattern (`logger.warning("… DEGRADED")` + `None`) is the house style.

### Frontend

No bundler, no build step. `static/index.html` loads 36 raw ES modules directly
(`static/js/` holds ~148 `.js` files across 13 subdirectories; the rest are imported by
those entry points). `{{CSP_NONCE}}` in the HTML is substituted per
request by `_serve_html_with_nonce` in `app.py`; inline scripts must carry the nonce.
`static/js/MODULE_SUMMARY.md` is explicitly a partial historical overview — trust the tree
and the script tags, not that file. Static sources are served with revalidation headers so
edits show up without a cache bust.

### CLI and integrations

`scripts/odysseus` is a git-style dispatcher over the `scripts/odysseus-*` siblings
(`odysseus mail list` ≡ `odysseus-mail list`). `companion/` is the LAN pairing bridge
(see `companion/README.md` for its auth matrix). `integrations/claude` and
`integrations/codex` ship skills for external coding agents.

## Conventions enforced in review

- **Never hardcode paths.** Import the named constant from `src/constants.py`
  (`AUTH_FILE`, `SETTINGS_FILE`, `CHROMA_DIR`, …); don't rebuild it from `DATA_DIR`,
  `Path(__file__)`, `/app/...`, or a relative `"data/..."`. Use `DATA_DIR` directly only
  for dynamic per-owner paths. Add a constant if one is missing. The source tree is
  read-only in Docker, so guard directory creation and degrade gracefully.
- **Never hardcode `http://localhost:7000`** — use `internal_api_base()` from
  `src.constants`.
- **Visual changes are held to the existing style.** Reuse CSS variables (`--red`, `--fg`,
  `--bg`, `--card`, `--border`) and existing button/input/card classes; no new color, font
  size, or spacing values; **no Unicode emoji in UI or code** (inline monochrome SVG or
  plain text); Fira Code is the UI font; dark is the default theme and light mode goes
  through the theme system. Extend an existing widget rather than adding a parallel one.
  Run the app in a browser and attach a screenshot for anything touching HTML/CSS/SVG or a
  `static/js/` module that draws to the DOM.
- **Commits:** Conventional Commits (`fix(search): …`, `feat(notes): …`). PRs target `dev`,
  never `main`. Keep them small and single-purpose; don't mix file moves with logic changes.
- **Tests:** behavior-first (not assertions on source text or AST), deterministic, isolated
  from `sys.modules`/`os.environ`/CWD, order-independent. Don't add `skip`/`xfail` to get
  CI green. Reach for a helper in `tests/helpers/` only when duplication is already proven.

## Gotchas

- Adding a long-running or SSE route? Add its prefix to `_TIMEOUT_EXEMPT_PREFIXES` in
  `app.py`, or the 45s `_RequestTimeoutMiddleware` will kill it with a 504. GZip already
  excludes `text/event-stream`, so streams aren't buffered.
- The pytest job in CI is `continue-on-error: true` — the suite has known flaky and
  environment-dependent failures, so a green check does not mean a green suite. Run the
  focused slice for what you touched and read the output.
- Merge-blocking CI is the security suite (gitleaks, actionlint/zizmor, dependency review,
  hadolint) plus the syntax jobs; Trivy and CodeQL are advisory. See `docs/security-ci.md`.
- Docs-only PRs skip pytest entirely by path detection in `.github/workflows/ci.yml`.
