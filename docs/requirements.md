# Requirements — Startup State Atlas

Distilled from `docs/hackathon-plan.md` (the full 870-line product spec
remains the source of truth for any ambiguity).

## Product summary

A polished founder/investor product on top of an agent-native data
layer. A founder describes their situation; the system returns a
precise 90-day resource plan with cited reasons, shows relevant
companies/investors/partners on a map, and exposes every resource,
company, and profile through a public REST API, CLI, MCP server, and
`/llms.txt`.

The product is really an **ecosystem graph** with three interfaces:

1. **Founder interface** — "Tell me your situation; I'll give you the
   right next moves."
2. **Investor interface** — "Show me what's being built in Utah."
3. **Agent interface** — "Let ChatGPT/Claude/Codex query and update
   the ecosystem directly."

## Personas (six required test cases per the brief)

We're shipping a real product. The six personas are the GOED brief's
**required test cases** — they gate the live flow, not replace it.
Real founders will hit the live site with arbitrary inputs; the
intake form, recommendation engine, and explanations have to hold up
for those founders too. The prefill-from-website path (below) is one
of the affordances that makes the live flow tolerable for users who
don't fit a persona.

Persona fixtures live in `db/seed/personas.ts` (Agent 1 writes these).
For the demo and for QA regression, each persona is one-click
testable on the founder page ("Try Jordan", "Try Maria", etc.).

| Persona     | One-line description                                                   |
| ----------- | ---------------------------------------------------------------------- |
| **Jordan**  | 20, Salt Lake City, idea but no business — needs start-business help. |
| **Maria**   | Rural / women-owned, scaling small ag operation in Washington County.  |
| **Marcus**  | Veteran, mid-stage, seeking workforce + customer connections.          |
| **Priya**   | 18 months in, paying customers, raising seed — needs angels/VCs.       |
| **David**   | Late-stage growth — exporting, talent pipeline, employer programs.     |
| **Dr. Amir**| University researcher commercializing IP.                              |

## Core features (grouped by surface)

### Founder Passport (intake)

- Structured form: county/city, business stage, industry, founder
  identity tags (student, veteran, woman-owned, rural, university
  researcher, …), goal (start, fundraise, hire, export,
  commercialize, find workspace, find mentors), urgency, company
  size/revenue, what they want (capital, customers, talent,
  regulatory help, operating support).
- Persisted as a `founder_passports` row; returns `passport_id`.
- Quick-test buttons for each persona.

#### Smart prefill from business website (optional)

Founders who already run a business shouldn't have to retype what's
on their public website. The intake form has an optional URL field
at the top: paste a business website, the form pre-populates the
fields it can infer (industry, stage, business_type, county/city,
likely identity tags and needs), and the founder reviews + edits
before submitting.

- The lookup hits `POST /api/v1/founder-passports/enrich` (Agent 2),
  which calls **Parallel.ai** against the supplied URL and returns a
  partial `FounderPassportInput`. No persistence happens on the
  enrich call — persistence is still on the existing intake POST.
- `founder_passports` carries a `website_url` (the founder's input)
  plus `enriched_at` / `enrichment_source` audit fields so we know
  which rows were touched by enrichment.
- Prefilled fields are visually marked (subtle "filled from your
  site" chip per field, dismissible). The founder is always the
  decider.
- The form must remain submittable without ever calling enrich —
  manual fill is the always-works path. Persona quick-test buttons
  bypass the URL field entirely.
- LinkedIn is **not** an accepted enrichment source; see Out of
  scope below.

### Recommendation engine

- `POST /api/v1/resources/recommend` returns ranked resources with
  per-resource scores, reasons, and suggested actions.
- **Deterministic field-match scoring first** (NOT an LLM):
  ```
  resource_score =
      25 * stage_match
    + 20 * location_match
    + 20 * goal_topic_match
    + 15 * industry_match
    + 10 * community_match
    + 10 * semantic_similarity (optional / cut if time-pressed)
  ```
- LLM (Anthropic, source-bound) generates the explanation **only**;
  it must cite resource IDs and may not invent eligibility or
  recommend resources outside the retrieved set.
- Output structure:
  - **Do this now:** top 3 resources.
  - **Do this next:** next 3 resources.
  - **Ignore for now:** noisy near-misses.
  - **Why these matched:** per-resource bullets.
  - **Saved plan URL:** shareable link.

### Investor / ecosystem map

- MapLibre GL map of companies, sector-colored clusters.
- Filters: sector, stage, employee count, hiring status, county/city.
- Click pin → side drawer with company profile.
- "Investor Brief" panel summarizes the current filtered set
  (Anthropic-generated, source-bound).
- "Investor tour mode" — curated 10-company tour.
- "Founder partner mode" — companies that might be customers,
  partners, suppliers, acquirers.
- "Talent mode" — companies hiring in the founder's geo/industry.

### Company profiles ("the business is the website")

- `/startups/:slug` — public profile page (HTML).
- `/startups/:slug.md` — markdown agent card.
- `/startups/:slug.json` — JSON agent card.
- Agent Card fields: canonical description, industry, location,
  stage, employee count, hiring status, jobs, founder/team links,
  funding (if known), what they sell, who should contact them, what
  agents should know before recommending them, update timestamp,
  verification status.

### Authentication (Better Auth)

- **Library**: [Better Auth](https://better-auth.com) with the
  Drizzle adapter against D1. CLI-managed migrations
  (`npx @better-auth/cli generate` + `migrate`). No external auth
  dashboard. Sessions stored in D1.
- **Roles** on `user.role`:
  - `founder` — default for self-serve sign-up. Building a
    company; uses the Founder Navigator.
  - `owner` — already running a Utah company; uses the claim
    flow to verify ownership and edit a profile.
  - `investor` — VC, angel, family office, scout, or LP looking
    at the Utah ecosystem; uses the map and saved filters.
  - `goeo_admin` — Utah Startup State / GOEO staff. **Invite-only**
    (no self-serve admin sign-up).
  - `superadmin` — bootstrapping role; promotes/demotes admins.
    **Invite-only** via the `bootstrap-superadmin` script.
- **Sign-up flow**: 3 steps on one URL with a stepper.
  1. Pick role (founder / owner / investor).
  2. Enter name, email, password (12+ chars).
  3. Verify with a 6-digit OTP emailed to that address (10-minute
     expiry, 30-second resend cooldown). Better Auth's `emailOTP`
     plugin handles storage/expiry.
- **Sign-in flow**: email + password on `/sign-in`.
  "Magic link" and "Continue with Google" appear as Phase-5
  Coming Soon stubs in Phase 4.
- **Reset flow**: `/forgot-password` (generic confirmation —
  don't leak whether the email is on file) → 6-digit code (same
  OTP plugin) → `/reset-password`.
- **Onboarding** (post-verify, role-specific):
  - **Founder** → `/founder?onboard=1` (the existing Navigator
    intake form, framed with a stepper).
  - **Owner** → `/onboarding/owner` (search the company index,
    pick one, redirect to `/companies/[slug]/claim`).
  - **Investor** → `/onboarding/investor` (preferences form;
    persists to `investor_profiles`).
  - All three end on `/onboarding/done` — single template with
    role-aware copy + CTA.
- **Settings**: single sectioned page at `/settings`. Sections:
  Profile (display name, email, time zone), Security (password,
  sessions, 2FA stub, connected accounts stub), role-specific
  (Founder Passport summary OR Investor preferences OR Claimed
  companies), Notifications (stub), Agent tokens (stub for Phase
  5 — distinct from the machine `X-Atlas-Admin-Token`), Danger
  zone (delete account; "Switch role" link).
- **Bootstrapping**: first `superadmin` is set by a one-shot
  `npm run bootstrap-superadmin <email>` script run by an
  operator with `wrangler` D1 access. After that, `superadmin`
  invites other admins via `/admin/admins` (writes
  `admin_invites`, sends a one-time link). `superadmin`
  promotes/demotes existing users between `owner` and
  `goeo_admin` from `/admin/users`.
- **Anonymous founder passports stay anonymous** — sign-up is
  not required to use the Founder Navigator. The persona
  quick-test buttons on `/founder` work without an account.
- **Read endpoints stay open.** Writes from the web app require a
  Better Auth session; writes from the CLI/MCP layer use the
  machine-only `X-Atlas-Admin-Token` (see Agent-native layer).

### Investor profile (preferences)

Investors are first-class users alongside founders and owners. On
sign-up they pick the `investor` role and complete a preferences
form (`/onboarding/investor`) that persists to `investor_profiles`,
keyed by `user.id`:

- **Firm / affiliation** (free text, e.g. "Pelion Ventures").
- **Investor type** (single-select):
  `vc | angel | family_office | corp_dev | scout | lp`.
- **Stages of interest** (multi-select):
  `pre_seed | seed | series_a | growth`.
- **Sectors of interest** (multi-select; mirrors the company
  sector taxonomy used on the map).
- **Check size** (range: optional min + max, USD integers).
- **Geographic focus** (multi-select):
  `wasatch_front | statewide | national`.

Phase 4 ships the data and the form. Phase 5 wires the
preferences into `/map` filter chips and a weekly cluster-brief
email. Investors can still see the whole map regardless — the
preferences personalize, they don't gate.

### Investor public surface + watchlists + intros (Phase 6 / post-MVP)

**Not part of the 24-hour hackathon ship.** Phase 6 builds on top of
Phase 4's investor identity. See `docs/agent-tasks/agent-8-investor.md`.

- **Public investor directory** at `/investors`. Anonymous-readable.
  Lists profiles with `verification_status = 'verified'` only.
- **Public investor profile** at `/investors/<slug>` with `.md` and
  `.json` agent variants (mirroring the company-profile triple).
  Investor email is **never** public — intros are admin-brokered.
- **Schema extension to `investor_profiles`** (Phase 4 owns the
  table; Phase 6 adds the public-facing columns): `slug` UNIQUE,
  `display_name`, `bio`, `tagline`, `website`, `linkedin`,
  `verification_status` ('unverified' default), `verified_at`,
  `last_updated_by`. Phase 4 rows stay valid — they default to
  `unverified` and don't appear in the directory until they
  publish + admin approves.
- **Self-serve public-profile editor** at
  `/investors/<slug>/edit`. Linked from the `/settings` Investor
  section ("Manage your public profile"). Field whitelist for
  owner edits; admin can override anything.
- **Saved-companies watchlist.** Investors save companies from
  the map / `/startups` to a personal list at `/me/saved`. Optional
  one-line note. Stored in `saved_companies` (`sc_*`).
- **Intro requests, always admin-brokered.** Both directions —
  founder/owner → investor, investor → founder/owner — flow
  through GOEO. Requester writes a short message, submits, lands
  in `/admin/intros` as `pending`. Admin accepts (system emails
  both parties via Resend with contact info), declines, or marks
  `introduced`. Stored in `intro_requests` (`irq_*`).
- **No CRM, no DMs.** Once admin accepts, the email is the
  handoff. Platform-side state ends.
- **Mobile-responsive** (375 / 768 / 1280px) and a11y-compliant
  on every new surface, like the rest of the product.

### Self-service claim flow (with ownership verification)

- "Claim this company" button on profile → routes the founder to
  sign-up / sign-in if not authenticated.
- **Real verification**: signed-in owner uploads a verification
  document (Secretary-of-State filing, business license, EIN
  letter — PDF or image) to the `atlas-ownership-docs` R2 bucket.
- Submission is recorded in `business_ownership_submissions`
  (`pending`).
- GOEO admin reviews the queue at `/admin/submissions`, opens the
  doc via a signed R2 URL, approves or rejects with notes.
- On approval: `companies.claimed_by_user_id` is set,
  `verified_at` and `claimed_at` are stamped, the submission row
  flips to `approved`. The owner can now edit the company at
  `/companies/[slug]/edit`.
- AI-generated draft profile on first edit-page open; founder
  approves/edits.

### Responsive design (desktop + mobile)

Every shipped UI surface — Founder Navigator, ecosystem map,
company profiles, claim flow, GOEO admin UI, `/agents` docs page —
must work cleanly at both desktop and mobile widths. Design
mobile-first; verify at 375 / 768 / 1280px. The demo will show
phones in the audience, and Utah founders & GOEO staff will open
this on mobile in the wild. See `AGENTS.md` § Coding Style for the
breakpoint policy and test checklist.

### GOEO admin UI

Utah Startup State / GOEO staff can keep the data current without
a developer. Auth is the same Better Auth system the regular users
use; admin pages require a session with role `goeo_admin` or
`superadmin` (Next.js middleware enforced — no separate token gate).
The admin shell uses a darker chrome (Auth.html#admin) to make role
context obvious.

- **Dashboard** (`/admin`) — landing page. Stats row (users,
  companies, resources, claim-queue size, open reports).
  Two-column body: pending claim queue (top of
  `/admin/submissions`) and a feed of recent agent edits drawn
  from `profile_updates` (Crew added 2 jobs via claude.ai;
  Routable description updated via chatgpt.com; etc.). Coverage
  gaps strip below — by county, sector, identity tag — surfaces
  what the catalog is missing.
- **Ownership-submission queue** (`/admin/submissions`) — review
  pending business-ownership submissions. View the uploaded doc
  via a signed R2 URL; approve (sets
  `companies.claimed_by_user_id` + `verified_at` + `claimed_at`)
  or reject (with notes). The single-submission page handles two
  modes: auto-approvable (domain match clean — one-click approve)
  and manual (no domain match — show claimant note, LinkedIn
  link, GOEO contact lookup, then Approve/Reject/Need-more-info).
- **Pending edits** review for owner-submitted profile changes.
- **Resources CRUD** — create, edit, delete entries in the
  resource directory (funding, mentoring, training, etc.) with
  all join-table fields (locations, industries, communities,
  topics). Status chips for live / stale / draft / broken-link.
- **Companies CRUD** — create, edit (no founder field whitelist),
  delete companies directly without a claim flow. Status chips
  for claimed / unclaimed / pending / flagged / duplicate.
- **Map curation** — click a pin to fix coordinates, edit the
  company, or hide a stale entry.
- **Users management** (`/admin/users`, read = admin, role-flip
  = `superadmin`) — list every user with role-filter chips
  (`all · founder · owner · investor · admin`) and counts. The
  superadmin can flip any user between `owner` and `goeo_admin`;
  the current logged-in superadmin's own row is disabled (no
  self-demotion). No "promote to superadmin" option from the UI —
  that's the bootstrap script's job.
- **Admin invites** (`/admin/admins`, `superadmin` only) —
  list current admins; "+ Invite admin" form writes a row to
  `admin_invites` (token hash + expiry) and emails a one-time
  link. Consuming the link flips the recipient's role to
  `goeo_admin` and marks the invite consumed.

### Agent-native layer

- **REST API** under `/api/v1/...` — versioned. OpenAPI spec at
  `/api/v1/openapi.json`.
- **CLI** (`startup-state` bin):
  - `startup-state recommend --persona priya --compact`
  - `startup-state map search --sector fintech --stage seed --json`
  - `startup-state company claim crew --domain trycrew.com --email founder@trycrew.com`
  - `startup-state profile build --company "NewCo" --from-url … --emit md,json,llms`
- **MCP server** (`startup-state-mcp` bin, stdio for local Claude
  Desktop demo). Tools: `recommend_resources`, `search_resources`,
  `search_companies`, `get_company`, `start_company_claim`,
  `update_company_profile`, `generate_founder_plan`,
  `generate_investor_tour`. Resources:
  `startupstate://resources/{id}`, `startupstate://companies/{slug}`,
  `startupstate://schemas/founder-passport`,
  `startupstate://schemas/company-profile`,
  `startupstate://datasets/resources`,
  `startupstate://datasets/companies`. Prompts: `founder_intake`,
  `investor_tour`, `company_profile_builder`,
  `resource_update_reviewer`.
- **`/llms.txt`** at site root with site description, links to API
  and schemas, and agent rules.
- **`/AGENTS.md`** at site root with end-user agent operating
  instructions (NOT the same as the repo-root AGENTS.md, which is
  for coding agents).
- **`/agents`** human-readable docs page that exposes the above
  surfaces in a way investors/judges can scan.

## Out of scope for this hackathon (explicit cuts)

Per `docs/hackathon-plan.md`, do not invest in:

- Real OAuth with ChatGPT/Claude (Better Auth covers our user /
  admin login; agents use `X-Atlas-Admin-Token`).
- Third-party social login providers (Google, GitHub, etc.) for
  Phase 4 — email + password is enough. The "Continue with
  Google" button on `/sign-up` and `/sign-in` renders as a
  Phase-5 stub (matches the wireframe; doesn't ship).
- Magic-link login for Phase 4 — same pattern: the `/sign-in`
  "Email me a magic link" button is a Phase-5 stub.
- 2FA, connected accounts, per-user agent tokens UI — `/settings`
  shows these sections with "coming soon" badges in Phase 4.
- Real-time notifications, weekly investor cluster-brief email,
  and map-personalization wired from `investor_profiles` —
  Phase 5.
- Complex CRM workflows.
- LinkedIn enrichment. Deferred. The GOED brief explicitly puts
  scraped LinkedIn enrichment out of scope; using Parallel.ai
  (which abstracts the scraping) on a LinkedIn URL doesn't change
  the spirit of that cut. Founder-supplied **business website**
  URLs *are* in scope and get enrichment via Parallel.ai (see
  § Founder Passport → Smart prefill).
- Perfect geocoding (use city/county centroids).
- Complicated vector-only RAG (defer embeddings).
- Calendar integration.
- Payment/funding application workflows.

In scope but kept simple: Better Auth email + password, file-upload
ownership verification (admin manually reviews), email verification
+ password reset via the `send-email` skill (Resend).

## Demo script (5 scenes — condensed from plan lines 449–528)

1. **Jordan (pre-seed)** — Click "Try Jordan". System produces a
   start-business checklist + student/community resources + "not
   yet" items.
2. **Priya (raising)** — Click "Try Priya". Capital-focused plan
   showing real Utah angels/VCs (Pelion, Grix, Tandem, Peterson,
   Kickstart, Red Rock Angels, Salt Lake Angels, Park City Angels).
   Pin those investors on the map.
3. **Investor view** — Switch to map. Filter "FinTech, Seed, 2-10
   employees, Salt Lake/Lehi/Utah County". Map clusters; sidebar
   generates an "Investor Brief" narrative.
4. **Business owner as website** — Claim a company profile. Update
   hiring status + add a job posting. Show that
   `/startups/crew.md` and `/api/v1/companies/crew` both updated
   from the same canonical source.
5. **Terminal / MCP proof** — Run
   `startup-state recommend --persona priya --compact`, then
   `startup-state company get crew --json`, then show the MCP tools
   list.

## Judging-rubric notes

- **Usability + design = 55% of judging.** Polish on the founder
  navigator and map matters more than feature breadth.
- One exceptional product beats two half-baked ones. Cut
  aggressively when behind.
- The agent layer is a *visible flourish in the demo*, not the main
  product. ~20 seconds of CLI/MCP airtime.
- "Why this matched" explanations build trust faster than generic
  AI prose. Show field-level reasons.
- Six persona buttons should produce *meaningfully different*
  outputs to validate against the organizers' test cases.

## Datasets

The state has provided complete datasets for both products — see
`docs/source_data/`. Agent 1 loads them into D1 directly from there;
no copy-into-`db/seed/data/` step.

- **`docs/source_data/Resources List - Builder Day - Sheet1.csv`** —
  226 resource rows. Columns: `id`, `Title`, `description`,
  `Communities`, `Industries`, `Locations`, `Topics`, `link`, `email`.
  Multi-values pipe-separated. Upstream IDs preserved as `r_<id>`.
- **`docs/source_data/Map Data for Builder Day  - Sheet1.csv`** —
  254 company rows (note **double space** in filename). Columns:
  `Display Type`, `LinkedIn Link (...)`, `Startup Name `,
  `Full Address`, `Description of startup`, `Website`, `Stage`,
  `# of Employees `, `Section`. Address-only — Agent 1 geocodes via
  city/county centroids.
- **`docs/source_data/page-2026-05-08-19-38-24.md`** — the canonical
  AI Builder Day brief from Utah GOED. Contains the verbatim persona
  descriptions, required company-profile fields, judging breakdown
  (30 / 25 / 25 / 20), and the link to the live site this build may
  replace: <https://startup.utah.gov/>.

The brief explicitly says: *"You don't need to research or compile
anything — focus every hour on the build."* Don't scrape startup.utah.gov,
LinkedIn, or pampam.city. Use what's in `docs/source_data/`.

## Source-of-truth pointers

- Implementation map: `docs/implementation-plan.md`.
- Screens / sitemap: `docs/screens.md`.
- Full plan: `docs/hackathon-plan.md`.
- Architecture: `docs/architecture.md`.
- Hackathon brief (canonical): `docs/source_data/page-2026-05-08-19-38-24.md`.
- Per-agent execution: `docs/agent-tasks/agent-<N>-<slice>.md`.
- Shared conventions: `docs/agent-tasks/00-shared-context.md`.
