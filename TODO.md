# SleepyLeo — Implementation & Fix Tracker

> Audit date: 2026-02-19
> Status legend: `[ ]` pending · `[x]` done

---

## sleepyleo-website

### Add — Missing Sections (High Impact for HR) · CMS-driven via DB

All three sections are content-managed: data lives in PostgreSQL, editable from the admin panel.
Follow the same pattern as `Project`: Prisma model → server action → server component → UI.

---

#### 1. About / Bio section — CMS

**Schema** (`sleepyleo-website/prisma/schema.prisma` — also mirror in admin schema)

```prisma
model Profile {
  id              String   @id @default(cuid())
  // Bio
  name            String
  bio             String   // markdown supported
  background      String   // 1-2 sentence origin story
  // Card highlights
  education       String   // e.g. "KMUTT — School of IT"
  location        String   // e.g. "Bangkok, Thailand"
  focus           String   // e.g. "Full-stack & DevOps"
  fuel            String   // e.g. "Coffee & Cats"
  // Timeline entries stored as JSON
  // e.g. [{ year: "2022", event: "Started at KMUTT" }, ...]
  timeline        Json     @default("[]")
  // Availability
  availableForHire Boolean @default(true)
  availableLabel  String   @default("Open to opportunities")
  updatedAt       DateTime @updatedAt
}
```

**Server action** (`sleepyleo-website/lib/actions.ts`)
- `getProfile()` → `prisma.profile.findFirst()` — returns the single profile row
- Called on the homepage alongside `getProjects()` and `getTotalCommits()`

**API routes** (`sleepyleo-admin/app/api/profile/route.ts`)
- `GET` — return current profile
- `PUT` — upsert the single profile row (admin-only, guarded by `isAuthenticated()`)

**Admin UI** (`sleepyleo-admin/app/page.tsx` or new `sleepyleo-admin/app/profile/page.tsx`)
- Form fields for all `Profile` columns
- Timeline editor: list of `{ year, event }` entries with add/remove
- Toggle for `availableForHire`

**Website component** (`sleepyleo-website/components/sections/about-section.tsx`)
- `"use client"` — receives `profile: Profile` prop from server component
- Renders bio, card highlights, timeline, availability badge
- Design: zinc-950 bg, indigo/violet gradients, Framer Motion entrance animations

---

#### 2. Skills section — CMS

**Schema** (`prisma/schema.prisma`)

```prisma
model Skill {
  id           String @id @default(cuid())
  name         String
  category     String // "Frontend" | "Backend" | "DevOps" | "Tools"
  proficiency  String // "daily_driver" | "comfortable" | "learning"
  projectUsage String @default("") // tooltip text — real project context
  order        Int    @default(0)  // display order within category
}
```

**Server action** (`lib/actions.ts`)
- `getSkills()` → `prisma.skill.findMany({ orderBy: [{ category: "asc" }, { order: "asc" }] })`
- Returns all skills; component groups client-side by `category`

**API routes** (`sleepyleo-admin/app/api/skills/route.ts`)
- `GET` — return all skills
- `POST` — create skill (body: `{ name, category, proficiency, projectUsage, order }`)
- `PATCH` — update skill by `?id=`
- `DELETE` — delete skill by `?id=`

**Admin UI** (`sleepyleo-admin/app/skills/page.tsx`)
- Table grouped by category (Frontend / Backend / DevOps / Tools)
- Inline edit: name, proficiency dropdown, projectUsage textarea
- Drag-to-reorder within category (updates `order` field)
- Quick-add row at the bottom of each category

**Website component** (`sleepyleo-website/components/sections/skills-section.tsx`)
- `"use client"` — receives `skills: Skill[]` prop
- Group by `category`, NO percentage bars
- Proficiency badge: `Daily Driver` / `Comfortable` / `Learning`
- Hover tooltip shows `projectUsage` text

---

#### 3. Contact section — CMS

Contact links are part of the `Profile` model (add these fields to the schema above):

```prisma
// Add to Profile model:
email       String  @default("")
github      String  @default("SleepyLe0")       // username only
linkedin    String  @default("kundids-khawmeesri-90814526a") // profile ID only
ctaCopy     String  @default("Let's build something.")
```

**Server action** — no new action needed; reuse `getProfile()`

**Admin UI** — add Email, GitHub, LinkedIn, CTA Copy fields to the existing Profile editor

**Website component** (`sleepyleo-website/components/sections/contact-section.tsx`)
- `"use client"` — receives contact fields from `profile` prop
- Constructs full URLs from stored usernames: `mailto:`, `github.com/`, `linkedin.com/in/`
- Availability badge synced with `profile.availableForHire`
- Short punchy CTA from `profile.ctaCopy`

---

#### Homepage wiring (`app/(website)/page.tsx`)

```ts
// Fetch in parallel with existing calls
const [result, totalCommits, profile, skills] = await Promise.all([
  getProjects(),
  getTotalCommits(),
  getProfile(),
  getSkills(),
]);
```

Render order: `Hero → Projects → About → Skills → Contact → DogBreedQuiz`

---

#### DB migration steps

1. Add `Profile` and `Skill` models to both `prisma/schema.prisma` files (website + admin share the same DB)
2. Run `npx prisma migrate dev --name add_profile_and_skills`
3. Seed initial data via admin panel or a `prisma/seed.ts` script
4. Re-generate client: `npx prisma generate`

### Add — Project Detail Pages

- [ ] **`/projects/[slug]` route** (`app/(website)/projects/[slug]/page.tsx`)
  - DB already has `slug` field — just needs the route
  - Show: name, description, tech stack badges, memeUrl GIF, repo link, live link, stars/forks
  - 404 redirect if slug not found or `visible = false`
  - Add `generateStaticParams` for static generation at build time

### Update — Existing Files

- [ ] **Navbar** (`components/navbar.tsx`)
  - Add nav items: `About` (`#about`), `Skills` (`#skills`), `Contact` (`#contact`)
  - Add to `navItems` array and update `sections` list in scroll observer

- [ ] **Homepage** (`app/(website)/page.tsx`)
  - Import and render: `<AboutSection />`, `<SkillsSection />`, `<ContactSection />`
  - Order: Hero → Projects → About → Skills → Contact → DogBreedQuiz

- [ ] **Sitemap** (`app/sitemap.ts`)
  - Currently only indexes `/`
  - Add `/projects` page
  - Add all visible project slugs: fetch from DB and map to `/projects/[slug]`

---

## sleepyleo-admin

### Fix — Security Issues

- [ ] **Weak session token** (`lib/auth.ts`)
  - Current: djb2 hash variant — collisions possible, not verifiable
  - Fix: Replace with `crypto.createHmac("sha256", AUTH_SECRET)` + `timingSafeEqual` for verification
  - Token format: `<random_hex>.<timestamp>.<hmac_signature>`
  - Update `isAuthenticated()` to verify the HMAC, not just check cookie existence

- [ ] **No auth check on `/api/exec`** (`app/api/exec/route.ts`)
  - Currently: anyone who knows the URL can execute commands on the VM
  - Fix: Call `isAuthenticated()` at the top of the handler, return 401 if false

- [ ] **No auth check on `/api/chat`** (`app/api/chat/route.ts`)
  - Same issue as exec — verify authentication before processing
  - Fix: Add `isAuthenticated()` guard at the top of the POST handler

- [ ] **Weak command blocklist** (`lib/actions.ts` → `executeCommand`)
  - Current: simple string includes check — bypassable via shell substitution, aliases, etc.
  - Fix: Extend blocklist with more patterns:
    - `eval`, `exec`, `base64`, `python -c`, `perl -e`, `bash -c`
    - `> /etc/`, `>> /etc/`, `/etc/passwd`, `/etc/shadow`
    - `shutdown`, `reboot`, `halt`, `poweroff`
    - `kill -9 1`, `killall`
  - Add output size cap: truncate stdout/stderr at 50 KB

- [ ] **No rate limiting on auth endpoint** (`app/api/auth/route.ts`)
  - Fix: In-memory Map tracking failed attempts per IP
  - Block IP after 10 failed attempts within 15 minutes
  - Return 429 with retry-after header when blocked

- [ ] **`AUTH_SECRET` default value** (`lib/auth.ts`)
  - Current default: `"default-secret-change-me"` — trivially guessable
  - Fix: Add startup validation — throw/warn loudly if `AUTH_SECRET` is missing or equals the default in production

---

## Both Projects

- [ ] **`prefers-reduced-motion`** (`components/ui/particle-field.tsx` in website)
  - Canvas particle animation ignores `prefers-reduced-motion` media query
  - Fix: Check `window.matchMedia("(prefers-reduced-motion: reduce)")` and skip animation if true
