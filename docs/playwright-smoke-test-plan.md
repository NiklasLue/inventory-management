# Playwright Smoke Test Plan

Captured from a Playwright MCP run on 2026-05-20. Use this as the working punch list when picking the work back up.

## Bugs to fix (priority order)

### 1. Backend `/api/tasks` endpoint is missing (404)
- **Symptom**: `App.vue:36` calls `GET /api/tasks` on mount; backend returns 404; console shows `Failed to load tasks: AxiosError`.
- **Decide first**: is the frontend calling a stale endpoint, or is the backend missing an endpoint that was planned? `CLAUDE.md` endpoint list does not mention `/api/tasks`.
- **Fix options**:
  - If the endpoint was planned → add `GET /api/tasks` to `server/main.py` (mirror the pattern of existing endpoints; data lives in `server/data/`).
  - If the call is stale → remove the loader from `client/src/App.vue` and any UI that consumes the response.

### 2. `PurchaseOrderModal` component not registered (Vue warning)
- **Symptom**: `[Vue warn]: Failed to resolve component: PurchaseOrderModal` fires twice during Dashboard mount/unmount.
- **Impact**: The "Create PO" buttons in the **Inventory Shortages** table on the dashboard render but do nothing visible — the modal element silently fails to mount.
- **Fix**: Either create the component at `client/src/components/PurchaseOrderModal.vue` and import + register it in `client/src/views/Dashboard.vue`, or remove the `<PurchaseOrderModal>` reference and the "Create PO" buttons until the feature is real.
- **Per CLAUDE.md**: any `.vue` file work must be delegated to the `vue-expert` subagent.

### 3. Missing favicon (cosmetic)
- **Symptom**: `GET /favicon.ico → 404`.
- **Fix**: drop a `favicon.ico` into `client/public/`. Low priority.

## Coverage gaps from this run

The run only verified that each route loads and that console state didn't worsen across navigation. The following were **not** exercised and should be in the next test pass:

- **Filters**: Time Period, Location, Category, Order Status — apply each filter and confirm dashboard KPIs / tables update via the query-param flow described in `CLAUDE.md`.
- **Table interactions**: row clicks on Top Products by Revenue, sorting (if any), "Create PO" button (will fail until bug #2 is fixed).
- **Inventory page**: filter combinations, low-stock highlighting.
- **Orders page**: status filter behavior, order detail navigation.
- **Restocking / Demand / Finance / Reports**: not opened with intent — just verified they render.
- **Backlog view**: `Backlog.vue` exists in `client/src/views/` but is not in the main nav. Either it's wired into another page (verify) or it's orphaned (delete or surface in nav).

## How to re-run this test

Prereqs (one-time, already done on this machine — see [windows-mcp-stdio-gotchas](../../.claude/projects/C--Users-User-Documents-Programming-Anthropic-Basecamp/memory/reference_windows_mcp_stdio_gotchas.md)):
- Node.js installed at `C:\Program Files\nodejs\`
- Chrome installed (Playwright MCP defaults to Chrome at the standard Google install path)
- Playwright MCP server configured in `.claude.json` with the working pattern from the gotchas memory

Steps:
1. Start backend: `cd server && uv run python main.py` (listens on `:8001`).
2. Start frontend: `cd client && npm run dev` (listens on `:3000`). PowerShell session must have `C:\Program Files\nodejs` on PATH — prepend it explicitly if `npm` is not found.
3. In Claude Code, drive Playwright MCP:
   - `browser_navigate` → `http://localhost:3000`
   - `browser_snapshot` → grab nav element refs
   - `browser_take_screenshot` per route
   - `browser_console_messages` with `level: error` to confirm no new errors per route
4. The 7 main nav routes to walk: `/`, `/inventory`, `/orders`, `/restocking`, `/spending`, `/demand`, `/reports`.

Screenshots from the 2026-05-20 run live in `.playwright-mcp/` next to the MCP process (`dashboard-overview.png`, `nav-inventory.png`, `nav-orders.png`, `nav-restocking.png`, `nav-finance.png`, `nav-demand.png`, `nav-reports.png`).
