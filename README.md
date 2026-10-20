# Tech Stacks — Recommended Stack for `zainpreneur-app`

Based on the brand specification (`BRAND.md`), domain-driven architecture (server/domain pure rules + server/repositories SQL hydration), and the "editorial / printed page not dashboard" aesthetic, here's the complete, modern stack.

## 🌐 Frontend

| Layer | Technology | Reason |
|---|---|---|
| **UI Framework** | **React 19** + **Vite 7** | Already proven; Vite HMR is instant; React 19 gives new APIs (concurrent roots, server components ready) |
| **Styling** | **Tailwind CSS 4** | Matches the brand's token system exactly; config-driven (colors, spacing, radii, fonts); no CSS files to drift from BRAND.md |
| **Components** | **Custom / htmx + Alpine.js** (or React components) | The brand doc praises the "printed page rather than glossy dashboard" aesthetic; htmx + Alpine keeps it lightweight and server-centric |
| **State / Data** | **TanStack Query** + **tRPC** (optional) | Caching, syncing, background refetches; tRPC gives end-to-end type safety if you add Zod schemas |

## 🛠️ Backend

| Layer | Technology | Reason |
|---|---|---|
| **Web Framework** | **Hono** (v4) | Lightweight, standards-compliant (Web Standard APIs), excellent TypeScript support, built-in OpenAPI/zod validation |
| **Language / Types** | **TypeScript 6** | End-to-end safety; BRAND.md tokens, Zod schemas, and DB types all stay in sync |
| **ORM / DB Layer** | **Drizzle ORM** | Zero-runtime, type-safe SQL, auto-inferred schemas from SQLite. Keeps the `server/domain` pure while giving compile-time safety. |
| **Database** | **better-sqlite3** (via Drizzle) | SQLite is fine for the business portfolio use case; Drizzle makes switching to Postgres painless later. |
| **Validation** | **Zod** | Schemas power: (1) API request validation, (2) tRPC type inference, (3) form parsing, (4) BRAND.md token enforcement. |

## 🔗 Client-Server Integration

| Pattern | Technology | Reason |
|---|---|---|
| **API Layer** | **tRPC** (optional) **or** Hono + Zod | Same Zod schemas validate API inputs AND generate React component types. Eliminates the "report and screen disagree" class of bugs. |
| **Form Handling** | **React Hook Form** + **Zod** | Standard, battle-tested form library that works beautifully with Zod. |
| **DOM Updates** | **htmx** + **Alpine.js** (if keeping current approach) **or** React state | Keeps the editorial, print-like UI without the overhead of a heavy UI library. |

## 🗄️ Database & Schema

| Concern | Solution |
|---|---|
| **Schema as code** | **Drizzle schema** (`db/schema.ts`) — one source of truth for tables, columns, and types |
| **Migrations** | **Drizzle Kit** (`drizzle-kit generate/migrate`) + CI-applied migrations |
| **Seed data** | Keep `seed.py`-style seed, or add Drizzle `seed()` calls in `db/init.ts` |

## 🧪 Testing & Quality

| Layer | Technology | Reason |
|---|---|---|
| **Unit / API tests** | **Vitest** + **happy-dom** | Same as current; keep the `npm run test` suite |
| **E2E / browser flows** | **Playwright** | Exercise the htmx + Alpine interactions, form imports/exports, and the brand token enforcement. Runs in CI. |
| **Linting** | **oxlint** (already configured) | Extend with custom rules: <br>• Disallow hex colors not in BRAND.md token list <br>• Enforce Zod schema usage on all API routes <br>• Validate Tailwind class names against `tailwind.config.ts` |
| **Type checking** | **`tsc --noEmit`** | Keep as the gate; add `noImplicitAny`, `strictNullChecks` if not already on. |
| **Coverage** | **Istanbul** (via Vitest) | Aim for ~80% on business logic, 60%+ on API routes. |

## 📦 DevOps & Build

| Concern | Solution |
|---|---|
| **Build** | `vite build` → static `dist/` served by Hono in production |
| **Environment** | `.env` variables for `DATABASE_URL`, `PORT`, `NODE_ENV` |
| **Process manager** | `pm2` or native `Node --watch` in dev; systemd/pm2 in production |
| **Containerization** | Optional: Dockerfile `node:22-alpine` → `COPY . /app` → `CMD ["node", "dist/index.js"]` |
| **CI/CD** | GitHub Actions: <br>• `npm install` <br>• `npm run lint` <br>• `npm run test` <br>• `npm run verify:all` <br>• Deploy static `dist/` to your hosting |

## 🎨 Brand & Token Enforcement

This is the **differentiator** for `zainpreneur-app`:

1. **Zod token schema** — a single source of truth for all BRAND.md tokens (`--bg`, `--gold`, `--teal`, `--radius`, `--fs-*`, etc.)
2. **oxlint custom rule** — flags any hex color not in the token list; ensures new components respect the brand
3. **Tailwind config** — pull all color/sizing tokens from the Zod schema; `theme: { extend: { colors: { ... }, spacing: { ... } } }` auto-generated from the schema
4. **Test assertions** — Vitest/Playwright tests that check rendered CSS `var(--gold)` usage, not hard-coded `#f5a524`

## 📋 Full Stack Diagram (Top to Bottom)

```
┌─────────────────────────────────────────────────────────────┐
│            PRODUCTION / DEVOPS LAYER                       │
│  + GitHub Actions CI/CD                                     │
│  + Docker (optional)                                        │
│  + Static `dist/` served by Hono + Node                     │
└─────────────────────────────────────────────────────────────┘
                                         │
┌─────────────────────────────────────────────────────────────┐
│              SERVER-SIDE LAYER                              │
│  + Hono (TypeScript)                                        │
│  + Drizzle ORM + SQLite (better-sqlite3)                    │
│  + Zod schemas (validation + type inference)                │
│  + server/domain (pure business rules)                      │
│  + server/repositories (SQL hydration)                      │
└─────────────────────────────────────────────────────────────┘
                                         │
┌─────────────────────────────────────────────────────────────┐
│              CLIENT-SIDE LAYER                              │
│  + React 19 + Vite 7                                        │
│  + Tailwind CSS 4 (config from Zod tokens)                  │
│  + TanStack Query (caching/sync)                           │
│  + optional tRPC (type-safe API)                           │
│  + htmx + Alpine.js (if keeping light DOM approach)         │
│  + React Hook Form + Zod (forms)                           │
└─────────────────────────────────────────────────────────────┘
                                         │
┌─────────────────────────────────────────────────────────────┐
│              BRAND & DESIGN SYSTEM                          │
│  + BRAND.md (tokens, palette, spacing, typography)          │
│  + Tailwind config derived from Zod schema                  │
│  + oxlint custom rules (no rogue hex, token compliance)     │
│  + Playwright/Vitest brand-enforcement tests               │
└─────────────────────────────────────────────────────────────┘

## 🛫 Migration Path (Incremental — No Breaking Changes)

You can adopt the new stack piece by piece:

1. **Add Zod schemas** → already have `zod 4.1.5`; use for API validation + tRPC types
2. **Add Drizzle ORM** → `npm i drizzle-orm better-sqlite3`; define schemas, keep existing `better-sqlite3` queries running in parallel
3. **Add Hono** → `npm i hono`; replace Express router gradually; Hono works with same middleware pattern
4. **Add tRPC (optional)** → `npm i @trpc/server @trpc@5`; if you want end-to-end types without a separate codegen step
5. **Add oxlint rules** → extend `oxlint` config to enforce BRAND.md tokens and Zod schema usage
6. **Add Playwright tests** → already configured; write brand-enforcement tests

## ✅ Bottom Line

**Everything is supported** in the current environment (Node v22, npm 10, TypeScript 5.9, React 19, Tailwind CSS 4, Vitest, Playwright, oxlint, better-sqlite3).

You can incrementally adopt:
- **Hono** + **Drizzle ORM** for a type-safe, lightweight backend
- **Zod** + **tRPC** for API validation + React type safety  
- **oxlint** custom rules for BRAND.md token compliance
- Keep the **React 19 + Tailwind CSS 4 + Vite 7** frontend exactly as-is (it already matches the brand spec perfectly)

No need to rewrite the UI — the stack improvements focus on the server-client type bridge and database layer, which is where the "report and screen disagree" bug class lives.