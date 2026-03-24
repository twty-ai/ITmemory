# Transway TMS - Project Memory

## Deployment
- **Live URL:** https://tms.transwaytransport.ca/
- **Hosting:** Vercel (auto-deploys on merge to `master`)
- **Repo:** https://github.com/gagansaggus-lgtm/transway-tms
- No `.env` in local dev — all DB/Supabase errors in dev are expected

## Key Email
- Primary dispatch inbox: dispatch@transwaytransport.ca
- Connected Microsoft account sends as dispatch@ (not the logged-in user's email)
- Graph API `/me` endpoint used to fetch actual sender email for replies/compose

## Tech Stack
- Next.js, Prisma 7, Supabase auth, Microsoft Graph API for email
- Sidebar uses lucide-react icons, shadcn/ui components
- TipTap rich text editor for email compose/reply
- `gh` CLI is NOT installed — use browser or git commands for GitHub operations

## Communications Hub (completed March 2026)
- See [communications.md](communications.md) for detailed architecture
- 6-phase build: bugs → core email → inbox UX → tickets → leads → performance
- Key patterns: sandboxed iframe for HTML emails, HTML tag regex for hasHtml detection
- Reply All filters out connected inbox emails via `connectedEmails` prop
- bodyPreview must strip HTML tags before storing: `.replace(/<[^>]*>/g, "")`
- Prisma enum values are singular: `CUSTOMER`, `LEAD`, `BROKER` (not plural)

## Orders Page — Critical Architecture
- **Sorting**: Orders sort by `autoNumber desc` (highest order number first). Uses `autoNumber` integer field for proper numeric sorting.
- **Page 1 = Highest order number**: Frontend starts at page 1, descending order
- **computeOrderDates auto-saves Start/End Date**: On order create and update, `computeOrderDates` runs to populate `startDate` (earliest pickup) and `endDate` (latest delivery)
- **Order number auto-increment**: Uses `findMany` + regex `/^\d+$/` to skip corrupted Zoho entries (e.g. "OR-46452,189.43"). `$queryRaw` does NOT work with `@prisma/adapter-pg`
- **autoNumber field**: Integer field on Order model used for sorting. Extracted from orderNumber by stripping "OR-" prefix. NULL for corrupted Zoho entries.
- **Report Config sorting**: Supports server-side sorting via `sortField` and `sortDir` query params. Report config saved to localStorage per user.

## Team Dispatch Board
- Route: `/dispatch/board/teams` → component `src/components/dispatch/team-dispatch-board.tsx`
- API: `GET /api/dispatch/board?team=HIGHWAY_US` returns orders, trucks, drivers filtered by dispatchTeam
- 5 teams: HIGHWAY_US, CANADIAN_TIRE, CANADIAN, PENSKE, CITY (enum `DispatchTeam`)
- Sub-tabs: Active Board, All Orders, All Stops, All Trips
- **SHUTTLES CANADIAN TIRE** → classified as CITY team (not CANADIAN_TIRE), all LOCAL orders
- **dispatchTeam auto-assignment**: Zoho sync and order creation auto-assign dispatchTeam based on customer.primaryTeam and order direction

## Zoho Sync
- **Manual sync**: POST `/api/zoho/sync` — triggered by "Sync Zoho" button on dashboard
- **Cron sync**: GET `/api/cron/zoho-sync` — daily automatic, incremental by Order_Id_an
- **Order sync resolves customers**: Looks up by Zoho ID → company name → auto-creates if not found
- **dispatchTeam auto-assigned on sync**: Based on customer primaryTeam and direction rules
- **Error tracking**: Sync logs store `errorDetails` array (up to 10 entries) in metadata
- **Trip sync**: NOT implemented — trips only come from initial Zoho import, no incremental sync

## Common Gotchas
- Worktrees can't checkout `master` — merge from main repo at `C:\Users\Dell PC\Desktop\transway-tms`
- After Prisma schema changes, run `npx prisma generate` before build
- `autoPort: true` in `.claude/launch.json` avoids port conflicts between worktrees
- **NEVER use `git checkout --theirs` for schema.prisma merge conflicts** — it drops fields/models silently
- **`$queryRaw` breaks with `@prisma/adapter-pg`** — always use Prisma client methods
- **Zoho order numbers are corrupted** — entries like "OR-46452,189.43" exist. Always validate with `/^\d+$/` regex
- **All orders have `zohoRecordId`** — TMS-created orders also sync TO Zoho
- **Prisma Decimal fields** — Must serialize with `.toNumber()` replacer before `NextResponse.json()`
- **Vercel build cache** — Use explicit `select` instead of `include: true` on relations to avoid stale Prisma client issues

## Database
- **No `.env` locally** — must prefix commands with `DIRECT_URL=...` for db push
- Connection string: `postgresql://postgres.dojszddhllidswhswnuw:s3XL6Vy66s3vFpjp@aws-1-ca-central-1.pooler.supabase.com:5432/postgres`
- Always run `prisma db push` after adding new columns/enums to schema
- Prisma 7 with `@prisma/adapter-pg` — no `datasources` in PrismaClient constructor

## Dispatcher-Grade Redesign (March 2026)
- **Phase 1: Dense Theme** — 3 density modes (Dense/Standard/Comfortable)
  - CSS variables in `globals.css` under `[data-density="dense"]` selectors
  - `useDensity` hook at `src/hooks/use-density.ts`, persists to `tms:theme-density`
  - `DensityToggle` dropdown at `src/components/layout/density-toggle.tsx`
  - Inline script in `src/app/layout.tsx` applies density BEFORE React hydrates (no flash)
  - Dense: 30px rows, 8px padding, 11-12px fonts
- **Phase 2: Saved Views** — DB-backed saved views per table
  - `SavedView` Prisma model: name, tableKey, userId, isShared, isDefault, config (JSON)
  - `SavedViewsDropdown` at `src/components/tables/saved-views-dropdown.tsx`
  - `SavedViewsWrapper` at `src/components/tables/saved-views-wrapper.tsx` (SSR-safe lazy loader)
  - Integrated on: Team Dispatch All Orders (per-team keys), Trips page
  - Orders page blocked by SSR TDZ issue — needs hook tree refactor
  - `useAuth` removed from SavedViewsDropdown, uses localStorage Supabase session instead
- **Phase 3: Enterprise Table Features**
  - Column pinning: `ColumnPinningState` in DataTable, sticky positioning
  - `getRowClassName` prop: conditional row highlighting per data
  - Team Dispatch row colors: HOLD=red, overdue=red, today=amber, tomorrow=yellow
  - Column visibility persists to localStorage: `table-cols-{tableKey}`
- **Phase 4: Resizable Side Panel**
  - `panel-shell.tsx` — drag-to-resize left edge, min 400px, max 85vw
  - Maximize/Restore toggle button
  - Width persists to `tms:panel-width` in localStorage
- **Phase 5: Dashboard Command Center**
  - Quick Actions bar: New Order, Team Dispatch, Trip Planner, Communications
  - Urgency Alerts: unassigned orders (amber), on hold (red), available drivers/trucks (green/blue)
  - All alerts clickable → navigate to relevant page

## HIGHWAY_US Team — Business Rules (Critical)
- **Active Board**: Uses Zoho-style business rules, NOT `dispatchTeam` column
  - direction ≠ LOCAL (OR direction IS NULL)
  - customer NOT contains PENSKE, CANADIAN TIRE, RYDER
  - Status active + date window (5 days past to 30 days ahead)
- **All Orders**: Same business rules, no date window, default Active status filter
- **Report Config defaults**: Shows 4 filters for visibility, but NOT sent to API (backend handles filtering). Only user-modified filters (isDirty) are sent.
- **Double-filter bug**: If Report Config defaults are sent to API alongside backend rules, `NOT` clauses conflict and reduce results. Fixed by checking `isDirty`.

## Zoho Direction Mapping
- `IN_OUT_fx` (formula field, always updated) preferred over `Out_In_dd` (manual dropdown, often empty)
- mapDirection handles: "IN"→IN_BOUND, "OUT"→OUT_BOUND, "In Bound"→IN_BOUND, "Local"→LOCAL, "USA"→USA
- Case-insensitive normalization via `.trim().toUpperCase()`
- 85 orders had NULL direction from missing Zoho field — backfilled, Highway filter includes NULL

## Sidebar Customization
- Cross-group page moves: `pageMoves` in `use-sidebar-preferences.ts` (restored after PR#50 removed it)
- `batchMovePageToGroup()`: atomic update of pageMoves + source/target pageOrder
- Drop on group header: accepts page drops for cross-group moves (not just module drops)
- Double-fire guard: `dropProcessedRef` prevents duplicate drop handler execution
- Favorites: only pinned items shown (not auto-favorites), `excluded` list prevents re-appearance after unpin

## Communications Hub — Per-Inbox Tabs
- 5 tabs: All, Dispatch (blue), Cantire (green), Penske (orange), Lactalis (purple)
- Per-tab colors with activeBg + border matching inbox color
- `inboxEmail` field on EmailThread for filtering
- Compose auto-selects From address based on active tab

## Report Config Filters
- `buildPrismaFilters()` in dispatch board API converts filter JSON to Prisma where clauses
- Supports: equals, notEquals, contains, notContains, startsWith, in, notIn, isNull, isNotNull
- Handles camelCase operators (from FilterCondition type) and snake_case
- Date filters: before/after/between with Date object conversion
- Nested fields: customer.companyName, trip.driver, trip.truck
- AND/OR group logic support

## Trip Activities & Mileage
- See [trip_activities_mileage.md](trip_activities_mileage.md) for details
- 3 mileage sources: Google (planned), PC Miler (billing), Samsara (actual GPS)
- OmniTrack for ELD/Logbook — separate system, no integration
- Auto-ETA cascade from Start Date-Time only
