# Data Model Suggestion 4: Tiered Multi-Tenancy with Tenant Isolation Promotion

> Project: Multi-Tenant SaaS Bootstrapper (Candidate #285)
> Generated: 2026-05-25

## Summary

A **tiered multi-tenancy** architecture that dynamically assigns tenants to one of three isolation levels based on their subscription plan, compliance requirements, or growth trajectory:

| Tier | Isolation Model | Target Segment |
|------|----------------|----------------|
| **Pool** (default) | Shared tables + RLS | Free, Starter, and Pro tenants |
| **Bridge** | Schema-per-tenant in the same database | Growth and Business tenants requiring logical isolation |
| **Silo** | Database-per-tenant (Neon branch or dedicated instance) | Enterprise tenants with contractual hard-isolation requirements |

A **tenant routing layer** resolves each request to the correct database/schema/connection based on the tenant's assigned tier, stored in a central **control plane** database. Tenants can be **promoted** from Pool to Bridge to Silo as they grow, without downtime or data loss, using a built-in migration pipeline.

This is the pattern used by mature B2B SaaS platforms (AWS, Salesforce, Shopify) and is uniquely suited to a bootstrapper that serves both indie hackers and enterprise buyers from the same codebase.

---

## Key Entities and Relationships

### Architecture Overview

```
                    ┌──────────────────────────────────┐
                    │       CONTROL PLANE DATABASE      │
                    │  (always shared, never tenant data)│
                    │                                    │
                    │  tenants / plans / routing_config  │
                    │  users / global_audit_logs         │
                    └──────────────┬─────────────────────┘
                                   │
                    ┌──────────────┴─────────────────────┐
                    │         TENANT ROUTER              │
                    │  Resolves tenant → connection info  │
                    └──┬───────────┬──────────────┬──────┘
                       │           │              │
              ┌────────▼──┐  ┌────▼─────┐  ┌─────▼──────┐
              │   POOL    │  │  BRIDGE  │  │    SILO    │
              │ Shared DB │  │ Schema/  │  │ Dedicated  │
              │ + RLS     │  │ tenant   │  │ DB/branch  │
              │           │  │          │  │            │
              │ (1000s of │  │ (10s-100s│  │ (1-10      │
              │  tenants) │  │  tenants)│  │  tenants)  │
              └───────────┘  └──────────┘  └────────────┘
```

### Control Plane Schema

```sql
-- =============================================================
-- CONTROL PLANE DATABASE
-- Contains no tenant business data, only routing and identity
-- =============================================================

CREATE TABLE tenants (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name                TEXT NOT NULL,
    slug                TEXT NOT NULL UNIQUE,
    billing_email       TEXT,
    stripe_customer_id  TEXT UNIQUE,
    status              TEXT NOT NULL DEFAULT 'active',
    -- active | suspended | migrating | churned

    -- Isolation tier assignment
    isolation_tier      TEXT NOT NULL DEFAULT 'pool',
    -- pool | bridge | silo

    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Where to find this tenant's data
CREATE TABLE tenant_routing (
    tenant_id           UUID PRIMARY KEY REFERENCES tenants(id),
    isolation_tier      TEXT NOT NULL DEFAULT 'pool',

    -- Pool tier: just the tenant_id (data lives in shared tables)
    -- Bridge tier: schema name in the shared database
    schema_name         TEXT,                  -- e.g. "tenant_abc123"

    -- Silo tier: dedicated connection details
    database_host       TEXT,                  -- e.g. "ep-xyz.neon.tech"
    database_name       TEXT,                  -- e.g. "tenant_abc123"
    database_port       INTEGER DEFAULT 5432,
    connection_pool_url TEXT,                  -- pooled connection string

    -- Neon-specific (for branch-per-tenant)
    neon_project_id     TEXT,
    neon_branch_id      TEXT,

    -- Migration tracking
    migration_status    TEXT DEFAULT 'none',
    -- none | scheduled | in_progress | validating | completed | failed
    migration_target    TEXT,                  -- target tier
    migration_started_at TIMESTAMPTZ,
    migration_completed_at TIMESTAMPTZ,

    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE plans (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name                TEXT NOT NULL,
    slug                TEXT NOT NULL UNIQUE,
    description         TEXT,
    is_active           BOOLEAN NOT NULL DEFAULT TRUE,
    trial_days          INTEGER NOT NULL DEFAULT 14,

    -- Which isolation tier this plan grants
    default_isolation_tier TEXT NOT NULL DEFAULT 'pool',
    -- pool (Free/Starter) | bridge (Business) | silo (Enterprise)

    features            JSONB NOT NULL DEFAULT '{}',
    pricing             JSONB NOT NULL DEFAULT '[]',
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Global user directory (users may belong to multiple tenants)
CREATE TABLE users (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email               TEXT NOT NULL UNIQUE,
    name                TEXT,
    password_hash       TEXT,
    email_verified_at   TIMESTAMPTZ,
    last_sign_in_at     TIMESTAMPTZ,
    auth_provider       TEXT DEFAULT 'email',
    auth_provider_id    TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE memberships (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id   UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    user_id     UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role        TEXT NOT NULL DEFAULT 'member',
    status      TEXT NOT NULL DEFAULT 'active',
    joined_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, user_id)
);

-- Subscriptions tracked in control plane (billing is not tenant data)
CREATE TABLE subscriptions (
    id                      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id               UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    plan_id                 UUID NOT NULL REFERENCES plans(id),
    stripe_subscription_id  TEXT UNIQUE,
    status                  TEXT NOT NULL DEFAULT 'trialing',
    current_period_start    TIMESTAMPTZ,
    current_period_end      TIMESTAMPTZ,
    cancel_at_period_end    BOOLEAN NOT NULL DEFAULT FALSE,
    created_at              TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at              TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Global audit log for control plane operations
CREATE TABLE control_plane_audit_logs (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id   UUID REFERENCES tenants(id),
    user_id     UUID REFERENCES users(id),
    action      TEXT NOT NULL,     -- tenant.created, tier.promoted, migration.started
    details     JSONB NOT NULL DEFAULT '{}',
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_routing_tier ON tenant_routing (isolation_tier);
CREATE INDEX idx_audit_tenant ON control_plane_audit_logs (tenant_id, created_at DESC);
```

### Pool Tier Schema (Shared Tables + RLS)

```sql
-- =============================================================
-- POOL TIER: all pool tenants share these tables
-- Identical to Suggestion 1, but scoped to pool tenants only
-- =============================================================

-- In the "pool" schema (or default public schema)

CREATE TABLE pool.projects (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id   UUID NOT NULL,
    name        TEXT NOT NULL,
    description TEXT,
    settings    JSONB NOT NULL DEFAULT '{}',
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE pool.api_keys (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id   UUID NOT NULL,
    name        TEXT NOT NULL,
    key_hash    TEXT NOT NULL UNIQUE,
    prefix      TEXT NOT NULL,
    scopes      TEXT[] NOT NULL DEFAULT '{}',
    last_used_at TIMESTAMPTZ,
    expires_at  TIMESTAMPTZ,
    revoked_at  TIMESTAMPTZ,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE pool.feature_flags (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id   UUID NOT NULL,
    feature_key TEXT NOT NULL,
    enabled     BOOLEAN NOT NULL DEFAULT FALSE,
    value       TEXT,
    UNIQUE (tenant_id, feature_key)
);

CREATE TABLE pool.webhook_endpoints (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id   UUID NOT NULL,
    url         TEXT NOT NULL,
    secret      TEXT NOT NULL,
    events      TEXT[] NOT NULL,
    is_active   BOOLEAN NOT NULL DEFAULT TRUE,
    config      JSONB NOT NULL DEFAULT '{}',
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE pool.audit_logs (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id   UUID NOT NULL,
    user_id     UUID,
    action      TEXT NOT NULL,
    resource    TEXT NOT NULL,
    resource_id UUID,
    details     JSONB NOT NULL DEFAULT '{}',
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- RLS on all pool tables
ALTER TABLE pool.projects          ENABLE ROW LEVEL SECURITY;
ALTER TABLE pool.api_keys          ENABLE ROW LEVEL SECURITY;
ALTER TABLE pool.feature_flags     ENABLE ROW LEVEL SECURITY;
ALTER TABLE pool.webhook_endpoints ENABLE ROW LEVEL SECURITY;
ALTER TABLE pool.audit_logs        ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation ON pool.projects
    USING (tenant_id = current_setting('app.current_tenant_id')::UUID);
-- (repeat for each pool table)

-- Composite indexes leading with tenant_id
CREATE INDEX idx_pool_projects_tenant ON pool.projects (tenant_id, created_at DESC);
CREATE INDEX idx_pool_api_keys_tenant ON pool.api_keys (tenant_id);
CREATE INDEX idx_pool_audit_tenant    ON pool.audit_logs (tenant_id, created_at DESC);
```

### Bridge Tier Schema Template

```sql
-- =============================================================
-- BRIDGE TIER: one schema per tenant, created dynamically
-- Same table structure, but no tenant_id column needed
-- =============================================================

-- Template: CREATE SCHEMA tenant_{slug};

-- Example for tenant "acme":
CREATE SCHEMA tenant_acme;

CREATE TABLE tenant_acme.projects (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name        TEXT NOT NULL,
    description TEXT,
    settings    JSONB NOT NULL DEFAULT '{}',
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT now()
    -- No tenant_id column needed: the schema IS the tenant boundary
);

CREATE TABLE tenant_acme.api_keys (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name        TEXT NOT NULL,
    key_hash    TEXT NOT NULL UNIQUE,
    prefix      TEXT NOT NULL,
    scopes      TEXT[] NOT NULL DEFAULT '{}',
    last_used_at TIMESTAMPTZ,
    expires_at  TIMESTAMPTZ,
    revoked_at  TIMESTAMPTZ,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE tenant_acme.feature_flags (
    feature_key TEXT PRIMARY KEY,
    enabled     BOOLEAN NOT NULL DEFAULT FALSE,
    value       TEXT
);

CREATE TABLE tenant_acme.webhook_endpoints (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    url         TEXT NOT NULL,
    secret      TEXT NOT NULL,
    events      TEXT[] NOT NULL,
    is_active   BOOLEAN NOT NULL DEFAULT TRUE,
    config      JSONB NOT NULL DEFAULT '{}',
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE tenant_acme.audit_logs (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id     UUID,
    action      TEXT NOT NULL,
    resource    TEXT NOT NULL,
    resource_id UUID,
    details     JSONB NOT NULL DEFAULT '{}',
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Tenant middleware sets: SET LOCAL search_path = 'tenant_acme', 'public';
```

### Tenant Router (Application Layer)

```typescript
// Middleware: resolve tenant and configure database connection

interface TenantRoute {
  tenantId: string;
  isolationTier: 'pool' | 'bridge' | 'silo';
  schemaName?: string;
  connectionUrl?: string;
}

class TenantRouter {
  private routeCache: Map<string, TenantRoute> = new Map();

  async resolveRoute(tenantSlug: string): Promise<TenantRoute> {
    // Check cache first (TTL: 60s)
    const cached = this.routeCache.get(tenantSlug);
    if (cached) return cached;

    // Query control plane
    const route = await this.controlPlaneDb.query(`
      SELECT t.id AS tenant_id, tr.isolation_tier, tr.schema_name,
             tr.connection_pool_url
      FROM tenants t
      JOIN tenant_routing tr ON tr.tenant_id = t.id
      WHERE t.slug = $1 AND t.status = 'active'
    `, [tenantSlug]);

    this.routeCache.set(tenantSlug, route);
    return route;
  }

  async getConnection(route: TenantRoute): Promise<DatabaseConnection> {
    switch (route.isolationTier) {
      case 'pool':
        // Use shared connection pool, set RLS context
        const poolConn = await this.sharedPool.connect();
        await poolConn.query(
          `SET LOCAL app.current_tenant_id = $1`,
          [route.tenantId]
        );
        return poolConn;

      case 'bridge':
        // Use shared connection pool, set search_path to tenant schema
        const bridgeConn = await this.sharedPool.connect();
        await bridgeConn.query(
          `SET LOCAL search_path = $1, 'public'`,
          [route.schemaName]
        );
        return bridgeConn;

      case 'silo':
        // Use dedicated connection pool for this tenant
        return await this.getSiloPool(route.connectionUrl!).connect();
    }
  }
}
```

### Tier Promotion Pipeline

```typescript
// Migrate a tenant from one isolation tier to another

class TierMigrationPipeline {
  async promoteTenant(
    tenantId: string,
    targetTier: 'bridge' | 'silo'
  ): Promise<void> {
    // 1. Mark migration as in_progress
    await this.controlPlane.query(`
      UPDATE tenant_routing
      SET migration_status = 'in_progress',
          migration_target = $1,
          migration_started_at = now()
      WHERE tenant_id = $2
    `, [targetTier, tenantId]);

    // 2. Create target infrastructure
    if (targetTier === 'bridge') {
      await this.createTenantSchema(tenantId);
    } else if (targetTier === 'silo') {
      await this.createNeonBranch(tenantId);
      // or: await this.createDedicatedInstance(tenantId);
    }

    // 3. Copy data from source to target
    await this.copyTenantData(tenantId, targetTier);

    // 4. Validate row counts and checksums
    await this.validateMigration(tenantId, targetTier);

    // 5. Atomic cutover: update routing in a single transaction
    await this.controlPlane.query(`
      UPDATE tenant_routing
      SET isolation_tier = $1,
          schema_name = $2,
          database_host = $3,
          migration_status = 'completed',
          migration_completed_at = now()
      WHERE tenant_id = $4
    `, [targetTier, schemaName, dbHost, tenantId]);

    // 6. Invalidate route cache
    this.router.invalidateCache(tenantId);

    // 7. Clean up source data (after grace period)
    // Scheduled job removes old pool rows after 7 days
  }

  private async createNeonBranch(tenantId: string): Promise<void> {
    // Neon API: create a branch for this tenant
    // Branch is a copy-on-write fork — instant, no data duplication
    const branch = await this.neonApi.createBranch({
      projectId: this.neonProjectId,
      parentId: this.templateBranchId,   // branch from a template with empty schema
      name: `tenant-${tenantId}`,
    });
    // Run migrations on the new branch
    await this.runMigrations(branch.connectionUri);
  }
}
```

---

## Pros

- **Serves all customer segments from one codebase.** Indie hackers get the cost efficiency of shared tables; enterprise buyers get the hard isolation they contractually require. No separate "enterprise edition" needed.
- **Smooth upgrade path.** Tenants migrate from Pool to Bridge to Silo as they grow, without downtime. The control plane routing change is atomic.
- **Compliance-ready.** Enterprise tenants in the Silo tier have physically separate databases, satisfying SOC 2 Type II, HIPAA, and contractual data isolation requirements.
- **Noisy-neighbor elimination for premium tenants.** Bridge and Silo tenants have dedicated compute and storage resources, insulating them from Pool-tier load spikes.
- **Cost-optimized.** Free and Starter tenants share infrastructure (Pool), keeping per-tenant cost near zero. Enterprise tenants in Silo tier have dedicated resources that are billed back through higher subscription fees.
- **Neon branching makes Silo tier affordable.** Neon's copy-on-write branching creates a "database per tenant" without the traditional per-instance provisioning cost. Branches scale to zero when idle, charging only for storage.
- **Natural upsell mechanism.** "Upgrade to Business for dedicated schema isolation" or "Upgrade to Enterprise for dedicated database" becomes a concrete, technical value proposition.

## Cons

- **Highest architectural complexity.** Three isolation modes, a tenant router, a migration pipeline, and a control plane database. This is the most complex option to build, test, and operate.
- **Schema migrations must run three ways.** DDL changes must be applied to: (1) the shared pool tables, (2) every Bridge schema, and (3) every Silo database/branch. Requires a migration orchestrator.
- **Testing matrix multiplies.** Every feature must be tested against all three tiers. Integration tests need fixtures for Pool, Bridge, and Silo tenants.
- **Operational overhead.** Monitoring, backup, and alerting must cover all three tiers. A Silo tenant's database going down is a single-tenant incident, not a platform-wide event.
- **Bridge tier hits PostgreSQL catalog limits.** PostgreSQL performance degrades with 5,000-10,000 schemas in a single database. Bridge tier is practical for hundreds of tenants, not thousands.
- **Connection pool management.** Pool and Bridge tenants share a connection pool; Silo tenants need dedicated pools. The total connection count across all pools requires careful management.
- **Overkill for MVP.** A bootstrapper's primary audience (indie hackers, seed-stage startups) rarely needs Bridge or Silo tiers initially. Building all three tiers from day one delays time-to-market.

---

## Technology Recommendations

| Component | Recommendation |
|-----------|---------------|
| Control plane DB | PostgreSQL on Supabase or AWS RDS (always available, not scale-to-zero) |
| Pool tier DB | Same PostgreSQL instance as control plane, separate schema (`pool.*`) |
| Bridge tier | Schemas in the same PostgreSQL instance (`tenant_*.*`) |
| Silo tier | **Neon branches** (instant, scale-to-zero, CoW, ~$0.35/GB-month storage) or dedicated RDS instances for largest tenants |
| Tenant router | Custom middleware with an in-memory LRU cache (60s TTL) backed by control plane queries |
| Migration orchestrator | Custom pipeline with Drizzle Kit for DDL; `pg_dump`/`pg_restore` for data migration |
| Auth | Clerk Organizations or WorkOS (JWT `tenant_id` claim drives router lookup) |
| Billing | Stripe Billing; plan.`default_isolation_tier` determines initial placement |
| Monitoring | Per-tier dashboards: Pool-level `pg_stat_statements`, Bridge-level schema metrics, Silo-level Neon dashboard |

---

## Migration and Scaling Considerations

1. **Start with Pool only.** Build the Pool tier (Suggestion 1) for MVP. Add Bridge tier when the first customer requests logical isolation. Add Silo tier when the first enterprise contract requires it. The control plane and routing layer can be built incrementally.

2. **Schema migration orchestrator.** Use a tool like Drizzle Kit or a custom migration runner that iterates over all Bridge schemas and Silo databases. Neon's branch API supports running migrations on all branches programmatically. Consider a "migration queue" that applies DDL to each tenant sequentially with error handling and rollback.

3. **Neon branch lifecycle management.** Neon branches scale to zero compute when idle (within 5 minutes by default). For Silo tenants, set a minimum compute size to avoid cold-start latency. Monitor branch storage growth and archive inactive branches.

4. **Bridge-to-Silo promotion.** When a Bridge tenant outgrows schema isolation (e.g., needs dedicated compute, regulatory requirement), promote to Silo by: (1) creating a Neon branch, (2) using `pg_dump --schema=tenant_xxx | pg_restore` to copy data, (3) validating, (4) updating `tenant_routing`. The Bridge schema is kept read-only for 7 days as a rollback safety net.

5. **Cross-tier analytics.** Platform-level analytics (revenue, usage, churn) must aggregate across all three tiers. Use a read replica or data pipeline (e.g., Fivetran, Airbyte) that pulls from Pool tables, Bridge schemas, and Silo databases into a unified analytics warehouse.

6. **Connection pool budgeting.** A single PostgreSQL instance supports ~500 concurrent connections (with PgBouncer). Budget: 400 for Pool+Bridge tenants (shared), 100 reserved for admin/migration operations. Each Silo tenant gets its own Neon connection pool (default 50 connections).

7. **Disaster recovery by tier.** Pool and Bridge: standard PostgreSQL backup (pg_basebackup, WAL archiving). Silo/Neon: branch-level point-in-time recovery (PITR) with 7-day retention on Neon Scale tier. Tenant-level restore is trivial for Silo (restore branch), harder for Pool (requires row-level filtering from backup).

8. **Regulatory compliance mapping.**
   - **Pool tier**: satisfies most SMB requirements, SOC 2 with RLS documentation
   - **Bridge tier**: satisfies logical isolation requirements (PCI DSS, some HIPAA)
   - **Silo tier**: satisfies physical isolation requirements (HIPAA BAA, FedRAMP, EU data residency with region-specific Neon projects)
