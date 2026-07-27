# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

Factory Inventory Management System — full-stack demo app with Vue 3 frontend, Python FastAPI backend, and in-memory mock data (no database).

> **This repository is PUBLIC.** Do not commit credentials, internal hostnames, or private registry URLs. `client/.npmrc` pins the public npm registry and `client/package-lock.json` is gitignored — leave both in place.

## Commands

```bash
# Backend (from repo root)
cd server && uv run python main.py          # starts on :8001, docs at /docs

# Frontend (from repo root)
cd client && npm install && npm run dev      # starts on :3000

# Tests (from repo root)
cd tests && uv run pytest backend/ -v        # all backend tests
cd tests && uv run pytest backend/test_inventory.py -v              # single file
cd tests && uv run pytest backend/test_inventory.py::test_name -v   # single test

# Frontend build
cd client && npm run build                   # outputs to client/dist/
```

No linters or formatters are configured. No frontend tests exist.

## Architecture

### Data Flow

```
FilterBar.vue (4 dropdowns)
    ↓ v-model binds to singleton refs in useFilters.js
View components watch filter refs → call getCurrentFilters()
    ↓ returns { warehouse, category, status, month }
api.js builds URLSearchParams (skips 'all' values) → GET request
    ↓
FastAPI receives query params → apply_filters() + filter_by_month()
    ↓ filters in-memory lists loaded from server/data/*.json at startup
Pydantic-validated response → Axios → stored in component refs → computed properties derive display data
```

### Filter System

Four global filters shared via singleton composable pattern: `client/src/composables/useFilters.js` declares `ref()` at **module scope** (outside the exported function), so all importers share the same state.

- **Time Period** → mapped to `month` query param. Supports `2025-01` (month) and `Q1-2025` (quarter) formats.
- **Warehouse** → `warehouse` param, exact match
- **Category** → `category` param, case-insensitive
- **Order Status** → `status` param, case-insensitive

**Inventory has no time dimension** — the `month` and `status` filters do not apply to `/api/inventory`.

Revenue goals: $800K/month for a single month, $9.6M when period is "all" (12 months).

### Backend (server/)

Single-file FastAPI app — all Pydantic models, filter logic, and routes live in `server/main.py`. Mock data loaded from JSON files via `server/mock_data.py` at import time. No persistence; restart resets state.

Key endpoints:
| Endpoint | Filters |
|---|---|
| `GET /api/inventory` | warehouse, category |
| `GET /api/orders` | warehouse, category, status, month |
| `GET /api/dashboard/summary` | all four |
| `GET /api/demand`, `/api/backlog` | none |
| `GET /api/spending/*` | none |
| `GET /api/reports/quarterly`, `/monthly-trends` | none |

### Frontend (client/)

Vue 3 + Composition API + Vue Router (HTML5 history) + Axios. No Vuex/Pinia — state managed through composable singletons and local component refs. Charts are custom SVG (no charting library). i18n supports English and Japanese with currency switching (USD/JPY at 150 rate in `client/src/utils/currency.js`).

Routes (all eagerly loaded in `client/src/main.js`): `/` Dashboard, `/inventory`, `/orders`, `/demand`, `/spending`, `/reports`.

Components use `export default { setup() {} }` pattern (not `<script setup>`).

Global styles in `client/src/App.vue` (unscoped, ~320 lines). Component styles use `<style scoped>`. Colors are hardcoded hex — no CSS custom properties, no dark mode.

### Testing (tests/)

pytest + FastAPI TestClient (synchronous). Config in `tests/pytest.ini`. Fixtures in `tests/backend/conftest.py`. ~36 tests across 3 files: `test_dashboard.py`, `test_inventory.py`, `test_misc_endpoints.py`.

## Subagents

- **vue-expert**: **Mandatory** for creating or significantly modifying `.vue` files
- **code-reviewer**: Use after writing significant code
- **Explore**: For codebase search and understanding

## Skills

- **backend-api-test**: Use when writing or modifying tests in `tests/backend/`

## MCP Tools

- **GitHub MCP** (`mcp__github__*`): Use for ALL GitHub operations. Exception: local branches use `git checkout -b`.
- **Playwright MCP** (`mcp__playwright__*`): Use for browser testing against `:3000` (frontend) and `:8001` (API).

## Code Style

- Always document non-obvious logic changes with comments

## Gotchas

1. Use unique keys in `v-for` (`sku`, `month`, etc.) — never array index
2. Validate dates before `.getMonth()` calls
3. Update Pydantic models in `main.py` when changing JSON data structure
4. Several frontend API calls (`/api/tasks/*`, `/api/purchase-orders/*`) have no backend implementation — they 404
5. `Backlog.vue` exists but has no route in the router
6. No emojis in UI
