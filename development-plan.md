# Multi-Tenant SaaS Bootstrapper — Phased Development Plan

> Project: 285-multi-tenant-saas-bootstrapper · Created: 2026-05-30
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

This plan synthesizes `research.md`, `features.md`, `standards.md`, `README.md`, and the four `data-model-suggestion-*.md` files into concrete technology decisions, an architecture, and a sequence of additive phases.

**Product summary.** An open-source, AI-native starter kit that lets indie hackers, agencies, and seed-stage startups ship a B2B multi-tenant SaaS without hand-wiring tenant isolation, authentication, billing, and an admin console. It packages the patterns proven by commercial starters (MakerKit, Supastarter, Shipfast) into a free, framework-coherent foundation that AI coding assistants can safely extend. The differentiator is twofold: (1) RLS-enforced tenant isolation, Stripe metered billing, and SCIM/SSO out of the box — features unevenly distributed across today's options; and (2) AI-native scaffolding (generate onboarding flows, billing tiers from plain English, a conversational admin, anomaly detection, and feature-module generation that respects the multi-tenant architecture), exposed both in-product and over MCP.

**Primary personas.** Indie hacker launching a first SaaS; agency building client products; seed-stage startup without a platform team; developer learning multi-tenant patterns.

**Deployment model.** Self-hosted and cloud via Docker, Vercel, and AWS. SaaS-style application + management API + admin console + MCP server. Not a CLI generator — it is a runnable, forkable application.

**Data model choice.** The MVP adopts **Data Model Suggestion 1 (Normalized Relational + PostgreSQL RLS)** as the canonical schema, with **Suggestion 3's JSONB escape hatches** (`tenants.settings`, `tenants.feature_overrides`, `audit_logs.details`) folded in where flexibility matters. **Suggestion 4 (tiered Pool/Bridge/Silo isolation)** is deferred to Phase 11 as an optional upgrade path — the MVP ships Pool tier only, exactly as Suggestion 4 itself recommends. **Suggestion 2 (event sourcing)** is explicitly rejected as architectural overkill for the target audience.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Language | TypeScript (strict) | The entire competitive set (MakerKit, Supastarter, ixartz, Gravity, Orbit) is JS/TS; buyers expect a TS stack. One language spans API, web UI, and edge. Best ecosystem fit for Stripe, Clerk/WorkOS, Resend, Svix, Neon, MCP SDK. |
| Runtime | Node.js 22 LTS | Stable LTS; native `fetch`, WebCrypto, test runner. Required by Next.js 16 and the Stripe/Svix/WorkOS SDKs. |
| Framework | Next.js 16 (App Router) | Single deployable that serves the web UI, the REST API (route handlers), and SSR admin console. First-class Vercel deploy target named in `README.md`. App Router + Server Components keep the bootstrapper modern in 2026. |
| Package manager | pnpm 9 (workspaces) | Fast, disk-efficient, monorepo-native. Lets us split `apps/web` from shared `packages/*` (db, auth, billing, core) so AI scaffolding edits one boundary at a time. |
| Monorepo tooling | Turborepo | Caches builds/tests across the workspace; standard in `next-forge` and Supastarter. |
| Database | PostgreSQL 16+ | RLS is the isolation mechanism mandated by `standards.md` (OWASP A01) and Suggestion 1. PG16 improves RLS plan performance. Runs locally via Docker, in cloud via Neon/Supabase/RDS. |
| ORM / migrations | Drizzle ORM + Drizzle Kit | Best JSONB ergonomics and typed SQL in TS (per Suggestions 1 & 3); migrations are plain SQL we can lint for RLS coverage. Lighter than Prisma for an RLS `SET LOCAL` workflow. |
| Connection pooling | PgBouncer (transaction mode) / Neon pooler | Suggestion 1 requires `SET LOCAL` tenant context; transaction-mode pooling is mandatory to prevent session-variable leakage between tenants. |
| Auth | Auth.js (NextAuth v5) for email/password + OAuth; pluggable adapter for Clerk/WorkOS | Auth.js is open-source (free, matches the project's positioning) and supports email/password, OAuth, PKCE (RFC 7636), and JWT sessions (RFC 7519). An adapter interface lets buyers swap in Clerk Organizations or WorkOS without touching app code. |
| Enterprise SSO/SCIM | BoxyHQ SAML Jackson (embedded) | Provides SAML 2.0 + OIDC + SCIM 2.0 (RFC 7643/7644) without bespoke crypto, exactly as `standards.md` recommends. Self-hostable, open-source, aligns with the free positioning. |
| Billing | Stripe Billing + Lemon Squeezy via a `BillingAdapter` interface | `standards.md` states metered-billing semantics are not standardized, so we abstract behind an adapter. Stripe is the default (subscriptions + metered usage + Customer Portal); Lemon Squeezy (Merchant of Record) is the second implementation for indie hackers needing global tax handling. |
| Background jobs / queue | BullMQ on Redis | Async workloads: webhook dispatch, email sends, usage aggregation, anomaly scans, tier migrations. Mature, TS-native, Docker-friendly. |
| Caching / rate limiting | Redis (same instance) | Token-bucket rate limiting per tenant/API key; route cache for tenant resolution; BullMQ backend. |
| Email | Resend + React Email | Developer-focused, used by MakerKit/Supastarter; templates are React components (invite, verify, reset, billing alerts). Abstracted behind an `EmailAdapter`. |
| Webhooks (outbound) | Standard Webhooks spec, HMAC-SHA256; Svix optional adapter | `standards.md` mandates the Standard Webhooks signature scheme to stay interoperable. A `WebhookDispatcher` ships in-house; a Svix adapter is offered for managed delivery. |
| API schema | OpenAPI 3.1 generated from Zod via `zod-to-openapi` | `standards.md` requires an OpenAPI schema for SDK generation and contract testing; Zod doubles as runtime request/response validation (JSON Schema 2020-12). |
| AI layer | Vercel AI SDK + Anthropic Claude (provider-agnostic via AI Gateway) | Powers onboarding generation, plain-English billing-tier configuration, conversational admin, anomaly remediation, and feature-module scaffolding. AI SDK gives streaming + structured output + tool calling. |
| MCP server | `@modelcontextprotocol/sdk` (TypeScript) | Exposes tenant-management operations as MCP tools (per `standards.md`) so AI assistants/agents can drive the admin plane. |
| Frontend UI | React 19 + Tailwind CSS + shadcn/ui | Matches ixartz/MakerKit conventions; shadcn gives composable, themeable components that support white-labeling. |
| Observability | OpenTelemetry (tenant_id on every span) + pino logs | `standards.md` "emerging standards" — per-tenant SLA/anomaly/chargeback signals come free if tenant context is attached to spans from day one. |
| Validation | Zod | Single source of truth for request validation, JSONB shape enforcement, env config, and OpenAPI generation. |
| Testing | Vitest (unit/integration) + Playwright (e2e) + Testcontainers (real Postgres/Redis) | Vitest is fast and TS-native; Testcontainers spins ephemeral Postgres for RLS tests; Playwright covers admin/web flows. |
| Lint / format / types | ESLint (typescript-eslint) + Prettier + `tsc --noEmit` | Standard TS quality gate; a custom ESLint/SQL lint rule enforces RLS-on-every-tenant-table. |
| Containerisation | Docker + docker-compose | `README.md` targets Docker/Vercel/AWS. Compose brings up Postgres + Redis + app for local dev and self-hosting. |
| CI | GitHub Actions | Lint, typecheck, unit, integration (Testcontainers), RLS lint, Docker build, OpenAPI diff. |
| Config | `.env` + Zod-validated `env.ts` | Fail-fast on missing secrets; documented in `.env.example`. |

### Project Structure

```
multi-tenant-saas-bootstrapper/
├── package.json                     # pnpm workspace root
├── pnpm-workspace.yaml
├── turbo.json
├── docker-compose.yml               # postgres, redis, app, jackson (SSO)
├── Dockerfile
├── .env.example
├── tsconfig.base.json
├── CLAUDE.md                        # AI rules: multi-tenant invariants, RLS rules, how to scaffold
├── .cursor/rules/                   # mirror of CLAUDE.md for Cursor compatibility
├── apps/
│   └── web/                         # Next.js 16 app: web UI + REST API + admin console
│       ├── app/
│       │   ├── (marketing)/         # public landing
│       │   ├── (auth)/              # sign-in, sign-up, accept-invite
│       │   ├── (app)/[tenant]/      # tenant-scoped product shell (dashboard, settings, billing, members)
│       │   ├── (admin)/admin/       # platform super-admin console
│       │   └── api/
│       │       ├── v1/              # public REST API (OpenAPI 3.1)
│       │       ├── webhooks/stripe/ # inbound Stripe webhook
│       │       ├── scim/v2/         # SCIM 2.0 provisioning endpoints
│       │       └── auth/            # Auth.js + SSO callbacks
│       ├── components/              # shadcn/ui + product components
│       └── middleware.ts            # tenant resolution, auth guard, rate limit
├── packages/
│   ├── db/                          # Drizzle schema, migrations, RLS policies, tenant-context client
│   │   ├── schema/                  # one file per domain (tenants, members, billing, …)
│   │   ├── migrations/              # generated SQL + RLS policy migrations
│   │   ├── rls.ts                   # SET LOCAL helpers, withTenant() transaction wrapper
│   │   └── lint-rls.ts              # CI check: every tenant-scoped table has an RLS policy
│   ├── core/                        # domain services (tenant, membership, audit, feature-flags)
│   ├── auth/                        # Auth.js config, JWT claims, role/permission engine, adapters
│   ├── billing/                     # BillingAdapter interface + stripe/ + lemonsqueezy/ impls
│   ├── email/                       # EmailAdapter + Resend impl + React Email templates
│   ├── webhooks/                    # outbound dispatcher (Standard Webhooks) + svix adapter
│   ├── jobs/                        # BullMQ queues, workers, schedulers
│   ├── ai/                          # AI SDK prompts, structured-output schemas, tool definitions
│   ├── mcp/                         # MCP server exposing admin-plane tools
│   ├── api-schema/                  # Zod schemas + OpenAPI 3.1 generator
│   └── config/                      # env.ts (Zod), shared constants, ESLint/TS base configs
├── scripts/
│   ├── seed.ts                      # seed plans, roles, demo tenant
│   └── create-tenant.ts
└── tests/
    ├── unit/
    ├── integration/                 # Testcontainers: Postgres + Redis
    └── e2e/                         # Playwright
```

The structure is grouped by concern, not by phase. Each phase adds files/packages without restructuring.

---

## Phase 1: Foundation & Tenant-Aware Data Layer

### Purpose
Establish the monorepo, the runnable app shell, and — most importantly — the RLS-enforced data layer that every later phase depends on. After this phase, the project boots via `docker-compose up`, migrations run, and any tenant-scoped query is provably isolated at the database level. This is the security backbone (OWASP A01:2021, ISO 27001 access control) and must be correct before any feature is built on top.

### Tasks

#### 1.1 — Monorepo & toolchain bootstrap

**What**: Initialise the pnpm/Turborepo workspace, Next.js 16 app, shared packages, Docker compose, and CI quality gate.

**Design**:
- `pnpm-workspace.yaml` includes `apps/*` and `packages/*`.
- `turbo.json` pipelines: `build`, `lint`, `typecheck`, `test`, `test:integration`.
- `apps/web` created with Next.js 16 App Router, React 19, Tailwind, shadcn/ui initialised.
- `packages/config/env.ts` validates environment with Zod; throws on boot if invalid:

```ts
export const env = z.object({
  DATABASE_URL: z.string().url(),
  REDIS_URL: z.string().url(),
  AUTH_SECRET: z.string().min(32),
  STRIPE_SECRET_KEY: z.string().startsWith("sk_").optional(),
  STRIPE_WEBHOOK_SECRET: z.string().startsWith("whsec_").optional(),
  RESEND_API_KEY: z.string().optional(),
  ANTHROPIC_API_KEY: z.string().optional(),
  APP_URL: z.string().url(),
  NODE_ENV: z.enum(["development", "test", "production"]).default("development"),
}).parse(process.env);
```

- `docker-compose.yml` services: `postgres:16`, `redis:7`, `jackson` (BoxyHQ SSO, wired in Phase 7), `web`.
- `Dockerfile` multi-stage build producing a standalone Next.js output.
- `.env.example` documents every variable.
- GitHub Actions workflow runs lint, typecheck, unit tests, integration tests, RLS lint, Docker build.

**Testing**:
- `Unit: env parse with valid vars → typed env object`
- `Unit: env parse missing AUTH_SECRET → ZodError naming AUTH_SECRET`
- `Integration: docker-compose up → web responds 200 on / within 60s`
- `CI: pnpm build succeeds for all workspaces`

#### 1.2 — Canonical schema & migrations (Suggestion 1 + JSONB)

**What**: Define the Drizzle schema for global and tenant-scoped tables and generate the initial migration.

**Design**: Implement the Suggestion-1 DDL as Drizzle tables, with Suggestion-3 JSONB columns folded in. Global tables: `plans`, `prices`, `plan_features`, `roles`, `permissions`. Tenant-scoped: `tenants`, `users`, `memberships`, `invitations`, `subscriptions`, `subscription_items`, `invoices`, `feature_flags`, `api_keys`, `webhook_endpoints`, `audit_logs`. Folded-in JSONB: `tenants.settings`, `tenants.feature_overrides`, `audit_logs.details`. Status enums modeled as TS union + DB `TEXT` with a check constraint:

```ts
export const tenants = pgTable("tenants", {
  id: uuid("id").primaryKey().defaultRandom(),
  name: text("name").notNull(),
  slug: text("slug").notNull().unique(),
  billingEmail: text("billing_email"),
  stripeCustomerId: text("stripe_customer_id").unique(),
  logoUrl: text("logo_url"),
  customDomain: text("custom_domain"),
  settings: jsonb("settings").$type<TenantSettings>().notNull().default({}),
  featureOverrides: jsonb("feature_overrides").$type<Record<string, string|number|boolean>>().notNull().default({}),
  status: text("status", { enum: ["active","suspended","churned"] }).notNull().default("active"),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp("updated_at", { withTimezone: true }).notNull().defaultNow(),
});
```

All composite indexes lead with `tenant_id` (Suggestion 1 §"Critical Indexes"). `password_hash` is nullable (OAuth-only users).

**Testing**:
- `Integration (Testcontainers): run migration → all tables and indexes exist (query information_schema)`
- `Integration: insert tenant with duplicate slug → unique violation`
- `Integration: insert membership with unknown role_id → FK violation`
- `Unit: every tenant-scoped table definition includes a tenant_id column (reflection test)`

#### 1.3 — RLS policies & tenant-context client

**What**: Enable RLS on every tenant-scoped table, add isolation policies, and provide a `withTenant()` transaction wrapper that sets `app.current_tenant_id` via `SET LOCAL`.

**Design**:
- A migration enables RLS and creates `tenant_isolation` policies for each tenant-scoped table (Suggestion 1 §"RLS Policies"). The application connects as a non-superuser role (`app_user`) that cannot bypass RLS.
- `packages/db/rls.ts`:

```ts
export async function withTenant<T>(
  tenantId: string,
  fn: (tx: DrizzleTx) => Promise<T>,
): Promise<T> {
  return db.transaction(async (tx) => {
    await tx.execute(sql`SET LOCAL app.current_tenant_id = ${tenantId}`);
    return fn(tx);
  });
}
// Admin/system queries use withPlatform() which sets a bypass role explicitly.
```

- `SET LOCAL` (transaction-scoped) is mandatory — never `SET` — to survive PgBouncer transaction pooling (Suggestion 1 §"Migration and Scaling").
- `packages/db/lint-rls.ts`: a CI script that fails if any table with a `tenant_id` column lacks an enabled RLS policy.

**Testing**:
- `Integration (real PG, app_user role): seed tenant A and tenant B rows; withTenant(A) select memberships → only A's rows` (the core isolation guarantee)
- `Integration: withTenant(A) attempt to update a row belonging to B → 0 rows affected`
- `Integration: query without SET LOCAL → 0 rows (policy denies)`
- `Integration: two interleaved transactions in the same pooled connection do not leak tenant context`
- `CI unit: lint-rls flags a deliberately-unpolicied test table`

#### 1.4 — Seed data & domain primitives

**What**: Seed default roles/permissions and starter plans; define core TS domain types.

**Design**: Seed `roles` (`owner`, `admin`, `member`, `viewer`) and their `permissions` matrix; seed `plans` (`free`, `starter`, `pro`) with `prices` and `plan_features` (`max_seats`, `api_requests_per_month`, `sso_enabled`, etc.). `scripts/seed.ts` is idempotent (upsert by slug).

**Testing**:
- `Integration: run seed twice → no duplicate rows, counts stable`
- `Unit: permission matrix for "viewer" grants read-only on all resources`

---

## Phase 2: Authentication & Identity

### Purpose
Add user identity: email/password and OAuth sign-in, email verification, sessions carrying tenant context, and the role/permission engine. After this phase a user can register, verify email, and sign in, receiving a JWT whose claims (`sub`, `tenant_id`, `role`) drive both middleware and RLS context. Implements RFC 6749/7636/7519 and aligns with NIST SP 800-63B and OWASP ASVS L2.

### Tasks

#### 2.1 — Auth.js configuration with pluggable provider adapter

**What**: Configure Auth.js (NextAuth v5) for credentials + OAuth, behind an `AuthProvider` interface.

**Design**:
- `packages/auth` exposes an `AuthProvider` interface; the default implementation is Auth.js, with documented swap-in adapters for Clerk Organizations and WorkOS.
- Credentials provider: Argon2id password hashing (NIST 800-63B compliant — no composition rules, length ≥ 8, breached-password check hook).
- OAuth providers: Google, GitHub (PKCE enforced per RFC 7636).
- Session strategy: JWT (RFC 7519). Token claims:

```ts
interface SessionClaims {
  sub: string;            // user id
  email: string;
  tenantId: string | null; // active tenant
  role: "owner"|"admin"|"member"|"viewer" | null;
  memberships: { tenantId: string; role: string }[];
}
```

- Active-tenant switching re-mints the JWT with the new `tenantId`/`role`.

**Testing**:
- `Unit: hash then verify password → true; wrong password → false`
- `Unit: weak/breached password → rejected with reason`
- `Integration (mocked OAuth): Google callback → user created, session issued`
- `Integration: sign-in issues JWT with correct tenantId and role claims`

#### 2.2 — Email verification, password reset, and email adapter

**What**: Implement verification and reset flows using the `EmailAdapter` (Resend default) and React Email templates.

**Design**: `EmailAdapter.send({ to, template, props })`. Templates: `VerifyEmail`, `ResetPassword`. Single-use, time-boxed tokens (hashed at rest, 24h verify / 1h reset). On verify, set `users.email_verified_at`.

**Testing**:
- `Integration (mocked Resend): request reset → email enqueued with one-time token`
- `Unit: expired token → rejected`
- `Unit: reused token → rejected (single-use)`

#### 2.3 — Role & permission engine

**What**: Central authorization check `can(role, resource, action)` backed by the `permissions` table.

**Design**:

```ts
function can(role: Role, resource: string, action: "read"|"write"|"delete"): boolean;
// Server Action / API guard:
function authorize(claims: SessionClaims, resource: string, action: Action): void; // throws ForbiddenError
```

Role hierarchy: `owner` ⊃ `admin` ⊃ `member` ⊃ `viewer`. Permissions cached per-role in memory.

**Testing**:
- `Unit: member cannot delete billing → false`
- `Unit: owner can write settings → true`
- `Unit: authorize() throws ForbiddenError for viewer writing members`

---

## Phase 3: Tenant Lifecycle, Membership & Middleware

### Purpose
Deliver the heart of multi-tenancy: creating tenants, resolving the active tenant on every request, inviting/managing members, and switching between tenants. After this phase, multiple isolated organizations coexist, each with its own members and roles, and the middleware guarantees tenant context is set for the entire request lifecycle.

### Tasks

#### 3.1 — Tenant resolution middleware

**What**: `apps/web/middleware.ts` resolves the active tenant (from path `/[tenant]`, custom domain, or JWT claim), authenticates the session, and rejects cross-tenant access.

**Design**: Resolution order: subdomain/custom domain → path slug → JWT `tenantId`. Verifies the user has a membership in the resolved tenant (else 403). Caches `slug → tenantId` in Redis (60s TTL). Sets request headers `x-tenant-id`, `x-user-id`, `x-role` consumed by route handlers and Server Components; downstream DB access uses `withTenant()`.

**Testing**:
- `Integration: request /acme by member of acme → 200, x-tenant-id set`
- `Integration: request /acme by non-member → 403`
- `Integration: unknown tenant slug → 404`
- `Integration: custom-domain request resolves to correct tenant`

#### 3.2 — Tenant service & provisioning

**What**: `TenantService.create()` provisions a tenant, owner membership, default feature flags (seeded from plan), and a Stripe customer stub.

**Design**:

```ts
async function createTenant(input: { name: string; slug: string; ownerUserId: string; planSlug?: string }): Promise<Tenant>;
```

Transactionally: insert tenant → insert owner membership (role `owner`) → copy `plan_features` into `feature_flags` → write `audit_log` (`tenant.created`). Slug validation: lowercase, `[a-z0-9-]`, reserved-word denylist (`admin`, `api`, `www`, …).

**Testing**:
- `Integration: createTenant → tenant + owner membership + seeded feature_flags + audit_log row`
- `Unit: reserved slug "admin" → ValidationError`
- `Integration: duplicate slug → conflict error, no partial rows (rollback)`

#### 3.3 — Invitations & member management

**What**: Invite by email, accept invite, change role, remove member — UI + API.

**Design**: `POST /api/v1/members/invitations` creates an `invitations` row (hashed token, 7-day expiry) and emails an accept link. Accepting (signed-in or after sign-up) creates a `membership` and marks `accepted_at`. Role changes/removal guard against demoting/removing the last `owner`.

**Testing**:
- `Integration (mocked email): invite → invitation row + email enqueued`
- `Integration: accept valid invite → membership created, accepted_at set`
- `Unit: accept expired invite → rejected`
- `Unit: remove last owner → blocked with explicit error`
- `Integration (RLS): tenant A cannot list tenant B's invitations`

#### 3.4 — Tenant settings & switcher UI

**What**: Settings page (name, branding `settings` JSONB) and an in-app tenant switcher.

**Design**: `settings` JSONB validated by a Zod `TenantSettings` schema (branding colors, logo URL, locale, timezone, notification prefs — Suggestion 3 shape). Switcher re-mints JWT with new active tenant.

**Testing**:
- `Unit: invalid hex color in settings → ZodError`
- `Integration: switch tenant → new JWT, subsequent queries scoped to new tenant`

---

## Phase 4: Public REST API, API Keys & Rate Limiting

### Purpose
Expose tenant operations over a versioned, OpenAPI-documented REST API authenticated by tenant API keys, with per-tenant rate limiting. After this phase the platform is programmable by tenants and external integrators, and the API surface is contract-tested.

### Tasks

#### 4.1 — API key management

**What**: Create, list, and revoke tenant API keys; authenticate requests by key.

**Design**: Key format `mtsb_<prefix8>_<secret>`. Store `key_hash` (SHA-256) + `prefix` only (Suggestion 1). Auth middleware looks up by prefix, constant-time compares hash, sets tenant context, updates `last_used_at`. Scopes array gates endpoints.

```
POST   /api/v1/api-keys        → { id, key }  (full key shown once)
GET    /api/v1/api-keys        → [{ id, name, prefix, scopes, lastUsedAt }]
DELETE /api/v1/api-keys/:id    → 204 (sets revoked_at)
```

**Testing**:
- `Integration: create key → full key returned once; refetch shows only prefix`
- `Integration: request with valid key → tenant context set`
- `Integration: revoked key → 401`
- `Unit: hash comparison is constant-time (no early return)`

#### 4.2 — OpenAPI 3.1 generation & contract tests

**What**: Generate `/api/v1/openapi.json` from Zod schemas; serve interactive docs.

**Design**: `packages/api-schema` registers each route's Zod request/response with `zod-to-openapi`; a build step emits OpenAPI 3.1 (JSON Schema 2020-12). Scalar/Swagger UI at `/api/v1/docs`. CI fails if the committed spec drifts from generated.

**Testing**:
- `Unit: generated spec validates against OpenAPI 3.1 meta-schema`
- `CI: openapi diff → no uncommitted drift`
- `Integration: each documented response matches its Zod schema (round-trip)`

#### 4.3 — Per-tenant rate limiting & quotas

**What**: Redis token-bucket rate limiter keyed by tenant + API key, with limits resolved from feature flags/plan.

**Design**: Sliding-window token bucket in Redis. Limit = `feature_flags["api_requests_per_month"]` resolved per tenant (overrides merged over plan defaults). Responses include `X-RateLimit-Remaining`/`Reset`; 429 with `Retry-After` on exhaustion.

**Testing**:
- `Integration: N requests under limit → 200; N+1 → 429 with Retry-After`
- `Integration: tenant A exhausting limit does not affect tenant B`
- `Unit: limit resolution merges tenant override over plan default`

---

## Phase 5: Billing & Subscriptions

### Purpose
Integrate Stripe Billing behind a `BillingAdapter`: plan checkout, subscription mirroring via webhooks, the Customer Portal, and metered usage reporting. After this phase tenants can subscribe, upgrade/downgrade, and be billed for usage; entitlements flow into feature flags. Implements Standard Webhooks signature verification (`standards.md`).

### Tasks

#### 5.1 — BillingAdapter interface + Stripe implementation

**What**: Define the provider-agnostic billing interface and implement Stripe.

**Design**:

```ts
interface BillingAdapter {
  createCustomer(t: Tenant): Promise<string>;
  startCheckout(p: { tenantId: string; priceId: string }): Promise<{ url: string }>;
  openPortal(p: { tenantId: string }): Promise<{ url: string }>;
  reportUsage(p: { subscriptionItemId: string; quantity: number; ts: Date }): Promise<void>;
  parseWebhook(raw: Buffer, sig: string): BillingEvent;  // verifies HMAC-SHA256
}
```

Stripe impl uses the `stripe` SDK, restricted key, idempotency keys. Lemon Squeezy impl stubbed as the second adapter (Phase 9 backlog-ready).

**Testing**:
- `Unit (Stripe mock): startCheckout → session URL`
- `Unit: parseWebhook with tampered signature → throws`

#### 5.2 — Subscription mirror & webhook handler

**What**: `/api/webhooks/stripe` verifies signatures and mirrors Stripe state into `subscriptions`/`invoices`.

**Design**: Verify signature → map events (`customer.subscription.created/updated/deleted`, `invoice.paid`, `invoice.payment_failed`) → upsert local mirror tables (Suggestion 1 recommends a synced mirror) → re-resolve feature flags → enqueue notification + outbound webhook. Idempotent on Stripe event id.

**Testing**:
- `Integration (mocked Stripe): subscription.created → subscription row + feature flags updated`
- `Integration: duplicate event id → processed once`
- `Integration: invalid signature → 400, no DB write`
- `Integration: payment_failed → subscription status past_due, alert enqueued`

#### 5.3 — Checkout, portal & metered usage

**What**: UI for plan selection/checkout, Customer Portal link, and a usage-reporting job for metered prices.

**Design**: Pricing page reads `plans`/`prices`. Metered usage aggregated daily by a BullMQ job from a `usage_events` source and reported via `reportUsage`. Entitlement check `canUse(feature)` reads resolved flags.

**Testing**:
- `Integration: checkout completes (mock) → tenant on Pro plan, max_seats raised`
- `Integration: usage job aggregates events → reportUsage called with correct quantity`
- `Unit: canUse(feature) false when over plan limit`

---

## Phase 6: Admin Console & Usage Analytics

### Purpose
Give platform operators a super-admin dashboard to manage all tenants, plus per-tenant usage analytics. After this phase operators can search, inspect, suspend, and impersonate tenants, and tenants/operators can view usage charts. Establishes the surface the AI conversational admin (Phase 8) will wrap.

### Tasks

#### 6.1 — Super-admin console

**What**: `/admin` console (platform role required) to list/search tenants, view detail, suspend/reactivate, and impersonate.

**Design**: Platform-admin access gated by a `platform_admins` allowlist (not tenant roles). Uses `withPlatform()` (RLS bypass role) for cross-tenant reads. Impersonation issues a time-boxed, audit-logged session (`audit_logs` action `tenant.impersonated`). Suspend sets `tenants.status='suspended'` (middleware then blocks tenant access).

**Testing**:
- `Integration: non-platform-admin hitting /admin → 403`
- `Integration: suspend tenant → members blocked at middleware`
- `Integration: impersonation writes audit log and expires after TTL`

#### 6.2 — Usage analytics & metrics

**What**: Per-tenant dashboards (active users, API calls, storage) and platform metrics (MRR, churn, signups).

**Design**: A daily BullMQ rollup writes `usage_daily` aggregates from `audit_logs`/rate-limiter counters. OpenTelemetry spans carry `tenant_id` for per-tenant latency. Charts via a server-rendered metrics endpoint.

**Testing**:
- `Integration: rollup job populates usage_daily for a tenant`
- `Unit: MRR computed from active subscriptions matches fixture`

---

## Phase 7: Enterprise SSO & SCIM Provisioning

### Purpose
Add the enterprise features that unlock larger buyers: SAML/OIDC SSO and SCIM 2.0 directory sync via BoxyHQ SAML Jackson. After this phase a tenant admin can connect Okta/Entra ID/Google Workspace, and users are provisioned/de-provisioned automatically. Implements RFC 7643/7644, SAML 2.0, OIDC.

### Tasks

#### 7.1 — SSO connection management (SAML Jackson)

**What**: Per-tenant SSO connection setup (metadata upload/URL) and SSO login flow.

**Design**: Embed `@boxyhq/saml-jackson` (Docker `jackson` service). Tenant admin configures a connection scoped to the tenant; login routes through Jackson's OAuth facade and maps the asserted identity to a `users`/`memberships` record. `sso_enabled` feature flag gates availability.

**Testing**:
- `Integration (mock IdP): SAML assertion → user signed in, membership ensured`
- `Integration: SSO disabled by plan → connection setup blocked`

#### 7.2 — SCIM 2.0 endpoints

**What**: Implement `/api/scim/v2/Users` and `/Groups` (RFC 7644) for automated provisioning.

**Design**: Bearer-token-secured per tenant. Support `POST/GET/PATCH/DELETE Users`, list with filters, `Groups` → roles. Deactivation (`active:false`) suspends the membership. Resources conform to RFC 7643 Core Schema.

**Testing**:
- `Integration: POST /Users → membership created (status invited)`
- `Integration: PATCH active:false → membership suspended`
- `Integration: filter userName eq → correct subset`
- `Unit: response conforms to SCIM User schema`

---

## Phase 8: AI-Native Layer

### Purpose
Deliver the project's core differentiator. After this phase, operators and developers can: generate onboarding/feature-flag configs from signup data, configure billing tiers from plain English, drive admin tasks conversationally, receive anomaly alerts with remediation, and scaffold new feature modules that respect the multi-tenant architecture. Built on the Vercel AI SDK with structured output and tool calling.

### Tasks

#### 8.1 — AI infrastructure & guardrails

**What**: Centralize model access, structured-output schemas, and tenant-scoped tool-calling.

**Design**: `packages/ai` wraps the AI SDK (Anthropic default, provider-agnostic via AI Gateway). All generations use Zod `schema` for structured output. A `Tool` registry binds each AI action to a tenant-scoped service call; tools never bypass RLS (`withTenant`). System prompt template encodes multi-tenant invariants:

```
You operate within tenant {tenantId}. Never reference or modify data outside this tenant.
All writes go through the provided tools, which enforce row-level security and audit logging.
```

**Testing**:
- `Unit: structured-output parse rejects payload failing Zod schema`
- `Integration (mock model): tool call executes within withTenant context`

#### 8.2 — Plain-English billing-tier & onboarding generation

**What**: Generate Stripe price objects + entitlement flags from a product description; generate onboarding/feature-flag config from signup data.

**Design**: `generatePlans(description)` → structured `{ plans: [{ name, priceCents, interval, features }] }`, previewed before applying to `plans`/`prices` + Stripe. `generateOnboarding(signupData, useCase)` → ordered onboarding steps + initial feature-flag values written to `tenants.settings.onboarding_state`.

**Testing**:
- `Integration (mock model): description → valid plan structures matching Zod schema`
- `Integration: apply generated plans → Stripe prices created (mock), plan_features written`
- `Unit: malformed model output → safe rejection, no DB writes`

#### 8.3 — Conversational admin & anomaly detection

**What**: A chat interface over admin operations (search/impersonate/suspend/escalate) and an anomaly scanner.

**Design**: Conversational admin exposes the Phase-6 admin operations as AI tools (platform-admin gated, every action audit-logged). Anomaly job scans per-tenant API/usage rates for outliers (z-score over rolling baseline) and emits alerts with a recommended remediation action (e.g., rate-limit, suspend, notify).

**Testing**:
- `Integration (mock model): "suspend tenant acme" → suspend tool invoked, audit logged`
- `Unit: usage spike beyond threshold → anomaly alert with remediation`
- `Integration: conversational admin denied for non-platform-admin`

#### 8.4 — Feature-module scaffolding

**What**: Generate a new feature module (Drizzle table + RLS policy + migration + API route + Zod schema + UI stub) from a natural-language description.

**Design**: `scaffoldModule(description)` produces files following repo conventions; generated tables automatically include `tenant_id` + an RLS policy and pass `lint-rls`. Output is written to a branch/diff for human review, never auto-merged.

**Testing**:
- `Integration (mock model): "add a projects feature" → migration with tenant_id + RLS policy`
- `Integration: generated module passes lint-rls and typecheck`

---

## Phase 9: Outbound Webhooks, Email Notifications & Audit Trail

### Purpose
Make the platform event-driven and compliant: emit tenant-facing webhooks (Standard Webhooks spec), send notification emails, and surface the audit trail. After this phase, tenants can subscribe to events, receive notifications, and review a complete activity log (ISO 27001 / SOC 2 / GDPR Art. 30 support).

### Tasks

#### 9.1 — Outbound webhook dispatcher

**What**: `WebhookDispatcher` delivers events to tenant `webhook_endpoints` with HMAC-SHA256 signatures and retries.

**Design**: Standard Webhooks envelope + `webhook-signature`/`webhook-id`/`webhook-timestamp` headers. BullMQ delivery with exponential backoff (config in `webhook_endpoints.config`, Suggestion 3). Optional Svix adapter for managed delivery. Events: `member.invited`, `member.joined`, `subscription.updated`, `invoice.paid`, etc.

**Testing**:
- `Integration: subscribed event → POST to endpoint with valid signature`
- `Unit: signature verifies against shared secret per Standard Webhooks`
- `Integration: failing endpoint retried with backoff, then dead-lettered`
- `Integration (RLS): tenant only receives its own events`

#### 9.2 — Notification emails

**What**: Transactional notifications (invite, billing alert, security alert) honoring tenant notification preferences.

**Design**: React Email templates; sends gated by `tenants.settings.notification_preferences`. Reuses `EmailAdapter`.

**Testing**:
- `Integration: billing_alerts disabled → no email on payment_failed`
- `Integration: security alert always sent regardless of prefs`

#### 9.3 — Audit log surface

**What**: Append-only audit logging helper and a tenant-facing/admin audit viewer.

**Design**: `audit(action, resource, resourceId?, details?)` writes to `audit_logs` within the current tenant context (captures user, IP, user-agent into `details` JSONB). Viewer paginates by `(tenant_id, created_at DESC)`. Append-only (no update/delete grants).

**Testing**:
- `Integration: tenant.created etc. produce audit rows`
- `Integration (RLS): tenant cannot read another tenant's audit logs`
- `Unit: audit_logs has no UPDATE/DELETE grant for app_user`

---

## Phase 10: White-Labeling, MCP Server & Deployment

### Purpose
Finish the differentiators and ship-readiness: per-tenant white-labeling/custom domains, an MCP server exposing the admin plane to AI agents, and production deployment configs for Docker/Vercel/AWS. After this phase the bootstrapper is forkable and deployable end-to-end.

### Tasks

#### 10.1 — White-labeling & custom domains

**What**: Per-tenant branding (logo, colors, favicon) and custom-domain routing.

**Design**: Branding from `tenants.settings.branding` injected as CSS variables in the app shell; emails use tenant logo. Custom domain verified via DNS TXT, then mapped in middleware tenant resolution.

**Testing**:
- `Integration: tenant with branding → CSS variables reflect colors`
- `Integration: verified custom domain resolves to tenant`
- `Unit: unverified domain → not routable`

#### 10.2 — MCP server for admin plane

**What**: Expose tenant-management operations as MCP tools (`standards.md` §MCP).

**Design**: `packages/mcp` server (stdio + HTTP) exposing tools: `create_tenant`, `list_tenants`, `list_members`, `update_entitlements`, `fetch_usage_metrics`, `suspend_tenant`. Auth via platform-admin token; every tool call audit-logged and RLS-respecting. Tool schemas published for AI assistants.

**Testing**:
- `Integration: MCP create_tenant tool → tenant created + audit log`
- `Integration: tool without valid token → rejected`
- `Unit: tool input schemas validate (JSON Schema)`

#### 10.3 — Deployment configs

**What**: Production Docker image, Vercel config, and AWS (ECS/RDS/ElastiCache) reference IaC.

**Design**: Hardened multi-stage Dockerfile; `vercel.json` (build + cron for jobs); AWS reference via Terraform or CDK (RDS Postgres, ElastiCache Redis, ECS Fargate). Migration step runs on deploy. Health check `/api/health`.

**Testing**:
- `Integration: docker image boots, /api/health 200`
- `CI: production build + image build succeed`
- `Smoke (Playwright): deployed preview completes sign-up → tenant create → invite`

---

## Phase 11: Tiered Isolation Upgrade Path (Optional / Enterprise)

### Purpose
Provide the upgrade path from Pool-only isolation to **Bridge** (schema-per-tenant) and **Silo** (database/Neon-branch-per-tenant) tiers, per Data Model Suggestion 4 — without forcing complexity on the MVP. After this phase, enterprise tenants can be promoted to hard isolation for SOC 2/HIPAA/data-residency contracts. This phase is opt-in and gated behind a config flag.

### Tasks

#### 11.1 — Control plane & tenant router

**What**: Introduce `tenant_routing` and a `TenantRouter` that resolves a tenant to Pool/Bridge/Silo connection info.

**Design**: Implement Suggestion 4's `tenant_routing` table and `TenantRouter` (LRU route cache, 60s TTL). Pool path = current `withTenant()`/RLS; Bridge = `SET LOCAL search_path`; Silo = dedicated connection pool. Default tier `pool` so existing behavior is unchanged.

**Testing**:
- `Integration: pool tenant routes through RLS as before (no regression)`
- `Integration: bridge tenant query scoped by search_path to its schema`
- `Unit: route cache invalidation on tier change`

#### 11.2 — Tier migration pipeline & multi-tier migrations

**What**: `TierMigrationPipeline.promoteTenant()` and a DDL orchestrator that applies migrations to pool tables, every bridge schema, and every silo database.

**Design**: Per Suggestion 4: mark `migration_status` → create target infra (schema or Neon branch) → copy data → validate checksums → atomic routing cutover → invalidate cache → grace-period cleanup. Migration orchestrator iterates all schemas/branches with per-target error handling.

**Testing**:
- `Integration: promote pool→bridge → data copied, routing updated, source kept 7d`
- `Integration: migration validation mismatch → abort, routing unchanged`
- `Integration: DDL orchestrator applies a new column to pool + all bridge schemas`

---

## Phase Summary & Dependencies

```
Phase 1: Foundation & RLS Data Layer        ─── required by everything
    │
Phase 2: Authentication & Identity          ─── requires 1
    │
Phase 3: Tenant Lifecycle & Middleware      ─── requires 1, 2  (core value ships here)
    │
    ├── Phase 4: REST API, API Keys, Rate Limiting ─── requires 3
    │       │
    │       └── Phase 5: Billing & Subscriptions   ─── requires 3 (uses 4 for usage)
    │               │
    │               └── Phase 6: Admin Console & Analytics ─── requires 3, 5
    │
    ├── Phase 7: Enterprise SSO & SCIM         ─── requires 2, 3 (parallel with 4/5/6)
    │
    └── Phase 9: Webhooks, Notifications, Audit ─── requires 3 (parallel with 4/5; uses jobs)

Phase 8: AI-Native Layer                     ─── requires 3, 5, 6 (wraps admin + billing)
Phase 10: White-label, MCP, Deployment       ─── requires 3, 6 (MCP wraps admin plane)
Phase 11: Tiered Isolation (optional)        ─── requires 1, 3 (build last / on demand)
```

**Parallelism opportunities** (after Phase 3 completes):
- Phases 4, 7, and 9 can be developed concurrently — they share only the Phase 1–3 foundation.
- Phase 5 depends on Phase 4 (metered usage) but its adapter/webhook work can start in parallel.
- Phase 8 must wait for Phases 5 and 6 (it orchestrates billing + admin operations).
- Phase 11 is independent and on-demand; it must not block the MVP (Phases 1–6 + 9).

**MVP boundary (per `features.md` Must-have):** Phases 1–6 plus Phase 9's audit trail constitute the MVP. Phases 7, 8 (partial), and 10 map to `Should-have`/`Differentiating`; Phase 11 and the backlog items map to `Nice-to-have`.

---

## Definition of Done (per phase)

A phase is complete only when all of the following hold:

1. All tasks in the phase are implemented.
2. All unit and integration tests for the phase pass (`pnpm test`, `pnpm test:integration` with Testcontainers).
3. ESLint and Prettier pass with no errors (`pnpm lint`).
4. Type checking passes (`tsc --noEmit`) across all touched workspaces.
5. **RLS lint passes** — every tenant-scoped table introduced in the phase has an enabled RLS policy (`pnpm lint:rls`). No exceptions.
6. The feature works end-to-end (verified by an integration or Playwright e2e test using real Postgres/Redis where applicable).
7. New configuration options are documented in `.env.example` and `CLAUDE.md`.
8. New API endpoints appear in the generated OpenAPI 3.1 spec, and the committed spec shows no drift (`pnpm openapi:check`).
9. Database migrations are generated, reversible where feasible, and applied cleanly to a fresh database.
10. Docker build succeeds and `/api/health` returns 200.
11. Tenant-isolation regression test (Phase 1.3) still passes — no phase may weaken cross-tenant isolation.
12. Every state-changing operation writes an `audit_logs` entry within the correct tenant context.
```
