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

## Trip Activities & Mileage
- See [trip_activities_mileage.md](trip_activities_mileage.md) for details
- 3 mileage sources: Google (planned), PC Miler (billing), Samsara (actual GPS)
- OmniTrack for ELD/Logbook — separate system, no integration
- Auto-ETA cascade from Start Date-Time only
