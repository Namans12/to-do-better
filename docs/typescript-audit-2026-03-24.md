# TypeScript Codebase Audit (2026-03-24)

Scope: `app/src` and `frontend/src` TypeScript code.

## Methodology

- Installed project dependencies in `app/` and `frontend/`.
- Ran strict unused-symbol audit in backend with:
  - `cd app && npx tsc --noEmit --noUnusedLocals --noUnusedParameters`
- Ran frontend build/type-check with:
  - `cd frontend && npm run build`
- Compared mounted backend route prefixes in `app/src/app.ts` against client calls in `frontend/src/api/client.ts`.
- Performed static symbol usage scan for exported declarations with one-reference-only candidates.

## 1) Unused imports, variables, and dead code

### Confirmed by TypeScript diagnostics

1. `app/src/index.ts`
   - `server` is assigned but never used (`const server = serve(...)`).
   - This is harmless runtime-wise but dead assignment for static analysis.

2. `app/src/routes/batch.ts`
   - `inArray` is imported from `drizzle-orm` but never used.

3. `app/src/routes/connections.ts`
   - `mergedKind` is computed but never used in merge logic.

4. `app/src/tests/helpers.ts`
   - `Database` import from `better-sqlite3` is unused.

### Likely unused exports (repository-wide symbol scan)

1. `frontend/src/api/client.ts`
   - `batchApi` appears exported but not referenced from other files.

2. `frontend/src/sync/auth.ts`
   - `isSupabaseVerifyLink` appears exported but not referenced.

3. `frontend/src/sync/auth.ts`
   - `sendMagicLink` appears exported but not referenced.

> Note: these export findings are static-text usage based; they should be validated before deletion if dynamic imports are used elsewhere.

## 2) API endpoint consistency between routes and controllers

### Result: route surface is consistent for REST mode

All client REST endpoints in `frontend/src/api/client.ts` map to mounted route groups in `app/src/app.ts`:

- `/groups*` ↔ `/api/groups`
- `/groups/:groupId/todos*` ↔ `/api/groups/:groupId/todos`
- `/todos*` and `/todos/batch/*` ↔ `/api/todos` and `/api/todos/batch`
- `/trash*` ↔ `/api/trash`
- `/connections*` ↔ `/api/connections`
- `/search*` ↔ `/api/search`
- `/activity*` ↔ `/api/activity`
- `/backups*` ↔ `/api/backups`
- `/templates*` ↔ `/api/templates`
- `/sync/export|import` ↔ `/api/sync`

No path-prefix mismatches were found in this audit.

## 3) Exported types/functions utilization

- Backend route factory exports in `app/src/routes/*.ts` are actively consumed by `app/src/app.ts`.
- Potentially unused exports called out above:
  - `batchApi`
  - `isSupabaseVerifyLink`
  - `sendMagicLink`

Recommendation: either wire these exports into active code paths or remove them to reduce surface area.

## 4) Comment-described or partially-implemented non-functional logic

### Confirmed feature gating / partial functionality

1. `frontend/src/api/client.ts`
   - `ensureRestOnlyFeatureAvailable(...)` blocks features while Supabase sync mode is enabled.
   - Affected API groups include Backups, Templates, and manual Sync package export/import.
   - These flows are intentionally non-functional in Supabase mode.

2. Connection merge/cut constraints
   - Both backend and sync repository explicitly reject certain graph operations:
     - Merging existing branch trees is not supported.
     - Cutting branch trees is not supported.
   - This indicates deliberately incomplete functionality for advanced branch graph operations.

### Potential incomplete implementation indicator

- `mergedKind` in `app/src/routes/connections.ts` is assigned but unused, suggesting unfinished or removed branch of merge-kind handling.

## Recommended remediation order

1. Remove obvious unused symbols (`inArray`, `Database`, `mergedKind`, or use `server` intentionally).
2. Confirm and prune unused exports (`batchApi`, `isSupabaseVerifyLink`, `sendMagicLink`) if not part of planned public API.
3. If Supabase mode should reach parity with REST mode, implement Backups/Templates/manual package flows in sync repository; otherwise document the mode limitation in user-facing settings/help text.
4. If branch-merge/cut limitations are intentional long-term, surface clearer UI affordances to avoid attempted unsupported actions.
