# Render deployment — Vtechiee GST Billing

## Current status: deployment intentionally blocked

The existing application is a local-first, single-tenant application. Its Express API reads and writes a shared JSON data directory and does not require customer authentication. Exposing it on Render would expose every customer's billing data to other visitors.

The Render blueprint in this branch is a **guarded migration target**, not a production-ready service. It provisions the intended Node web service and PostgreSQL database, but the application must refuse cloud startup until the SaaS API migration and security tests are complete. Do not change `SAAS_AUTH_ENABLED` to true to bypass this guard.

## Target Render configuration

- Repository: `saraiffath/Free-GST-Billing-Software`
- Branch: `cloud-saas`
- Runtime: Node
- Build command: `npm ci && npm run build`
- Start command: `npm run server`
- Host: `0.0.0.0`
- Port: Render-provided `PORT`
- Database: Render PostgreSQL
- Health check: `/api/health`
- Domain target: `billing.vtechiee.com`

## Required before a public release

1. Complete PostgreSQL schema and migrations for every data domain currently stored under `data/`.
2. Implement registration, verified email, password reset, secure sessions, logout, CSRF protection and rate limiting.
3. Add organisation membership and role checks.
4. Migrate every read/write/delete/import/export/report/background task to enforce the authenticated organisation ID at the database query layer.
5. Replace shared file-backed business data. Do not use a Render persistent disk as a substitute for tenant-aware storage.
6. Disable authenticated API caching in the service worker and clear account-specific browser state on logout/account switching.
7. Test cross-tenant IDOR attempts, exports, backups, invoice numbering concurrency, restore, and all billing regression tests.
8. Configure email delivery, backup retention, monitoring, and a tested database restore procedure.
9. Only after the above, enable the cloud startup gate and connect the custom domain.

## Render dashboard steps after the release gate is cleared

1. Create a Blueprint from this repository and select branch `cloud-saas`.
2. Confirm the Node web service and PostgreSQL instance are created in the same region.
3. Set the production secrets and email-provider variables in Render's environment settings.
4. Verify health checks, migrations, backups, tenant-isolation tests and the full billing workflow in staging.
5. Add `billing.vtechiee.com` under the service's Custom Domains and create the exact DNS records Render displays.
6. Verify HTTPS and test with two separate test organisations before entering real customer data.

Never paste database URLs, SMTP passwords, or session secrets into GitHub or chat.
