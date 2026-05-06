# Multi-Tenant SaaS Bootstrapper

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> A full-stack, AI-native multi-tenant SaaS starter with auth, billing, and an admin dashboard built in.

The Multi-Tenant SaaS Bootstrapper is an open-source starter kit for indie hackers, agencies, and seed-stage startups who want to ship a B2B SaaS product without spending weeks wiring up tenant isolation, authentication, billing, and an admin console. It packages the patterns proven by commercial starters into a free, framework-flexible foundation that AI coding assistants can extend safely.

---

## Why Multi-Tenant SaaS Bootstrapper?

- Leading commercial starters (MakerKit ~$299+, Supastarter, Shipfast ~$199, Gravity, Tenancy for Laravel) charge per-developer licence fees and lock teams into a single framework — Next.js, Laravel, or NestJS only.
- The main free option (ixartz SaaS Boilerplate) is actively maintained but has a less battle-tested billing integration than paid kits.
- Enterprise-grade open-source alternatives like Ultimate Backend impose CQRS/GraphQL/event-sourcing complexity that small teams do not need.
- AI coding assistant compatibility (Cursor, Claude Code rules files) is becoming a 2026 differentiator, yet most existing starters were not designed with AI-driven scaffolding in mind.
- Buyers increasingly expect SCIM 2.0 provisioning, RLS-based tenant isolation, and Stripe metered billing out of the box — features unevenly distributed across today's options.

---

## Key Features

### Multi-Tenant Foundation

- Multi-tenant architecture with strict tenant isolation
- Database schema isolation
- Row-level security (RLS) implementation
- API endpoints for tenant operations
- User invitation and management

### Authentication & Access

- Email/password and OAuth authentication
- Single Sign-On (SSO) support
- API key management for tenants
- Rate limiting and quota management

### Billing & Monetisation

- Subscription and billing integration via Stripe
- Usage-based billing calculation
- Feature flagging per tenant
- White-labeling and custom branding

### Operations & Admin

- Admin dashboard for tenant management
- Basic usage analytics
- Email integration for notifications
- Backup and disaster recovery
- Deployment configuration for Docker, Vercel, and AWS

### Backlog Capabilities

- Automatic tenant onboarding workflow
- Tenant migration and data export tools
- Cost allocation and chargeback
- Compliance and audit trail logging
- Webhooks for tenant events
- Integration marketplace for tenant plugins
- Multi-region deployment

---

## AI-Native Advantage

AI lets this bootstrapper go beyond static templates. Tenant-specific onboarding flows and feature flag configurations can be generated from signup data and a stated use case. Billing tiers and entitlement rules can be auto-configured from a plain-English product description, producing Stripe price objects and permission rules directly. A conversational admin interface allows operators to search, impersonate, suspend, or escalate tenant cases without writing SQL, while anomaly detection surfaces unusual API or data access patterns with recommended remediation. New feature modules — schema migrations, API routes, UI components, and permissions — can be scaffolded from a natural-language description that respects the existing multi-tenant architecture.

---

## Tech Stack & Deployment

The project targets self-hosted and cloud deployment via Docker, Vercel, and AWS. It aligns with established standards: OAuth 2.0 / OIDC for authentication, SCIM 2.0 for enterprise user provisioning against Okta, Azure AD, and Google Workspace, Stripe Connect / Billing APIs for subscription and metered billing, and Postgres Row-Level Security for tenant data isolation. Infrastructure choices are intended to support SOC 2 and ISO 27001 readiness for buyers who require it.

---

## Market Context

The SaaS starter kit niche sits within the broader low-code and developer tools segment estimated at USD 26 billion in 2025, driven by indie hackers, micro-ISVs, and venture-backed startups. Commercial competitors are typically bootstrapped lifestyle businesses generating $200K–$2M ARR, with one-time licence fees of $99–$499 per developer. Primary buyers are indie hackers launching their first SaaS, agencies building client products, seed-stage startups without a full platform team, and developers learning multi-tenant architecture patterns.

---

## Project Status

> This project is in the **research and specification phase**.  
> Contributions, feedback, and domain expertise are welcome.

---

## Contributing

We welcome contributions from developers, domain experts, and potential users.
See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

**Important:** All contributions must be your own original work or clearly attributed
open-source material with a compatible licence. Copyright infringement and licence
violations will not be tolerated and will result in immediate removal of the offending
contribution. If you are unsure whether a piece of code, text, or other material is
safe to contribute, open an issue and ask before submitting.

---

## Licence

Licence to be determined. See [discussion](#) for context.
