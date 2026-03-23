# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Critical Tool Usage Rules

### Subagents
- **vue-expert**: **MANDATORY** for creating or significantly modifying any `.vue` file
- **code-reviewer**: Use after writing significant code
- **Explore**: Use for codebase exploration and pattern searches
- **general-purpose**: Use for complex multi-step tasks

### Skills
- **backend-api-test**: Use when writing or modifying tests in `tests/backend/`
- **saas-ui-redesign**: Use when redesigning the UI to a vertical sidebar / SaaS-style layout

### MCP Tools
- **ALWAYS use GitHub MCP tools** (`mcp__github__*`) for ALL GitHub operations
  - Exception: Local-only branches — use `git checkout -b` instead of `mcp__github__create_branch`
- **ALWAYS use Playwright MCP tools** (`mcp__playwright__*`) for browser testing
  - Frontend: `http://localhost:3000` | API: `http://localhost:8001`

## Stack

- **Frontend**: Vue 3 + Composition API + Vue Router + Axios (Vite, port 3000)
- **Backend**: Python FastAPI + Pydantic v2 + Uvicorn (port 8001)
- **Data**: JSON files in `server/data/` loaded into memory at startup via `server/mock_data.py` — no database, no persistence

## Commands

```bash
# Backend (requires uv; fallback: python3 -m pip install -r requirements.txt && python3 main.py)
cd server && uv run python main.py

# Frontend
cd client && npm install && npm run dev

# Backend tests (run from repo root)
cd tests && uv run pytest backend/ -v

# Run a single test file
cd tests && uv run pytest backend/test_inventory.py -v

# Run a single test by name
cd tests && uv run pytest backend/ -k "test_get_all_inventory" -v
```

## Architecture

### Data Flow
```
FilterBar.vue (useFilters composable)
  → getCurrentFilters() maps UI state to API params
  → client/src/api.js (axios, hardcoded to http://localhost:8001/api)
  → FastAPI (apply_filters + filter_by_month helpers)
  → In-memory lists from mock_data.py
  → Pydantic validation
  → View component refs → computed properties → template
```

### Filter System
`useFilters.js` is a singleton composable (state declared at module level, shared across all components). It holds 4 filter refs: `selectedPeriod`, `selectedLocation`, `selectedCategory`, `selectedStatus`. `getCurrentFilters()` maps these to API query params (`warehouse`, `category`, `status`, `month`).

- **Inventory** (`/api/inventory`): supports `warehouse`, `category` only — no month/time dimension
- **Orders** (`/api/orders`): supports all 4 filters; `month` accepts `YYYY-MM` or `Q1-2025` format
- **Dashboard summary** (`/api/dashboard/summary`): supports all 4 filters
- **Demand/Backlog** (`/api/demand`, `/api/backlog`): no filters

### Frontend Structure
- `App.vue` — shell layout + global styles (CSS variables, sidebar nav)
- `main.js` — router setup (6 routes: `/`, `/inventory`, `/orders`, `/demand`, `/spending`, `/reports`)
- Views in `client/src/views/` own their data loading and computed state
- Composables: `useFilters` (global filter state), `useAuth` (auth state), `useI18n` (en/ja localization)
- `client/src/utils/currency.js` — shared number formatting

### Backend Structure
- `server/main.py` — all endpoints + Pydantic models + `apply_filters` / `filter_by_month` helpers
- `server/mock_data.py` — loads all JSON files at import time; data lives in module-level lists
- All mutation endpoints (POST purchase orders, tasks CRUD) update the in-memory list only; restart resets data

### Testing
- Tests live in `tests/backend/` using pytest + FastAPI `TestClient`
- `conftest.py` adds `server/` to `sys.path` and provides `client` fixture + sample data fixtures
- Tests import directly from `server/main.py`

## Code Style

- Always document non-obvious logic changes with comments

## Key Gotchas

1. **`uv` may not be on PATH** — if `uv` is missing, install deps with `pip` and run with `python3` directly
2. **Data mutations don't persist** — POST/PATCH endpoints update in-memory lists; a server restart reloads original JSON
3. **Inventory has no time dimension** — don't add `month` filter to `/api/inventory`
4. **Pydantic models must stay in sync with JSON** — changing JSON structure requires updating the model in `main.py`
5. **v-for keys**: use unique fields (`sku`, `id`, `month`), never array index
6. **Date validation**: always check `!isNaN(date.getTime())` before calling `.getMonth()` etc.
7. **Revenue goals in data**: $800K/month (single month), $9.6M YTD (all months)

## Design System

- Colors: slate/gray (`#0f172a`, `#64748b`, `#e2e8f0`); status colors green/blue/yellow/red
- Charts: custom SVG; layouts use CSS Grid
- No emojis in UI
- Localization: English (`client/src/locales/en.js`) and Japanese (`client/src/locales/ja.js`) via `useI18n`
