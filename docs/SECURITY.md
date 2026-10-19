# Security

StudioOS holds Studio North's most sensitive business data: client contact details, invoices, revenue, expenses, and staff cost rates. This doc lists what we protect, who from, and the controls that do it.

## Assets and threats

| Asset | Sensitivity | Main threats |
| --- | --- | --- |
| Company financials (revenue, expenses, margins) | High | Exposure to staff below Admin, or to outsiders |
| Staff cost rates | High (personal, HR) | Exposure to colleagues |
| Client PII and invoices | High (GDPR/UK-GDPR) | Data breach, tampering with issued invoices |
| Accounts and sessions | High | Phishing, credential stuffing, session theft, ex-staff keeping access |
| Supabase secret key | Critical | Leak into client bundle, logs or repo, which bypasses all RLS |

| Threat actor | Example |
| --- | --- |
| Outsider | Hits the public Supabase API directly with the publishable key, trying to read tables |
| Authenticated Member | Calls the API directly to read `invoices` or `staff_rates`, bypassing the UI |
| Former employee | Tries to sign in after leaving |
| Compromised dependency | Malicious npm package exfiltrates env vars at build time |

## Controls

### 1. Authentication

- **Invite-only.** Public sign-ups are **disabled** in Supabase Auth. An Admin invites a user by email from `/settings/users` (server-side, secret-key client, Admin-checked).
- **Sign-in methods:** email magic link and Google OAuth.
  - Google sign-in is restricted to Studio North's Workspace domain. The auth callback rejects any `hd` claim/email domain not in `ALLOWED_EMAIL_DOMAINS`, and a `before-user-created` auth hook enforces the same rule at the database level.
  - Even with a valid Google account, no `profiles` row means no role, and no role means no access.
- **MFA for Admins (recommended):** Supabase TOTP MFA required for the `admin` role. Finance policies additionally check `auth.jwt() ->> 'aal' = 'aal2'`, so a stolen magic-link session without the second factor can't read financials.
- **Offboarding:** setting `profiles.is_active = false` cuts all data access **immediately** (role helpers return null). The Admin action also signs the user out of all sessions and bans the auth user.
- **Sessions:** short-lived access JWTs (1 hour), with rotating refresh tokens in `HttpOnly`, `Secure`, `SameSite=Lax` cookies managed by `@supabase/ssr`. Server code verifies identity with `getClaims()`/`getUser()`, never by trusting the cookie's session payload.

### 2. Authorisation: RLS is the boundary

- **Every table in `public` has RLS enabled**, with explicit policies per role. Default deny.
- Role checks use `SECURITY DEFINER` helpers (`current_role()`, `is_admin()`), which read `profiles` with a pinned `search_path`.
- Sensitive columns are split into sibling tables (`staff_rates`, `project_budgets`) instead of relying on column-level privileges.
- Column-level write restrictions (e.g. Members may only change a task's status) are enforced by `BEFORE UPDATE` triggers.
- `SECURITY DEFINER` RPCs (`issue_invoice`, `dashboard_summary`, …) **check the caller's role first** and set `search_path = ''`.
- **Tests, not hope:** every table and RPC has pgTAP tests that sign in as each role and assert allowed and denied operations. CI also fails if any `public` table has RLS disabled, and runs Supabase's security advisor (`get_advisors`) on staging.

### 3. Secrets and configuration

| Variable | Exposure | Notes |
| --- | --- | --- |
| `NEXT_PUBLIC_SUPABASE_URL` | Public | |
| `NEXT_PUBLIC_SUPABASE_PUBLISHABLE_KEY` | Public | Safe *only because* RLS is enforced everywhere |
| `SUPABASE_SECRET_KEY` | **Server only** | Used only in `lib/supabase/admin.ts`, which imports `server-only` |
| `RESEND_API_KEY` | Server only | |
| `ALLOWED_EMAIL_DOMAINS` | Server only | |

- Never prefix a secret with `NEXT_PUBLIC_`. Never log env vars.
- Secrets live in Vercel project env settings (scoped per environment) and in local `.env.local` (git-ignored). `.env.example` lists names only.
- Production and staging use **separate** Supabase projects and keys.
- GitHub secret scanning and push protection are enabled on the repo.

### 4. Application layer

- **Input validation:** every Server Action parses input with Zod before touching the database. Unknown keys are stripped.
- **CSRF:** Server Actions only accept same-origin POSTs (Next.js built-in origin check). Route handlers that mutate do the same.
- **XSS:** React escapes output by default. No `dangerouslySetInnerHTML` with user content. A strict Content-Security-Policy (nonce-based scripts, `frame-ancestors 'none'`) is set in `next.config` headers.
- **Headers:** HSTS, `X-Content-Type-Options: nosniff`, `Referrer-Policy: strict-origin-when-cross-origin`, `Permissions-Policy` minimal.
- **Files:** `invoices` and `receipts` Storage buckets are **private**, with `storage.objects` RLS limited to Admins. Downloads use short-lived signed URLs (≤ 5 min). Receipt uploads are restricted by MIME type (PDF, JPEG, PNG, HEIC) and size (≤ 10 MB).
- **Errors:** users see generic messages. Details go to server logs with no PII or tokens.

### 5. Data integrity

- Issued invoices are immutable (trigger-enforced), and numbers are gap-free and never reused. Corrections happen via void + reissue.
- Money totals are calculated by the database, never accepted from the client.
- `audit_log` captures who changed what on financial tables. It is append-only, even for Admins.

### 6. Operations

- **Backups:** Supabase Pro plan with daily backups. Point-in-time recovery is recommended for production. Restores are tested once per quarter on staging.
- **Dependencies:** Dependabot / Renovate, `pnpm audit` in CI, lockfile committed, minimal dependency footprint.
- **Monitoring:** Vercel runtime logs and error alerts, Supabase auth and Postgres logs, and alerts on repeated auth failures.
- **Access to infrastructure:** Vercel, Supabase and GitHub accounts all require 2FA. Production database access is limited to named Admins.
- **Privacy:** client data is processed for contract and invoicing purposes. Data export and deletion requests are handled by Admins. Archived clients with no invoices can be hard-deleted. Invoice records are retained for the statutory period (UK: 6 years).

## Security checklist for every PR

- [ ] New table? RLS enabled, policies for each role, pgTAP tests for allow **and** deny.
- [ ] New `SECURITY DEFINER` function? Role check first, `set search_path = ''`, deny test.
- [ ] No secret-key usage outside `lib/supabase/admin.ts`.
- [ ] Server Action validates input with Zod.
- [ ] No sensitive data in client components' props beyond what the user's role may see.

## Reporting a vulnerability

Contact the project maintainers privately. Don't open a public issue.
