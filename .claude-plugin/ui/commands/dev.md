---
description: Enter the build loop — code, review, iterate
arguments:
  - name: component
    description: Start from a specific component (optional, picks next planned)
    required: false
---

# /treble:dev — Build Loop

You are Treble's build router. Your job is to set up a solid project foundation, determine the target stack, and hand off to the correct build skill.

## Step 0: Project Setup (FIRST PRIORITY)

Before writing any components, ensure the project is a well-organized, **runnable** repository. If `package.json` already exists and the dev server starts, skip to "Determine the target".

### 0a. Determine the target stack

Check in this order:

1. `.treble/analysis.json` → `metadata.target` field (if already set)
2. `package.json` with `next` dependency → **Next.js** (shadcn target)
3. `package.json` with `astro` dependency → **Astro** (shadcn target)
4. `package.json` with `react` (but no framework) → **Vite + React** (shadcn target)
5. `style.css` containing `Theme Name:` or `functions.php` present → **wordpress** target
6. If unclear, **ask the user**: "Which framework do you want? Next.js (recommended for apps), Astro (recommended for content sites), or WordPress?"

For **shadcn** targets, also ask: **Next.js or Astro?**
- **Next.js** — best for apps, dynamic content, API routes, SSR
- **Astro** — best for content/marketing sites, static-first, islands architecture

### 0b. Scaffold or verify the project

If starting fresh (no `package.json`):

**Next.js:**
```bash
npx create-next-app@latest . --typescript --tailwind --app --src-dir
npx shadcn@latest init
```

**Astro:**
```bash
npm create astro@latest . -- --template basics --typescript strict
npx astro add react tailwind
npx shadcn@latest init
```

**WordPress:** existing theme root is fine, skip scaffold.

**Verify it runs** — `npm run dev` must start without errors. Fix any issues before moving on.

If `package.json` exists, verify: `npm install && npm run dev` works. If it doesn't, fix it first.

### 0c. Project structure

Set up the feature-based directory structure (see `skills/dev-shadcn.md` for full rules):

```
src/
├── components/
│   ├── ui/              # shadcn primitives ONLY (managed by shadcn CLI)
│   ├── common/          # truly reusable across 2+ features (Logo, SocialLinks)
│   └── layout/          # page shells (Header, Footer, PageLayout, SectionContainer)
├── features/
│   └── [feature-name]/  # one per page/domain area
│       ├── components/  # feature-specific components
│       └── feature.tsx  # main export — mounted in pages/routes
├── lib/                 # utilities, helpers, cn()
└── app/ or pages/       # thin route files that mount features
public/
├── images/              # extracted Figma images
└── fonts/               # local font files (if any)
```

**Rule:** If you're about to write a file to `src/components/`, stop and ask: "Is this used by 2+ features?" If not, it belongs in `src/features/{name}/components/`.

### 0d. Testing setup (if appropriate)

Add a basic test runner. Skip for simple landing pages — add for apps with logic, forms, or interactivity.

```bash
npm install -D vitest @testing-library/react @testing-library/jest-dom jsdom
```

Add to `vite.config.ts` (or `vitest.config.ts` for Next.js/Astro):
```ts
test: {
  environment: 'jsdom',
  setupFiles: './src/test/setup.ts',
}
```

Create `src/test/setup.ts`:
```ts
import '@testing-library/jest-dom'
```

Add `"test": "vitest"` to `package.json` scripts. Run `npm test` to verify.

### 0e. Database / backend services (if needed)

If the project needs a database (CMS, auth, etc.), use Docker so the repo is self-contained:

```yaml
# docker-compose.yml
services:
  db:
    image: postgres:16-alpine
    ports: ["5432:5432"]
    environment:
      POSTGRES_DB: app
      POSTGRES_USER: app
      POSTGRES_PASSWORD: app
    volumes:
      - pgdata:/var/lib/postgresql/data

volumes:
  pgdata:
```

Add to `package.json` scripts: `"db:up": "docker compose up -d"`, `"db:down": "docker compose down"`.

For simpler needs (Payload CMS, small apps), prefer **SQLite** — no Docker required.

### 0f. Git hygiene

```bash
git init  # if not already a repo
```

Ensure `.gitignore` covers: `node_modules/`, `dist/`, `.env.local`, `.treble-tmp/`, `.next/` (Next.js), `.astro/` (Astro).

**Commit the scaffold:** `git add -A && git commit -m "chore: initial project setup"`

This is your clean baseline. Every component build after this gets its own commit.

---

## Hand off

Once the project is set up and runnable, hand off to the correct build skill from the plugin's `skills/` directory:

- **shadcn** (Next.js or Astro) → read and execute `skills/dev-shadcn.md`
- **wordpress** → read and execute `skills/dev-basecoat-wp.md`

Pass through any arguments the user provided (e.g. component name).
