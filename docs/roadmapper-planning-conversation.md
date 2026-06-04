# Roadmapper — Planning Conversation

## User

In this session you only think. If you need to iterate or read or search the web, send a Sonnet agent to do that. The goal of this app is actually to create and store roadmaps for different purposes. Personal or not. For example: in my work whenever there is a new worker, he needs to learn a lot of stuff, and the roadmapper would be very useful for that. So I want to optimize it for creating new roadmaps and organizing libraries of roadmaps according to categories or something like that. In the end I want to make this product open source with this background I want to hear your opinion before we get to planning

## Assistant

Got it. Here's my honest read before we plan:

**The pivot makes sense, but the current app is fighting it.** FitPath right now is a *marketing site for prebuilt fitness paths* — static `paths.ts` / `path-details.ts` / `guides.ts` data files, landing sections (Hero, FAQ), and read-only `PathCard` / `PathGrid` components. What you're describing is a *roadmap authoring + library tool*: create, edit, organize, share. Those are almost disjoint products. The fitness content becomes one example seed, not the spine.

**The core question is single-player vs multi-player.** An onboarding-roadmap use case implies at least two roles (author + learner), persistence beyond localStorage, and probably progress tracking per learner. That decision drives everything else: auth, DB, whether Vercel + a Marketplace Postgres (Neon) is enough, or whether you need real user accounts from day one. For an open-source product this also shapes self-hostability — people will want to BYO database.

**My recommendation, as a starting position to push back on:** treat this as a near-greenfield rebuild *inside* the existing repo. Keep the design tokens, Nav/Footer shell, and `PathCard` visual language — throw out the static data layer and landing-page-as-product framing. Build around three primitives: **Roadmap** (ordered steps, optional nesting), **Category/Library** (tagging + collections), **Step** (markdown + resources + optional checkable state). Start single-user with a real DB (Neon via Vercel Marketplace), design the schema as if multi-user from day one (a `user_id` column that's nullable/default now), and defer auth until you actually need sharing. Open-source-ready means: clean schema, env-var-driven config, no vendor lock beyond Postgres.

**Three things I'd want your take on before planning:**

1. Is the "library" yours alone (curated by you, others consume), or should anyone be able to author? That's the single biggest scope fork.
2. Do learners need *progress tracking* (checkboxes that persist per person), or is the roadmap purely a reference document?
3. Are you willing to drop the current fitness landing page entirely, or do you want to keep it as a public showcase with the authoring tool behind a route like `/app`?

## User

If a user creates a library he can give admin to other users he chooses, otherwise only he can edit it. To view the library created by someone else you need to have an access key. The personalized progress tracking is one of the main features of the app so its essential the app tracks the progress of each user. I didn't understand the question, but I want to keep the fitness app stored as backup and work in the roadmapper as a remix of it. I would duplicate it, rename the new to roadmapper, change icon and favicon and then start working on the changes of the concept of the app. What do you think

## Assistant

Good answers — that clarifies the shape a lot. Reactions:

**Access model is clean and sensible.** Owner + invited admins + key-gated viewers is a well-known pattern (think Notion share links). It's also friendly to open source because it doesn't require a heavy permissions system on day one — three roles, one share token per library, done. Just decide early whether the access key grants *library-wide* view or *per-roadmap* view; I'd say library-wide, simpler mental model.

**Progress tracking as a core feature changes the data model meaningfully.** It means viewers aren't anonymous — even key-gated users need *some* identity so their checkmarks persist. Two options: (a) require real accounts to view (simpler, cleaner, but adds friction), or (b) allow "guest" progress tied to a browser/device with optional account upgrade later. For an onboarding tool used at work, (a) is almost certainly right — you *want* the manager to see which new hire finished which step. So: auth is not deferrable after all. It's day-one.

**The fork-and-rename approach is the right call.** Keeping FitPath frozen as a backup repo and starting Roadmapper as a fresh repo (duplicated, renamed, rebranded) is cleaner than trying to evolve one into the other in-place. You get: clean git history for the new product, freedom to delete fitness-specific code aggressively without worrying about breaking the live site, and a real "v0" moment. The shared design tokens and component shell come along for free.

**Where I'd push back gently:** "remix" suggests keeping a lot of the existing UI. I'd treat the visual language (tokens, typography, card aesthetic) as the only thing worth preserving. The information architecture is completely different — FitPath is *browse → pick a path → read*, Roadmapper is *log in → my libraries → pick library → pick roadmap → track progress / edit*. That's a different app skeleton, not a reskin. The Nav, Hero, FAQ, Footer sections largely don't survive.

**Revised mental model for planning:**

- **New repo:** `roadmapper`, forked from Fitpath, rebranded.
- **Auth from day one** (Clerk via Vercel Marketplace is the path of least resistance; open-source-friendly alternative is Auth.js + Postgres if self-hosting matters more).
- **Postgres from day one** (Neon via Marketplace).
- **Core entities:** User, Library, LibraryMember (role: owner/admin/viewer), AccessKey, Roadmap, Step, StepProgress (per-user-per-step).
- **Two surfaces:** authenticated app (`/app/*`) and a public landing page you can ship later. Don't waste time on landing copy now.

**One open question before we plan:** for open-sourcing — is the goal that *others self-host their own instance* (org-internal onboarding tool, like Outline or Plane), or that *this becomes a hosted multi-tenant SaaS* with the code public (like Cal.com)? Both are valid but they shape decisions: self-host-first pushes you toward Auth.js + plain Postgres + Docker; SaaS-first lets you lean on Clerk/Vercel without guilt. Which one is the real target?

## User

The goal that others self-host their own instance org-internal onboarding tool. What do you think? What else do we need to think before planning the app?

## Assistant

Self-host-first is the right call for this use case and it sharpens a lot of decisions. Onboarding content is *internal company knowledge* — orgs will not want it on someone else's SaaS. Plane, Outline, AppFlowy all won by being self-hostable first. It also means you should be designing for a "one Docker compose up and you have it running" experience as the north star.

**What that locks in:**

- **Auth must be self-hostable.** Clerk is out as the default — it's a hosted service. Auth.js (NextAuth) with a Postgres adapter is the standard answer; supports email magic links, OAuth providers the org configures themselves, and credentials. Optionally support SSO (SAML/OIDC) later for enterprise.
- **Database: plain Postgres.** No Neon-specific features, no Vercel-only APIs. Drizzle or Prisma as ORM — both are self-host-friendly. I'd lean Drizzle (lighter, SQL-first, easier for contributors to read).
- **No vendor-locked storage.** If you need file uploads (images in steps, attachments), abstract storage behind an interface with an S3-compatible default. Users plug in MinIO, R2, S3, whatever.
- **Config via env vars only.** No hardcoded URLs, no "works on Vercel only" assumptions.
- **Dockerfile + docker-compose.yml are first-class deliverables**, not afterthoughts. Your own dev loop should use them.

---

**Things to think through before planning, in roughly priority order:**

**1. Content model for a step.** This is the actual product. Is a step: plain markdown? Rich blocks (like Notion)? Markdown + embedded resources (links, files, videos)? Can a step have sub-steps (nested roadmaps) or is it strictly flat? My instinct: markdown body + a typed "resources" array (link, file, video, embed) + optional sub-steps one level deep. Notion-style block editors are a tarpit — don't go there in v1.

**2. Roadmap structure: linear vs DAG vs tree.** Onboarding is usually mostly linear with some parallel tracks ("week 1: setup" branches into "frontend track" / "backend track"). A pure list is too restrictive; a full DAG is overkill and hard to render. Suggestion: ordered list of *sections*, each containing ordered steps. Two-level hierarchy. Ship that, see if anyone asks for more.

**3. Progress semantics.** Per-step boolean done/not-done is the floor. Do you also want: time-to-complete tracking, "blocked" status, comments per step, manager sign-off ("your manager must verify this step")? Sign-off is a real onboarding need but adds complexity. I'd ship boolean-only in v1, design the schema to allow status enums later.

**4. Visibility model edge cases.** You said library-wide access keys + invited admins. Some real questions: (a) can a viewer's progress be seen by the owner/admins? (almost certainly yes — that's the whole point), (b) can keys be revoked / expire?, (c) can one library have multiple keys with different roles?, (d) what happens to a viewer's progress if the key is rotated? Decide these now, they shape the schema.

**5. Org / team concept — or not?** Right now you're describing User → Library. But onboarding is org-scoped. Does "User" belong to an Org, and libraries belong to Orgs? Or is it flat (a user owns libraries directly, and shares them ad-hoc)? Flat is simpler to ship. Org-scoped is what real companies need. My recommendation: ship flat, but put a `org_id` column on User and Library from day one (nullable, defaults to a personal org). Migrating to real orgs later is then a config change, not a schema migration.

**6. Public vs private libraries.** Your model is "you need a key to view." Do you also want *fully public* libraries (community-shared roadmaps — "learn Rust", "onboard a junior PM")? For an open-source onboarding tool, probably not v1, but it'd be a natural extension. Don't build it now, don't preclude it.

**7. Editing UX.** Authoring is the *hard* part of this product. Drag-to-reorder steps, inline edit, markdown preview, autosave, undo. The market is full of tools that have great viewing UX and terrible authoring UX. Decide now if v1 is "edit JSON in a form" (ugly but fast to ship) or "polished drag-and-drop editor" (slow but is the actual product). I'd go middle: clean form-based editor with reordering, no fancy block editor, ship it, iterate.

**8. Import / export.** For self-hosters this matters more than you think. Markdown export of a whole roadmap. JSON export/import of a library (for backups, sharing across instances). This is a v1 nice-to-have, v2 must-have.

**9. Telemetry and privacy.** Open-source onboarding tools must default to *zero phone-home*. No Vercel Analytics in the core, no Sentry without env-var opt-in. Decide the telemetry posture now so you don't have to rip it out later.

**10. Naming.** "Roadmapper" is descriptive but generic and SEO-crowded (roadmap.sh exists, lots of "roadmap" tools). Worth 30 minutes thinking of a real name before you fork — renaming later in an OSS project is painful (repo URLs, Docker images, docs). Not a blocker but flag it.

**11. License.** Open source has tradeoffs. MIT is permissive (anyone can fork and SaaS-ify it). AGPL prevents that but scares off some contributors. If you don't care about someone hosting it commercially, MIT. If you want to be the only commercial host, AGPL or BSL. Decide before first commit.

**12. What's *not* in v1.** Equally important. My nominations to cut: notifications/email, SSO, audit logs, analytics dashboards, mobile app, real-time collab, AI features. All tempting, all v2+.

---

**My summary recommendation:** the planning doc should answer #1–#5 explicitly (they're schema-shaping), make decisions on #9 + #11 (one-line policies), and explicitly defer #6, #8, #10, #12 with a note. #7 is the biggest UX risk and deserves its own thinking.

Want to work through #1–#5 next, or is there one of these where your gut already has an answer?

## User

1. A combination of notion style blocks and "roadmap.sh" (take a look at the website using playwright if you're not familiar with its style)
2. I like that suggestion. Ordered list of sections, each containing ordered steps
3. Boolean is okay, but I want to keep it open for other options, so like this: DONE, NOT YET, OTHER OPTIONS
4. a. Yes b. No c. Only two types: admin, user (the user only fills the library in his account) d. Not relevant since keys are not rotated
5. Okay do that but everything related to the org build that shows in the UI should have an * that when hovered shows a small text describing why that's there
6. Yes I also want fully public libraries
7. Let's go with the middle, but keep in mind that I want to upgrade to the polished drag and drop editor futurely
8. OK then we should have that in the app
9. No telemetry. Is that an option?
10. I will think a little bit more about the name
11. MIT is okay
12. Didn't get it

## Assistant

Good answers — most of these tighten things up. Reactions:

**1. Notion blocks + roadmap.sh style.** I know roadmap.sh — node-graph layout, color-coded boxes (yellow=primary, gray=optional, etc.), click a node → side panel with content. That's a *visualization* layer on top of the data, not a replacement for it. The Notion-block part is the *step content*; the roadmap.sh part is the *navigation/overview*. Good combo, but be aware: a real block editor (drag handles, slash menu, nested blocks, inline formatting) is 2-3 months of work on its own. Tiptap or BlockNote are the realistic libraries. I'd cut it down for v1: **markdown with a live preview + a small set of "block" types as fenced shortcuts** (callout, checklist, embed, image). Full Notion-style blocks is a v2 milestone. For the roadmap.sh-style overview: ship list view first, graph view second — graph layout (React Flow) is also a significant undertaking.

**2. Two-level hierarchy.** Locked in.

**3. Status as enum, not boolean.** Smart. I'd ship with three values — `not_started`, `in_progress`, `done` — and design the column as a free-form short string (or a proper enum that's easy to extend). "OTHER OPTIONS" can be added later without migration pain. Don't ship a literal "OTHER" value in v1; pick the three above.

**4. Access model.** Clean. So: owner + admins (edit), viewers via key (read + track own progress). One key per library, non-rotating. Got it.

**5. Asterisk + hover tooltip for org-related UI.** Works. Concretely: anywhere "Organization" appears in v1, it'll be "Organization*" with a tooltip like *"Future feature — currently every user has a personal organization."* Good UX hint, sets expectations.

**6. Public libraries.** OK. Adds one field (`visibility: private | key | public`) and a discovery surface (`/explore` or similar). Not much extra schema, but the discovery UI is real work. I'd ship public libraries as *viewable via direct URL* in v1 (no discovery page), and add a browse/search page in v2.

**7. Form editor now, drag-and-drop later.** Locked in. Schema-wise we just need to make sure step ordering is a clean integer column (or fractional indexing) so the future drag editor doesn't require a migration.

**8. Import/export in v1.** Good. JSON export/import of a whole library is the minimum. Markdown export per-roadmap is nice. I'd scope v1 to JSON only — Markdown export gets messy because of block types.

**9. Zero telemetry.** Yes, totally an option. The default state is: nothing is sent anywhere. No analytics packages installed, no error reporting in the core build. If you ever want optional telemetry later, it goes behind an env var that defaults to off (the Plane/Outline pattern). For v1: just don't install any of it.

**10. Name.** Take your time. One quick thought: searchable + ungeneric matters more than clever. "Pathwise", "Trailguide", "Onboardly" — all bad, just illustrating that you want something that's *not* a common word. Sleep on it.

**11. MIT.** Locked in.

**12. "What's NOT in v1."** I meant: explicitly write down the things we're *deciding not to build* in v1, so we don't accidentally scope-creep into them. My v1 cut list: email notifications, SSO/SAML, audit logs, analytics dashboards, mobile app, real-time multi-user editing, AI features (auto-generate roadmaps, suggest steps), comments on steps, @mentions, version history. All of these are reasonable v2+ features. Naming them now means when you're tempted mid-build to add "just a quick comments feature," you can point at the list and say no.

---

**Where I think we are.** The product shape is clear enough to plan now. Before we move into actual planning, two small things to lock:

**A. Tech stack confirmation.** My read: Next.js App Router (keep), Postgres + Drizzle, Auth.js (email magic link + GitHub OAuth as defaults), Tailwind + existing tokens, shadcn/ui (already there), Tiptap *deferred* to v2. Docker compose for self-host. Anything you want different?

**B. The block editor question.** I want to make sure you're aligned on "v1 = markdown with a live preview + a few special block shortcuts" not "v1 = real Notion-style editor." The latter doubles the project timeline. OK to defer the polished editor?

Once those two are settled, we're ready to draft the actual plan: schema, route structure, milestones.

## User

a. What other options do I have
b. That's OK

## Assistant

Here are the realistic alternatives at each layer, with tradeoffs framed for a self-hosted open-source onboarding tool:

**Framework**

- **Next.js App Router** *(my default)* — what you have, great Vercel story, but self-hosters need to run a Node server (fine, but heavier than a static SPA). React Server Components are a learning curve for contributors.
- **Remix / React Router v7** — simpler mental model, no RSC complexity, runs on any Node host. Smaller ecosystem of examples though, and you'd be rewriting more of the existing FitPath code.
- **SvelteKit** — much smaller bundles, friendlier for contributors who don't love React, very nice DX. But you'd throw away all your existing components and shadcn/ui.
- **Plain Vite + React SPA + separate API** — most self-host-friendly architecture (static frontend + small API server), easiest to understand for contributors. But you give up SSR/SEO for the public landing page and lose the "one app, one deploy" simplicity.

My recommendation stays Next.js — you already have it, the team behind it cares about self-hosting now (standalone output mode is solid), and shadcn/Tailwind/your tokens come along for free.

**Database**

- **Postgres** *(my default)* — universal, every host supports it, every contributor knows it.
- **SQLite** — *strong* alternative for a self-hosted tool. One file, zero ops, Litestream for backups. Plane and many self-hosted tools support it as the "just run it on a Pi" option. Downside: weaker concurrent write story, no advanced features (full-text search is meh).
- **Postgres + SQLite dual support** via Drizzle — possible but adds complexity. Probably not worth it for v1.

I'd actually nudge you toward **SQLite as the dev default and Postgres as the production recommendation** if you want the lowest possible barrier to "try it locally." But pure Postgres is simpler. Your call.

**ORM / DB layer**

- **Drizzle** *(my default)* — SQL-first, lightweight, TS types are excellent, easy migrations, easy for contributors to read.
- **Prisma** — better tooling (Studio is great), bigger community, but heavyweight runtime, slower cold starts on serverless, schema-first DSL is a love-it-or-hate-it.
- **Kysely** — pure query builder, no schema management. Very clean but you write more code.
- **Raw SQL + a tiny migrator** — most "hackable" for contributors but more boilerplate.

Drizzle is the modern sweet spot. Prisma is the safe boring choice. I'd stay with Drizzle.

**Auth**

- **Auth.js (NextAuth v5)** *(my default)* — battle-tested, supports magic links, OAuth (GitHub, Google, etc.), credentials, and SSO providers. Self-host friendly. DB adapter for Drizzle exists.
- **Better Auth** — newer, TypeScript-first, much nicer DX than Auth.js, growing fast, framework-agnostic. Real contender. Slightly less mature.
- **Lucia** — was the popular minimal alternative, but the maintainer is sunsetting it in favor of guides. Skip.
- **Roll your own with `@oslojs/*`** — Lucia's author now recommends this. Total control, more code to maintain. Only worth it if you have unusual requirements.
- **Clerk / WorkOS / Stack Auth** — hosted, great DX, but violate the self-host-first principle. Out.

I'd actually consider **Better Auth** here over Auth.js — the DX is materially better, the magic-link + OAuth + SSO story is cleaner, and for a greenfield project there's no migration cost. It's the choice I'd make today. Auth.js is the safer choice if you want maximum stability.

**Styling / components**

- **Tailwind + shadcn/ui** *(my default, already in place)* — keep.
- **Panda CSS / vanilla-extract** — more performant runtime, type-safe. Niche.
- **Mantine / MUI / Chakra** — full component libraries. You'd throw away your existing design tokens. Not worth it.

Stay with what you have.

**State / data fetching**

- **React Server Components + Server Actions** *(default in App Router)* — simplest for CRUD-heavy app. Most code runs on the server.
- **TanStack Query** on top — when you want client-side caching, optimistic updates, real-time-ish UI. You'll want this for the progress-tracking UX.
- **tRPC** — typed RPC, great DX, but somewhat redundant when Server Actions exist now.

Recommend: Server Actions for mutations, TanStack Query for any list/grid that needs optimistic updates (progress toggles especially).

**Editor (deferred to v2 but worth knowing now)**

- **Tiptap** — most popular, most extensible, ProseMirror-based, what Linear / many tools use.
- **BlockNote** — Notion-style out of the box, built on Tiptap, faster to ship a "Notion clone."
- **Lexical** — Meta's editor, used by Notion itself now. More flexible but lower-level.
- **Plate** — Slate-based, also Notion-style.

For v1 (markdown + preview), use `react-markdown` + a small custom block-shortcode parser. For v2, **BlockNote** is the fastest path to the look you want.

**Self-host packaging**

- **Docker + docker-compose** *(default)* — universal, what every self-hoster expects.
- **Single binary (Next standalone + embedded SQLite)** — best DX if you go the SQLite route. One executable, run it.
- **Helm chart** — k8s users. Nice-to-have, not v1.

---

**My honest stack pick if I were starting today:**

Next.js App Router (keep) · Drizzle ORM · **Postgres in prod, SQLite optionally for dev** · **Better Auth** · Tailwind + shadcn + your tokens · Server Actions + TanStack Query · `react-markdown` + custom shortcodes in v1, BlockNote in v2 · Docker Compose for self-host · zero telemetry.

The two real swaps vs my earlier default: **Better Auth over Auth.js**, and **optional SQLite for dev simplicity**. Both improve the contributor + self-hoster experience, which is the whole point of going OSS.

Want to take that stack, or swap any of it?

## User

Let's go with that stack.
