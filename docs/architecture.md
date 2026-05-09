# Architecture — Startup State Atlas

This is the load-bearing reference for every coding agent. If a brief
disagrees with this file, this file wins.

## Stack

| Layer            | Choice                                                                                   |
| ---------------- | ---------------------------------------------------------------------------------------- |
| Framework        | **Next.js 15** App Router via **`@opennextjs/cloudflare`**                               |
| Runtime          | **Cloudflare Workers** (with Static Assets for the Next.js build)                        |
| DB               | **Cloudflare D1** (SQLite at edge) + **Drizzle ORM**                                     |
| Object store     | **Cloudflare R2** — `atlas-ownership-docs` (verification docs, required) + optional `atlas-photos` (Agent 4) |
| Auth             | **Better Auth** (email + password, self-hosted in D1 via Drizzle adapter; Web Crypto password hashing; CLI migrations) |
| Email            | **Resend** (via the `send-email` skill) — used for Better Auth verification + password-reset mail |
| LLM              | **Anthropic Claude `claude-opus-4-7`** (use prompt caching where possible)               |
| Enrichment       | **Parallel.ai** — founder-website context extraction during intake (`PARALLEL_API_KEY`). Optional path; the form remains submittable without it. |
| Map              | **MapLibre GL** (open tiles via OpenStreetMap or CARTO basemap; no API token)            |
| Errors / logs    | **Cloudflare Workers Observability** (built-in, free) — `wrangler tail` + Workers Logs UI |
| Provisioning     | **Stripe Projects CLI** (`stripe projects`) for SaaS credentials and account linkage     |
| Cloud ops        | **Wrangler CLI** for D1, R2, Workers, secrets                                            |
| Code host        | **GitHub** via `gh` CLI                                                                  |
| Test runners     | **Vitest** (unit), **Playwright** (E2E, scope-permitting)                                |
| UI               | **Tailwind CSS** + **shadcn/ui** primitives                                              |
| Lint / format    | ESLint (Next.js defaults) + Prettier (`~/.prettierrc.yaml`, printWidth 80)               |
| Repo agent infra | Vendored Claude/Codex skills, hooks, and GitHub workflows under `.agents/`, `.claude/`, `.github/` — see `AGENTS.md` |

## System diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                            Browser / CLI / MCP host                  │
│  ┌──────────────────┐  ┌──────────────┐  ┌──────────────────────┐   │
│  │ Founder Navigator │  │ Investor map │  │ External agent       │   │
│  │ (Next.js client)  │  │ (MapLibre)   │  │ (Claude / ChatGPT)   │   │
│  └──────────────────┘  └──────────────┘  └──────────────────────┘   │
└──────────────────────────┬──────────────────────────────────────────┘
                           │ HTTPS
┌──────────────────────────▼──────────────────────────────────────────┐
│                Cloudflare Worker (OpenNext build)                    │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │ Next.js App Router                                          │    │
│  │  ├── app/page.tsx, app/founder/, app/map/, app/startups/    │    │
│  │  ├── app/api/v1/resources/recommend/route.ts                │    │
│  │  ├── app/api/v1/companies/route.ts (and :slug, :slug.md, …) │    │
│  │  ├── app/api/v1/openapi.json/route.ts                       │    │
│  │  ├── app/AGENTS.md, app/llms.txt (or under public/)         │    │
│  │  └── public/AGENTS.md, public/llms.txt (end-user agents)    │    │
│  └─────────────────────────────────────────────────────────────┘    │
│             │                  │                  │                  │
│   ┌─────────▼────────┐  ┌──────▼──────────────┐  ┌────────▼─────────┐│
│   │ D1 binding `DB`  │  │ R2 `OWNERSHIP_DOCS` │  │ env: ANTHROPIC_…, ││
│   │ (SQLite + auth)  │  │ (verification docs) │  │ BETTER_AUTH_…,    ││
│   │                  │  │ optional photos R2  │  │ RESEND_…,         ││
│   │                  │  │                     │  │ ATLAS_ADMIN_…,    ││
│   │                  │  │                     │  │ PARALLEL_API_KEY  ││
│   └─────────┬────────┘  └─────────────────────┘  └────────┬─────────┘│
└─────────────┼──────────────────────────────────────┼─────────────────┘
              │                                       │
        ┌─────▼──────┐                       ┌────────▼─────────┐
        │ D1 DB:     │                       │ Anthropic API    │
        │ resources, │                       │ (claude-opus-4-7)│
        │ companies, │                       │ Parallel.ai      │
        │ passports  │                       │ (founder website │
        └────────────┘                       │  enrichment)     │
                                             └──────────────────┘

Side surfaces (same package, different bin):
- cli/index.ts → `startup-state` bin (talks to the Worker via HTTPS)
- mcp/server.ts → `startup-state-mcp` bin (stdio MCP for Claude Desktop)
```

## Repo layout (target — Agent 0 creates the app skeleton)

```
startup-state-atlas/
├── app/                          # Next.js App Router
│   ├── layout.tsx
│   ├── page.tsx
│   ├── founder/                  # Founder Navigator UI (Agent 3)
│   │   ├── page.tsx              # intake form
│   │   └── results/[id]/page.tsx # results page
│   ├── map/page.tsx              # ecosystem map (Agent 4)
│   ├── startups/[slug]/          # company profile pages (Agent 4)
│   │   ├── page.tsx
│   │   ├── route.md/route.ts     # markdown profile endpoint
│   │   └── route.json/route.ts   # JSON profile endpoint
│   ├── sign-in/page.tsx          # Better Auth sign-in (Agent 5)
│   ├── sign-up/page.tsx          # role select — step 1 of 3 (Agent 5)
│   ├── sign-up/account/page.tsx  # email + password — step 2 (Agent 5)
│   ├── sign-up/verify/page.tsx   # 6-digit OTP — step 3 (Agent 5)
│   ├── login/sent/page.tsx       # "code/link sent" confirmation (Agent 5)
│   ├── forgot-password/page.tsx  # password reset request (Agent 5)
│   ├── reset-password/page.tsx   # password reset (Agent 5)
│   ├── onboarding/founder/page.tsx     # redirects /founder?onboard=1 (Agent 5)
│   ├── onboarding/owner/page.tsx       # search-and-claim shortcut (Agent 5)
│   ├── onboarding/investor/page.tsx    # investor preferences form (Agent 5)
│   ├── onboarding/done/page.tsx        # role-aware "you're ready" (Agent 5)
│   ├── settings/page.tsx               # single-page sectioned settings (Agent 5)
│   ├── companies/[slug]/claim/page.tsx  # ownership submission (Agent 5)
│   ├── companies/[slug]/edit/page.tsx   # owner edits claimed co (Agent 5)
│   ├── me/submissions/page.tsx   # owner's submission status (Agent 5)
│   ├── admin/                    # GOEO admin UI (Agent 5)
│   │   ├── layout.tsx            # role gate + dark sidebar shell
│   │   ├── page.tsx              # dashboard: stats + claim queue + edit feed
│   │   ├── submissions/page.tsx  # ownership-submission queue
│   │   ├── submissions/[id]/page.tsx  # review one submission (auto + manual)
│   │   ├── users/page.tsx        # users list, role filters; superadmin role flip
│   │   ├── admins/page.tsx       # superadmin: list + invite admins
│   │   ├── resources/page.tsx    # resource list + create
│   │   ├── resources/[id]/page.tsx
│   │   ├── companies/page.tsx    # company list + create
│   │   ├── companies/[slug]/page.tsx  # direct edit (no claim)
│   │   └── map/page.tsx          # map curation
│   ├── agents/page.tsx           # /agents docs page (Agent 6)
│   ├── api/auth/[...all]/route.ts         # Better Auth handler (Agent 5)
│   └── api/v1/
│       ├── resources/route.ts             # GET (Agent 2), POST (Agent 5)
│       ├── resources/[id]/route.ts        # GET, PATCH, DELETE (Agent 5)
│       ├── resources/recommend/route.ts   # POST (Agent 2)
│       ├── companies/route.ts             # GET list, POST create (Agent 5)
│       ├── companies/[slug]/route.ts      # GET, PATCH, DELETE (Agent 5)
│       ├── ownership-submissions/route.ts # POST + GET own (Agent 5)
│       ├── ownership-submissions/[id]/route.ts # GET + PATCH approve/reject (Agent 5)
│       ├── investor-profiles/route.ts       # POST + GET own (Agent 5)
│       ├── admin/invites/route.ts           # POST invite, GET list (Agent 5)
│       ├── admin/invites/[token]/route.ts   # GET consume token (Agent 5)
│       ├── admin/users/[id]/route.ts        # PATCH role flip (Agent 5)
│       ├── founder-passports/route.ts     # POST
│       ├── founder-passports/[id]/plan/route.ts # GET
│       ├── search/route.ts                # GET
│       └── openapi.json/route.ts          # spec endpoint (Agent 6)
├── db/                           # Drizzle (Agent 1 lays initial)
│   ├── schema.ts                 # any agent may extend post-Agent-1
│   ├── migrations/0000_*.sql
│   ├── seed/personas.ts
│   ├── seed/resources.ts
│   ├── seed/companies.ts
│   ├── seed/investor-profiles.ts  # demo investor rows (Agent 5)
│   ├── seed/data/resources.csv  # user provides (Google Sheets)
│   └── seed/data/companies.csv  # user provides
├── auth.ts                       # Better Auth config (Agent 5)
├── middleware.ts                 # role-gated route protection (Agent 5)
├── scripts/
│   └── bootstrap-superadmin.ts   # promote a user to superadmin (Agent 5)
├── lib/
│   ├── db.ts                     # drizzle client wired to D1 binding
│   ├── auth-client.ts            # Better Auth React client (Agent 5)
│   ├── anthropic.ts              # Claude client (claude-opus-4-7)
│   ├── api-error.ts              # ApiError class — frozen error shape
│   ├── ids.ts                    # newId(prefix) helper
│   ├── cf.ts                     # getRequestContext().env accessor
│   ├── email.ts                  # Resend wrapper for Better Auth mail (Agent 5)
│   └── recommend.ts              # scoring lib (Agent 2)
├── schemas/
│   ├── founder-passport.ts       # zod schema
│   └── company-profile.ts        # zod schema
├── types/
│   └── api.ts                    # zod-derived TS types for every endpoint
├── cli/
│   └── index.ts                  # `startup-state` bin entry (Agent 6)
├── mcp/
│   └── server.ts                 # `startup-state-mcp` bin entry (Agent 6)
├── public/
│   ├── AGENTS.md                 # END-USER-FACING agent docs (Agent 6)
│   └── llms.txt                  # /llms.txt (Agent 6)
├── docs/
│   ├── hackathon-plan.md
│   ├── architecture.md           # this file
│   ├── requirements.md
│   └── agent-tasks/00-…6-…md
├── wrangler.jsonc                # CF Workers config
├── open-next.config.ts           # OpenNext for CF
├── drizzle.config.ts
├── next.config.mjs
├── tsconfig.json
├── package.json
└── .env.example
```

## Data flow — the three demo paths

### A) Founder intake → personalized plan

```
Browser /founder
  → (optional) POST /api/v1/founder-passports/enrich  (Agent 2)
      → call Parallel.ai with founder-supplied website URL
      → return partial FounderPassportInput shape (county, city,
        stage, industry, business_type, identity tags, needs)
      → front-end pre-populates the form; founder reviews + edits
  → POST /api/v1/founder-passports        (Agent 2)
      → insert into D1 founder_passports (incl. website_url,
        enriched_at, enrichment_source if the enrich path ran)
  → POST /api/v1/resources/recommend      (Agent 2)
      → SELECT scored resources from D1 with field-match scoring
      → Anthropic call (source-bound explanation, IDs only from set)
      → return { passport_id, recommendations[] }
  → GET /api/v1/founder-passports/:id/plan (cached "do this now / next / ignore")
  ← render results page (Agent 3)
```

### B) Investor / map view

```
Browser /map
  → MapLibre loads with sector-clustered company markers
  → User filters: sector=fintech, stage=seed, county=Salt Lake
  → GET /api/v1/companies?...                (Agent 2/4)
      → SELECT from D1, JOIN locations, filter
  → Click pin → drawer shows profile
  → "Investor Brief" sidebar runs Anthropic on retrieved companies
  → "View profile" → /startups/:slug
      → ALSO available at /startups/:slug.md and .json
```

### C) External agent calls the API

```
Claude/ChatGPT reads /llms.txt and /AGENTS.md
  → POST /api/v1/resources/recommend  (no auth — read endpoints are open)
  → cites resource IDs in its response to the user
  → optionally: PATCH /api/v1/companies/:slug
      (with X-Atlas-Admin-Token machine token; the route handler
       still enforces ownership rules from D1 — owner_user_id match
       or admin role on the calling user, when known)
```

## Cloudflare bindings

`wrangler.jsonc` (Agent 0 sets this up):

```jsonc
{
  "name": "startup-state-atlas",
  "main": ".open-next/worker.js",
  "compatibility_date": "2025-05-01",
  "compatibility_flags": ["nodejs_compat"],
  "assets": { "directory": ".open-next/assets", "binding": "ASSETS_BUCKET" },
  "d1_databases": [
    // database_id is only required for `--remote` and `wrangler deploy`.
    // Local dev uses .wrangler/state/v3/d1/ keyed by database_name —
    // each worktree has its own SQLite file.
    { "binding": "DB", "database_name": "startup-state-atlas-db", "database_id": "<from `wrangler d1 create` at deploy time>" }
  ],
  "r2_buckets": [
    { "binding": "OWNERSHIP_DOCS", "bucket_name": "atlas-ownership-docs" }
    // Optional photo bucket (Agent 4 only):
    // { "binding": "ASSETS", "bucket_name": "startup-state-atlas-assets" }
  ],
  "observability": { "enabled": true },
  "vars": {}      // public
  // Secrets via `wrangler secret put`:
  //   ANTHROPIC_API_KEY
  //   BETTER_AUTH_SECRET
  //   RESEND_API_KEY
  //   ATLAS_ADMIN_TOKEN  (machine-only; CLI/MCP write path)
  //   PARALLEL_API_KEY   (founder-website enrichment during intake)
}
```

In code, read bindings via the OpenNext context helper:

```ts
import { getRequestContext } from '@opennextjs/cloudflare';
const env = getRequestContext().env;
const db = drizzle(env.DB);
```

Wrap that in `lib/cf.ts` so handlers don't import OpenNext directly.

## Observability (free, no Sentry)

- Enable via `observability.enabled = true` in `wrangler.jsonc`.
- Live tail: `wrangler tail` (during dev).
- Search/retention: Cloudflare Workers dashboard → Logs.
- No third-party error tracker for the hackathon. If we want richer
  grouping later, add Sentry's free tier or self-host GlitchTip — but
  not now.

## Deploys

- **Local dev:** `npm run dev` (Next.js dev server) for fast UI
  iteration. `npm run preview` (or `npx wrangler dev`) when you need
  live D1/R2 bindings.
- **Push deploy:** `npm run deploy` → `wrangler deploy`. Agent 0 ships
  the empty scaffold to confirm wiring works on day one (catches
  binding/env surprises early).
- **GitHub Action:** optional `.github/workflows/deploy.yml` using
  `cloudflare/wrangler-action` if we want push-to-deploy. Defer until
  after Agent 0 has a green deploy locally.

## Contracts (FROZEN before agents fork)

- **Schema** lives in `db/schema.ts`. Agent 1 lays the initial
  version; later agents may extend it from their own worktrees
  against per-worktree local D1. Rebase against `main` before
  running `npm run db:generate` to avoid migration-number
  collisions (`0003_*.sql` clashes).
- **API contract** lives in `app/api/v1/openapi.yaml` (committed) and
  is also served at `/api/v1/openapi.json`. Agent 6 owns it.
- **Error shape:** `{ error: { code, message, details? } }`.
- **ID prefixes:** `fp_*`, `co_*`, `r_*`, `rec_*`, `bos_*`,
  `inv_*` (investor profiles). Phase 6 (post-MVP) adds
  `sc_*` (saved companies) and `irq_*` (intro requests) — see
  `docs/agent-tasks/agent-8-investor.md`. Use `lib/ids.ts`.
  (Better Auth's own IDs — `user`, `session`, `account`,
  `verification` — are managed by Better Auth.)
- **Casing:** snake_case API ↔ camelCase TS. Convert at the
  Drizzle/zod boundary.
- **Auth (dual model):**
  - **Web users (humans):** Better Auth (email + password,
    self-hosted in D1). `auth.ts` configures the Drizzle adapter
    and an `additionalFields.role` column on `user` with values
    `founder` / `owner` / `investor` / `goeo_admin` / `superadmin`.
    Default for self-serve sign-up is `founder`; founders, business
    owners, and investors pick their role on the first signup
    step and may switch in Settings. `goeo_admin` and `superadmin`
    are **invite-only** — no self-serve admin sign-up; admins are
    promoted via the `admin_invites` flow or the bootstrap script.
    Sessions are cookie-based and stored in D1. Next.js middleware
    enforces role checks on `/admin/*` and ownership-bound write
    routes. First `superadmin` is bootstrapped via
    `npm run bootstrap-superadmin <email>` (Agent 5 ships this
    script). Email-verification uses Better Auth's `emailOTP`
    plugin (6-digit code, 10-minute expiry, 30-second resend
    cooldown). Verification + reset mail goes through the
    `send-email` skill (Resend).
  - **Machine clients (CLI / MCP / scripts):**
    `X-Atlas-Admin-Token` header, validated against
    `env.ATLAS_ADMIN_TOKEN`. This is a service-account-style
    token, **not** a human admin login. It bypasses Better Auth
    and assumes a privileged caller; route handlers must still
    enforce business rules (e.g. only the user matched to
    `companies.claimed_by_user_id` can edit a claimed company).
  - Read endpoints stay unauthenticated.
- **R2 (ownership docs):** the `OWNERSHIP_DOCS` binding holds
  uploaded verification documents. They are never served as
  public URLs — admins fetch via short-lived signed URLs.

## Open questions to escalate

If an agent needs answers it can't infer from these docs, the user is
the source of truth. Don't guess on:

- A new database column (Agent 1 only).
- A new public API endpoint (must update OpenAPI; Agent 6 awareness).
- A new third-party SaaS dependency (run it past the user — keep the
  free/CLI-only constraint).
- A change to the error response shape, ID prefixes, casing, or auth
  model (these are architectural; the user decides).
