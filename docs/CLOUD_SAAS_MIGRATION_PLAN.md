# Vtechiee Cloud SaaS Migration Plan

Status: architecture and repository audit; production hosting is NOT enabled by this document.

## Product target

Turn the existing single-computer GST billing application into a multi-customer SaaS at `https://billing.vtechiee.com`, hosted on Render.

## Repository audit (2026-11-02)

- Current package version: 1.10.83.
- Stack: React 19, Vite 7, Express 5, Node >=18.
- The Express application is a local-first server. It binds to `127.0.0.1`.
- The server stores bills, clients, products, expenses, receipts, profiles, purchases, recurring invoices, cash-book records and related data as JSON files below a single `data/` directory.
- API routes are not protected by a server-side user authentication layer.
- The browser application includes local-first/PWA behaviour and API response caching. This must be reviewed and changed for authenticated, tenant-specific cloud data.
- `vite.config.js` uses a local fixed-port API proxy for development; this must remain local-development behaviour and must not determine the Render production port.
- The current `docs/DEPLOY_ONLINE.md` explicitly warns that the application is not safe or ready for online hosting. Do not deploy the current server publicly.

## Required architecture

- Render Node web service for the Express API and built React frontend.
- Render managed PostgreSQL for durable business data.
- Custom domain `billing.vtechiee.com` with HTTPS.
- Server-side authentication with secure password hashing, session rotation, HttpOnly/Secure/SameSite cookies, CSRF protection, rate limiting, email verification and password-reset flow.
- Organisations/tenants, users, memberships and role-based access.
- Every tenant-owned row has a non-null `organization_id`; every read, update, delete, export, report and background job is scoped to the authenticated organisation.
- Separate platform-admin authorization, with audited and time-limited support access.
- Audit trail for sensitive operations.
- Automated database backups and tested restore procedure.

## Suggested relational model

- Organization(id, name, slug, status, created_at)
- User(id, email_normalized, password_hash, email_verified_at, created_at)
- Membership(id, organization_id, user_id, role, status)
- BusinessProfile(id, organization_id, ...)
- Client(id, organization_id, ...)
- Product(id, organization_id, ...)
- Invoice(id, organization_id, business_profile_id, client_id, invoice_number, ...)
- InvoiceLine(id, organization_id, invoice_id, ...)
- Expense, Purchase, Receipt, RecurringInvoice, CashBookEntry and other existing business records, each with organization_id and appropriate foreign keys.
- Session, PasswordResetToken, EmailVerificationToken, AuditEvent and PlatformAdmin records.

Use database constraints and indexes to reinforce tenant boundaries. Prefer composite foreign keys that include organization_id where practical. Do not rely solely on UI hiding or client-supplied tenant IDs.

## Migration work packages

1. Inventory every Express route, file helper, background timer, export/import path, and browser store function. Build a route-to-data-domain migration matrix.
2. Define the PostgreSQL schema and migrations; add tenant-aware repository/service modules.
3. Implement authentication, session lifecycle, account recovery, email verification, CSRF defence, rate limiting and authorization middleware.
4. Convert all data routes and background jobs from JSON files to tenant-scoped PostgreSQL transactions. Preserve invoice numbering with a database-backed, concurrency-safe counter per organisation/business profile/financial year.
5. Add registration, login, logout, onboarding, organisation switcher and account settings UI.
6. Review localStorage, IndexedDB, service-worker API caching and offline mode. Disable shared-cache behaviour for authenticated API responses; prevent data from one account surviving logout or account switching.
7. Add automated tenant-isolation tests, authorization tests, migration tests, backup/restore tests and full end-to-end billing regression tests.
8. Deploy to a non-production Render service and test with two separate test organisations before any real GST or customer data is entered.

## Render deployment target (after implementation)

- Build: `npm ci && npm run build`
- Start: `npm run server`
- Runtime: current supported Node LTS.
- Database: managed Render PostgreSQL; use the internal database URL from the same region.
- Environment: `NODE_ENV=production`, `DATABASE_URL`, `SESSION_SECRET`, `APP_BASE_URL=https://billing.vtechiee.com`, plus email-provider credentials.
- Bind Express to `0.0.0.0` and `process.env.PORT` in production.
- Configure the custom domain in Render and add the DNS records Render specifies at the domain's DNS provider.
- Do not store customer books in an ephemeral service filesystem or use a shared JSON data directory.

## Release-blocking acceptance criteria

- Unauthenticated requests cannot access any business API.
- Organisation A cannot read, update, delete, export, enumerate or infer Organisation B's records, including by guessing IDs.
- All API endpoints and background jobs enforce tenant scope.
- Concurrent invoice creation cannot issue duplicate invoice numbers.
- Logout and account switching clear or isolate all tenant-specific browser caches and service-worker data.
- CSRF, session fixation, brute-force, IDOR and privilege-escalation tests pass.
- Backup restore is demonstrated in a separate environment.
- Existing GST calculations, invoice PDFs, GSTR exports, inventory, cash book, purchase, recurring invoice and import/export workflows pass regression tests.

## Safety decision

This branch is intentionally not configured for public production deployment yet. The current application is a single-user local-first file-backed program. Merely changing the bind address, adding a shared password, or attaching a persistent disk would expose customer financial data and would not create tenant isolation. Do not connect this branch to a public Render service until all release-blocking criteria above are implemented and verified.
