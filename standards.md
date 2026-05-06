# Standards & API Reference

> Project: Multi-Tenant SaaS Bootstrapper · Generated: 2026-05-03

## Industry Standards & Specifications

### ISO Standards

**ISO/IEC 27001:2022 — Information Security Management Systems**
- URL: https://www.iso.org/standard/27001
- The dominant international certification for information security management. Enterprise buyers frequently require their SaaS vendors to be ISO 27001 certified or on a credible certification path. A bootstrapper should scaffold the ISMS-aligned controls (access control, audit logging, encryption, incident response) that make certification achievable.

**ISO/IEC 27017:2015 — Cloud Security Controls**
- URL: https://www.iso.org/standard/43757.html
- Supplements ISO 27001 with cloud-specific security guidance, including controls for shared environments, virtual machine hardening, and cloud service customer/provider responsibilities. Directly relevant to multi-tenant SaaS deployments.

**ISO/IEC 27018:2019 — Protection of Personally Identifiable Information (PII) in Public Clouds**
- URL: https://www.iso.org/standard/76559.html
- Establishes controls for public cloud service providers that process personal data as data processors. Governs consent, purpose limitation, and data subject rights — all of which must be supported in a bootstrapper's tenant data model.

---

### W3C & IETF Standards

**RFC 6749 — The OAuth 2.0 Authorization Framework**
- URL: https://www.rfc-editor.org/rfc/rfc6749.html
- The foundational authorization protocol used by every major SaaS authentication library (Clerk, Auth.js, Supabase Auth, WorkOS). Defines authorization flows, token types, and scope negotiation that a bootstrapper's auth layer must implement correctly.

**RFC 7636 — Proof Key for Code Exchange (PKCE)**
- URL: https://www.rfc-editor.org/rfc/rfc7636.html
- Mandatory extension to the OAuth 2.0 Authorization Code flow that prevents authorization code interception attacks. Required for SPAs, mobile clients, and any public OAuth client. OAuth 2.1 (in draft) makes PKCE mandatory for all clients.

**RFC 7519 — JSON Web Token (JWT)**
- URL: https://datatracker.ietf.org/doc/html/rfc7519
- Defines the compact, self-contained token format used throughout modern SaaS for conveying identity claims, tenant context, and entitlements between services. The bootstrapper's middleware and RLS policies depend on JWT claims for tenant scoping.

**RFC 7643 — SCIM 2.0 Core Schema**
- URL: https://www.rfc-editor.org/rfc/rfc7643.html
- Defines the standard User, Group, and EnterpriseUser resource schemas for cross-domain identity management. Required for enterprise integrations with Okta, Microsoft Entra ID, and Google Workspace.

**RFC 7644 — SCIM 2.0 Protocol**
- URL: https://datatracker.ietf.org/doc/html/rfc7644
- Defines the HTTP-based CRUD API for SCIM resources. Implementing this protocol is the prerequisite for automated user provisioning and de-provisioning demanded by enterprise buyers.

**OpenID Connect Core 1.0 (OIDC)**
- URL: https://openid.net/specs/openid-connect-core-1_0.html
- An identity layer built on OAuth 2.0 that standardises the ID token and UserInfo endpoint. The bootstrapper's auth integration must support OIDC for federated identity with corporate IdPs. OIDC discovery (`.well-known/openid-configuration`) enables zero-config integration with third-party providers.

**SAML 2.0 — Security Assertion Markup Language**
- URL: https://docs.oasis-open.org/security/saml/v2.0/saml-core-2.0-os.pdf
- XML-based SSO standard still widely required by large enterprise buyers, particularly in regulated industries. While OIDC is preferred for new integrations, the bootstrapper should reference a library (e.g., BoxyHQ SAML Jackson) that provides SAML support without bespoke implementation.

**W3C Web Authentication (WebAuthn) — Level 2**
- URL: https://www.w3.org/TR/webauthn-2/
- The browser API for passkey and hardware security key authentication. Increasingly adopted as a phishing-resistant MFA replacement. Bootstrappers targeting security-conscious buyers should include or plan for WebAuthn support via Clerk, Auth.js, or Supabase.

---

### Data Model & API Specifications

**OpenAPI Specification 3.1.0**
- URL: https://spec.openapis.org/oas/v3.1.0
- The industry-standard format for describing RESTful APIs. A bootstrapper should generate an OpenAPI schema for its tenant management and provisioning endpoints to enable SDK generation, interactive docs, and API contract testing.

**JSON Schema (draft-07 / 2020-12)**
- URL: https://json-schema.org/specification.html
- Used for validating API request/response payloads, webhook event envelopes, and feature flag configurations. Widely supported by Zod, Ajv, and TypeScript toolchains common in SaaS starters.

**Standard Webhooks Specification**
- URL: https://www.standardwebhooks.com/
- An emerging industry specification (backed by Svix, Stripe, and others) for consistent webhook payload formats and HMAC-SHA256 signature verification. Adopting this standard makes a bootstrapper's event system interoperable with webhook tooling and avoids bespoke signature schemes.

---

### Security & Authentication Standards

**OWASP Application Security Verification Standard (ASVS) 4.0**
- URL: https://owasp.org/www-project-application-security-verification-standard/
- Defines three tiers of security verification requirements for web applications. A SaaS bootstrapper should target Level 2 compliance, covering authentication, session management, access control, and API security controls that enterprise security reviews will check.

**OWASP Top 10 (2021)**
- URL: https://owasp.org/Top10/
- The canonical list of the most critical web application security risks (injection, broken access control, misconfigured security, etc.). Multi-tenant architectures are especially vulnerable to tenant data leakage via broken access control — the bootstrapper's RLS patterns and middleware guards directly address A01:2021.

**NIST SP 800-63B — Digital Identity Guidelines (Authentication)**
- URL: https://pages.nist.gov/800-63-3/sp800-63b.html
- Federal guidance on authenticator assurance levels (AAL1–AAL3), password policies, MFA requirements, and session management. Referenced by SOC 2 auditors and US government-adjacent buyers.

**SOC 2 Type II (AICPA Trust Services Criteria)**
- URL: https://www.aicpa.org/resources/article/soc-2
- The attestation report most commonly required by US SaaS buyers evaluating vendor security posture. The bootstrapper's scaffolded architecture should support the five Trust Services Criteria: security, availability, processing integrity, confidentiality, and privacy.

**GDPR — General Data Protection Regulation (EU) 2016/679**
- URL: https://eur-lex.europa.eu/legal-content/EN/TXT/?uri=CELEX%3A32016R0679
- Requires the bootstrapper to support per-tenant data subject rights (access, erasure, portability), data residency routing, and Data Processing Agreement (DPA) templates. Tenant isolation models must ensure cross-tenant data leakage is architecturally impossible.

---

### MCP Server Specifications

**Model Context Protocol (MCP) — Anthropic**
- URL: https://modelcontextprotocol.io/specification/
- An open protocol for exposing tools and data sources to AI agents. A multi-tenant SaaS bootstrapper could expose tenant management operations (create tenant, list users, update entitlements, fetch usage metrics) as MCP tools, enabling AI coding assistants and autonomous agents to interact with the platform's admin plane programmatically.

---

## Similar Products — Developer Documentation & APIs

### Stripe Billing
- **Description:** The de-facto payment infrastructure for SaaS subscriptions, supporting flat-rate, per-seat, metered/usage-based, and tiered pricing models with a customer portal for self-service billing management.
- **API Documentation:** https://docs.stripe.com/billing
- **SDKs/Libraries:** Node.js (`stripe`), Python (`stripe`), Ruby (`stripe`), Go (`github.com/stripe/stripe-go`), PHP (`stripe/stripe-php`), Java, .NET
- **Developer Guide:** https://docs.stripe.com/saas — https://docs.stripe.com/billing/subscriptions/build-subscriptions
- **Standards:** REST/JSON, OpenAPI-described, webhook events use HMAC-SHA256 signature verification
- **Authentication:** Secret API key (Bearer token); restricted keys for least-privilege service access

### Stripe Connect
- **Description:** Extension of Stripe Billing for multi-sided platforms; routes payments between the platform account and tenant-owned connected accounts, supporting marketplace and SaaS-plus-marketplace models.
- **API Documentation:** https://docs.stripe.com/connect
- **SDKs/Libraries:** Same as Stripe Billing
- **Developer Guide:** https://docs.stripe.com/connect/subscriptions
- **Standards:** REST/JSON; OAuth 2.0 for connecting accounts
- **Authentication:** Platform secret key + per-account access via `Stripe-Account` header

### Lemon Squeezy
- **Description:** Merchant of Record payment platform for indie developers and small SaaS businesses; handles global tax compliance, subscriptions, and licensing without needing a Stripe Atlas or local entity.
- **API Documentation:** https://docs.lemonsqueezy.com/api
- **SDKs/Libraries:** Official JavaScript/TypeScript SDK; Python, PHP, and community libraries
- **Developer Guide:** https://docs.lemonsqueezy.com/guides/developer-guide/getting-started
- **Standards:** REST/JSON, JSON:API encoded responses, HMAC-SHA256 webhook signatures
- **Authentication:** Bearer token (API key from dashboard)

### WorkOS
- **Description:** Enterprise authentication platform providing SSO (SAML + OIDC), Directory Sync (SCIM), Admin Portal, Audit Logs, and Fine-Grained Authorization — all via a single integration point. Widely used as the "enterprise features" layer in SaaS starters.
- **API Documentation:** https://workos.com/docs/reference
- **SDKs/Libraries:** Node.js, Python, Ruby, PHP, Go, Java, .NET — https://workos.com/docs/sdks
- **Developer Guide:** https://workos.com/docs — https://workos.com/blog/developers-guide-saas-multi-tenant-architecture
- **Standards:** REST/JSON, OAuth 2.0 / OIDC, SAML 2.0, SCIM 2.0
- **Authentication:** API key (per-environment); SSO connections use per-tenant configuration

### Clerk
- **Description:** Full-stack authentication and user management service with first-class support for Organizations (multi-tenant teams), role-based access, and session management. Provides pre-built UI components and hooks for React/Next.js.
- **API Documentation:** https://clerk.com/docs/reference/backend-api
- **SDKs/Libraries:** `@clerk/nextjs`, `@clerk/react`, `@clerk/clerk-sdk-node`, Expo, Astro, Remix, React Router
- **Developer Guide:** https://clerk.com/docs/guides/organizations/overview — https://clerk.com/docs/guides/how-clerk-works/multi-tenant-architecture
- **Standards:** REST/JSON, JWT-based session tokens, OIDC for SSO
- **Authentication:** Publishable key (frontend) + Secret key (backend); JWT verification using JWKS endpoint

### Supabase
- **Description:** Open-source Firebase alternative built on PostgreSQL, providing database, authentication, storage, real-time subscriptions, and Edge Functions. Row-Level Security (RLS) is the primary mechanism for tenant data isolation in pooled-schema architectures.
- **API Documentation:** https://supabase.com/docs/reference/javascript/introduction
- **SDKs/Libraries:** `@supabase/supabase-js`, Python (`supabase`), Dart/Flutter, Swift, Kotlin
- **Developer Guide:** https://supabase.com/docs/guides/database/postgres/row-level-security — https://supabase.com/docs/guides/auth
- **Standards:** REST (PostgREST), GraphQL (optional), WebSockets (Realtime), OpenAPI schema auto-generated from schema
- **Authentication:** JWT from Supabase Auth; anon key (public) + service_role key (admin)

### Neon
- **Description:** Serverless PostgreSQL with storage/compute separation, instant branching, and scale-to-zero. Its branching model (one branch per tenant) supports database-per-tenant isolation without per-instance provisioning overhead.
- **API Documentation:** https://api-docs.neon.tech/reference/getting-started-with-neon-api
- **SDKs/Libraries:** `@neondatabase/serverless` (HTTP/WebSocket driver); compatible with all standard Postgres drivers
- **Developer Guide:** https://neon.com/docs/guides/multitenancy — https://neon.com/docs/guides/database-per-user
- **Standards:** PostgreSQL wire protocol; REST API for management plane; HTTP/WebSocket for serverless query execution
- **Authentication:** API key (management API); database connection strings (application queries)

### Resend
- **Description:** Developer-focused transactional email API built around React Email for template authoring. Used by MakerKit, Supastarter, and other starters for password reset, invite, and notification emails.
- **API Documentation:** https://resend.com/docs/api-reference/introduction
- **SDKs/Libraries:** `resend` (Node.js/TypeScript), Python, Ruby, PHP, Go, Elixir, Java, .NET, Rust
- **Developer Guide:** https://resend.com/docs/introduction
- **Standards:** REST/JSON; React Email templates compile to HTML; webhook events for delivery status
- **Authentication:** Bearer token (API key)

### Svix
- **Description:** Managed webhook infrastructure service that handles delivery retries, signature verification, event log UI, and consumer portal — replacing the need to build a bespoke webhook system. Used by BoxyHQ SaaS Starter Kit and others.
- **API Documentation:** https://api.svix.com/docs
- **SDKs/Libraries:** `svix` (Node.js), Python, Ruby, Go, Java, Kotlin, C#, PHP, Rust
- **Developer Guide:** https://docs.svix.com/ — https://docs.svix.com/receiving/verifying-payloads/how-manual
- **Standards:** Standard Webhooks Specification; HMAC-SHA256 payload signatures; REST/JSON management API
- **Authentication:** Bearer token (Svix API key); per-application secrets for payload signing

### BoxyHQ (SAML Jackson)
- **Description:** Open-source enterprise SSO and Directory Sync service that wraps SAML 2.0 and OIDC complexity behind a simple OAuth-like API. Used as the SSO layer in BoxyHQ's open-source Enterprise SaaS Starter Kit.
- **API Documentation:** https://boxyhq.com/docs/jackson/overview
- **SDKs/Libraries:** `@boxyhq/saml-jackson` (embedded Node.js service); Docker image for standalone deployment
- **Developer Guide:** https://boxyhq.com/guides/jackson — https://github.com/boxyhq/saas-starter-kit
- **Standards:** SAML 2.0, OIDC, OAuth 2.0 facade over SAML; SCIM 2.0 for directory sync
- **Authentication:** API key for management endpoints; standard OAuth 2.0 flows for end-user SSO

---

## Notes

**Emerging standards to monitor:**

- **OAuth 2.1** — A consolidating draft profile of OAuth 2.0 that mandates PKCE, removes implicit grant and ROPC, and tightens redirect URI handling. Expected to be finalised in 2026; bootstrappers should align auth libraries accordingly.

- **FastFed Enterprise SCIM Profile 1.0** — An OpenID Foundation profile that standardises how IdPs and SaaS apps negotiate and establish SCIM provisioning connections automatically (https://openid.net/specs/fastfed-scim-1_0-03.html). Reduces the manual configuration overhead of per-tenant SCIM setup.

- **OpenTelemetry (OTel) for multi-tenant observability** — The CNCF-standard API/SDK for traces, metrics, and logs (https://opentelemetry.io/). Attaching tenant context to every OTel span enables per-tenant SLA monitoring, anomaly detection, and chargeback without bespoke instrumentation.

- **Standard Webhooks v1** — An industry effort backed by Stripe, Svix, and others to unify webhook payload format and signature verification across SaaS providers (https://www.standardwebhooks.com/). Adopting it from the start avoids a proprietary event format that tenants must adapt to.

**Areas where standards are still thin:**

- Per-tenant feature flagging has no formal standard; implementations vary between LaunchDarkly SDK conventions, Unleash's toggle API, and bespoke flag tables.
- Tenant-level audit logging formats lack a universal schema; best practice converges on CloudEvents (https://cloudevents.io/) for event envelope structure, but adoption is uneven.
- Metered/usage-based billing API semantics are not standardised across Stripe, Lago, and open-source alternatives — a bootstrapper must abstract this behind a billing adapter interface.
