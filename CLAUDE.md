# CLAUDE.md

Guidance for AI coding agents working in this repository.

## Project

StudioOS: business-management web app for Studio North (small creative agency). It covers clients and project budgets, invoicing, teams and tasks, time tracking, revenue and expenses, and a business-health dashboard.

**Stack:** Next.js App Router + TypeScript, Tailwind + shadcn/ui, Supabase (Postgres, Auth, Storage, RLS), Zod, deployed on Vercel.

**Status:** planning. Read these before writing code:
- `docs/ARCHITECTURE.md`: structure, request flow, where logic lives
- `docs/DATA_MODEL.md`: schema, RPCs, metrics, **access matrix**
- `docs/SECURITY.md`: controls and the per-PR security checklist
- `CONTRIBUTING.md`: conventions, migration rules, scripts

## Non-negotiable rules

1. **RLS is the security boundary.** Every `public` table has RLS enabled with per-role policies. Never "fix" an access error by disabling RLS, widening a policy beyond the access matrix, or switching to the secret-key client.
2. **The secret key (`SUPABASE_SECRET_KEY`) is used only in `src/lib/supabase/admin.ts`**, only for admin user management, after an explicit admin check. Never import it into client components, and never prefix it `NEXT_PUBLIC_`.
3. **Money is computed in Postgres** (`numeric`, triggers, RPCs). Never calculate stored totals, tax or burn in TypeScript.
4. **Issued invoices are immutable.** State changes go through `issue_invoice` / `mark_invoice_paid` / `void_invoice` RPCs only.
5. **Never edit a merged migration.** Add a new one, then regenerate types (`pnpm db:types`).
6. **Members must never receive financial data**: not in queries, not in props passed to client components, not in RPC results.
7. Validate every Server Action input with the domain's Zod schema.

## Conventions

- Domain code: `src/features/<domain>/{queries.ts,actions.ts,schemas.ts,components/}`.
- Server Components by default. `'use client'` only where interactivity requires it.
- Verify identity server-side with `requireUser()` / `requireRole()` from `src/lib/auth.ts`.
- Sensitive columns go in sibling tables with stricter RLS (pattern: `projects` / `project_budgets`).
- New table or `SECURITY DEFINER` function ⇒ pgTAP allow **and** deny tests in `supabase/tests/`.
- Conventional Commits. Small, focused PRs.

## Commands (once scaffolded)

```bash
pnpm dev            # Next.js dev server
pnpm lint           # ESLint
pnpm typecheck      # tsc --noEmit
pnpm test           # Vitest
pnpm test:db        # pgTAP via supabase test db
pnpm test:e2e       # Playwright
pnpm db:reset       # re-apply migrations + seed locally
pnpm db:types       # regenerate src/lib/database.types.ts
```

Before declaring work done: lint, typecheck, and the relevant tests pass. If you changed the schema, `test:db` passes and the types are regenerated.
