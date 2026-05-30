# Data Model Suggestion 1: Normalized Relational (PostgreSQL with RLS)

> Project: Multi-Tenant SaaS Bootstrapper (Candidate #285)
> Generated: 2026-05-25

## Summary

A fully normalized relational schema on PostgreSQL, using **Row-Level Security (RLS)** as the primary tenant isolation mechanism. All tenants share the same database and the same set of tables, with a `tenant_id` column on every tenant-scoped table. PostgreSQL RLS policies enforce data boundaries at the database layer, making isolation independent of application-layer query filtering.

This is the most widely adopted pattern for multi-tenant SaaS products targeting SMB and mid-market customers (up to tens of thousands of tenants), and the default recommendation from Supabase, Citus, Neon, and AWS RDS documentation.

---

## Key Entities and Relationships

### Entity Overview

```
tenants ──┬── users (via memberships)
          ├── invitations
          ├── api_keys
          ├── subscriptions ── subscription_items
          ├── invoices ── invoice_line_items
          ├── feature_flags
          ├── audit_logs
          ├── webhook_endpoints
          └── [domain-specific tables]

plans ── plan_features
         prices

roles ── permissions (global, not tenant-scoped)
```

### Core Schema

```sql
-- =============================================================
-- GLOBAL TABLES (no tenant_id, shared across all tenants)
-- =============================================================

CREATE TABLE plans (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,              -- e.g. "Starter", "Pro", "Enterprise"
    slug            TEXT NOT NULL UNIQUE,
    description     TEXT,
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    trial_days      INTEGER NOT NULL DEFAULT 14,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE prices (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    plan_id             UUID NOT NULL REFERENCES plans(id),
    stripe_price_id     TEXT UNIQUE,
    currency            TEXT NOT NULL DEFAULT 'usd',
    amount_cents        INTEGER NOT NULL,          -- 0 for free tier
    interval            TEXT NOT NULL DEFAULT 'month',  -- month | year | one_time
    metered             BOOLEAN NOT NULL DEFAULT FALSE,
    usage_type          TEXT,                       -- licensed | metered
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE plan_features (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    plan_id     UUID NOT NULL REFERENCES plans(id),
    feature_key TEXT NOT NULL,                  -- e.g. "max_seats", "api_requests_per_month"
    value       TEXT NOT NULL,                  -- e.g. "10", "true", "unlimited"
    UNIQUE (plan_id, feature_key)
);

CREATE TABLE roles (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name        TEXT NOT NULL UNIQUE,           -- owner | admin | member | viewer
    description TEXT
);

CREATE TABLE permissions (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    role_id     UUID NOT NULL REFERENCES roles(id),
    resource    TEXT NOT NULL,                  -- e.g. "billing", "members", "settings"
    action      TEXT NOT NULL,                  -- e.g. "read", "write", "delete"
    UNIQUE (role_id, resource, action)
);

-- =============================================================
-- TENANT-SCOPED TABLES (tenant_id on every row, RLS enforced)
-- =============================================================

CREATE TABLE tenants (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name                TEXT NOT NULL,
    slug                TEXT NOT NULL UNIQUE,
    billing_email       TEXT,
    stripe_customer_id  TEXT UNIQUE,
    logo_url            TEXT,
    custom_domain       TEXT,
    settings            JSONB NOT NULL DEFAULT '{}',  -- branding, locale, etc.
    status              TEXT NOT NULL DEFAULT 'active', -- active | suspended | churned
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE users (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email               TEXT NOT NULL UNIQUE,
    name                TEXT,
    avatar_url          TEXT,
    password_hash       TEXT,                  -- null if OAuth-only
    email_verified_at   TIMESTAMPTZ,
    last_sign_in_at     TIMESTAMPTZ,
    auth_provider       TEXT DEFAULT 'email',  -- email | google | github | saml
    auth_provider_id    TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE memberships (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id   UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    user_id     UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role_id     UUID NOT NULL REFERENCES roles(id),
    status      TEXT NOT NULL DEFAULT 'active',  -- active | invited | suspended
    joined_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, user_id)
);

CREATE TABLE invitations (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id   UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    email       TEXT NOT NULL,
    role_id     UUID NOT NULL REFERENCES roles(id),
    invited_by  UUID REFERENCES users(id),
    token       TEXT NOT NULL UNIQUE,
    expires_at  TIMESTAMPTZ NOT NULL,
    accepted_at TIMESTAMPTZ,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE subscriptions (
    id                      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id               UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    plan_id                 UUID NOT NULL REFERENCES plans(id),
    stripe_subscription_id  TEXT UNIQUE,
    status                  TEXT NOT NULL DEFAULT 'trialing',
    -- trialing | active | past_due | canceled | unpaid
    current_period_start    TIMESTAMPTZ,
    current_period_end      TIMESTAMPTZ,
    cancel_at_period_end    BOOLEAN NOT NULL DEFAULT FALSE,
    created_at              TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at              TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE subscription_items (
    id                          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    subscription_id             UUID NOT NULL REFERENCES subscriptions(id) ON DELETE CASCADE,
    price_id                    UUID NOT NULL REFERENCES prices(id),
    stripe_subscription_item_id TEXT UNIQUE,
    quantity                    INTEGER NOT NULL DEFAULT 1,
    created_at                  TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE invoices (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL REFERENCES tenants(id),
    subscription_id     UUID REFERENCES subscriptions(id),
    stripe_invoice_id   TEXT UNIQUE,
    amount_cents        INTEGER NOT NULL,
    currency            TEXT NOT NULL DEFAULT 'usd',
    status              TEXT NOT NULL,      -- draft | open | paid | void | uncollectible
    period_start        TIMESTAMPTZ,
    period_end          TIMESTAMPTZ,
    paid_at             TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE feature_flags (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id   UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    feature_key TEXT NOT NULL,
    enabled     BOOLEAN NOT NULL DEFAULT FALSE,
    value       TEXT,                          -- override value, e.g. limit
    UNIQUE (tenant_id, feature_key)
);

CREATE TABLE api_keys (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id   UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    name        TEXT NOT NULL,
    key_hash    TEXT NOT NULL UNIQUE,           -- never store plaintext
    prefix      TEXT NOT NULL,                  -- first 8 chars for identification
    scopes      TEXT[] NOT NULL DEFAULT '{}',
    last_used_at TIMESTAMPTZ,
    expires_at  TIMESTAMPTZ,
    revoked_at  TIMESTAMPTZ,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE webhook_endpoints (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id   UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    url         TEXT NOT NULL,
    secret      TEXT NOT NULL,                 -- HMAC signing secret
    events      TEXT[] NOT NULL,               -- e.g. {"user.created", "invoice.paid"}
    is_active   BOOLEAN NOT NULL DEFAULT TRUE,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE audit_logs (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id   UUID NOT NULL REFERENCES tenants(id),
    user_id     UUID REFERENCES users(id),
    action      TEXT NOT NULL,
    resource    TEXT NOT NULL,
    resource_id UUID,
    metadata    JSONB DEFAULT '{}',
    ip_address  INET,
    user_agent  TEXT,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### RLS Policies

```sql
-- Create an application role that always goes through RLS
CREATE ROLE app_user;
ALTER TABLE memberships ENABLE ROW LEVEL SECURITY;
ALTER TABLE invitations  ENABLE ROW LEVEL SECURITY;
ALTER TABLE subscriptions ENABLE ROW LEVEL SECURITY;
ALTER TABLE feature_flags ENABLE ROW LEVEL SECURITY;
ALTER TABLE api_keys      ENABLE ROW LEVEL SECURITY;
ALTER TABLE audit_logs    ENABLE ROW LEVEL SECURITY;
ALTER TABLE webhook_endpoints ENABLE ROW LEVEL SECURITY;

-- Example policy (repeated for each tenant-scoped table)
CREATE POLICY tenant_isolation ON memberships
    USING (tenant_id = current_setting('app.current_tenant_id')::UUID);

CREATE POLICY tenant_isolation ON subscriptions
    USING (tenant_id = current_setting('app.current_tenant_id')::UUID);

-- Set tenant context per request (in middleware, using SET LOCAL for transaction scope)
-- SET LOCAL app.current_tenant_id = '<tenant-uuid>';
```

### Critical Indexes

```sql
-- Lead every tenant-scoped index with tenant_id for RLS performance
CREATE INDEX idx_memberships_tenant_user ON memberships (tenant_id, user_id);
CREATE INDEX idx_subscriptions_tenant    ON subscriptions (tenant_id, status);
CREATE INDEX idx_feature_flags_tenant    ON feature_flags (tenant_id, feature_key);
CREATE INDEX idx_api_keys_tenant         ON api_keys (tenant_id);
CREATE INDEX idx_audit_logs_tenant_time  ON audit_logs (tenant_id, created_at DESC);
CREATE INDEX idx_invoices_tenant         ON invoices (tenant_id, created_at DESC);
CREATE INDEX idx_invitations_tenant      ON invitations (tenant_id, email);
CREATE INDEX idx_webhook_endpoints_tenant ON webhook_endpoints (tenant_id);
```

---

## Pros

- **Simplest to operate.** One database, one connection pool, one migration pipeline. Schema changes apply to all tenants atomically.
- **Cost-effective at scale.** A single PostgreSQL instance can serve thousands of tenants without per-tenant infrastructure overhead.
- **RLS enforces isolation at the database layer.** Even if application code has a bug, the database will not return another tenant's data. This satisfies most SOC 2 and GDPR auditor expectations.
- **Excellent ORM and framework support.** Prisma, Drizzle, TypeORM, and Supabase all support RLS-based multi-tenancy natively.
- **Simple cross-tenant analytics.** Platform-level reports (churn, usage trends, revenue) query the same tables without ETL.
- **Battle-tested pattern.** Supabase, Neon, Citus/Azure, and AWS all recommend this as the default multi-tenancy model.

## Cons

- **Every new table needs an RLS policy.** A missed policy is a security hole. Requires discipline and CI checks (e.g., a migration lint that verifies every tenant-scoped table has RLS enabled).
- **Noisy-neighbor risk.** A single tenant running expensive queries can degrade performance for all tenants. Mitigation requires per-tenant rate limiting or connection pooling with `pg_stat_statements` monitoring.
- **Limited per-tenant schema customization.** All tenants share the same DDL; custom fields require a JSONB escape hatch (e.g., `tenants.settings`) or a separate `custom_fields` table.
- **Large-table maintenance.** Tables with hundreds of millions of rows across all tenants require partitioning (e.g., by `tenant_id` hash) for vacuum and index maintenance.
- **May not satisfy enterprise "hard isolation" requirements.** Some regulated buyers (healthcare, finance) contractually require dedicated infrastructure, which this model cannot provide without a separate deployment.

---

## Technology Recommendations

| Component | Recommendation |
|-----------|---------------|
| Database | PostgreSQL 16+ (for improved RLS performance) on Supabase, Neon, or AWS RDS |
| ORM | Drizzle ORM or Prisma with RLS middleware |
| Migrations | Drizzle Kit or Prisma Migrate, with CI lint to verify RLS on all tenant-scoped tables |
| Connection pooling | PgBouncer (transaction mode) or Supabase Pooler; always use `SET LOCAL` for tenant context |
| Auth | Clerk Organizations or Supabase Auth + custom JWT claims with `tenant_id` |
| Billing | Stripe Billing API, with a local `subscriptions` mirror table synced via webhooks |
| Feature flags | Local `feature_flags` table seeded from `plan_features` on subscription creation |
| Audit logging | Append-only `audit_logs` table; consider partitioning by month for retention policies |

---

## Migration and Scaling Considerations

1. **Composite indexes are non-negotiable.** Every tenant-scoped index must lead with `tenant_id`. Without this, RLS policy evaluation degrades from ~1ms to ~120ms on tables with millions of rows.

2. **Connection pooling with `SET LOCAL`.** Always set tenant context with `SET LOCAL app.current_tenant_id = '...'` (transaction-scoped), never `SET` (session-scoped). With PgBouncer in transaction mode, session-scoped variables leak between tenants.

3. **Table partitioning.** When any single table exceeds ~100M rows, partition by `tenant_id` hash (e.g., 32 or 64 partitions). This improves vacuum performance and enables partition-level maintenance windows.

4. **Upgrade path to schema-per-tenant.** If an enterprise customer requires hard isolation, provision them a dedicated PostgreSQL schema (or Neon branch) and route their requests via a schema resolver in middleware. The normalized schema design is identical in both modes.

5. **Horizontal scaling.** For very large deployments (100K+ tenants), consider Citus (distributed PostgreSQL) which shards by `tenant_id` across worker nodes while maintaining the same SQL interface and RLS policies.

6. **Backup and restore.** Tenant-level backup is possible via `pg_dump` with `--schema` or by querying `COPY ... WHERE tenant_id = '...'`, but full-database backups are the primary DR strategy.
