# qstart-rwsdk

A production-ready starter for [RedwoodSDK](https://docs.rwsdk.com/) on Cloudflare Workers.

Built by [qntbr](https://qntbr.com) after shipping a real app with this stack. The goal is to skip the painful setup and get straight to building.

---

## What's included

- **RedwoodSDK** — full-stack framework on Cloudflare Workers
- **BetterAuth** — email/password auth with org support, session management, password reset
- **Prisma + D1** — type-safe SQLite database on Cloudflare D1
- **Organization scoping** — subdomain-based multi-tenancy (`org.yourdomain.com`)
- **Durable Objects** — `UserSessionDO` as a working example with WebSocket hibernation
- **Stripe** — checkout + webhook handler wired up and ready
- **Rate limiting** — KV-backed rate limiter on sensitive routes
- **Turnstile** — bot protection on signup
- **Middleware chain** — session → org → autoCreateOrg, documented and battle-tested

---

## Why I made this

RedwoodSDK is powerful but the gap between "hello world" and "production app" is wide. Auth alone took days to get right — the middleware order matters, the org scoping is subtle, and BetterAuth's CF Workers integration has quirks.

This starter is what I wished existed. It's extracted from a real app (`qntbr.com`) that runs in production, not a toy example.

---

## Examples included

### UserSessionDO (`src/durableObjects/userSessionDO.ts`)
A Durable Object using the **Hibernation API** — the correct pattern for WebSocket DOs on CF Workers. Demonstrates:
- `state.acceptWebSocket()` for hibernation
- `webSocketMessage` / `webSocketClose` as class methods
- Tag-based device identification
- Broadcasting to all connections
- Persistent storage across hibernations
- HTTP + WebSocket in the same `fetch()`

Connect from the client: `ws://yourdomain/__user-session?userId=xxx&deviceId=yyy`

### `useUserSession` hook (`src/app/hooks/useUserSession.ts`)
Client-side hook that connects to `UserSessionDO`. Shows the full client ↔ DO WebSocket pattern.

### Middleware chain (`src/worker.tsx`, `src/lib/middlewareFunctions.ts`)
The middleware order is documented with warnings. Do not reorder without reading the comments — it has bitten us before.

### Catch-all API loader
Drop a file in `src/app/api/` and it's automatically available at `/api/filename`. No route registration needed.

### Stripe webhook + checkout
Working handlers in `src/app/api/webhooks/stripe-wh.ts` and `src/app/api/stripe/create-checkout.ts`.

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
npx wrangler secret put BETTER_AUTH_SECRET     # random 32+ char string
npx wrangler secret put RESEND_API_KEY         # from resend.com
npx wrangler secret put STRIPE_SECRET_KEY      # from stripe.com
npx wrangler secret put STRIPE_WEBHOOK_SECRET  # from stripe dashboard
npx wrangler secret put TURNSTILE_SECRET_KEY   # from cloudflare dashboard
```

For local dev, copy `.dev.vars.example` to `.dev.vars` and fill in values.

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
| `BETTER_AUTH_SECRET` | Generate: `openssl rand -hex 32` |
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
├── worker.tsx                  # Entry point — routes, middleware, DO exports
├── durableObjects/
│   └── userSessionDO.ts        # Example DO with hibernation pattern
├── session/
│   └── durableObject.ts        # BetterAuth session DO (don't modify)
├── lib/
│   ├── auth.ts                 # BetterAuth config
│   ├── middlewareFunctions.ts  # Session + org context setup
│   └── middleware/
│       └── autoCreateOrgMiddleware.ts
├── app/
│   ├── pages/
│   │   ├── user/               # Login, signup, password reset
│   │   ├── landing/            # Public landing page
│   │   ├── sanctum/            # Authenticated home (rename to dashboard)
│   │   └── errors/             # OrgNotFound, NoAccess
│   ├── api/
│   │   ├── stripe/             # Checkout session
│   │   └── webhooks/           # Stripe + LemonSqueezy
│   └── hooks/
│       └── useUserSession.ts   # WebSocket hook for UserSessionDO
└── db.ts                       # Prisma client
```

---

## Adding a new Durable Object

1. Create `src/durableObjects/myDO.ts` — model on `userSessionDO.ts`
2. Export it from `worker.tsx`: `export { MyDO } from './durableObjects/myDO'`
3. Add binding in `wrangler.jsonc` under `durable_objects.bindings`
4. Add migration in `wrangler.jsonc` under `migrations`
5. Wire a route in `worker.tsx`: `route("/__mydo", async ({ request }) => { ... })`

## Adding a new API endpoint

Drop a file at `src/app/api/my-endpoint.ts`:

```ts
export default async function({ request, ctx, params }: any) {
  return Response.json({ hello: "world" });
}
```

Available immediately at `/api/my-endpoint`. No registration needed.

---

## Further reading

- [RedwoodSDK docs](https://docs.rwsdk.com/)
- [BetterAuth docs](https://www.better-auth.com/)
- [Cloudflare Durable Objects](https://developers.cloudflare.com/durable-objects/)
- [Cloudflare D1](https://developers.cloudflare.com/d1/)