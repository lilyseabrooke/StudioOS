# Contributing to StudioOS

## Workflow

1. Branch from `main`: `feat/<short-name>`, `fix/<short-name>`, `chore/<short-name>`, `docs/<short-name>`.
2. Keep PRs small and focused on one feature or fix. A schema change and the UI that uses it may ship together.
3. Open a PR. Vercel creates a preview deployment against the staging database.
4. CI must be green: lint, typecheck, unit tests, pgTAP, build, E2E.
5. One approving review, then **squash merge**. `main` is always deployable.

> The initial planning docs were committed directly to `main`. All work from here on goes through PRs.

## Commit messages

[Conventional Commits](https://www.conventionalcommits.org): `feat(invoices): add void action`, `fix(time): reject entries over 24h`, `chore(deps): …`, `docs: …`.

## Code conventions

- **TypeScript strict mode.** No `any` without a comment explaining why.
- **Feature folders:** domain code lives in `src/features/<domain>/` (`queries.ts`, `actions.ts`, `schemas.ts`, `components/`). Shared UI lives in `src/components/`.
- **Server first:** default to Server Components. Add `'use client'` only for interactivity, and keep client components small.
- **Reads** go in `queries.ts` (server-only) and **writes** in `actions.ts` (`'use server'`). Every action parses input with the domain's Zod schema and returns `{ ok: true, data } | { ok: false, error }`.
- **Supabase clients:** `lib/supabase/server.ts` in server code, `lib/supabase/client.ts` only for auth UI in the browser. `lib/supabase/admin.ts` (secret key) is reserved for admin-only user-management paths.
- **Money:** never sum, multiply or round money in TypeScript for anything that gets stored. The database computes totals. `lib/money.ts` is for **display formatting** only.
- **Dates:** store `date` for calendar dates (invoice dates, entry dates) and `timestamptz` for moments. Display in the company's time zone.
- **Styling:** Tailwind + shadcn/ui components. Don't add another component library.
- **Accessibility:** use semantic HTML and labelled form fields, make everything keyboard reachable, and check colour contrast. shadcn/Radix primitives handle most of this.

## Database changes

- Create a migration: `supabase migration new <description>`. Write SQL by hand, and keep it reviewable.
- **Never edit a migration that has been merged.** Write a new one.
- Every new table needs: RLS enabled, policies for admin/manager/member, `created_at`/`updated_at`, and pgTAP tests in `supabase/tests/` covering allowed **and** denied access per role.
- Every `SECURITY DEFINER` function needs `set search_path = ''`, a role check as its first statement, and a deny test.
- Regenerate types after migrating: `pnpm db:types`. Commit `src/lib/database.types.ts`.
- `supabase/seed.sql` is for local/dev demo data only. It never runs against production.

## Testing

| Layer | Tool | Expectation |
| --- | --- | --- |
| Database / RLS / RPC | pgTAP (`pnpm test:db`) | Required for every table, policy and function |
| Pure logic, schemas | Vitest (`pnpm test`) | Required for non-trivial logic |
| Critical journeys | Playwright (`pnpm test:e2e`) | Sign in, create invoice → issue → mark paid, log time, member can't see finances |

## Planned scripts

| Script | Does |
| --- | --- |
| `pnpm dev` | Run Next.js locally |
| `pnpm lint` / `pnpm typecheck` | ESLint / `tsc --noEmit` |
| `pnpm test` / `pnpm test:e2e` | Vitest / Playwright |
| `pnpm db:reset` | `supabase db reset`: re-apply migrations and seed locally |
| `pnpm db:types` | Regenerate `database.types.ts` from the local schema |
| `pnpm test:db` | `supabase test db`: run pgTAP tests |

## PR checklist

- [ ] Scoped to one change. Description explains *why*.
- [ ] Lint, typecheck and tests pass locally.
- [ ] Migrations + pgTAP tests + regenerated types included (if schema changed).
- [ ] [Security checklist](docs/SECURITY.md#security-checklist-for-every-pr) reviewed.
- [ ] Docs updated if behaviour, schema or roles changed.
- [ ] Screenshots for UI changes (desktop + mobile width).
