# Port "space8tt" (Spaces social app) into this project

## What the repo is

A single-page social network app ("Spaces"): feed, stories, live audio spaces, DMs, notifications, explore, profiles, pricing/billing, monetization, and a full admin console.

- ~120 files, ~9.8k lines of app code.
- Frontend already uses React 19 + TanStack Router file routes + TanStack Query + Tailwind v4 + shadcn/ui — very close to this project's stack.
- Backend is a hand-rolled Express server (`server.ts` + `server/routes/api.ts`, 1.8k lines, ~110 endpoints) with swappable "drivers" for database (sqlite / json file / Supabase), storage (local disk / S3 / R2 / Supabase), auth (local / Supabase), realtime (SSE / Supabase), AI (Gemini / mock), and cache.

## What cannot be carried over as-is

| In the repo | Why it breaks here | Replacement |
| --- | --- | --- |
| Express app, `tsx server.ts`, `dist/server.cjs` | This project serves through TanStack Start on an edge runtime; no long-lived Node process | TanStack Start server functions + `src/routes/api/public/*` for external callers |
| SQLite / JSON-file / local-disk drivers | No writable OS filesystem at runtime | Lovable Cloud Postgres as the single database |
| Local password auth, `switch-user`, client-held session id | Insecure and not portable | Lovable Cloud auth (email/password, optional Google) with RLS |
| Local `public/uploads` + S3/R2 driver | No disk; no external buckets configured | Lovable Cloud storage buckets |
| SSE `/api/realtime/events` long-lived stream | Edge functions cannot hold open streams reliably | Cloud realtime subscriptions from the browser |
| `@google/genai` with own API key | Bring-your-own key not needed | Lovable AI gateway for drafts, summaries, smart replies, story text, gradients |
| `/api/system/switch-driver`, `/api/supabase/sync`, driver config UI | Multi-driver abstraction is dead weight once there is one backend | Delete; keep a small `/api/public/health` |
| `src/main.tsx`, `src/App.tsx`, `index.html`, custom `vite.config.ts` env `define` | Start owns the entry, shell, and env handling | Existing `src/routes/__root.tsx`, `src/router.tsx`, `import.meta.env.VITE_*` |
| Roles/flags on the user profile row (`admin` console reads them) | Privilege escalation risk | Separate `user_roles` table + `has_role()` security-definer function |

## Target shape

```text
src/
  routes/
    index.tsx            landing (replaces placeholder)
    feed.tsx explore.tsx spaces.tsx messages.tsx notifications.tsx
    bookmarks.tsx profile.tsx pricing.tsx settings.tsx auth.tsx
    _authenticated/      gated subtree for feed/messages/settings/admin
    api/public/health.ts
  components/ social/*, admin/*, ui/*      (ported nearly unchanged)
  lib/
    *.functions.ts       typed server functions, grouped by domain
    *.server.ts          data access, AI calls, storage helpers
    types.ts formatters.ts utils.ts        (ported)
```

`src/lib/api-client.ts` (808 lines of `fetch("/api/...")`) is replaced domain by domain with server functions, keeping the same function names and return types so component code barely changes.

## Migration phases

1. **Foundations** — enable Lovable Cloud; port `src/styles.css` design tokens, `src/lib/types.ts`, formatters, shadcn `ui/*`, and `hooks/use-mobile`. Nothing user-visible yet.
2. **Schema** — one migration creating profiles, follows, posts, comments, likes, bookmarks, reposts, poll votes, impressions, stories, spaces + participants + space messages, conversations + messages, notifications, reports, audit logs, settings, plans/subscriptions, monetization, feed preferences, plus `user_roles`. Every table gets GRANTs and RLS policies, and the migration includes literal seed rows so the feed is not empty on first load.
3. **Auth** — port `auth.tsx` onto Cloud auth, add the `_authenticated` gate, replace `auth-state.ts` client session juggling with a real session hook.
4. **Read paths** — feed, explore, profile, notifications, bookmarks: server functions + route loaders/`useSuspenseQuery`.
5. **Write paths** — composer, likes/bookmarks/reposts/votes, comments, follows, profile edit, reports.
6. **Media** — upload through Cloud storage, signed URLs, avatar/cover/post images.
7. **Realtime** — DMs, notifications, and space rooms via Cloud realtime subscriptions instead of SSE polling.
8. **AI** — draft, summarize-space, smart-reply, story, gradient endpoints on the Lovable AI gateway.
9. **Admin** — overview, users, moderation, content, audit logs, system settings, all behind `has_role(auth.uid(),'admin')` checked server-side.
10. **Billing/monetization** — port the pricing page and plan gating; real payments only if you want Stripe wired in later.
11. **Polish** — per-route `head()` metadata (unique titles/descriptions/OG), remove the last driver-abstraction leftovers, verify each route in the browser.

## Refactor decisions worth flagging

- `src/routes/index.tsx` (1138 lines) and `messages.tsx` (1238 lines) get split into route file + feature components under `src/components/social/`.
- The nine `*-state.ts` modules (plan, branding, monetization, support, workspace, developer, theme, unread, feed-cache) are localStorage singletons; the ones representing real data move into Cloud tables, the rest collapse into TanStack Query cache.
- One `plans.ts` source of truth for plan tiers, shared by pricing, upgrade modal, and server-side gating.
- Admin, billing, and moderation authority always verified server-side — never from client state.

## Open questions

1. Should the pricing page stay presentational for now, or do you want real Stripe checkout in this port?
2. Keep the "developer portal", "team workspaces", "custom branding", and "priority support" screens, or drop them from the first pass?
3. Does any existing data in the repo's sqlite/json store need importing, or is seeded demo content fine?
