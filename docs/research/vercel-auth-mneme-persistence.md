# Survey: Vercel-friendly auth + Mneme persistence

**Ticket:** [#6](https://github.com/LayishSieger/euodia/issues/6) · **Map:** [#2](https://github.com/LayishSieger/euodia/issues/2)  
**Scope:** Facts and trade-offs only — no stack choice (that is [#7](https://github.com/LayishSieger/euodia/issues/7)).  
**Product constraint:** V0 needs authenticated users and a saved **Mneme** document (YAML/JSON from a Zod schema) per user on Next.js / Vercel.

---

## Naming note: there is no first-party “Vercel Auth” product

As of the sources below, Vercel does **not** ship a standalone end-user auth product named “Vercel Auth.” Relevant Vercel-adjacent pieces:

| Name | What it is | Fit for public app users |
| --- | --- | --- |
| **Auth.js / NextAuth** | Open-source auth library; Auth.js docs state the project is now part of Better Auth | Yes — documented for serverless deploys including Vercel |
| **Better Auth** | Framework-agnostic TypeScript auth library (self-hosted in your app) | Yes — Vercel docs also show Connect adapters for it |
| **Neon Managed Better Auth** | Neon-hosted Better Auth; users/sessions in `neon_auth` schema | Yes — designed for apps; beta |
| **Vercel Connect + Auth.js/Better Auth** | OAuth connectors (e.g. Linear/Slack) into app sessions | Narrow — signs users in via Connect connectors, not a general consumer IdP suite |
| **Vercel Passport** | Deployment protection (Enterprise); OIDC before the app runs | No for product Mneme — protects deployments, does not create an app user store |

Sources: [Auth.js Deployment](https://authjs.dev/getting-started/deployment), [Better Auth intro](https://www.better-auth.com/docs/introduction), [Neon Managed Better Auth](https://neon.com/docs/auth/overview), [Vercel Connect + Auth.js](https://vercel.com/docs/connect/ecosystem/authjs), [Vercel Passport KB](https://vercel.com/kb/guide/vercel-passport), [Vercel auth KB (Auth.js on Vercel)](https://vercel.com/kb/guide/complete-guide-authentication-vercel).

---

## Auth options × trade-offs (minimal public V0)

| Option | How it works (facts) | Pros for minimal V0 | Cons / constraints |
| --- | --- | --- | --- |
| **Auth.js (`next-auth`) — JWT session** | Default when no DB adapter: encrypted JWT in `HttpOnly` cookie; needs `AUTH_SECRET` | No auth DB required for sessions; fewer moving parts; Edge-friendlier for session reads | Cannot revoke a JWT before expiry without a blocklist; cookie size limits (~4KB; Auth.js can chunk); OAuth callback / preview-URL friction |
| **Auth.js — database session + adapter** | Session row in DB; cookie holds opaque session id; official adapters include Postgres (`@auth/pg-adapter`) | Server-side revoke / “sign out everywhere”; persists users/accounts for linking Mneme to `userId` | DB roundtrip on session use; many adapters not Edge-native (Auth.js documents split JWT middleware + DB instance pattern) |
| **Auth.js on Vercel (ops facts)** | `AUTH_TRUST_HOST` auto-inferred on Vercel; OAuth often needs stable callback URL; `AUTH_REDIRECT_PROXY_URL` for preview proxy | First-class serverless deploy story | Preview deploys need shared `AUTH_SECRET` + stable redirect proxy for many providers; some providers (e.g. Apple) disallow redirect proxy |
| **Better Auth (self-hosted)** | App-owned auth server; typically backed by a database (`Pool` / `DATABASE_URL` in Vercel’s Connect tutorial) | Rich plugin surface; Auth.js project now under Better Auth org | You run/configure auth in-app; still need a DB for durable users/sessions in usual setups |
| **Neon Managed Better Auth** | Managed REST auth; data in `neon_auth`; branches with Neon DB; Next.js via `@neondatabase/auth` | Auth + Postgres co-located; branch-aware preview auth; MAU included on Neon plans (Free up to 60k MAU) | **Beta**; AWS regions only; no IP Allow / Private Networking; Managed vs self-host feature gap; couples auth to Neon |
| **Clerk** | Hosted IdP; `@clerk/nextjs`, middleware, UI components | Fast path for sign-in/up UI and session helpers on Next.js | External IdP; Mneme still needs separate storage; vendor + billing surface |
| **Supabase Auth** | JWT auth; users in project Postgres; RLS for authorization; Next.js SSR via `@supabase/ssr` / `with-supabase` template | Auth + Postgres + RLS in one vendor; Marketplace-installable from Vercel | Product/auth coupling to Supabase; RLS policy work is required for safe client access |
| **Vercel Connect as IdP** | `connect()` provider for Auth.js or Better Auth generic OAuth | Useful if sign-in is via Connect-supported apps | Not a general email/social consumer auth suite; distinct from provider API tokens / Passport |

Sources: [Auth.js session strategies](https://authjs.dev/concepts/session-strategies), [Auth.js database adapters](https://authjs.dev/getting-started/database), [Auth.js Edge compatibility](https://authjs.dev/guides/edge-compatibility), [Auth.js Postgres adapter](https://authjs.dev/getting-started/adapters/pg), [Auth.js Deployment / preview proxy](https://authjs.dev/getting-started/deployment), [Neon Auth overview](https://neon.com/docs/auth/overview), [Clerk Next.js quickstart](https://clerk.com/docs/nextjs/getting-started/quickstart), [Supabase Auth](https://supabase.com/docs/guides/auth), [Supabase + Next.js](https://supabase.com/docs/guides/getting-started/quickstarts/nextjs), [Vercel Connect Auth.js](https://vercel.com/docs/connect/ecosystem/authjs), [Vercel Connect Better Auth](https://vercel.com/docs/connect/frameworks/better-auth).

---

## Mneme persistence options × trade-offs

Mneme is a **per-user structured document** (YAML/JSON). Needs durable read/write keyed by authenticated user id, not CDN config.

### Retired / renamed Vercel products (do not plan on these)

| Product | Status | Replacement direction |
| --- | --- | --- |
| **Vercel Postgres** | No longer available; existing DBs moved to Neon (Dec 2024) | Marketplace Postgres (e.g. Neon) |
| **Vercel KV** | Product/package deprecated / gone | Marketplace KV (e.g. Upstash Redis) |
| **Edge Config** | Renamed **Global Config** | Same product family; still config-oriented |

Sources: [Postgres on Vercel](https://vercel.com/docs/postgres), [vercel/storage README](https://github.com/vercel/storage/blob/main/README.md), [Neon Vercel Postgres transition](https://neon.com/docs/guides/vercel-postgres-transition-guide), [Global Config limits](https://vercel.com/docs/storage/edge-config/edge-config-limits).

### Viable storage options

| Storage | Mechanism | Fit for Mneme document | Trade-offs for minimal V0 |
| --- | --- | --- | --- |
| **Marketplace Postgres (Neon, Supabase, Prisma Postgres, Aurora, …)** | Relational DB; credentials injected via Vercel Marketplace / `vercel install` | Strong: one row (or versioned rows) per user with `user_id` + `jsonb`/`text` Mneme; ACID; queryable | Needs schema/migrations; serverless needs pooling / serverless drivers (e.g. Neon HTTP/WebSocket driver); region choice affects latency |
| **Neon specifically** | Serverless Postgres; Vercel-managed or Neon-managed integration; preview DB branching | Same as Postgres + optional Managed Better Auth + branch cleanup differences | Preview branch cleanup timing differs (Vercel-managed vs Neon-managed); choose billing path |
| **Supabase Postgres (+ optional Storage)** | Postgres + Auth + RLS (+ file Storage if used) | Mneme as table row with RLS `auth.uid()` policies | Auth/storage coupling; RLS must be correct before exposing Data API |
| **Upstash Redis (Marketplace KV)** | Redis-compatible key/value; REST-friendly for serverless | Possible: `mneme:{userId}` → JSON string | No rich relational querying; document size/TTL/eviction policies are Redis concerns; better as cache than system of record unless designed carefully |
| **Vercel Blob (private store)** | Object storage; `put`/`get` with `access: 'private'`; OIDC or token auth | Possible: one private object path per user (e.g. `mneme/{userId}.json`) | Object store, not a DB — listing/query/versioning/indexing are app-built; private blobs need app route + auth to stream; suited to files more than transactional docs |
| **Vercel Blob (public)** | Public URLs | Poor for private career memory | Anyone with URL can read |
| **Global Config (ex–Edge Config)** | Ultra-fast global reads; max **1 MB per store**; writes propagate up to ~10s | Poor as Mneme store | Documented for rare-write config/flags; size + write latency make per-user mutable career memory a mismatch |

Sources: [Vercel Storage overview](https://vercel.com/docs/storage), [Marketplace storage](https://vercel.com/docs/marketplace-storage), [Neon ↔ Vercel integration](https://neon.com/docs/guides/vercel), [Neon serverless driver](https://neon.tech/docs/serverless/serverless-driver), [Vercel Blob private storage](https://vercel.com/docs/storage/vercel-blob/private-storage), [Upstash Redis get started](https://upstash.com/docs/redis/overall/getstarted), [Global Config limits](https://vercel.com/docs/storage/edge-config/edge-config-limits).

---

## Auth × storage coupling patterns (factual combinations)

| Pattern | Auth | Mneme store | Coupling notes |
| --- | --- | --- | --- |
| A | Auth.js JWT only | Postgres (Marketplace) | App maps session `user.id`/email → Mneme row; may still want adapter later for durable users |
| B | Auth.js + Postgres adapter | Same Postgres | Auth tables + Mneme tables in one DB |
| C | Neon Managed Better Auth | Neon Postgres | Users in `neon_auth`; Mneme in app schema; RLS-compatible identity in same DB |
| D | Better Auth self-hosted | Any Marketplace Postgres | Auth tables + Mneme in app DB (as in Vercel Better Auth + `DATABASE_URL` examples) |
| E | Clerk | Postgres or Blob | External `userId` foreign key into your store |
| F | Supabase Auth | Supabase Postgres | JWT + RLS policies on Mneme table |
| G | Any of the above | Private Blob | Store only path/URL metadata if using a DB; enforce auth on every `get` |

---

## Vercel operational facts that apply regardless of choice

- Marketplace storage: provision via dashboard or `vercel install neon|upstash|supabase`; env vars injected into the project ([Marketplace storage](https://vercel.com/docs/marketplace-storage)).
- Prefer DB region near Functions ([Storage overview best practices](https://vercel.com/docs/storage)).
- Auth.js on Vercel: secret + OAuth callback hygiene; preview deploys often need redirect proxy ([Auth.js Deployment](https://authjs.dev/getting-started/deployment); [Vercel auth KB](https://vercel.com/kb/guide/complete-guide-authentication-vercel)).
- Passport / Connect are **not** substitutes for “save Mneme per product user” unless the product intentionally uses those identity models ([Connect Auth.js “choose the correct integration”](https://vercel.com/docs/connect/ecosystem/authjs)).

---

## Out of scope

- Choosing the V0 stack (issue #7).
- Schema of Mneme fields, RLS policy text, or implementation.

---

## Primary source index

1. https://authjs.dev/getting-started/deployment  
2. https://authjs.dev/concepts/session-strategies  
3. https://authjs.dev/getting-started/database  
4. https://authjs.dev/guides/edge-compatibility  
5. https://authjs.dev/getting-started/adapters/pg  
6. https://www.better-auth.com/docs/introduction  
7. https://vercel.com/kb/guide/complete-guide-authentication-vercel  
8. https://vercel.com/docs/connect/ecosystem/authjs  
9. https://vercel.com/docs/connect/frameworks/better-auth  
10. https://vercel.com/kb/guide/vercel-passport  
11. https://vercel.com/docs/postgres  
12. https://vercel.com/docs/storage  
13. https://vercel.com/docs/marketplace-storage  
14. https://vercel.com/docs/storage/vercel-blob/private-storage  
15. https://vercel.com/docs/storage/edge-config/edge-config-limits  
16. https://github.com/vercel/storage/blob/main/README.md  
17. https://neon.com/docs/auth/overview  
18. https://neon.com/docs/guides/vercel  
19. https://neon.com/docs/guides/vercel-postgres-transition-guide  
20. https://neon.tech/docs/serverless/serverless-driver  
21. https://supabase.com/docs/guides/auth  
22. https://supabase.com/docs/guides/getting-started/quickstarts/nextjs  
23. https://clerk.com/docs/nextjs/getting-started/quickstart  
24. https://upstash.com/docs/redis/overall/getstarted  
