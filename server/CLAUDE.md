# CLAUDE.md - Server

FastAPI backend for the inventory management system.

## Running

```bash
uv run python main.py     # http://localhost:8001, docs at /docs
```

## Project-Specific Patterns

**Single-file app**: All Pydantic models, filter functions, and route handlers live in `main.py`. No routers or separate modules.

**Mock data**: `mock_data.py` loads 7 JSON files from `data/` into module-level variables at import time. No persistence — server restart resets everything. Use `generate_data.py` to regenerate `orders.json`.

**Filter functions**: Two functions handle all filtering:
- `apply_filters(items, warehouse, category, status)` — skips filter if `None` or `'all'`, case-insensitive for category/status
- `filter_by_month(items, month)` — supports direct month (`2025-01`) and quarter (`Q1-2025`) via `QUARTER_MAP`, filters on `order_date` string

**Adding an endpoint**: Define Pydantic model → add `@app.get` handler → apply filters if needed → write tests in `tests/backend/`.

**CORS**: Wide open (`allow_origins=["*"]`) for local dev.

## Tests

```bash
cd tests && uv run pytest backend/ -v
```

Synchronous TestClient tests. Fixtures in `conftest.py` provide `client`, `sample_inventory_item`, `sample_order`.
