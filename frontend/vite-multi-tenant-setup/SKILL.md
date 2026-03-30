---
name: vite-multi-tenant-setup
description: Use when converting a single-tenant Vite + React app to multi-tenant architecture, or when adding a new tenant — each requiring isolated Tailwind themes, Vite configs, AppConfig constants, and npm scripts
---

# Vite Multi-Tenant Setup

## Overview

Each tenant gets its own Vite config (extending a shared base via `mergeConfig`), its own Tailwind v4 CSS theme, its own `AppConfig` constant, and its own `dev`/`build`/`preview` npm scripts. Tenant selection uses `import.meta.env.VITE_CLIENT` loaded via Vite's `--mode` mechanism — no shell env vars, no `cross-env`, works on Windows.

Start with a tenant called `simple-tenant`; clone the pattern for every additional tenant.

## Directory Structure

```
project-root/
├── vite.config.base.ts              ← factory function, never imported directly by Vite
├── vite.config.ts                   ← untenanted fallback (npm run dev)
├── .env.simple-tenant               ← VITE_CLIENT=simple-tenant
├── src/
│   ├── main.tsx                     ← imports '@tenant-css' alias
│   ├── config/
│   │   ├── types.ts                 ← AppConfig interface
│   │   ├── index.ts                 ← reads VITE_CLIENT, exports active config
│   │   └── simple-tenant.ts        ← AppConfig constant for this tenant
└── tenants/
    └── simple-tenant/
        ├── vite.config.ts           ← merges base config + tenant overrides
        └── theme.css                ← @import "tailwindcss" + @theme { ... }
```

## Installation

**Do this before starting the dev server — skipping it causes `ERR_MODULE_NOT_FOUND`.**

```bash
npm install -D tailwindcss @tailwindcss/vite
# If TypeScript complains about 'path' module:
npm install -D @types/node
```

No PostCSS config needed — `@tailwindcss/vite` handles everything.

## AppConfig Interface

```ts
// src/config/types.ts
export interface AppConfig {
  appName: string
  // Add fields here as needed
}
```

Keep `AppConfig` in its own file to avoid circular imports: `simple-tenant.ts` imports `AppConfig` from `types.ts`, and `index.ts` imports both.

## Per-Tenant Config Constant

```ts
// src/config/simple-tenant.ts
import type { AppConfig } from './types'

export const config: AppConfig = {
  appName: 'Simple Tenant',
}
```

## Config Index (active tenant selection)

```ts
// src/config/index.ts
import type { AppConfig } from './types'
import { config as simpleTenant } from './simple-tenant'

const configs = {
  'simple-tenant': simpleTenant,
} as const

export type ClientKey = keyof typeof configs
// → 'simple-tenant'  (grows automatically as you add entries)

export const client = (import.meta.env.VITE_CLIENT ?? 'simple-tenant') as ClientKey
export const config: AppConfig = configs[client]
export type { AppConfig }
```

`ClientKey` is derived from `configs` keys — no manual union to maintain when adding tenants.

## .env File per Tenant

```bash
# .env.simple-tenant  (at project root)
VITE_CLIENT=simple-tenant
```

Vite loads this automatically when `--mode simple-tenant` is passed to any `vite` command.

## Base Config (factory function)

```ts
// vite.config.base.ts
import { defineConfig, type UserConfig } from 'vite'
import react from '@vitejs/plugin-react'
import tailwindcss from '@tailwindcss/vite'
import path from 'path'

export function createBaseConfig(tenantId: string): UserConfig {
  return defineConfig({
    plugins: [react(), tailwindcss()],
    resolve: {
      alias: {
        '@': path.resolve(__dirname, './src'),
        // Resolves to THIS tenant's theme.css at build time
        '@tenant-css': path.resolve(__dirname, `./tenants/${tenantId}/theme.css`),
      },
    },
    build: {
      outDir: `dist/${tenantId}`,
      emptyOutDir: true,
    },
  }) as UserConfig
}
```

Note: no `define` needed — `VITE_CLIENT` comes from the `.env.[tenant]` file loaded by `--mode`.

## Per-Tenant Vite Config

```ts
// tenants/simple-tenant/vite.config.ts
import { mergeConfig, defineConfig } from 'vite'
import { createBaseConfig } from '../../vite.config.base'

export default mergeConfig(
  createBaseConfig('simple-tenant'),
  defineConfig({
    server: { port: 5174 },   // avoid port collision with default dev server
  })
)
```

## Per-Tenant Tailwind Theme (v4)

```css
/* tenants/simple-tenant/theme.css */
@import "tailwindcss";

@theme {
  --color-primary-500: #0ea5e9;
  --color-primary-700: #0369a1;
  --color-accent-500:  #f59e0b;
  --color-surface:     #ffffff;
  --font-sans:         "Inter", ui-sans-serif, system-ui;
  --radius-brand:      0.5rem;
}
```

These become Tailwind utility classes: `text-primary-500`, `bg-accent-500`, etc.

**Coexistence with Bootstrap / other CSS resets:** Tailwind v4's `@import "tailwindcss"` injects its own preflight reset, which will conflict with Bootstrap. Use this instead:

```css
/* Bootstrap-safe variant */
@import "tailwindcss/theme";
@import "tailwindcss/utilities";

@theme { ... }
```

**Bootstrap variable bridging — REQUIRED when the app uses Bootstrap components:**

`@theme` tokens (`--color-primary-500`, etc.) are Tailwind's namespace. Bootstrap components read `--bs-primary`, `--bs-primary-rgb`, etc. — they are entirely separate. Without an explicit bridge, `@theme` colors have **zero visual effect** on any Bootstrap button, badge, link, or text.

Always add a `:root {}` block that covers all three layers:

1. **App-level CSS tokens** — any `--color-primary` / `--color-primary-dark` etc. defined in the app's `index.css` that components reference via `var(--color-primary)`
2. **Bootstrap component variables** — `--bs-primary`, `--bs-primary-rgb`, etc.
3. **Link colors** — `--bs-link-color`, `--bs-link-hover-color`

```css
@theme {
  --color-primary-500: #c20016;
  --color-primary-700: #8b0010;
  --color-primary-300: #e8334a;
  /* ... */
}

:root {
  /* 1. Override app-level tokens (used by var(--color-primary) in index.css classes) */
  --color-primary:       #c20016;
  --color-primary-dark:  #8b0010;
  --color-primary-light: #e8334a;

  /* 2. Bridge to Bootstrap component variables */
  --bs-primary:          #c20016;
  --bs-primary-rgb:      194, 0, 22;   /* must be raw R, G, B — no var() here */
  --bs-link-color:       #c20016;
  --bs-link-hover-color: #8b0010;
}
```

Note: `--bs-primary-rgb` must be a raw `R, G, B` value (used inside `rgba()`), not a hex or `var()`.

**Which tokens to override:** Read `index.css` and note every `--color-*` variable defined in its `:root` block that is used in component classes. Include all of them in the tenant `:root` — any you omit will fall back to the app's default color.


## main.tsx CSS Import Order — CRITICAL

**`@tenant-css` must be the last CSS import.** CSS custom properties follow cascade order: the last `:root {}` declaration with the same variable name wins. If the app has a base `index.css` that defines color tokens (e.g. `--color-primary: #0024c1`), importing it after `@tenant-css` silently overwrites the tenant's values.

```tsx
// src/main.tsx — correct order
import './index.css'    // app base styles + default color tokens
import '@tenant-css'    // tenant overrides — MUST come last
import App from './App'
```

Never hardcode `import '../tenants/simple-tenant/theme.css'` — that bakes one tenant's styles into every build.

**Untenanted fallback (`npm run dev`):** The root `vite.config.ts` does not define `@tenant-css`. Add a fallback alias there pointing to a shared base CSS, or point it at the default tenant's theme.

**Note:** if `index.css` already contains `@import 'bootstrap/...'`, remove any separate Bootstrap import from `main.tsx` to avoid loading it twice.

## package.json Scripts

```json
{
  "scripts": {
    "dev":                   "vite",
    "build":                 "vite build",
    "dev:simple-tenant":     "vite --mode simple-tenant --config tenants/simple-tenant/vite.config.ts",
    "build:simple-tenant":   "vite build --mode simple-tenant --config tenants/simple-tenant/vite.config.ts",
    "preview:simple-tenant": "vite preview --mode simple-tenant --config tenants/simple-tenant/vite.config.ts"
  }
}
```

`--mode simple-tenant` loads `.env.simple-tenant`, which sets `VITE_CLIENT=simple-tenant`. Works on Windows without `cross-env`.

## TypeScript: type VITE_CLIENT and __dirname

Add to `src/vite-env.d.ts` to get autocomplete and avoid strict-mode complaints:

```ts
/// <reference types="vite/client" />
interface ImportMetaEnv {
  readonly VITE_CLIENT: string
}
interface ImportMeta {
  readonly env: ImportMetaEnv
}
```

If `__dirname is not defined` at runtime (ES module context on some Node versions), add to `vite.config.base.ts`:

```ts
import { fileURLToPath } from 'url'
import { dirname } from 'path'
const __dirname = dirname(fileURLToPath(import.meta.url))
```

## TypeScript Config

Add `vite.config.base.ts` and tenant configs to `tsconfig.node.json`:

```json
{
  "include": [
    "vite.config.ts",
    "vite.config.base.ts",
    "tenants/*/vite.config.ts"
  ]
}
```

## Preserving Existing Vite Config Settings

Before replacing `vite.config.ts`, note any project-specific settings (e.g. `base: './'`, `optimizeDeps`, `server.proxy`) and carry them into `createBaseConfig`. The template only shows the minimum.

## Adding a New Tenant

1. Create `src/config/[new-tenant].ts` with `AppConfig` constant
2. Add `'[new-tenant]': newTenantConfig` entry to `configs` in `src/config/index.ts`
3. Create `.env.[new-tenant]` with `VITE_CLIENT=[new-tenant]`
4. Copy `tenants/simple-tenant/` → `tenants/[new-tenant]/`, update theme + port
5. Add three scripts to `package.json` with `--mode [new-tenant]`

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| `ERR_MODULE_NOT_FOUND: Cannot find package '@tailwindcss/vite'` | Run `npm install -D tailwindcss @tailwindcss/vite` before starting the dev server |
| Object-spreading base config instead of `mergeConfig` | `mergeConfig` understands arrays (plugins); spread silently drops them |
| Root-level `postcss.config.js` loading Tailwind | Remove it — `@tailwindcss/vite` conflicts with it |
| Hardcoded tenant CSS import in `main.tsx` | Use `import '@tenant-css'` alias resolved by `createBaseConfig` |
| Inline env var in scripts: `VITE_CLIENT=x vite` | Fails on Windows — use `--mode x` + `.env.x` file instead |
| `AppConfig` in `index.ts` imported by `simple-tenant.ts` | Circular import — put `AppConfig` in `types.ts` |
| `ClientKey` as a manually maintained union | Derive it: `export type ClientKey = keyof typeof configs` |
| Tailwind CSS `content` paths in v3 style | Unnecessary in v4 — `@tailwindcss/vite` auto-detects |
| Two dev servers on the same port | Add `server: { port: 5174+ }` override in each tenant Vite config |
| `VITE_CLIENT` not typed in `vite-env.d.ts` | Add `ImportMetaEnv` augmentation as shown above |
| `__dirname is not defined` in base config | Add `fileURLToPath` / `dirname` boilerplate as shown above |
| `vite.config.base.ts` not in `tsconfig.node.json` | Widen `include` as shown above |
| Dropping existing `base`, `optimizeDeps`, `server.proxy` | Carry them into `createBaseConfig` |
| `@import "tailwindcss"` conflicts with Bootstrap/other resets | Use `@import "tailwindcss/theme"` + `@import "tailwindcss/utilities"` |
| Tenant colors defined in `@theme` but not visible in Bootstrap UI | `@theme` tokens don't affect Bootstrap — add `:root { --bs-primary: ...; --bs-primary-rgb: R, G, B; }` bridge |
| `@tenant-css` imported before `index.css` in `main.tsx` | `index.css` `:root` vars overwrite the tenant theme — `@tenant-css` must be the **last** CSS import |
| Tenant `:root` only has `--bs-primary` but components stay the wrong color | Also override app-level tokens (`--color-primary`, `--color-primary-dark`, etc.) used by `var()` in `index.css` classes |
