# CLAUDE.md - Client

Vue 3 frontend for the inventory management system.

## Dev Server

```bash
npm run dev    # http://localhost:3000
npm run build  # production build to dist/
```

## Project-Specific Patterns

**Component style**: All components use `export default { setup() {} }` with explicit `return` — not `<script setup>`. Stay consistent.

**Singleton composables**: `useFilters.js`, `useAuth.js`, `useI18n.js` declare `ref()` at module scope so state is shared across all consumers. Don't move refs inside the exported function.

**Filter → API mapping**: `getCurrentFilters()` maps frontend names to API params (`selectedLocation` → `warehouse`, `selectedPeriod` → `month`). The `'all'` value means "no filter" and is stripped in `api.js` before sending.

**Inventory view strips time/status filters** before calling the API — inventory has no time dimension.

**i18n**: Two locales (`en.js`, `ja.js`). Currency auto-switches: EN → USD, JA → JPY (fixed 150 rate in `utils/currency.js`). Translations use dot-notation keys with `{placeholder}` interpolation.

**Charts**: All custom SVG in templates — no charting library. Keep chart calculations in `computed()`.

**Styles**: Global styles in `App.vue` (unscoped). Component styles use `<style scoped>`. All colors are hardcoded hex values (slate/gray palette). Status colors: green (success), blue (info), yellow/orange (warning), red (danger).
