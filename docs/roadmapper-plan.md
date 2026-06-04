# Roadmapper — Unified Implementation Plan

> Synthesized from four parallel research streams: schema design, Better Auth integration, fork/rebrand inventory, editor/content model. Working name "Roadmapper" — final name TBD.

---

## 0. Product summary

Self-hosted, open-source onboarding/roadmap tool. Authors create **libraries** of **roadmaps**; each roadmap is an ordered list of **sections**, each section an ordered list of **steps** with markdown content + resources. **Viewers** join via per-library access keys (or public URLs) and track personal progress through steps.

**Locked decisions:**
- Stack: Next.js App Router, Drizzle ORM, Postgres (prod) + SQLite (dev), Better Auth, Tailwind + shadcn/ui, Server Actions + TanStack Query, `react-markdown` + `remark-directive` shortcodes (v1) → BlockNote (v2), Docker Compose for self-host.
- Two-level hierarchy: Library → Roadmap → Section → Step.
- Roles: owner, admin, viewer. One non-rotating access key per library. Library visibility: `private` | `key` | `public`.
- Step status enum: `not_started` | `in_progress` | `done` (extensible).
- Org concept stubbed (`org_id` nullable; every user gets a personal org); UI uses `Organization*` tooltips.
- Zero telemetry, MIT license, JSON library import/export in v1.

**Deferred to v2+:** real block editor (BlockNote), graph view (React Flow), comments, email notifications, SSO/SAML, audit logs, analytics dashboards, mobile app, real-time collab, AI features, version history.

---

## 1. Repository fork & rebrand

### 1.1 Fork procedure

```powershell
robocopy "C:\Users\danys\MyGithubRepos\Fitpath" `
         "C:\Users\danys\MyGithubRepos\Roadmapper" `
         /E /XD node_modules .next .git /XF package-lock.json

Set-Location "C:\Users\danys\MyGithubRepos\Roadmapper"
git init
git add .
git commit -m "chore: initial fork from FitPath"
gh repo create DanySued/roadmapper --private --source=. --remote=origin --push
```

After push: link a new Vercel project (`vercel link`) — do **not** reuse the fitpath-dev project.

### 1.2 File disposition (high signal only — full table in research transcript)

**KEEP:** `src/lib/motion.ts`, `src/components/sections/ThemeToggle.tsx`, Tailwind config, ESLint config, tsconfig, `template.tsx`.

**REWRITE:** `layout.tsx` (metadata, drop banner), `globals.css` (rename tokens), `Nav.tsx` / `Footer.tsx` (replace brand + nav items), `login/page.tsx` / `signup/page.tsx` (wire to Better Auth), `dashboard/page.tsx` (rebuild as workspace), `auth-context.tsx` (gut mock, wire to real auth).

**DELETE:** every marketing section (`Hero`, `FAQ`, `Problem`, `BrainOptimized`, `Research`, `FeatureCarousel`, `LogoMarquee`, `AnnouncementBanner`), all of `src/components/fitpath/`, all of `src/lib/data/`, `paths/*`, `guides/*`, `videos/*`, `about/`, all FitPath brand assets.

### 1.3 Design token rename

Replace all `--fp-*` → `--rm-*` in `globals.css` and the `@theme inline` block. Color values unchanged. Also rename utility classes: `.fp-container` → `.rm-container`, etc. Delete `.hero-drift-*` keyframes (no marketing hero in v1). Full rename table in `roadmapper-planning-conversation.md` companion notes.

Rename Lucide icon `<Dumbbell>` → `<Map>` (or `<Route>` / `<Milestone>`) wherever it appears.

### 1.4 Dependency changes

**Add:** `better-auth`, `drizzle-orm`, `drizzle-kit` (dev), `postgres`, `better-sqlite3`, `@tanstack/react-query`, `zod`, `react-hook-form`, `@hookform/resolvers`, `react-markdown`, `remark-gfm`, `remark-directive`, `rehype-raw`, `rehype-sanitize`, `rehype-highlight`, `@dnd-kit/core`, `@dnd-kit/sortable`, `nanoid`.

**Optional/later:** `resend` (email), `nodemailer` (SMTP fallback).

**Remove:** nothing FitPath-specific in deps — current package set is all general-purpose.

### 1.5 Brand assets to create

`src/app/favicon.ico`, `src/app/icon.svg`, `src/app/opengraph-image.png` (1200x630), `brand/logos/icon.svg`. Defer until skeleton compiles.

---

## 2. Database schema (Drizzle)

Single schema source, dialect-agnostic where possible. Better Auth owns four tables; Roadmapper owns the rest.

### 2.1 Better Auth tables (managed via `npx better-auth generate`)

| Table | Notes |
|---|---|
| `user` | `id`, `name`, `email` (unique), `emailVerified`, `image`, `createdAt`, `updatedAt` |
| `session` | `id`, `userId → user.id`, `token` (unique), `expiresAt`, `ipAddress`, `userAgent` |
| `account` | OAuth/credential provider rows; `userId → user.id`, `providerId`, `password` (scrypt hash for credentials) |
| `verification` | Email/magic-link tokens |

SQLite note: `timestamp` columns must use Drizzle's `integer({ mode: "timestamp" })`. The `lib/db.ts` selector branches on `DATABASE_URL` prefix (`file:` → SQLite, else Postgres).

### 2.2 Roadmapper tables

| Table | Purpose | Key columns |
|---|---|---|
| `org` | Personal/team org stub | `id`, `name`, `slug` (unique), `ownerId → user.id`, `personal` bool default true |
| `library` | Top-level container | `id`, `orgId → org.id` (nullable v1), `name`, `slug`, `description`, `visibility` enum `private \| key \| public` |
| `library_member` | Owner/admin/viewer membership | composite PK `(libraryId, userId)`, `role` enum `owner \| admin \| viewer`, `accessKeyId` nullable |
| `access_key` | One per library | `id`, `libraryId` (unique FK), `token` (unique opaque) |
| `roadmap` | Named roadmap in a library | `id`, `libraryId`, `name`, `slug`, `description`, `position` int |
| `section` | Ordered group of steps | `id`, `roadmapId`, `title`, `position` int |
| `step` | Leaf node, holds content | `id`, `sectionId`, `title`, `body` text (markdown), `position` int |
| `step_resource` | Typed resources (link/file/video/embed) | `id`, `stepId`, `type` enum, `label`, `url`, `position` int, `meta` json-string |
| `step_progress` | Per-user-per-step | composite PK `(stepId, userId)`, `status` enum, `updatedAt` |
| `step_checklist_state` | Checked items inside `:::checklist` blocks | composite PK `(stepId, userId, itemIndex)`, `checked` bool |

**Indexes:** position indexes on `(libraryId, position)`, `(roadmapId, position)`, `(sectionId, position)`, `(stepId, position)`. User index on `library_member.userId` and `step_progress.userId`. Token index on `access_key.token`.

**Cascade rules:** all `*_id` FKs cascade on delete except `library_member.accessKeyId` which is `ON DELETE SET NULL` (revoking a key does not remove memberships).

### 2.3 Key design decisions

- **Viewers are a role on `library_member`**, not a separate table. Single join model; `WHERE role = 'viewer'`.
- **Ordering is integer `position` with gaps of 1000.** Universally supported on Postgres + SQLite; future drag-and-drop reorders only the moved row. Rebalance job when gaps exhaust.
- **Step content is a single markdown column.** v2 adds nullable `body_json jsonb` for BlockNote; both columns coexist during migration. Renderer prefers `body_json` when present.
- **Resources live in a sibling table**, not a JSON column. Enables ORDER BY, type filtering, per-resource edits.
- **Personal org auto-created on first sign-in** via Better Auth callback. `library.orgId` stays nullable for v1 flexibility; tightened to `NOT NULL` when multi-org ships.
- **Access keys grant viewer role via `/join/:token`.** Idempotent: existing members are not re-inserted.

### 2.4 Conflict resolution from research

- **Resources storage:** schema report (sibling table) **wins** over editor report's JSONB-on-step idea. Cleaner schema, future-friendly.
- **Checklist state:** added `step_checklist_state` as a small table (editor report needed it; schema report didn't include it). Item identity is by positional `itemIndex` for v1, with the known caveat that author edits to the checklist can misalign existing checks.

### 2.5 Migrations

Drizzle Kit. `drizzle/` folder with numbered SQL files committed to git. Generate the initial `init` migration after the schema is complete. Never `drizzle-kit push` in prod. CI runs migrations against a fresh Postgres container per PR.

### 2.6 Open schema questions (need decisions before first migration)

1. **Access key revocation cascade.** Does deleting a key also delete viewer memberships granted by it, or just null the FK? Recommendation: null only (less destructive).
2. **Roadmap-level visibility.** Currently visibility is per-library. Should individual roadmaps be independently public? Recommendation: no for v1.
3. **Slug URL structure.** Library slugs are unique per-org. URL shape: `/l/:slug` (single-org assumed) or `/o/:org/l/:slug` (multi-org ready). Recommendation: ship `/l/:slug` v1, plan to add the org prefix in v2 via redirects.
4. **Soft deletes.** None modeled. Recommendation: skip v1; revisit if/when "undo" or audit features come up.

---

## 3. Authentication (Better Auth)

### 3.1 Methods enabled in v1

- **Email + password** (always on). Self-host-friendly: works with zero third-party services.
- **GitHub OAuth** (off unless `GITHUB_CLIENT_ID` + `GITHUB_CLIENT_SECRET` set).
- **Google OAuth** (off unless `GOOGLE_CLIENT_ID` + `GOOGLE_CLIENT_SECRET` set).
- **Magic link** (off unless `MAGIC_LINK_ENABLED=true` AND email is configured).

**Deferred:** SAML/SSO, 2FA, passkeys, Better Auth's organization plugin (we hand-roll the org stub).

### 3.2 Env var contract

| Var | Required | Purpose |
|---|---|---|
| `BETTER_AUTH_SECRET` | yes | Session signing (32+ chars; `openssl rand -base64 32`) |
| `BETTER_AUTH_URL` | yes | App's base URL (matters for OAuth callbacks + cookies) |
| `DATABASE_URL` | yes | `file:./local.db` (SQLite) or `postgres://…` |
| `GITHUB_CLIENT_ID` / `GITHUB_CLIENT_SECRET` | no | Enables GitHub OAuth |
| `GOOGLE_CLIENT_ID` / `GOOGLE_CLIENT_SECRET` | no | Enables Google OAuth |
| `MAGIC_LINK_ENABLED` | no | Activates magic-link plugin (requires email) |
| `RESEND_API_KEY` | no | Preferred email transport |
| `SMTP_HOST` / `SMTP_PORT` / `SMTP_USER` / `SMTP_PASS` / `SMTP_FROM` | no | SMTP fallback |
| `ALLOW_REGISTRATION` | no | Default `true`; set `false` for invite-only mode |
| `REQUIRE_EMAIL_VERIFICATION` | no | Default `false`; opt-in for stricter posture |

### 3.3 File layout

```
lib/
  auth.ts                       — Better Auth server instance, all plugin/provider config
  auth-client.ts                — createAuthClient() for React
  db.ts                          — Drizzle db; dialect-selects on DATABASE_URL
  email.ts                       — sendEmail() abstraction: Resend → SMTP → console.log
app/
  api/auth/[...all]/route.ts    — toNextJsHandler(auth) catch-all
middleware.ts                    — Guards /app/* with getSessionCookie()
db/schema/
  auth.ts                        — Better Auth tables
  roadmapper.ts                  — All Roadmapper tables
  index.ts                       — Re-exports for Drizzle Kit
drizzle.config.ts
```

### 3.4 Email strategy

`lib/email.ts` decision tree:
1. If `RESEND_API_KEY` set → Resend SDK.
2. Else if `SMTP_HOST` set → Nodemailer.
3. Else → `console.log` the link (dev fallback).

Magic-link send and password-reset send are not awaited in the request handler (use `waitUntil` on Vercel, or fire-and-forget on Node).

### 3.5 Route protection

`middleware.ts` does an **optimistic** check (`getSessionCookie`) to redirect unauthenticated requests away from `/app/*` to `/sign-in`. Sensitive Server Actions and API routes always do a **full** check via `auth.api.getSession({ headers })` — never trust middleware alone for authorization.

### 3.6 Bootstrap & gotchas

- **First admin:** with `ALLOW_REGISTRATION=true` (default), the first user to sign up is the system's first user. After that, operators can set `ALLOW_REGISTRATION=false` for invite-only. A `scripts/seed-admin.ts` is provided for ops who want to skip the UI step.
- **BETTER_AUTH_URL must match the public origin exactly** — including scheme + port. Reverse proxies must forward `X-Forwarded-Host` and `X-Forwarded-Proto`.
- **GitHub OAuth has no refresh tokens** (intentional).
- **Better Auth is ESM-only** — already fine in Next.js, but custom scripts need `"type": "module"`.

### 3.7 Open auth questions

1. **SMTP vs Resend precedence** when both are set — recommend Resend wins (current plan).
2. **Session lifetime** — recommend 7-day default; expose `SESSION_MAX_AGE_DAYS` env var if anyone asks.
3. **Magic-link-only mode** — supported by disabling email/password + enabling magic-link; document in self-host guide.

---

## 4. Step content & editor (v1)

### 4.1 Block syntax — `remark-directive`

```markdown
:::callout{type="warn"}
Body with **markdown** supported.
:::

:::checklist
- [ ] First item
- [ ] Second item
:::

::embed{url="https://www.youtube.com/watch?v=…" caption="Optional"}

::image{src="/uploads/diagram.png" alt="Required alt" caption="Optional"}
```

Plus all standard markdown (headings, lists, links, code blocks, blockquotes, tables via `remark-gfm`).

**Why directives over fenced code blocks:** dedicated MDAST node types (`containerDirective` / `leafDirective`), structured attributes, clean mapping to BlockNote JSON in v2, no conflict with syntax highlighting.

### 4.2 Markdown pipeline

```ts
remarkPlugins: [remarkGfm, remarkDirective, remarkCustomBlocks]
rehypePlugins: [rehypeRaw, rehypeHighlight, [rehypeSanitize, customSchema]]
```

- `remarkCustomBlocks` (local) maps directive nodes → custom component names.
- `rehype-sanitize` runs last with an extended schema allowing `figure`, `figcaption`, and `iframe` (with allowlisted `src` origins: YouTube, Loom, Vimeo).
- CSP header `frame-src` matches the allowlist.

### 4.3 Editor UI

- **Desktop (≥ 768px):** side-by-side textarea + live preview with a draggable divider.
- **Mobile:** Write/Preview tab toggle.
- **Toolbar:** four buttons only — Callout, Checklist, Embed, Image. No bold/italic — authors using a markdown editor type those.
- **Autosave:** debounced Server Action, 1500ms. Indicator: Saving / Saved / Error. `Cmd+S` triggers immediate save.
- **Dirty guard:** `beforeunload` + Next router guard prompts on unsaved changes.

### 4.4 Resources panel

Lives in `step_resource` table (not on the step body). Editor has a "Resources" tab; viewer sees a collapsible right-side panel. Add/edit/reorder via `@dnd-kit/sortable`.

### 4.5 Interactive checklists

Per-user persistence via `step_checklist_state(stepId, userId, itemIndex)`. Optimistic toggle via TanStack Query. Known v1 limitation: if an author reorders checklist items, existing checks misalign by index (acceptable for v1; v2 may move to stable hashes).

### 4.6 Image uploads

**v1 decision needed:** local disk (`/uploads`) is simplest but breaks on stateless containers. S3-compatible (Supabase Storage, R2, MinIO) is portable but adds env vars.

**Recommendation for v1:** **URL-only.** The Image toolbar button accepts a URL (no file picker). Local-disk uploads ship in v1.1 with `STORAGE_DRIVER=local|s3` env var. Defers the storage abstraction without blocking the editor.

### 4.7 v1 → v2 migration (BlockNote)

Markdown is re-parsed through the same `remark` + `remark-directive` pipeline → MDAST → BlockNote block JSON. `containerDirective` exposes `name` + `attributes` matching BlockNote's `type` + `props`. Mapping is mechanical, not heuristic. Step rows gain nullable `body_json`; renderer prefers it when present. v1 and v2 steps coexist.

### 4.8 Component file plan

```
components/step/
  MarkdownRenderer.tsx         — react-markdown + pipeline
  StepEditor.tsx               — full screen, autosave, dirty tracking
  EditorPane.tsx               — textarea
  PreviewPane.tsx              — preview wrapper
  EditorToolbar.tsx            — 4 block buttons + save status
  SaveStatus.tsx               — indicator
  ResourcesPanel.tsx           — viewer-side panel
  ResourceEditor.tsx           — author-side add/remove/reorder
  ResourceItem.tsx
  blocks/CalloutBlock.tsx
  blocks/ChecklistBlock.tsx
  blocks/EmbedBlock.tsx
  blocks/ImageBlock.tsx
lib/
  remark-custom-blocks.ts      — directive → component name mapper
  sanitize-schema.ts           — extended rehype-sanitize schema
app/actions/
  step.ts                      — saveStepContent, saveStepResources
  checklist.ts                 — toggleChecklistItem
```

---

## 5. App routes & shell

### 5.1 Route structure

```
/                              — redirect to /app or /sign-in based on session
/sign-in                       — Better Auth email/password + OAuth buttons
/sign-up                       — Registration (gated by ALLOW_REGISTRATION)
/forgot-password               — Password reset request
/reset-password                — Password reset (token from email)
/join/:token                   — Access key claim → adds viewer membership → redirects to library

/app                           — Dashboard: list of user's libraries (owned, admin, viewer)
/app/libraries/new             — Create library
/app/l/:librarySlug            — Library home: list roadmaps + members + access key (if admin+)
/app/l/:librarySlug/settings   — Library settings (admins only)
/app/l/:librarySlug/r/:roadmapSlug
                               — Roadmap viewer: sections + steps + progress
/app/l/:librarySlug/r/:roadmapSlug/edit
                               — Roadmap editor (admins only): reorder sections/steps
/app/l/:librarySlug/r/:roadmapSlug/s/:stepId
                               — Step viewer with resources panel
/app/l/:librarySlug/r/:roadmapSlug/s/:stepId/edit
                               — Step editor (StepEditor component)

/l/:librarySlug                — Public/unauthenticated read view (only if library.visibility = public)
/l/:librarySlug/r/:roadmapSlug
/l/:librarySlug/r/:roadmapSlug/s/:stepId

/api/auth/[...all]             — Better Auth catch-all
```

Server Actions live in `app/actions/*.ts` colocated with routes that use them.

### 5.2 Public surfaces

- Public libraries are viewable by direct URL only in v1. No `/explore` discovery page.
- Public viewing does **not** persist progress (no userId). Sign-in prompt encourages account creation to track.

### 5.3 Authorization helpers

`lib/authz.ts` exports:
- `requireSession()` → throws/redirects if no session.
- `requireLibraryMember(libraryId, minRole)` → `'owner' | 'admin' | 'viewer'`, throws on insufficient.
- `requireLibraryAdmin(libraryId)` → shortcut for `'admin'` or above.

Every Server Action and route handler calls one of these. Middleware only handles redirect UX.

### 5.4 Org tooltip pattern

Anywhere the UI shows "Organization" in v1, render `<OrgLabel />` which is the word + `*` + a shadcn Tooltip explaining: *"Future feature — currently every user has a personal organization."*

---

## 6. Self-host packaging

### 6.1 Docker setup

- **`Dockerfile`** — Multi-stage: build with full deps → runtime with Next.js `output: 'standalone'`. Final image runs `node server.js` on port 3000.
- **`docker-compose.yml`** — Two services: `app` and `db` (Postgres 16). Volume for Postgres data. `.env` mounted from host.
- **`docker-compose.dev.yml`** — Single service running in dev mode against SQLite (no db service needed). For "try it in 30 seconds" UX.

### 6.2 `next.config.ts` additions

- `output: 'standalone'` for Docker.
- CSP `frame-src` allowlist matching embed providers.
- Image domains for self-hosted upload origins (added in v1.1 with storage driver).

### 6.3 Telemetry posture

Zero. No `@vercel/analytics`, no Sentry, no PostHog in the core build. README explicitly states this. If anyone wants telemetry later, env-gated and off-by-default.

### 6.4 README essentials

- One-paragraph what-is-it.
- "Run with Docker in 30s" quickstart (SQLite, no email config).
- Production setup (Postgres + email).
- Env var reference (link to env.example).
- Contributing guide.
- MIT license badge.

---

## 7. Milestones

### M0 — Fork & rebrand (1 session)
- Run fork procedure → new repo on GitHub
- Rename tokens (`--fp-*` → `--rm-*`) and classes
- Delete marketing sections, FitPath data files, fitpath components
- Replace brand assets (favicon, icon, logo)
- Update package.json name, layout metadata
- Verify `npm run build` passes
- **Output:** empty Roadmapper skeleton that compiles and shows a placeholder Nav + Dashboard

### M1 — Auth + DB foundation (1–2 sessions)
- Add deps: better-auth, drizzle-orm/kit, postgres, better-sqlite3
- Build `lib/db.ts` (dialect-selecting), `lib/auth.ts`, `lib/email.ts`
- Generate Better Auth schema, define Roadmapper schema in `db/schema/`
- Initial Drizzle migration
- `app/api/auth/[...all]/route.ts`, sign-in/sign-up pages wired to authClient
- `middleware.ts` guarding `/app/*`
- Personal-org auto-create on first sign-in
- **Output:** can sign up, log in, log out; session persists; `/app` requires auth

### M2 — Library + roadmap CRUD (2 sessions)
- Dashboard listing user's libraries (owner/admin/viewer separated)
- Create library flow + settings page (visibility, access key generation)
- Library home: list roadmaps + members + access key
- Create roadmap / section / step (form-based; no drag yet)
- `/join/:token` endpoint adds viewer membership
- Authz helpers (`lib/authz.ts`) used on every mutation
- **Output:** can create a library, add roadmaps with sections + steps, share key, join as viewer

### M3 — Step editor + viewer (2 sessions)
- `MarkdownRenderer` with full plugin pipeline
- `StepEditor` with split layout + autosave
- All four block components (Callout, Checklist, Embed, Image)
- `step_resource` CRUD + Resources panel
- `step_progress` toggling (Server Action + TanStack Query optimistic update)
- `step_checklist_state` for in-content checkboxes
- Owner/admin view of all viewers' progress
- **Output:** authoring + tracking flow end-to-end

### M4 — Public libraries + import/export (1 session)
- Public read views at `/l/:slug` (no auth)
- JSON export of entire library
- JSON import (validates schema, creates new library)
- **Output:** v1 feature-complete

### M5 — Self-host polish (1 session)
- Dockerfile + docker-compose.yml + docker-compose.dev.yml
- `.env.example` with full reference
- README with quickstart + prod guide
- `scripts/seed-admin.ts`
- MIT LICENSE file
- **Output:** v1.0 ship

### v1.1 (post-launch)
- Image uploads with `STORAGE_DRIVER=local|s3`
- Drag-and-drop reorder for sections + steps
- Markdown export per-roadmap

### v2 (later)
- BlockNote editor migration
- React Flow graph view of roadmap overview
- `/explore` discovery page for public libraries
- Real organization plugin (multi-org)
- Comments, version history, email notifications, SSO

---

## 8. Cross-cutting decisions still open

These are not blockers for starting M0–M2 but need answers before the milestone they touch:

| # | Question | Blocks | Recommendation |
|---|---|---|---|
| 1 | Access key revocation: cascade memberships or null only? | M2 | Null only |
| 2 | Roadmap-level visibility (independent of library)? | M2 | No, v1 |
| 3 | URL slug shape now vs multi-org future? | M2 | `/l/:slug` now, add `/o/:org/...` redirect in v2 |
| 4 | Soft deletes? | M2 | No, v1 |
| 5 | Image storage in v1 — URL-only or local-disk? | M3 | URL-only; storage driver in v1.1 |
| 6 | Step body size cap? | M3 | 100 KB enforced server-side |
| 7 | Code-block syntax theme — follow app dark/light or fixed? | M3 | Follow app theme (CSS-variable hljs theme) |
| 8 | Real project name? | M5 (README, Docker image name) | Sleep on it |
| 9 | Open registration vs invite-only default? | M1 | Open (`ALLOW_REGISTRATION=true` default) |
| 10 | Email verification required by default? | M1 | No (env opt-in) |

---

## 9. What this plan deliberately does NOT include in v1

Don't add these even when tempted:

- Real Notion-style block editor (BlockNote) — v2
- Roadmap graph visualization — v2
- Discovery / search for public libraries — v2
- Comments on steps — v2
- @mentions — v2
- Version history / step revisions — v2
- Real-time collaborative editing — v2
- Email notifications — v2
- SSO / SAML — v2
- 2FA / passkeys — v2
- Audit logs — v2
- Analytics dashboards — v2
- Mobile app — never (PWA-friendly is enough)
- AI features (auto-generate, suggest steps) — v2
- Markdown export — v1.1
- Image file uploads — v1.1
- Drag-and-drop reordering — v1.1
- Multi-org — v2

---

## 10. Next concrete action

**M0, step 1:** Run the fork procedure (section 1.1). Once the new `Roadmapper` repo exists, push to GitHub, and verify a clean `npm run build` on the unmodified copy — then start the rename + deletion pass.

Decide question #8 (real name) before the fork if possible, otherwise commit as `roadmapper` and plan a one-time rename later.
