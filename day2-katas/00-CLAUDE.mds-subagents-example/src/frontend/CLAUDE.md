# Frontend — CLAUDE.md

Next.js 14 App Router with Tailwind CSS v4 and shadcn/ui components.

## Structure

```
src/frontend/
  app/                   # Next.js App Router pages and layouts
    (auth)/              # Route group: login, signup (no auth required)
    (dashboard)/         # Route group: main app (auth required)
    api/                 # Next.js API routes (BFF, thin proxies to backend)
    layout.tsx           # Root layout
    error.tsx            # Root error boundary
  components/
    ui/                  # shadcn/ui primitives (Button, Dialog, etc.) — don't modify
    composed/            # Project-specific composed components
    forms/               # Form components using react-hook-form + zod
    layouts/             # Layout shells (sidebar, nav, etc.)
  hooks/                 # Custom React hooks
  lib/
    api-client.ts        # Typed fetch wrapper for backend API
    utils.ts             # Generic utilities (cn(), formatDate, etc.)
  styles/
    globals.css          # Tailwind directives, CSS custom properties
  public/                # Static assets (favicon, og-images)
    assets/              # UI assets (illustrations, icons not in lucide)
```

## Patterns

**Data fetching**: Server Components fetch data. Client Components receive it as props.
Use `api-client.ts` for all backend calls — never raw `fetch`.

```tsx
// app/(dashboard)/users/page.tsx — Server Component
export default async function UsersPage() {
  const users = await api.users.list();
  return <UserTable users={users} />;
}
```

**Client interactivity**: Mark `"use client"` only on components that need browser APIs,
event handlers, or hooks. Keep client boundaries as low in the tree as possible.

**Forms**: react-hook-form + zod. Schema shared with backend when possible (from `src/shared/types/`).

**State**: Server state via React Server Components. Client state via `useState`/`useReducer`.
No global state library. If you think you need one, you probably need a Server Component instead.

**Styling**: Tailwind utility classes. Use `cn()` from `lib/utils.ts` for conditional classes.
No CSS modules, no styled-components. Component variants via `cva()` (class-variance-authority).

## Assets

- Icons: Use `lucide-react`. Only add custom SVGs to `public/assets/` if lucide doesn't have it.
- Images: Use Next.js `<Image>` component. All images in `public/` with descriptive names.
- Fonts: Loaded in root layout via `next/font`. Currently Inter (sans) and JetBrains Mono (mono).

## UI component conventions

- New UI primitives: add via `npx shadcn@latest add <component>`. Don't hand-write primitives.
- Composed components go in `components/composed/` and combine multiple primitives.
- Every composed component gets a `*.stories.tsx` for Storybook (optional but encouraged).
- Accessible by default — use semantic HTML, aria attributes where needed, keyboard navigation.

## Adding a new page

1. Create route directory in `app/(group)/your-route/`
2. Add `page.tsx` (Server Component by default)
3. Add `loading.tsx` if data fetching is needed (Suspense boundary)
4. Add `error.tsx` for route-specific error handling
5. Extend `api-client.ts` if calling a new backend endpoint
6. Add E2E test in `tests/e2e/` for critical user flows
