# Agent 8 — Investor public surface + watchlists + intro brokerage

**Phase 6 — post-MVP.** Not part of the 24-hour hackathon ship.

You build three coordinated surfaces **on top of** Agent 5's
investor identity work:

1. **Public investor presence.** A directory at `/investors`, a
   public profile page at `/investors/<slug>` with `.md` / `.json`
   agent variants (mirroring `/startups/<slug>`), and a self-serve
   public-profile editor at `/investors/<slug>/edit` linked from
   the `/settings` Investor section.
2. **Saved-companies watchlist.** Investors save companies from
   the map / `/startups` to a personal list at `/me/saved`.
3. **Intro brokerage.** Both directions (founder/owner →
   investor, investor → founder/owner) flow through GOEO admin.
   Admin reviews in `/admin/intros`, accepts → both parties
   emailed (Resend) with each other's contact info.

Aim for **~150 minutes**. Most of the work is reuse of patterns
already in the codebase (`/startups/<slug>` triple → mirror it for
investors; `/admin/submissions` queue → mirror it for intros).

## What Phase 4 already shipped (don't redo)

Read Agent 5's brief (`agent-tasks/agent-5-claim.md`) and the
merged code before starting. **Do not rebuild any of this:**

- `investor_profiles` table (preferences-shaped: `firm_name`,
  `investor_type`, `stages_json`, `sectors_json`,
  `check_size_min/max`, `geo_focus_json`).
- `inv_*` ID prefix in `lib/ids.ts`.
- `'investor'` role in `auth.ts`, sign-up role chooser,
  email-OTP verify, `/onboarding/investor`.
- `/settings` page with an Investor section that edits the
  preferences fields (firm, type, stages, sectors, check size,
  geo focus).
- `POST /api/v1/investor-profiles` (upsert own preferences),
  `GET /api/v1/investor-profiles` (own profile fetch).
- 3 demo investor seed rows (Pelion, etc.) in
  `db/seed/investor-profiles.ts`.

Phase 5 polish wires the preferences into `/map` filter chips —
that's also someone else's work, not yours.

## Branch + worktree

- **Worktree:** any free of wt1-3.
- **Branch:** `feat/investor-public`. First action:
  `git checkout -b feat/investor-public`.

## Reads first

1. `docs/implementation-plan.md` § Phase 6 + the Agent 5↔8 and
   Agent 4↔8 coordination matrix rows.
2. `docs/agent-tasks/00-shared-context.md` — `sc_*` and `irq_*`
   prefixes are reserved for you.
3. `docs/architecture.md` — repo layout + dual-auth model.
4. `docs/requirements.md` § Investor public surface + watchlists
   + intros — single source of truth for what ships.
5. `docs/screens.md` § Phase 6 / post-MVP — URL matrix.
6. **`docs/agent-tasks/agent-5-claim.md`** — your closest analog.
   Auth wiring, role-gated middleware, admin queue with
   pending/accepted/declined states + email-on-status-change.
   You reuse all of this.
7. `docs/agent-tasks/agent-4-map.md` — the
   `/startups/<slug>` + `.md` + `.json` triple. Mirror it for
   `/investors/<slug>`.
8. The shipped `db/schema.ts` — read the actual table definitions
   for `investor_profiles`, `companies`, `user`. You extend
   `investor_profiles`, you reference `companies` and `user`.
9. `docs/design-guidelines.md` — brand tokens + primitives. Your
   pages must use the same `Tile`, `Chip`, etc.

## Depends on

**All of Phase 1–5 should be merged before you start.** This
isn't a hackathon-time agent — pick this up post-demo. If Phase 5
hasn't fully landed, the `/map` filter integration may still be
in flight, but that's not your concern.

## Owns (write surface)

### Schema + types

- `db/schema.ts`:
  - **Add columns to `investor_profiles`** (extension, no data
    loss; rebase before generate):
    ```
    slug                TEXT UNIQUE             -- nullable until publish
    display_name        TEXT
    bio                 TEXT
    tagline             TEXT
    website             TEXT
    linkedin            TEXT
    verification_status TEXT NOT NULL DEFAULT 'unverified'
                        -- 'unverified' | 'verified'
    verified_at         INTEGER (timestamp)
    last_updated_by     TEXT REFERENCES user.id
    ```
    Add an index on `verification_status` for the directory
    query.
  - **New table `saved_companies`:**
    ```
    id           TEXT PK ($defaultFn newId('sc'))
    investor_id  TEXT FK -> investor_profiles.id  NOT NULL
    company_id   TEXT FK -> companies.id          NOT NULL
    note         TEXT
    saved_at     INTEGER (timestamp) NOT NULL
    ```
    Indexes: `(investor_id)`, UNIQUE `(investor_id, company_id)`.
  - **New table `intro_requests`:**
    ```
    id                   TEXT PK ($defaultFn newId('irq'))
    requester_user_id    TEXT FK -> user.id   NOT NULL
    requester_role       TEXT                 NOT NULL
                         -- snapshot at request time:
                         -- 'founder' | 'owner' | 'investor'
    target_investor_id   TEXT FK -> investor_profiles.id
    target_company_id    TEXT FK -> companies.id
    -- EXACTLY ONE of target_* is non-null
    message_text         TEXT NOT NULL
    status               TEXT NOT NULL DEFAULT 'pending'
                         -- 'pending' | 'accepted' | 'declined' | 'introduced'
    submitted_at         INTEGER (timestamp) NOT NULL
    reviewed_by_user_id  TEXT FK -> user.id
    reviewed_at          INTEGER (timestamp)
    admin_notes          TEXT
    ```
    Indexes: `(requester_user_id)`, `(target_investor_id)`,
    `(target_company_id)`, `(status)`.
- `lib/ids.ts` — extend the `IdPrefix` union with `'sc'`,
  `'irq'`. Do **not** redefine `'inv'` — Phase 4 owns it.
- `schemas/investor-public.ts` — zod for the public-profile
  PATCH (slug, display_name, bio, tagline, website, linkedin).
- `schemas/intro-request.ts` — zod for the request POST body.
- `types/api.ts` — add types for the new endpoints.

### Public / signed-in UI

- `app/investors/page.tsx` — directory; lists profiles with
  `verification_status = 'verified'` only. Anonymous-readable.
  Filters: focus area, stage focus, location (optional v2;
  ship a flat grid first).
- `app/investors/[slug]/page.tsx` — public profile (HTML).
  Show display_name, tagline, location, focus areas, stage focus,
  check-size band, bio, external links. **No email.** Big primary
  CTA: "Request intro through GOEO" (signed-in users only — anon
  visitors get prompted to sign in).
- `app/investors/[slug]/edit/page.tsx` — owner self-edit (gated:
  `investor_profiles.user_id === session.user.id` OR admin).
  Editable: slug, display_name, bio, tagline, website, linkedin.
  **Not** editable here: verification_status (admin-only).
  Linked from `/settings` Investor section as "Manage your
  public profile."
- `app/investors/[slug]/route.md/route.ts` — markdown agent card
  (mirrors `/startups/<slug>.md` exactly — same content,
  different content-type).
- `app/investors/[slug]/route.json/route.ts` — JSON agent card.
- `app/me/saved/page.tsx` — investor's own watchlist. Table on
  desktop, cards on mobile. Each row links to the company
  profile + has an "unsave" button + an inline edit-note
  affordance.
- `app/me/intros/page.tsx` — own intros, sent + received. Status
  badges. Recipient sees the intro only after admin accepts
  (so contact info is visible).

### Admin UI

- `app/admin/intros/page.tsx` — intro-request queue, pending
  first. Each row: requester name + role, target name,
  submitted_at, status, message preview.
- `app/admin/intros/[id]/page.tsx` — review one intro. Full
  message, links to both parties' profiles, three buttons:
  Accept / Decline / Mark introduced + admin-notes textarea.

### API

- `app/api/v1/investors/route.ts` — `GET` (verified investor
  list, public).
- `app/api/v1/investor-profiles/[slug]/route.ts` — `GET`
  (public; redacts user.email), `PATCH` (three-auth-mode like
  `companies/[slug]`: owner, admin, machine).
- `app/api/v1/saved-companies/route.ts` — `POST` (save with
  optional note), `DELETE` (unsave by `?company_id=`), `GET`
  (own list, joined to companies). All signed-in only.
- `app/api/v1/intro-requests/route.ts` — `POST` (create), `GET`
  (own — sent + received).
- `app/api/v1/intro-requests/[id]/route.ts` — `GET` (own or
  admin), `PATCH` (admin only — moves status to
  accepted/declined/introduced; fires emails).

### Edits to existing surfaces

- `app/companies/[slug]/page.tsx` (Agent 4's surface) — add a
  small button group: "Save" (signed-in investor only) and
  "Request intro through GOEO" (any signed-in user). Coordinate
  with whoever currently owns the file via the PR.
- `app/settings/page.tsx` (Agent 5's surface) — within the
  Investor section, add a "Manage your public profile" link
  that routes to `/investors/<own-slug>/edit` (creates the
  slug from `display_name` on first click if not yet set).
  Coordinate with Agent 5.

### Helpers

- `lib/investor-card.ts` — shared formatter for HTML / .md /
  .json output (mirrors `lib/company-card.ts`).
- `lib/email.ts` (Agent 5's surface) — extend with templates:
  - `sendIntroPendingEmail(requester)` — confirmation.
  - `sendIntroAcceptedEmail(party_a, party_b, admin_note)` —
    sent to both parties, each with the other's contact info.
  - `sendIntroDeclinedEmail(requester, admin_note)` — polite no.
  - `sendIntroIntroducedEmail(requester)` — admin made the
    intro out-of-band; closes the loop.

### Tests

- `tests/intro-requests.test.ts` (Vitest) — happy path
  (request → admin accepts → both emails fire), decline path,
  authorization (you can't accept someone else's intro,
  recipients can't see pending intros).

## Deliverables

### 1. Schema migration

Rebase against `main` first. Run `npm run db:generate` →
`npm run db:migrate:local` → re-run `npm run seed` (the existing
seed will skip rows that exist; the new public-profile columns
on existing investor seed rows stay null until each demo investor
"publishes").

The migration must be backward-compatible: the new columns on
`investor_profiles` are nullable (or have defaults), and Phase 4
rows continue to satisfy `POST /api/v1/investor-profiles` (which
only writes the preference fields).

### 2. Slug bootstrapping

When an investor first opens `/investors/<auto-slug>/edit`, if
their row's `slug` is null, generate one from `display_name`
(or `firm_name` as fallback) using the same `slugify` Agent 1
shipped for company slugs. On collision, append `-2`, `-3`, etc.

### 3. Public directory + profile

`/investors` lists `verified` profiles only. Each row links to
`/investors/<slug>`. The profile page is anonymous-readable but
the "Request intro" CTA is signed-in only.

### 4. Self-serve public-profile editor

`/investors/<slug>/edit`. Field whitelist (no slug change after
verification, no `verification_status`). On save: PATCH
`/api/v1/investor-profiles/<slug>`. If the investor was
`verified` and changes a sensitive field (`display_name`,
`website`), optionally re-flip to `unverified` for re-review —
punt on this for v1 unless time allows.

### 5. Markdown + JSON agent variants

Mirror `app/startups/<slug>/route.md/route.ts` and `route.json`
exactly. Use `lib/investor-card.ts` to share the formatter
across HTML / markdown / JSON. **Email is never in the
output** — same redaction rule as Phase 4's
`GET /api/v1/investor-profiles`.

### 6. Saved companies

- `POST /api/v1/saved-companies` `{ company_id, note? }` →
  inserts a row keyed on `(investor_id, company_id)`. The
  UNIQUE index returns `409 CONFLICT` on duplicates.
- `DELETE /api/v1/saved-companies?company_id=co_xxx`.
- `GET /api/v1/saved-companies` → own list joined to
  `companies` for display.
- `/me/saved` renders a table-on-desktop, cards-on-mobile UI
  with an inline note field.
- Add a `<SaveButton companyId>` to
  `app/companies/[slug]/page.tsx` — only renders for sessions
  with `role === 'investor'`.

### 7. Intro brokerage — request creation

`POST /api/v1/intro-requests` body:

```jsonc
{
  "target": { "type": "investor", "id": "inv_..." }
  //  or:   { "type": "company",  "id": "co_..." }
  ,
  "message_text": "..."
}
```

Behavior:

1. Require a Better Auth session.
2. Validate the body. Reject if both targets set or neither set.
3. Insert `intro_requests` row with `status = 'pending'`,
   `requester_user_id = session.user.id`,
   `requester_role = session.user.role` (snapshot at request
   time — even if the role changes later, the audit trail
   reflects what they were when they asked).
4. Email the requester via `sendIntroPendingEmail(...)` —
   confirmation that the request is queued.
5. Return `{ id, status }`.

### 8. Intro brokerage — admin queue + actions

`/admin/intros` lists all rows, `pending` first. Click → detail
page with three buttons:

- **Accept.** PATCH `{ status: 'accepted', admin_notes? }`.
  Flips status, stamps `reviewed_by_user_id` + `reviewed_at`,
  fires `sendIntroAcceptedEmail` to **both** parties with the
  other's contact info + the admin note (if any).
- **Decline.** PATCH `{ status: 'declined', admin_notes? }`.
  Fires `sendIntroDeclinedEmail` to the requester only.
- **Mark introduced.** PATCH
  `{ status: 'introduced', admin_notes? }`. Admin already made
  the intro out-of-band; fires `sendIntroIntroducedEmail` to
  the requester (closes the loop).

`/me/intros` shows the requester's own queue (status visible),
plus inbound intros (target's own — only after admin moves them
to `accepted`).

### 9. Email templates

Five new templates in `lib/email.ts`. Plain HTML, no React
Email. Resend `from` is the same address Agent 5 uses for
verification. Templates include:

- the message text the requester wrote (HTML-escaped),
- the admin note (if any),
- in the accepted case, both parties' name + email + each
  other's profile link,
- the GOEO admin's name (the reviewer).

If `RESEND_API_KEY` is unset (dev / time-pressed), fall back
to `console.log` — same pattern as Agent 5's verification mail.

### 10. PR

```bash
git add db/schema.ts db/migrations \
        lib/ids.ts lib/email.ts lib/investor-card.ts \
        schemas/investor-public.ts schemas/intro-request.ts \
        types/api.ts \
        app/investors app/me \
        app/companies/\[slug\]/page.tsx \
        app/settings/page.tsx \
        app/admin/intros \
        app/api/v1/investors app/api/v1/investor-profiles \
        app/api/v1/saved-companies app/api/v1/intro-requests \
        tests/intro-requests.test.ts
git commit -m "feat(investor-public): directory + watchlists + admin-brokered intros"
git push -u origin feat/investor-public
gh pr create --base main --title "Investor public surface + intro brokerage"
```

## DONE when

1. **Profile publishing:** sign in as a Phase-4 investor, open
   `/investors/<auto-slug>/edit` from `/settings`, fill in the
   public fields, save. The row gains a slug + bio + etc.;
   `verification_status` stays `'unverified'`.
2. **Admin verification:** sign in as a `goeo_admin`, find the
   investor in `/admin/users` (or wherever you list them) or via
   the admin investor edit, flip to `verified`. The profile now
   appears in `/investors`.
3. **Public agent variants:** `/investors/<slug>`,
   `/investors/<slug>.md`, `/investors/<slug>.json` all return
   the same data with the right content-type. No email exposed.
4. **Save flow:** sign in as the investor, visit `/startups/crew`,
   click Save. `/me/saved` lists Crew. Unsave removes it.
5. **Founder requests intro:** sign in as a `founder`, visit
   `/investors/<slug>`, click "Request intro through GOEO",
   write a message, submit. `/admin/intros` shows pending.
6. **Admin accepts:** click Accept with a note. Both parties
   get emails (or both are logged to console if Resend is
   unset). `intro_requests` row is `accepted`. Both
   `/me/intros` views show the closed-loop record.
7. **Authorization:** another investor can't PATCH someone
   else's intro request. Anon users can't list other users'
   saved companies. Tests cover both.
8. **Mobile (375px):** every new surface
   (`/investors`, `/investors/<slug>`, `/investors/<slug>/edit`,
   `/me/saved`, `/me/intros`, `/admin/intros`,
   `/admin/intros/<id>`) works without horizontal scroll.
   Verified with `mcp__playwright__browser_resize`.
9. PR open.

## Cuts allowed if time-pressed (in priority order)

1. **Skip the `introduced` final state.** Treat `accepted` as
   terminal; admin manually closes out-of-band intros via the
   queue without a status change.
2. **Skip the public directory** — `/investors` returns 404;
   only direct profile links work. Verified investors are still
   reachable via `/investors/<slug>` and the agent variants.
3. **Skip the `.md` / `.json` agent variants for investors.**
   The HTML profile is enough for v1.
4. **Skip the watchlist `note` field.** Just save the
   `(investor, company)` join.
5. **Skip filtering on `/investors`.** Single flat grid; let
   founders scan.
6. **Skip the in-page "Save" / "Request intro" buttons on
   `/companies/[slug]`.** Users can navigate to `/investors`
   directly; the company-side buttons are convenience.

## Common pitfalls

- **Don't expose investor emails.** Public pages, `.md`,
  `.json`, OpenAPI all redact `user.email`. The only way to
  surface contact info is via the admin-accepted intro email.
- **Admin double-action.** Two admins might Accept the same
  intro near-simultaneously. Wrap the PATCH in a check that
  current status is still `'pending'`; otherwise return `409`.
- **Slug uniqueness.** Reuse Agent 1's `slugify`. On collision,
  append `-2`, `-3`, …
- **Defense in depth on `/me/*`.** Middleware gates "signed-in,"
  but the route handler must scope to `session.user.id`. Don't
  trust the URL for ownership.
- **Don't break Phase 4 endpoints.** `POST /api/v1/investor-profiles`
  must keep working with the preferences-only payload — the new
  public columns are nullable, with defaults where appropriate.
- **`requester_role` snapshot.** Store the role at request time
  — the requester's `user.role` may change later (e.g., admin
  flips them); the audit trail should reflect what they were
  when they asked.
- **Don't add `inv_*` to `lib/ids.ts`.** Phase 4 already shipped
  it; you only add `'sc'` and `'irq'`.
