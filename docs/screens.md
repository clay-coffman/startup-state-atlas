# Screens — Startup State Atlas

The bridge between the wireframes (`design/startup-state-atlas-wireframes/`)
and the agent briefs. Every URL the app ships with one row.

If you're building a UI, find your URL here, follow the wireframe
pointer, and check the role gate. Wireframe variants we'd ship in
Phase 5 (complement variants) are noted but not first-pass scope.

The content shown on these screens comes from the GOED-provided
datasets in `docs/source_data/` (loaded by Agent 1). When you mock
state during development, use those columns and values — not invented
ones.

## How to use this doc

- **Wireframe column** — relative path under
  `design/startup-state-atlas-wireframes/project/` plus the variant
  letter (per the v2 chosen direction). For pages designed in
  `Auth.html`, the column points at the relevant tab anchor
  (e.g. `Auth.html#signup`).
- **Role** — what auth gate the route enforces. `anon` = no session
  required; `signed-in` = any role; `founder|owner|investor` = the
  matching `user.role`; `owner-of-co` = session user matches
  `companies.claimed_by_user_id`; `admin` = role
  `goeo_admin | superadmin`; `superadmin` = role `superadmin`;
  `machine` = `X-Atlas-Admin-Token` header.
- **Agent** — the agent brief that owns the file at
  `agent-tasks/agent-<N>-<slice>.md`.

## Public screens (anonymous)

| URL | Agent | Wireframe variant | Role | Notes |
|-----|-------|------------------|------|-------|
| `/` | 7 | `wireframes/v2/hero.js` (B + G hybrid) | anon | Hero, persona tiles, activity ticker stub, secondary CTAs |
| `/founder` | 3 | `wireframes/v2/intake.js` (Variant D) | anon | Two-pane form + live passport JSON, 6 quick-test persona buttons |
| `/plan/:id` | 3 | `wireframes/v2/plan.js` (Variant A + memo complement) | anon | Now / Next / Ignore columns, share URL, "draft email" actions |
| `/map` | 4 | `wireframes/v2/map.js` (Variant B + cluster gazetteer toggle) | anon | Map-first MapLibre, floating filter chips, Investor Brief panel |
| `/startups/:slug` | 4 | `wireframes/v2/profile.js` (A classic + B agent reveal) | anon | Profile page with `.md` / `.json` / `.html` toggle |
| `/startups/:slug.md` | 4 | — | anon | Markdown agent card. Same data, different content-type. |
| `/startups/:slug.json` | 4 | — | anon | JSON agent card. Same data, different content-type. |
| `/agents` | 6 | `wireframes/v2/agents.js` (C install hero + B tabs + A reference) | anon | API / CLI / MCP / llms.txt docs page |
| `/llms.txt` | 6 | — | anon | Static under `public/`. Site description + agent rules. |
| `/AGENTS.md` | 6 | — | anon | End-user agent rules (distinct from this repo's `AGENTS.md` for coding agents) |
| `/api/v1/openapi.json` | 6 | — | anon | OpenAPI 3.1 spec |

## Auth screens

| URL | Agent | Wireframe variant | Role | Notes |
|-----|-------|------------------|------|-------|
| `/sign-in` | 5 | `project/Auth.html#login` (2.1) | anon | Email + password. Magic-link / Google buttons render as Phase-5 stubs. |
| `/sign-up` | 5 | `project/Auth.html#signup` (1.1) | anon | Step 1: role select (founder · owner · investor). Default role: `founder`. |
| `/sign-up/account` | 5 | `project/Auth.html#signup` (1.2) | anon | Step 2: name + email + password (12+ chars, strength meter). |
| `/sign-up/verify` | 5 | `project/Auth.html#signup` (1.3) | anon-with-token | Step 3: 6-digit OTP (10-min expiry, 30-sec resend). Better Auth `emailOTP` plugin. |
| `/forgot-password` | 5 | `project/Auth.html#login` (2.2) | anon | Generic confirmation — does not leak whether email is on file. |
| `/reset-password` | 5 | `project/Auth.html#login` (2.3) | anon-with-token | OTP-driven; same plugin as signup verify. |
| `/login/sent` | 5 | `project/Auth.html#login` (2.3) | anon | Shared "code/link sent" confirmation. Phase 4: reset only. Phase 5: magic-link too. |
| `/api/auth/[...all]` | 5 | — | varies | Better Auth catch-all handler (incl. OTP endpoints) |

## Onboarding screens (signed-in, role-specific)

| URL | Agent | Wireframe variant | Role | Notes |
|-----|-------|------------------|------|-------|
| `/onboarding/founder` | 5 (redirect) + 3 (page) | `project/Auth.html#onboard` (Founder) | founder | 302 to `/founder?onboard=1`; the existing intake (Variant D) renders a stepper when the param is set. Submit → `/onboarding/done`. |
| `/onboarding/owner` | 5 | `project/Auth.html#onboard` (Owner) | owner | Search-and-claim shortcut. Pick a company → redirect to `/companies/[slug]/claim`. Builds on existing claim flow. |
| `/onboarding/investor` | 5 | `project/Auth.html#onboard` (Investor) | investor | Preferences form; persists to `investor_profiles` via `POST /api/v1/investor-profiles`. Submit → `/onboarding/done`. |
| `/onboarding/done` | 5 | `project/Auth.html#onboard` (final) | signed-in | Shared template; copy + CTA swap by `session.user.role` (founder → `/plan/<id>`, owner → claim, investor → `/map`). |

## Account settings (signed-in)

| URL | Agent | Wireframe variant | Role | Notes |
|-----|-------|------------------|------|-------|
| `/settings` | 5 | `project/Auth.html#settings` | signed-in | Single-page sectioned: Profile, Security, role-specific (Founder Passport view OR Investor preferences OR Claimed companies), Notifications (stub), Agent tokens (stub), Danger zone. Includes "Switch role" link. |

## Owner screens (signed-in)

| URL | Agent | Wireframe variant | Role | Notes |
|-----|-------|------------------|------|-------|
| `/companies/:slug/claim` | 5 | `wireframes/v2/claim.js` (Acts 1 + 2) | signed-in | Upload verification doc to R2; submission row inserted |
| `/companies/:slug/edit` | 5 | `wireframes/v2/claim.js` (Act 3) | owner-of-co OR admin | Profile editor with field whitelist for owners |
| `/me/submissions` | 5 | — | signed-in | Owner's submission queue (pending / approved / rejected) |

## GOEO admin screens

| URL | Agent | Wireframe variant | Role | Notes |
|-----|-------|------------------|------|-------|
| `/admin` | 5 | `project/Auth.html#admin` (dashboard) | admin | Stats row (5 cards) + claim-queue summary + recent agent edits feed + coverage-gaps strip |
| `/admin/submissions` | 5 | — | admin | Ownership-submission queue |
| `/admin/submissions/:id` | 5 | `project/Auth.html#admin` (claim review · manual) | admin | Two modes: auto-approve (domain-match clean) and manual (no match — claimant note + LinkedIn + GOEO contact). 60-second signed R2 URL for the doc. |
| `/admin/resources` | 5 | `project/Auth.html#admin` (resources) | admin | Resource CRUD with status chips (live / stale / link-broken / draft) |
| `/admin/resources/:id` | 5 | — | admin | Resource edit |
| `/admin/companies` | 5 | `project/Auth.html#admin` (companies) | admin | Company CRUD without whitelist; status chips (claimed / pending / unclaimed / flagged / duplicate) |
| `/admin/companies/:slug` | 5 | — | admin | Direct company edit |
| `/admin/map` | 5 | — | admin | Map curation — fix coordinates, hide stale entries |
| `/admin/users` | 5 | `project/Auth.html#admin` (users) | admin (read) / superadmin (role flip) | Role-filter chips: `all · founder · owner · investor · admin`. Superadmin flips between `owner` and `goeo_admin`. |
| `/admin/admins` | 5 | (implied by `Auth.html#admin` "+ Invite admin") | superadmin | List current admins + invite form. Writes `admin_invites` row, sends one-time link. |

## API routes

OpenAPI is the source of truth at `app/api/v1/openapi.yaml` (Agent 6).
This list is for orientation — when in doubt, check the spec.

| Method + path | Agent | Auth | Notes |
|---------------|-------|------|-------|
| POST `/api/v1/founder-passports` | 2 | anon | Anonymous intake — no session required |
| GET `/api/v1/founder-passports/:id/plan` | 2 | anon | Cached plan for shareable URL |
| POST `/api/v1/resources/recommend` | 2 | anon | Returns scored recommendations |
| GET `/api/v1/resources` | 2 / 5 | anon | Agent 2 ships GET; Agent 5 may add admin-side filters |
| POST `/api/v1/resources` | 5 | admin | Resource create |
| PATCH `/api/v1/resources/:id` | 5 | admin | Resource update |
| DELETE `/api/v1/resources/:id` | 5 | admin | Resource delete |
| GET `/api/v1/companies` | 4 | anon | List + filters (sector, stage, county, hiring, etc.) |
| POST `/api/v1/companies` | 5 | admin | Direct create |
| GET `/api/v1/companies/:slug` | 4 | anon | Single company |
| PATCH `/api/v1/companies/:slug` | 5 | owner-of-co OR admin OR machine | Three auth paths — see Agent 5 brief |
| DELETE `/api/v1/companies/:slug` | 5 | admin | Soft-delete or hide |
| POST `/api/v1/ownership-submissions` | 5 | signed-in | Multipart upload to R2 |
| GET `/api/v1/ownership-submissions` | 5 | signed-in | Owner's own submissions |
| GET `/api/v1/ownership-submissions/:id` | 5 | owner OR admin | Owner sees own; admin sees any |
| PATCH `/api/v1/ownership-submissions/:id` | 5 | admin | Approve / reject |
| GET `/api/v1/admin/users` | 5 | admin | Users list (filterable by role) |
| PATCH `/api/v1/admin/users/:id` | 5 | superadmin | Role flip (owner ⇄ goeo_admin) |
| POST `/api/v1/admin/invites` | 5 | superadmin | Issue admin-invite token (writes `admin_invites`, emails link) |
| GET `/api/v1/admin/invites` | 5 | superadmin | List active + consumed invites |
| GET `/api/v1/admin/invites/:token` | 5 | signed-in | Consume token; flips role to `goeo_admin` once |
| POST `/api/v1/investor-profiles` | 5 | investor (or signed-in) | Upsert investor preferences keyed by `user_id` |
| GET `/api/v1/investor-profiles` | 5 | signed-in | Caller's own investor profile (or 404) |
| GET `/api/v1/search` | 6 | anon | Generic search across resources + companies |
| GET `/api/v1/openapi.json` | 6 | anon | Spec |

Read endpoints are unauthenticated. Writes branch inside the route
handler — see the Agent 5 brief for the dual-auth precedence on
`PATCH /api/v1/companies/:slug`.

## Phase 6 / post-MVP — investor public surface + intros

Not part of the 24-hour hackathon ship. Documented here so the URL
matrix stays the source of truth when Agent 8 picks up the work.
See `agent-tasks/agent-8-investor.md` for the brief and
`requirements.md` § Investor public surface for the feature spec.
Phase 4 already ships the investor sign-up, role, onboarding
preferences, and `/settings` Investor section.

| URL | Agent | Wireframe variant | Role | Notes |
|-----|-------|------------------|------|-------|
| `/investors` | 8 | — (mirror `/startups` directory pattern) | anon | Public directory; only `verified` profiles appear |
| `/investors/:slug` | 8 | — (mirror `/startups/:slug` profile) | anon | Public investor profile; **no email exposed** |
| `/investors/:slug.md` | 8 | — | anon | Markdown agent card |
| `/investors/:slug.json` | 8 | — | anon | JSON agent card |
| `/investors/:slug/edit` | 8 | — (mirror company edit) | owner-of-investor OR admin | Public-fields editor; linked from `/settings` Investor section |
| `/me/saved` | 8 | — | signed-in (`investor`) | Investor's saved-companies watchlist |
| `/me/intros` | 8 | — | signed-in | Own intros — sent + received (after admin accepts) |
| `/admin/intros` | 8 | — | admin | Intro-request queue, pending first |
| `/admin/intros/:id` | 8 | — | admin | Review one intro: accept / decline / mark introduced + notes |

New API routes added in Phase 6:

| Method + path | Agent | Auth | Notes |
|---------------|-------|------|-------|
| GET `/api/v1/investors` | 8 | anon | Verified investor list |
| GET `/api/v1/investor-profiles/:slug` | 8 | anon | Public investor profile by slug |
| PATCH `/api/v1/investor-profiles/:slug` | 8 | owner OR admin OR machine | Three auth paths, like `companies/:slug` |
| POST `/api/v1/saved-companies` | 8 | signed-in | Save a company |
| DELETE `/api/v1/saved-companies?company_id=…` | 8 | signed-in | Unsave |
| GET `/api/v1/saved-companies` | 8 | signed-in | Own watchlist |
| POST `/api/v1/intro-requests` | 8 | signed-in | Create intro request |
| GET `/api/v1/intro-requests` | 8 | signed-in | Own intros (sent + received) |
| GET `/api/v1/intro-requests/:id` | 8 | own OR admin | Single intro |
| PATCH `/api/v1/intro-requests/:id` | 8 | admin | Accept / decline / mark introduced |

Role gates: `/investors/:slug/edit` is gated on
`investor_profiles.user_id === session.user.id` OR admin role.
`/me/saved` and `/me/intros` are gated on any signed-in session
(content scopes itself to the session user). `/admin/intros`
reuses Agent 5's `goeo_admin | superadmin` middleware.

## Wireframe variant disposition

We're not picking variants now. For each screen with multiple v2
variants, the **chosen** direction is the first implementation; the
**complement** variant is a Phase 5 polish target only if there's
budget.

| Screen | Phase 3/4 ships | Phase 5 polish (if time) |
|--------|----------------|--------------------------|
| Hero (`/`) | B + G hybrid | — |
| Intake (`/founder`) | D (two-pane + live passport) | — |
| Plan (`/plan/:id`) | A (Now / Next / Ignore) | D printable memo toggle |
| Map (`/map`) | B (map-first floating filters) | Cluster gazetteer toggle |
| Profile (`/startups/:slug`) | A classic + B agent reveal toggle | — |
| Claim (`/companies/:slug/claim` + `/edit`) | A magic + B editor | C "Update via Claude/ChatGPT" handoff |
| Agents (`/agents`) | C install hero + B tabs + A reference | — |

## Mobile compliance

Every screen above must work at **375 / 768 / 1280px**. The wireframes
are desktop-only; mobile layouts are designer's-discretion within
the rules from `AGENTS.md` § Coding Style:

- No horizontal scroll at 375px.
- Tap targets ≥ 44×44 px.
- Tables → cards on mobile.
- Map drawer → bottom sheet (not side panel).
- Two-pane forms → stacked single-column with the live preview
  collapsed under the form (not beside it).

Verify with `mcp__playwright__browser_resize` (or agent-browser
device toolbar) before marking your brief DONE.

## Things specced but with no wireframe

Worth flagging — these need invented UI without a designer reference:

- OTP-error states ("expired", "wrong code", "rate-limited") on
  `/sign-up/verify` — wireframe shows the success path only.
- `/me/submissions` (owner's submission status list).
- The admin shell layout (dark sidebar) is in `Auth.html#admin` but
  individual admin pages beyond dashboard / users / resources /
  companies / claim-review need invented detail (e.g. `/admin/map`,
  `/admin/admins` form, the audit-log drill-down).
- Loading / empty / error / 404 states across the app.
- "Update via Claude/ChatGPT" claim-flow third act.
- Ownership document file-upload UI (drag-and-drop, progress, error).
- Settings sub-modals (change password, sign out other sessions,
  delete-account confirm) — `Auth.html#settings` shows the section
  layout but not the destructive-confirm dialogs.
- Phase-5 stub appearance (greyed-out "coming soon" buttons) on
  `/sign-in`, `/sign-up`, and the agent-tokens / 2FA / connected
  accounts blocks in `/settings`.

Default to "shadcn primitive + paper/ink theme tokens, mobile-first,
keep it boring" for these.
