# Data Model Suggestion 3: Hybrid Relational + JSONB/Document Approach

> Project: Multi-Tenant SaaS Bootstrapper (Candidate #285)
> Generated: 2026-05-25

## Summary

A **hybrid relational + document** schema on PostgreSQL that uses normalized tables for core entities (tenants, users, subscriptions) while leveraging **JSONB columns** for flexible, tenant-customizable, and rapidly-evolving data. This approach treats PostgreSQL as both a relational database and a document store, avoiding the need for a separate NoSQL database while giving tenants the ability to extend the data model without DDL migrations.

This is the pragmatic middle ground: you get referential integrity and SQL joins for structured data, plus schemaless flexibility for tenant settings, custom fields, feature configurations, webhook payloads, and metadata -- all in one database, queryable with standard SQL and GIN indexes.

---

## Key Entities and Relationships

### Design Principles

1. **Relational for stable, queryable, cross-tenant data** -- foreign keys, constraints, and indexes on columns that are well-defined and frequently filtered/joined.
2. **JSONB for flexible, tenant-specific, or rapidly-evolving data** -- settings, preferences, custom fields, metadata, and any data shape that varies per tenant or changes frequently during development.
3. **Validate JSONB at the application layer** -- use Zod, JSON Schema, or Pydantic to enforce structure before insertion, even though the database column is schemaless.

### Core Schema

```sql
-- =============================================================
-- GLOBAL TABLES
-- =============================================================

CREATE TABLE plans (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL UNIQUE,
    description     TEXT,
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    trial_days      INTEGER NOT NULL DEFAULT 14,
    -- JSONB: feature entitlements, limits, and metadata
    -- Avoids a separate plan_features join table
    features        JSONB NOT NULL DEFAULT '{}',
    /*  Example features value:
        {
          "max_seats": 10,
          "max_projects": 5,
          "api_requests_per_month": 10000,
          "sso_enabled": false,
          "custom_domain": false,
          "white_label": false,
          "support_tier": "email",
          "storage_gb": 5
        }
    */
    pricing         JSONB NOT NULL DEFAULT '[]',
    /*  Example pricing value:
        [
          {"interval": "month", "currency": "usd", "amount_cents": 2900, "stripe_price_id": "price_xxx"},
          {"interval": "year",  "currency": "usd", "amount_cents": 29000, "stripe_price_id": "price_yyy"}
        ]
    */
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- GIN index for querying plan features
CREATE INDEX idx_plans_features ON plans USING GIN (features);

CREATE TABLE roles (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name        TEXT NOT NULL UNIQUE,
    -- JSONB: permission matrix instead of a separate permissions table
    permissions JSONB NOT NULL DEFAULT '{}',
    /*  Example permissions value:
        {
          "billing":  {"read": true, "write": true, "delete": false},
          "members":  {"read": true, "write": true, "delete": true},
          "settings": {"read": true, "write": true, "delete": false},
          "api_keys": {"read": true, "write": false, "delete": false}
        }
    */
    description TEXT
);

-- =============================================================
-- TENANT-SCOPED TABLES (RLS enforced)
-- =============================================================

CREATE TABLE tenants (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name                TEXT NOT NULL,
    slug                TEXT NOT NULL UNIQUE,
    billing_email       TEXT,
    stripe_customer_id  TEXT UNIQUE,
    status              TEXT NOT NULL DEFAULT 'active',

    -- JSONB: branding, locale, and custom configuration
    settings            JSONB NOT NULL DEFAULT '{}',
    /*  Example settings value:
        {
          "branding": {
            "logo_url": "https://...",
            "primary_color": "#3B82F6",
            "favicon_url": "https://..."
          },
          "locale": "en-US",
          "timezone": "America/New_York",
          "custom_domain": "app.example.com",
          "notification_preferences": {
            "billing_alerts": true,
            "weekly_digest": false,
            "security_alerts": true
          }
        }
    */

    -- JSONB: tenant-specific feature overrides (merged with plan defaults)
    feature_overrides   JSONB NOT NULL DEFAULT '{}',
    /*  Example: {"max_seats": 25, "api_requests_per_month": 50000}
        These override the plan-level defaults for this specific tenant
    */

    -- JSONB: custom fields defined by the tenant for their own data
    custom_fields_schema JSONB NOT NULL DEFAULT '[]',
    /*  Example: defines what custom fields this tenant has added
        [
          {"key": "department", "label": "Department", "type": "select",
           "options": ["Engineering", "Sales", "Marketing"]},
          {"key": "employee_id", "label": "Employee ID", "type": "text", "required": true}
        ]
    */

    onboarding_state    JSONB NOT NULL DEFAULT '{}',
    /*  Tracks onboarding progress without a dedicated table:
        {"completed_steps": ["create_org", "invite_team"], "current_step": "connect_billing"}
    */

    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_tenants_settings ON tenants USING GIN (settings);
CREATE INDEX idx_tenants_features ON tenants USING GIN (feature_overrides);

CREATE TABLE users (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email               TEXT NOT NULL UNIQUE,
    name                TEXT,
    password_hash       TEXT,
    email_verified_at   TIMESTAMPTZ,
    last_sign_in_at     TIMESTAMPTZ,

    -- JSONB: auth provider details and profile metadata
    auth_profile        JSONB NOT NULL DEFAULT '{}',
    /*  Example:
        {
          "provider": "google",
          "provider_id": "google-uid-123",
          "avatar_url": "https://...",
          "oauth_metadata": {"access_token_expires": "..."}
        }
    */

    -- JSONB: user preferences (global, not tenant-specific)
    preferences         JSONB NOT NULL DEFAULT '{}',
    /*  Example: {"theme": "dark", "language": "en", "notifications": {"email": true, "push": false}} */

    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE memberships (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id   UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    user_id     UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role_id     UUID NOT NULL REFERENCES roles(id),
    status      TEXT NOT NULL DEFAULT 'active',
    joined_at   TIMESTAMPTZ NOT NULL DEFAULT now(),

    -- JSONB: tenant-specific custom field values for this member
    custom_fields JSONB NOT NULL DEFAULT '{}',
    /*  Values matching the tenant's custom_fields_schema:
        {"department": "Engineering", "employee_id": "EMP-042"}
    */

    -- JSONB: per-member metadata
    metadata    JSONB NOT NULL DEFAULT '{}',
    /*  {"invited_by": "user-uuid", "invitation_source": "email", "notes": "..."} */

    UNIQUE (tenant_id, user_id)
);

CREATE INDEX idx_memberships_tenant ON memberships (tenant_id, user_id);
CREATE INDEX idx_memberships_custom ON memberships USING GIN (custom_fields);

CREATE TABLE invitations (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id   UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    email       TEXT NOT NULL,
    role_id     UUID NOT NULL REFERENCES roles(id),
    invited_by  UUID REFERENCES users(id),
    token       TEXT NOT NULL UNIQUE,
    expires_at  TIMESTAMPTZ NOT NULL,
    accepted_at TIMESTAMPTZ,
    metadata    JSONB NOT NULL DEFAULT '{}',  -- custom message, source, etc.
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE subscriptions (
    id                      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id               UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    plan_id                 UUID NOT NULL REFERENCES plans(id),
    stripe_subscription_id  TEXT UNIQUE,
    status                  TEXT NOT NULL DEFAULT 'trialing',
    current_period_start    TIMESTAMPTZ,
    current_period_end      TIMESTAMPTZ,
    cancel_at_period_end    BOOLEAN NOT NULL DEFAULT FALSE,

    -- JSONB: Stripe subscription metadata synced via webhooks
    stripe_metadata         JSONB NOT NULL DEFAULT '{}',
    /*  Mirrors relevant Stripe subscription fields:
        {
          "items": [{"price_id": "price_xxx", "quantity": 5}],
          "discount": {"coupon_id": "LAUNCH20", "percent_off": 20},
          "payment_method": {"brand": "visa", "last4": "4242", "exp_month": 12, "exp_year": 2027}
        }
    */

    -- JSONB: usage tracking for metered billing
    usage_summary           JSONB NOT NULL DEFAULT '{}',
    /*  {"api_requests": 4523, "storage_bytes": 1073741824, "active_users": 8} */

    created_at              TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at              TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_subscriptions_tenant ON subscriptions (tenant_id, status);

CREATE TABLE api_keys (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id   UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    name        TEXT NOT NULL,
    key_hash    TEXT NOT NULL UNIQUE,
    prefix      TEXT NOT NULL,
    scopes      TEXT[] NOT NULL DEFAULT '{}',
    last_used_at TIMESTAMPTZ,
    expires_at  TIMESTAMPTZ,
    revoked_at  TIMESTAMPTZ,
    metadata    JSONB NOT NULL DEFAULT '{}',  -- rate limits, IP allowlist, etc.
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE webhook_endpoints (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id   UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    url         TEXT NOT NULL,
    secret      TEXT NOT NULL,
    events      TEXT[] NOT NULL,
    is_active   BOOLEAN NOT NULL DEFAULT TRUE,

    -- JSONB: delivery configuration and stats
    config      JSONB NOT NULL DEFAULT '{}',
    /*  {
          "retry_policy": {"max_retries": 5, "backoff_ms": [1000, 5000, 30000]},
          "headers": {"X-Custom-Header": "value"},
          "last_delivery": {"status": 200, "at": "2026-05-20T..."}
        }
    */
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE audit_logs (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id   UUID NOT NULL REFERENCES tenants(id),
    user_id     UUID REFERENCES users(id),
    action      TEXT NOT NULL,
    resource    TEXT NOT NULL,
    resource_id UUID,

    -- JSONB: captures the full context of the action
    details     JSONB NOT NULL DEFAULT '{}',
    /*  {
          "changes": {"name": {"from": "Old Co", "to": "New Co"}},
          "ip_address": "203.0.113.42",
          "user_agent": "Mozilla/5.0...",
          "request_id": "req_abc123"
        }
    */

    created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_logs_tenant ON audit_logs (tenant_id, created_at DESC);
CREATE INDEX idx_audit_logs_details ON audit_logs USING GIN (details);

-- =============================================================
-- RLS POLICIES (same pattern as Suggestion 1)
-- =============================================================

ALTER TABLE memberships       ENABLE ROW LEVEL SECURITY;
ALTER TABLE invitations       ENABLE ROW LEVEL SECURITY;
ALTER TABLE subscriptions     ENABLE ROW LEVEL SECURITY;
ALTER TABLE api_keys          ENABLE ROW LEVEL SECURITY;
ALTER TABLE webhook_endpoints ENABLE ROW LEVEL SECURITY;
ALTER TABLE audit_logs        ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation ON memberships
    USING (tenant_id = current_setting('app.current_tenant_id')::UUID);
-- (repeat for each tenant-scoped table)
```

### Feature Resolution Logic

```typescript
// Merge plan features with tenant-specific overrides
function resolveFeatures(plan: Plan, tenant: Tenant): FeatureSet {
  return {
    ...plan.features,          // Plan defaults: {"max_seats": 10, "sso_enabled": false}
    ...tenant.featureOverrides // Tenant overrides: {"max_seats": 25}
    // Result: {"max_seats": 25, "sso_enabled": false}
  };
}

// Check a feature limit
function canAddMember(tenant: Tenant, plan: Plan, currentMemberCount: number): boolean {
  const features = resolveFeatures(plan, tenant);
  if (features.max_seats === 'unlimited') return true;
  return currentMemberCount < features.max_seats;
}
```

### Custom Fields System

```typescript
// Validate custom field values against the tenant's schema
import { z } from 'zod';

function buildCustomFieldsValidator(schema: CustomFieldDef[]): z.ZodType {
  const shape: Record<string, z.ZodType> = {};
  for (const field of schema) {
    let validator: z.ZodType;
    switch (field.type) {
      case 'text':    validator = z.string(); break;
      case 'number':  validator = z.number(); break;
      case 'boolean': validator = z.boolean(); break;
      case 'select':  validator = z.enum(field.options as [string, ...string[]]); break;
      case 'date':    validator = z.string().datetime(); break;
      default:        validator = z.unknown();
    }
    shape[field.key] = field.required ? validator : validator.optional();
  }
  return z.object(shape);
}

// Usage: validate before inserting into memberships.custom_fields
const validator = buildCustomFieldsValidator(tenant.customFieldsSchema);
const result = validator.safeParse(incomingCustomFields);
```

---

## Pros

- **Best of both worlds.** Relational integrity for core entities (foreign keys, unique constraints, joins) plus document flexibility for everything that varies per tenant or evolves rapidly.
- **Single database technology.** No need to operate a separate MongoDB, DynamoDB, or Redis alongside PostgreSQL. Fewer moving parts means fewer failure modes.
- **Tenant-extensible without migrations.** Tenants can define custom fields, configure branding, and set preferences without any DDL changes. The `custom_fields_schema` + `custom_fields` pattern gives each tenant a tailored experience.
- **Faster iteration during early development.** JSONB columns let developers ship features before the schema is fully settled. When a JSONB field stabilizes, it can be promoted to a dedicated column with a simple migration.
- **Efficient feature flag resolution.** Storing plan features as JSONB and tenant overrides as JSONB enables a simple merge operation at the application layer, avoiding the N+1 queries of a normalized `plan_features` join table.
- **Rich querying with GIN indexes.** PostgreSQL GIN indexes on JSONB columns support containment queries (`@>`) and existence checks (`?`) with good performance, enabling queries like "find all tenants where SSO is enabled."
- **Reduced table count.** By collapsing metadata, preferences, and configuration into JSONB columns on existing entities, the schema has fewer tables, fewer joins, and simpler ORM mappings.

## Cons

- **No referential integrity inside JSONB.** Foreign key relationships cannot span into JSONB values. If `pricing[].stripe_price_id` references a Stripe object, that reference is not database-enforced.
- **JSONB validation is application-layer only.** The database does not enforce the shape of JSONB values. A bug in the application can insert malformed data. Mitigated by Zod/JSON Schema validation in middleware, but this is a discipline requirement.
- **Query complexity for deep JSONB paths.** While simple JSONB queries are fast with GIN indexes, queries involving multiple nested paths or aggregations across JSONB arrays can become complex and slower than normalized equivalents.
- **ORM support varies.** Not all ORMs handle JSONB ergonomically. Prisma's `Json` type works but lacks typed access; Drizzle has better JSONB support. Custom TypeScript types are needed to get type safety on JSONB columns.
- **Schema documentation burden.** The actual data model is split between DDL (visible in migrations) and JSONB contracts (documented in code/comments). Without discipline, JSONB columns become a dumping ground for unstructured data.
- **Document size limits.** While PostgreSQL supports JSONB documents up to 1GB, performance degrades for documents over ~1MB. Tenant settings and feature configs are typically small, but audit log `details` could grow if not bounded.

---

## Technology Recommendations

| Component | Recommendation |
|-----------|---------------|
| Database | PostgreSQL 16+ (Supabase, Neon, or AWS RDS) |
| ORM | Drizzle ORM (best JSONB ergonomics in TypeScript) or Prisma with typed JSONB helpers |
| JSONB validation | Zod schemas co-located with each JSONB column definition |
| Indexing | GIN indexes on frequently queried JSONB columns; expression indexes on specific keys (e.g., `CREATE INDEX ON tenants ((settings->>'custom_domain'))`) |
| Auth | Clerk Organizations with JWT claims for `tenant_id` |
| Billing | Stripe Billing with `stripe_metadata` JSONB column synced via webhooks |
| Feature management | JSONB-based plan features + tenant overrides; no external feature flag service needed for MVP |
| Custom fields | Application-layer Zod validation against `custom_fields_schema` |

---

## Migration and Scaling Considerations

1. **Promote JSONB to columns when stable.** When a JSONB key is queried frequently and its schema is stable, extract it to a dedicated column. For example, if `settings->>'custom_domain'` is checked on every request, add a `custom_domain TEXT` column and backfill from the JSONB.

2. **JSONB column size monitoring.** Add application-level checks that warn if a JSONB column exceeds a size threshold (e.g., 100KB). This prevents a single tenant's settings from degrading write performance.

3. **GIN index maintenance.** GIN indexes on JSONB columns are write-amplified (each write updates the index). For write-heavy tables like `audit_logs`, consider a partial GIN index or skip the GIN index entirely if full-text JSONB search is not needed.

4. **Migration from pure JSONB to hybrid.** If starting with maximum JSONB flexibility (e.g., entire entities as JSONB), refactoring toward the hybrid model is straightforward: add columns, backfill from JSONB, add constraints, and remove the JSONB key.

5. **Cross-tenant analytics.** JSONB aggregation functions (`jsonb_each`, `jsonb_array_elements`) enable platform-level analytics without ETL, but complex aggregations should be pre-computed into a materialized view or analytics table.

6. **Compatibility with AI scaffolding.** The JSONB-heavy model is particularly well-suited for AI-generated schema extensions. An AI assistant can add new keys to `settings` or `features` JSONB without generating migration files, reducing the risk of AI-generated DDL errors.
