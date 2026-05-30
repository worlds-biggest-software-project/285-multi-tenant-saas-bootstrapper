# Data Model Suggestion 2: Event-Sourced / CQRS Approach

> Project: Multi-Tenant SaaS Bootstrapper (Candidate #285)
> Generated: 2026-05-25

## Summary

An **event-sourced** data model with **Command Query Responsibility Segregation (CQRS)**, where every state change in the system is captured as an immutable domain event in an append-only event store. Read-optimized projections (materialized views) are built from the event stream to serve queries. Tenant isolation is enforced at the event stream level, with each tenant's events stored in a logically separate stream.

This approach is inspired by the architecture of **Ultimate Backend** (the open-source NestJS multi-tenant SaaS starter that uses CQRS, GraphQL, and event sourcing) and is appropriate for teams that need a complete audit trail, complex domain logic, or eventual consistency across distributed services.

---

## Key Entities and Relationships

### Architectural Overview

```
Commands ──> Command Handlers ──> Event Store (append-only)
                                       │
                                       ▼
                                  Event Bus (Kafka / NATS / in-process)
                                       │
                              ┌────────┴────────┐
                              ▼                  ▼
                       Read Projections    Side Effects
                       (PostgreSQL views)  (Email, Stripe sync,
                                            webhook dispatch)

Queries ──> Query Handlers ──> Read Projections
```

### Event Store Schema

```sql
-- =============================================================
-- EVENT STORE (append-only, the source of truth)
-- =============================================================

CREATE TABLE event_store (
    id              BIGSERIAL PRIMARY KEY,
    stream_id       TEXT NOT NULL,                -- e.g. "tenant:abc123" or "user:xyz789"
    stream_type     TEXT NOT NULL,                -- e.g. "Tenant", "User", "Subscription"
    tenant_id       UUID NOT NULL,                -- partition key for tenant isolation
    event_type      TEXT NOT NULL,                -- e.g. "TenantCreated", "MemberInvited"
    event_data      JSONB NOT NULL,               -- the event payload
    metadata        JSONB NOT NULL DEFAULT '{}',  -- correlation_id, causation_id, user_id, ip
    version         INTEGER NOT NULL,             -- optimistic concurrency per stream
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (stream_id, version)                   -- prevents concurrent writes to same stream
);

CREATE INDEX idx_event_store_tenant    ON event_store (tenant_id, created_at);
CREATE INDEX idx_event_store_stream    ON event_store (stream_id, version);
CREATE INDEX idx_event_store_type      ON event_store (event_type, created_at);

-- Partition by tenant_id hash for large-scale deployments
-- CREATE TABLE event_store ... PARTITION BY HASH (tenant_id);
```

### Domain Events Catalogue

```typescript
// Tenant lifecycle events
type TenantCreated = {
  tenantId: string;
  name: string;
  slug: string;
  billingEmail: string;
  createdBy: string;
};

type TenantSettingsUpdated = {
  tenantId: string;
  settings: Record<string, unknown>;
  updatedBy: string;
};

type TenantSuspended = {
  tenantId: string;
  reason: string;
  suspendedBy: string;
};

// Membership events
type MemberInvited = {
  tenantId: string;
  email: string;
  role: string;
  invitedBy: string;
  token: string;
  expiresAt: string;
};

type MemberJoined = {
  tenantId: string;
  userId: string;
  role: string;
  invitationId?: string;
};

type MemberRoleChanged = {
  tenantId: string;
  userId: string;
  previousRole: string;
  newRole: string;
  changedBy: string;
};

type MemberRemoved = {
  tenantId: string;
  userId: string;
  removedBy: string;
  reason?: string;
};

// Subscription events
type SubscriptionCreated = {
  tenantId: string;
  planId: string;
  stripeSubscriptionId: string;
  status: string;
  periodStart: string;
  periodEnd: string;
};

type SubscriptionPlanChanged = {
  tenantId: string;
  previousPlanId: string;
  newPlanId: string;
  effectiveAt: string;
};

type SubscriptionCanceled = {
  tenantId: string;
  cancelAtPeriodEnd: boolean;
  reason?: string;
};

type PaymentSucceeded = {
  tenantId: string;
  invoiceId: string;
  amountCents: number;
  currency: string;
};

type PaymentFailed = {
  tenantId: string;
  invoiceId: string;
  amountCents: number;
  failureReason: string;
};

// Feature flag events
type FeatureFlagToggled = {
  tenantId: string;
  featureKey: string;
  enabled: boolean;
  value?: string;
  changedBy: string;
};

// API key events
type ApiKeyCreated = {
  tenantId: string;
  keyId: string;
  name: string;
  prefix: string;
  scopes: string[];
  expiresAt?: string;
};

type ApiKeyRevoked = {
  tenantId: string;
  keyId: string;
  revokedBy: string;
};
```

### Read Projections (Query Side)

```sql
-- =============================================================
-- READ PROJECTIONS (rebuilt from event store, optimized for queries)
-- =============================================================

-- Tracks which events have been projected
CREATE TABLE projection_checkpoints (
    projection_name TEXT PRIMARY KEY,
    last_event_id   BIGINT NOT NULL DEFAULT 0,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Tenant read model
CREATE TABLE tenant_projections (
    id                  UUID PRIMARY KEY,
    name                TEXT NOT NULL,
    slug                TEXT NOT NULL UNIQUE,
    billing_email       TEXT,
    stripe_customer_id  TEXT,
    status              TEXT NOT NULL DEFAULT 'active',
    settings            JSONB NOT NULL DEFAULT '{}',
    member_count        INTEGER NOT NULL DEFAULT 0,
    current_plan_id     UUID,
    current_plan_name   TEXT,
    subscription_status TEXT,
    created_at          TIMESTAMPTZ NOT NULL,
    updated_at          TIMESTAMPTZ NOT NULL
);

-- Membership read model
CREATE TABLE membership_projections (
    id          UUID PRIMARY KEY,
    tenant_id   UUID NOT NULL,
    user_id     UUID NOT NULL,
    user_email  TEXT NOT NULL,
    user_name   TEXT,
    role_name   TEXT NOT NULL,
    status      TEXT NOT NULL,
    joined_at   TIMESTAMPTZ NOT NULL,
    UNIQUE (tenant_id, user_id)
);

-- Subscription read model
CREATE TABLE subscription_projections (
    id                      UUID PRIMARY KEY,
    tenant_id               UUID NOT NULL,
    plan_id                 UUID NOT NULL,
    plan_name               TEXT NOT NULL,
    stripe_subscription_id  TEXT,
    status                  TEXT NOT NULL,
    current_period_start    TIMESTAMPTZ,
    current_period_end      TIMESTAMPTZ,
    cancel_at_period_end    BOOLEAN NOT NULL DEFAULT FALSE,
    monthly_amount_cents    INTEGER,
    currency                TEXT DEFAULT 'usd'
);

-- Invoice read model
CREATE TABLE invoice_projections (
    id                  UUID PRIMARY KEY,
    tenant_id           UUID NOT NULL,
    stripe_invoice_id   TEXT,
    amount_cents        INTEGER NOT NULL,
    currency            TEXT NOT NULL,
    status              TEXT NOT NULL,
    period_start        TIMESTAMPTZ,
    period_end          TIMESTAMPTZ,
    paid_at             TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL
);

-- Feature flag read model
CREATE TABLE feature_flag_projections (
    tenant_id   UUID NOT NULL,
    feature_key TEXT NOT NULL,
    enabled     BOOLEAN NOT NULL DEFAULT FALSE,
    value       TEXT,
    PRIMARY KEY (tenant_id, feature_key)
);

-- API key read model (excludes secret material)
CREATE TABLE api_key_projections (
    id          UUID PRIMARY KEY,
    tenant_id   UUID NOT NULL,
    name        TEXT NOT NULL,
    prefix      TEXT NOT NULL,
    scopes      TEXT[] NOT NULL DEFAULT '{}',
    is_active   BOOLEAN NOT NULL DEFAULT TRUE,
    last_used_at TIMESTAMPTZ,
    expires_at  TIMESTAMPTZ,
    created_at  TIMESTAMPTZ NOT NULL
);

-- RLS on all projection tables
ALTER TABLE membership_projections ENABLE ROW LEVEL SECURITY;
ALTER TABLE subscription_projections ENABLE ROW LEVEL SECURITY;
ALTER TABLE invoice_projections ENABLE ROW LEVEL SECURITY;
ALTER TABLE feature_flag_projections ENABLE ROW LEVEL SECURITY;
ALTER TABLE api_key_projections ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation ON membership_projections
    USING (tenant_id = current_setting('app.current_tenant_id')::UUID);
-- (repeat for each projection table)
```

### Projection Rebuilder

```typescript
// Pseudocode: a projection handler that processes events sequentially
class TenantProjectionHandler {
  async handle(event: DomainEvent): Promise<void> {
    switch (event.eventType) {
      case 'TenantCreated':
        await this.db.insert('tenant_projections', {
          id: event.eventData.tenantId,
          name: event.eventData.name,
          slug: event.eventData.slug,
          billing_email: event.eventData.billingEmail,
          status: 'active',
          member_count: 0,
          created_at: event.createdAt,
          updated_at: event.createdAt,
        });
        break;

      case 'MemberJoined':
        await this.db.query(
          `UPDATE tenant_projections
           SET member_count = member_count + 1, updated_at = $1
           WHERE id = $2`,
          [event.createdAt, event.eventData.tenantId]
        );
        break;

      case 'SubscriptionCreated':
        await this.db.query(
          `UPDATE tenant_projections
           SET current_plan_id = $1, subscription_status = $2, updated_at = $3
           WHERE id = $4`,
          [event.eventData.planId, event.eventData.status,
           event.createdAt, event.eventData.tenantId]
        );
        break;

      case 'TenantSuspended':
        await this.db.query(
          `UPDATE tenant_projections SET status = 'suspended', updated_at = $1 WHERE id = $2`,
          [event.createdAt, event.eventData.tenantId]
        );
        break;
    }
  }
}
```

---

## Pros

- **Complete audit trail by design.** Every state change is an immutable event. Compliance (SOC 2, GDPR Article 30) is built into the architecture, not bolted on. There is no need for a separate `audit_logs` table -- the event store *is* the audit log.
- **Time-travel and replay.** The state of any tenant at any point in time can be reconstructed by replaying events up to that timestamp. This is invaluable for debugging, dispute resolution, and forensic analysis.
- **Read/write optimization.** Write-side is append-only (extremely fast). Read-side projections are denormalized and tuned for specific query patterns. The two sides scale independently.
- **Decoupled side effects.** Sending emails, syncing with Stripe, dispatching webhooks, and updating analytics are all event consumers. Adding a new side effect requires zero changes to the write model.
- **Schema evolution without migrations.** New event types can be added without altering existing tables. Old events remain intact; new projections can be rebuilt from scratch.
- **Natural fit for distributed systems.** If the bootstrapper evolves to microservices, the event bus becomes the integration backbone between services.

## Cons

- **Significantly higher complexity.** Event sourcing requires a fundamentally different mental model from CRUD. Most indie hackers and seed-stage developers have never built an event-sourced system.
- **Eventual consistency.** Read projections lag behind writes. This is confusing for UI flows that expect immediate consistency (e.g., "I just invited a member, why don't they appear in the list?"). Requires careful handling of read-your-writes consistency.
- **Projection management overhead.** Every new query pattern requires a new projection. Projections must be rebuilt when their logic changes, which can take hours for large event stores.
- **Event versioning is hard.** As the domain evolves, event schemas change. Upcasting old events to new formats (or maintaining multiple event versions) adds ongoing maintenance burden.
- **Overkill for a bootstrapper.** The primary audience (indie hackers, small teams) needs to ship fast. Event sourcing adds weeks of architecture work before the first user-facing feature is built.
- **Tooling maturity.** The Node.js/TypeScript event sourcing ecosystem is less mature than Java/C# (which have Axon, EventStoreDB, Marten). Teams may need to build custom projection infrastructure.
- **Storage growth.** The event store grows indefinitely. Snapshotting (periodic state checkpoints) is needed to keep replay times manageable, adding more infrastructure.

---

## Technology Recommendations

| Component | Recommendation |
|-----------|---------------|
| Event store | PostgreSQL (with append-only `event_store` table) for simplicity, or EventStoreDB for dedicated event streaming |
| Event bus | Apache Kafka (production), NATS JetStream (lightweight), or in-process EventEmitter (MVP) |
| Projection database | PostgreSQL (same instance for MVP; separate read replica for scale) |
| Framework | NestJS CQRS module (`@nestjs/cqrs`), or custom handlers with TypeScript |
| Serialization | JSON (JSONB in PostgreSQL); consider Avro/Protobuf for Kafka topics at scale |
| Auth | Clerk or Supabase Auth (events record who performed each action) |
| Billing | Stripe webhooks consumed as events into the event store |
| Snapshot store | Periodic aggregate snapshots in a `snapshots` table to avoid replaying full history |

---

## Migration and Scaling Considerations

1. **Start simple, evolve to events.** A practical approach for a bootstrapper is to start with the normalized relational model (Suggestion 1) and introduce event sourcing incrementally for specific domains (e.g., billing events, audit trail) rather than making it the universal architecture from day one.

2. **Event store partitioning.** Partition the `event_store` table by `tenant_id` hash to distribute storage and query load. For very high-throughput tenants, consider stream-level partitioning within Kafka topics.

3. **Snapshot strategy.** Create aggregate snapshots every N events (e.g., every 100 events per stream). On replay, load the latest snapshot and replay only subsequent events.

4. **Projection rebuild pipeline.** Maintain a projection rebuild mechanism that can recreate any read model from scratch. This is essential for schema evolution and bug fixes in projection logic. Budget 1-2 hours rebuild time per 10M events.

5. **Event versioning.** Adopt an upcasting strategy from the start: when an event schema changes, write an upcaster function that transforms old event formats to the new format during replay. Never modify historical events.

6. **GDPR right to erasure.** Event sourcing and GDPR's "right to be forgotten" are in tension. The recommended approach is **crypto-shredding**: encrypt PII fields in events with a per-tenant key, and destroy the key when erasure is requested. The events remain but PII becomes unreadable.

7. **Transition path.** If the bootstrapper starts with CRUD and later adds event sourcing, use the **Strangler Fig pattern**: introduce an event store alongside the existing database, dual-write during migration, and gradually shift read paths to projections.
