# Multi-Tenant SaaS Bootstrapper — Feature & Functionality Survey

> Candidate #285 · Researched: 2026-05-03

## Core Features

Primary solutions: Next.js starter templates, Vercel multi-tenant blueprints, Wasp, Create-React-App, Rails + ActiveAdmin.

**Must-have**: Multi-tenant architecture, user authentication, tenant isolation, admin dashboard, billing integration, database schema isolation, deployment configuration.

**Differentiating**: Row-level security (RLS) implementation, automatic API generation, feature flagging per tenant, usage analytics, white-labeling support, Single Sign-On (SSO).

**Underserved**: Automatic tenant onboarding workflows, tenant migration tools, cost allocation across tenants, compliance and audit trail.

**AI-augmentation**: Intelligent feature recommendations per tenant, usage-based billing optimization.

## Legal & IP Summary

Open-source frameworks (Next.js, Rails) are MIT/Apache licensed. Starter templates may have custom licenses. No patent encumbrances identified.

## Recommended Feature Scope

**Must-have (MVP)**
- Multi-tenant architecture with tenant isolation
- User authentication (email/password, OAuth)
- Admin dashboard for tenant management
- Subscription and billing integration (Stripe)
- Database schema isolation
- API endpoints for tenant operations
- User invitation and management
- Basic usage analytics
- Deployment configuration (Docker, Vercel, AWS)

**Should-have (v1.1)**
- Row-level security (RLS) implementation
- Single Sign-On (SSO) support
- Feature flagging per tenant
- White-labeling and custom branding
- Usage-based billing calculation
- API key management for tenants
- Rate limiting and quota management
- Email integration for notifications
- Backup and disaster recovery

**Nice-to-have (backlog)**
- Automatic tenant onboarding workflow
- Tenant migration and data export tools
- Cost allocation and chargeback
- Compliance and audit trail logging
- Webhooks for tenant events
- Advanced analytics per tenant
- Integration marketplace for tenant plugins
- Multi-region deployment
