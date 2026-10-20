# Tech Stack A — PWA + Brand-First Stack for `zainpreneur-app`

This is the first in a series of stack definition files for the Zainpreneur project. File `a` focuses on the **PWA-enabled, brand-compliant frontend + type-safe backend** stack.

---

## 🌐 Frontend (React + Vite + Tailwind 4 + PWA)

| Layer | Technology | PWA Status |
|---|---|---|
| **UI Framework** | **React 19** + **Vite 7** | — |
| **Styling** | **Tailwind CSS 4** | Token-driven from BRAND.md; no CSS drift |
| **PWA Plugin** | **`vite-plugin-pwa`** (Workbox) | ✅ Enabled in `vite.config.ts` |
| **Service Worker** | **`injectManifest` strategy** | Custom `src/sw.ts` for offline queue draining (not Workbox's cache-only SW) |
| **Web App Manifest** | Configured with theme_color, background_color, display, icons | ✅ Generated from `vite-plugin-pwa` config |
| **State Management** | **TanStack Query** + **Zod** | — |
| **DOM Updates** | **htmx** + **Alpine.js** (lightweight, server-centric) | Matches "printed page, not dashboard" aesthetic |

### PWA Configuration (vite.config.ts)

```ts
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';
import tailwindcss from '@tailwindcss/vite';
import { VitePWA } from 'vite-plugin-pwa';

export default defineConfig({
  plugins: [
    react(),
    tailwindcss(),
    VitePWA({
      strategies: 'injectManifest', // custom SW, not auto-generated
      srcDir: 'src',
      filename: 'sw.ts',
      registerType: 'autoUpdate',
      injectRegister: null,
      includeAssets: ['icons/apple-touch-icon-180.png'],
      manifest: {
        name: 'Zainpreneur Services',
        short_name: 'Zainpreneur',
        description:
          'Service catalogue, service requests and help desk tickets — online or off.',
        theme_color: '#484ad7',
        background_color: '#f8fafc',
        display: 'standalone',
        orientation: 'portrait-primary',
        start_url: '/',
        scope: '/',
        categories: ['business', 'productivity'],
        icons: [
          { src: 'icons/icon-192.png', sizes: '192x192', type: 'image/png', purpose: 'any' },
          { src: 'icons/icon-512.png', sizes: '512x512', type: 'image/png', purpose: 'any' },
          { src: 'icons/icon-512-maskable.png', sizes: '512x512', type: 'image/png', purpose: 'maskable' },
        ],
      },
      injectManifest: {
        globPatterns: ['**/*.{js,css,html,svg,png,ico,webmanifest}'],
      },
      workbox: {
        navigateFallback: 'index.html',
        cleanupOutdatedCaches: true,
        clientsClaim: true,
        skipWaiting: true,
      },
      devOptions: { enabled: true, type: 'module' },
    }),
  ],
  // ... rest of config (server proxy, build, etc.)
});
```

### Service Worker Template (src/sw.ts)

```ts
// Custom hand-written SW (injectManifest strategy)
// Why not Workbox-generated? Because Workbox's backgroundSync only replays fetch requests
// it was caching; our offline queue lives in IndexedDB as rows, and nothing would move them
// unless we explicitly drain them here.

import { clientsClaim, skipWaiting } from 'workbox-skypager';

clientsClaim();
skipWaiting();

// Optional: manually process IndexedDB offline queue on SW install/activate
self.addEventListener('activate', (event) => {
  event.waitUntil(
    // Example: purge old caches, re-process queued rows, etc.
    caches.keys().then((keys) => Promise.all(keys.map((k) => caches.delete(k))))
  );
});
```

---

## 🛠️ Backend (Hono + Drizzle ORM + Zod)

| Layer | Technology | Reason |
|---|---|---|
| **Web Framework** | **Hono** (v4) | Lightweight, Web Standard APIs, excellent TS support, built-in OpenAPI/zod |
| **Language** | **TypeScript 5.9** | End-to-end safety; BRAND.md tokens, Zod schemas, DB types stay in sync |
| **ORM / DB** | **Drizzle ORM** + **better-sqlite3** | Zero-runtime, type-safe SQL, auto-inferred schemas from SQLite |
| **Validation** | **Zod** | Schemas power: (1) API validation, (2) tRPC type inference, (3) form parsing, (4) BRAND.md token enforcement |
| **Pure Rules** | **server/domain** | Business rules kept pure, independent of DB/hydration layer |
| **Hydration** | **server/repositories** | SQL → domain objects; keeps domain testable without DB |

### Example Drizzle Schema (`db/schema.ts`)

```ts
import { sqliteTable, text, integer, real } from 'drizzle-orm/sqlite-core';
import { db } from './index';

export const businesses = sqliteTable('businesses', {
  id: integer('id').primaryKey({ autoIncrement: true }),
  name: text('name').notNull(),
  legal_name: text('legal_name'),
  industry: text('industry'),
  status: text('status').notNull().default('active'),
  equity_total: real('equity_total').default(100),
  created_at: text('created_at').notNull(),
  updated_at: text('updated_at').notNull(),
});

export type Business = typeof businesses.$inferSelect;
export type NewBusiness = typeof businesses.$inferInsert;
```

### Example Hono Route (`server/index.ts`)

```ts
import { Hono } from 'hono';
import { db } from '../db';
import { businesses } from '../db/schema';
import { z } from 'zod';

const app = new Hono();

// Zod schema for API validation + type inference
const businessSchema = z.object({
  name: z.string().min(1),
  industry: z.string().optional(),
  status: z.enum(['active', 'prospect', 'exited']).optional(),
});

app.get('/api/businesses', async (c) => {
  const data = await db.select().from(businesses);
  return c.json(data);
});

app.post('/api/businesses', async (c) => {
  const body = await c.req.json();
  const result = businessSchema.safeParse(body);
  if (!result.success) {
    return c.json({ error: result.error.format }, 400);
  }
  const { name, industry, status } = result.data;
  const newBiz = await db.insert(businesses).values({
    name,
    industry,
    status: status || 'active',
    created_at: new Date().toISOString(),
    updated_at: new Date().toISOString(),
  }).returning();
  return c.json(newBiz[0]);
});

export { app };
```

---

## 🔗 Client-Server Integration

| Pattern | Technology | Reason |
|---|---|---|
| **API Layer** | **Hono + Zod** (no separate tRPC codegen needed) | Same Zod schema validates API inputs; types inferred at compile time |
| **Forms** | **React Hook Form** + **Zod** | Standard, battle-tested; errors stay in sync between browser and API |
| **Data Fetching** | **TanStack Query** | Caching, background refetches, stale-while-revalidate; types come from Zod |
| **DOM Updates** | **htmx** + **Alpine.js** | Keeps the editorial, print-like UI; each HTMX response just needs minimal HTML |
| **Type Safety** | **Zod → React Component Props** | `type Business = z.infer<typeof businessSchema>` — one source of truth |

---

## 🎨 Brand & Token Enforcement (BRAND.md)

This is the **differentiator** for `zainpreneur-app`:

1. **Zod Token Schema** — a single source of truth for all BRAND.md tokens:
   ```ts
   const brandTokens = z.object({
     bg: z.string().regex(/^#?[0-9a-f]{6}$/i),
     gold: z.string().regex(/^#?[0-9a-f]{6}$/i),
     teal: z.string().regex(/^#?[0-9a-f]{6}$/i),
     radius: z.number().int().min(0),
     // ... all other `--fs-*`, `--sp-*`, `--radius-*` tokens
   });
   ```
2. **oxlint Custom Rule** — flags any hex color not in the token list; ensures new components respect the brand
3. **Tailwind Config** — all `colors`, `spacing`, `borderRadius` pulled from the Zod schema; `theme: { extend: { ... } }` auto-generated
4. **Test Assertions** — Vitest/Playwright tests check rendered `var(--gold)` usage, not hard-coded `#f5a524`

### oxlint Rule Sketch (`.oxlintrc.json`)

```json
{
  "rules": {
    "no-hardcoded-colors": "error",
    "zod-schema-usage": "error"
  },
  "plugins": ["custom"],
  "plugins/custom": {
    "no-hardcoded-colors": {
      "meta": {
        "type": "problem",
        "docs": { "description": "Disallow hex colors not in BRAND.md token list" }
      },
      "create": (context) => {
        const allowedTokens = ['--gold', '--teal', '--bg', '--surface', '--ink', /* ... */];
        return {
          "Literal[node]": (node) => {
            if (node.value && typeof node.value === 'string' && node.value.startsWith('#')) {
              if (!allowedTokens.some((t) => node.value.includes(t))) {
                context.report({ message: `Hard-coded color not in BRAND.md tokens: ${node.value}` });
              }
            }
          }
        };
      }
    }
  }
}
```

---

## 📦 Migration Path (Incremental — No Breaking Changes)

| Step | Action | Risk |
|---|---|---|
| **1** | Add Zod schemas for all API inputs | Low — Zod already a dependency (`zod 4.1.5`) |
| **2** | Add Drizzle ORM + define schemas | Low — run in parallel with existing `better-sqlite3` queries |
| **3** | Replace Express router with Hono | Medium — Hono API is similar; keep same route paths |
| **4** | Add tRPC (optional) | Medium — adds codegen step, but gives compile-time API types on React side |
| **5** | Add oxlint BRAND.md rules | Low — static analysis; catches drift early |
| **6** | Add Playwright brand-enforcement tests | Low — CI gate; prevents "report and screen disagree" bugs |

---

## ✅ Bottom Line

**Everything is supported** in the current environment (Node v22, npm 10, TypeScript 5.9, React 19, Tailwind CSS 4, Vitest, Playwright, oxlint, better-sqlite3, Zod).

This file (`tech-stack-a`) serves as the **foundation**. Future files (`b`, `c`, `d`) will cover:

- `tech-stack-b` → Full database schema + migration scripts
- `tech-stack-c` → Complete component library + brand token mapping
- `tech-stack-d` → CI/CD pipeline + Docker configuration + performance budget

To be continued...