# qstart-rwsdk

A production-ready starter for [RedwoodSDK](https://docs.rwsdk.com/) on Cloudflare Workers.

Built by [qntbr](https://qntbr.com) after shipping a real app with this stack. The goal is to skip the painful setup and get straight to building.

---

## What's included

- **RedwoodSDK** — full-stack React framework on Cloudflare Workers
- **Tailwind CSS** — utility-first styling, configured and ready
- **BetterAuth** — email/password auth with org support, session management, password reset
- **Prisma + D1** — type-safe SQLite on Cloudflare D1
- **Organization scoping** — subdomain-based multi-tenancy (`org.yourdomain.com`)
- **Durable Objects** — `UserSessionDO` as a working hibernation example with WebSocket
- **Stripe** — real checkout session + webhook handler, wired end to end
- **Rate limiting** — KV-backed rate limiter on sensitive routes
- **Turnstile** — Cloudflare bot protection on signup
- **Middleware chain** — session → org → autoCreateOrg, documented and battle-tested
- **Server actions** — thin wrappers over services, usable from both RSC and API routes

---

## Why this stack

I wanted React Server Components without paying Next.js/Vercel prices, and without the GraphQL layer that always felt like overhead for solo and small-team projects.

RedwoodSDK gives you RSC and SSR on Cloudflare Workers. That means your server components render at the edge, close to your users, and the cacheable parts stay cached. You get to decide what's server-rendered and what's client — no framework forcing your hand. And because it's just Cloudflare Workers, you can reuse the same service functions in server actions, API routes, and webhooks. No duplication, no special cases.

The Cloudflare infra piece is genuinely great:
- **Workers** compute is cheap and distributed globally by default
- **D1** is SQLite at the edge — easy to reason about, Prisma makes it portable to other DBs if needed
- **Durable Objects** are the killer feature — persistent stateful compute with WebSocket hibernation. You pay for active processing time only (~10-50ms per message), not connection time. A WebSocket DO hibernates between messages and costs essentially nothing at rest
- **KV** for fast reads on things like auth cache and rate limits
- **R2** has no egress fees — store assets without the AWS S3 egress tax

Prisma on top of all this means if you ever need to move off D1, it's a config change, not a rewrite.

---

## Architecture

### Request flow

```
Browser → Cloudflare Worker (src/worker.tsx)
            │
            ├── Middleware chain
            │     1. URL normalization (www strip, HTTPS enforce)
            │     2. initializeServices()       — DB singleton
            │     3. setupSessionContext()      — BetterAuth cookie → ctx.user
            │     4. setupOrganizationContext() — subdomain → ctx.organization
            │     5. autoCreateOrgMiddleware()  — create org for new users
            │
            ├── Route matching
            │     /__user-session   → UserSessionDO (WebSocket)
            │     /api/auth/*       → BetterAuth handler
            │     /api/stripe/*     → Stripe checkout
            │     /api/webhooks/*   → Stripe / LemonSqueezy webhooks
            │     /api/*            → catch-all dynamic loader
            │     /*                → RSC render
```

### Server actions

Server actions live in `src/app/serverActions/` and are the standard way to run server-side logic from React components. They can also be called from API routes — same function, no duplication.

```
src/app/serverActions/
├── admin/
│   ├── signup.ts                  # Create user + org in one transaction
│   ├── createOrgForExistingUser.ts
│   └── getFirstOrgSlugOfUser.ts
├── orgs/
│   └── createOrg.ts               # Create organization, set membership
├── stripe/
│   ├── createCheckoutSession.ts   # Stripe checkout session
│   └── createPortalSession.ts     # Stripe billing portal
└── user/
    └── setPassword.ts             # Password update
```

Server actions are thin — they validate inputs, call the DB or a service, and return a typed result. They don't own routing or response formatting. The same action works whether called from a server component, a client component via RSC, or directly from an API handler.

### Durable Objects

`UserSessionDO` is the working example in this starter. It demonstrates the **Hibernation API** pattern — the correct and cost-efficient way to run WebSocket DOs on Cloudflare.

Key properties of the hibernation pattern:
- The DO sleeps between messages — you only pay for active processing (~10-50ms per message)
- `state.acceptWebSocket(server, [tag])` registers the socket with the CF runtime, not your code
- `webSocketMessage` and `webSocketClose` are class methods, not event listeners
- The DO can broadcast to all connected sockets with `state.getWebSockets()`
- Storage (`state.storage`) persists across hibernations

At 10,000 users each maintaining a persistent WebSocket, this pattern costs roughly $0.75/month.

Connect from the client:
```
ws://yourdomain/__user-session?userId=xxx&deviceId=yyy
```

### API routes (catch-all loader)

Drop a file in `src/app/api/` and it's live — no route registration needed:

```ts
// src/app/api/my-endpoint.ts
export default async function({ request, ctx, params }: any) {
  return Response.json({ hello: "world" });
}
// → available at /api/my-endpoint
```

### Organization scoping

Every org gets a subdomain: `myorg.yourdomain.com`. The middleware reads the subdomain, looks up the org in D1 (with KV cache), and populates `ctx.organization`. Routes branch on `ctx.organization`, `ctx.user`, and `ctx.userRole`. New users are automatically redirected to create an org on first login.

---

## Getting live fast

### 1. Clone and install

```bash
git clone https://github.com/QuinnsCode/qstart-rwsdk-26.git my-app
cd my-app
pnpm install
```

### 2. Create Cloudflare resources

```bash
# D1 database
npx wrangler d1 create my-app-db

# KV namespaces
npx wrangler kv namespace create RATELIMIT_KV
npx wrangler kv namespace create AUTH_CACHE_KV
```

Paste the IDs into `wrangler.jsonc`.

### 3. Update wrangler.jsonc

```jsonc
{
  "name": "my-app",
  "routes": ["yourdomain.com/*", "*.yourdomain.com/*"],
  "vars": {
    "BETTER_AUTH_URL": "https://yourdomain.com"
  }
}
```

### 4. Set secrets

```bash
npx wrangler secret put BETTER_AUTH_SECRET     # openssl rand -hex 32
npx wrangler secret put RESEND_API_KEY         # resend.com
npx wrangler secret put STRIPE_SECRET_KEY      # stripe.com dashboard
npx wrangler secret put STRIPE_WEBHOOK_SECRET  # stripe dashboard → webhooks
npx wrangler secret put TURNSTILE_SECRET_KEY   # cloudflare dashboard → turnstile
```

For local dev, copy `.dev.vars.example` to `.dev.vars` and fill in the same values.

### 5. Run migrations

```bash
pnpm exec prisma generate
npx wrangler d1 execute my-app-db --local --file=prisma/migrations/0001_init.sql
```

### 6. Dev

```bash
pnpm dev
```

### 7. Deploy

```bash
pnpm deploy
```

---

## Required env vars

| Variable | Where to get it |
|---|---|
| `BETTER_AUTH_SECRET` | `openssl rand -hex 32` |
| `BETTER_AUTH_URL` | Your production URL |
| `RESEND_API_KEY` | [resend.com](https://resend.com) |
| `STRIPE_SECRET_KEY` | [stripe.com](https://stripe.com) |
| `STRIPE_WEBHOOK_SECRET` | Stripe dashboard → Webhooks |
| `TURNSTILE_SECRET_KEY` | Cloudflare dashboard → Turnstile |

Optional:

| Variable | Purpose |
|---|---|
| `PRIMARY_DOMAIN` | Used for www redirect (defaults to example.com) |
| `API_ENCRYPTION_KEY` | 32-byte hex key for encrypted fields |

---

## Project structure

```
src/
├── worker.tsx                         # Entry point — routes, middleware, DO exports
├── durableObjects/
│   └── userSessionDO.ts               # Example DO with hibernation + WebSocket
├── session/
│   └── durableObject.ts               # BetterAuth session DO (don't modify)
├── lib/
│   ├── auth.ts                        # BetterAuth server config
│   ├── auth-client.ts                 # BetterAuth client config
│   ├── middlewareFunctions.ts         # Session + org context middleware
│   ├── rateLimit.ts                   # KV-backed rate limiter
│   ├── turnstile.ts                   # Cloudflare Turnstile verification
│   ├── encrypt.ts                     # Field encryption utility
│   └── middleware/
│       └── autoCreateOrgMiddleware.ts
├── app/
│   ├── pages/
│   │   ├── user/                      # Login, signup, password reset
│   │   ├── landing/                   # Public landing page
│   │   ├── sanctum/                   # Authenticated home (rename to /dashboard)
│   │   ├── settings/                  # User settings
│   │   └── errors/                    # OrgNotFound, NoAccess
│   ├── api/
│   │   ├── stripe/                    # create-checkout.ts
│   │   └── webhooks/                  # stripe-wh.ts, lemonsqueezy-wh.ts
│   ├── serverActions/
│   │   ├── admin/                     # signup, createOrg, getOrgSlug
│   │   ├── orgs/                      # createOrg
│   │   ├── stripe/                    # checkout, portal
│   │   └── user/                      # setPassword
│   ├── hooks/
│   │   └── useUserSession.ts          # WebSocket hook for UserSessionDO
│   └── components/
│       ├── theme/FantasyTheme.tsx     # UI component library (replace with your own)
│       ├── Organizations/             # Org creation flow
│       └── settings/                  # Account linking, settings UI
└── db.ts                              # Prisma client singleton
```

---

## Extending the starter

### Adding a new Durable Object

1. Create `src/durableObjects/myDO.ts` — model on `userSessionDO.ts`
2. Export from `worker.tsx`: `export { MyDO } from './durableObjects/myDO'`
3. Add binding in `wrangler.jsonc` under `durable_objects.bindings`
4. Add migration in `wrangler.jsonc` under `migrations`
5. Wire a route: `route("/__mydo", async ({ request }) => { ... })`

### Adding a new API endpoint

```ts
// src/app/api/my-feature.ts
export default async function({ request, ctx, params }: any) {
  return Response.json({ ok: true });
}
```

Live at `/api/my-feature` immediately.

### Adding a new page

Create a component in `src/app/pages/` and register it in `src/worker.tsx`:

```ts
import MyPage from "@/app/pages/MyPage";
route("/my-page", MyPage),
```

---

## Further reading

- [RedwoodSDK docs](https://docs.rwsdk.com/)
- [BetterAuth docs](https://www.better-auth.com/)
- [Cloudflare Durable Objects](https://developers.cloudflare.com/durable-objects/)
- [Cloudflare D1](https://developers.cloudflare.com/d1/)
- [Prisma D1 adapter](https://www.prisma.io/docs/orm/overview/databases/cloudflare-d1)