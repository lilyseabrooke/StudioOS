# StudioOS

A lightweight business-management platform for **Studio North**, a small creative agency. It replaces a patchwork of separate tools with one secure web app for:

1. **Clients & projects:** client records, project budgets, and budget burn
2. **Invoicing:** create invoices with line items, send them as PDFs, track payment
3. **Teams & tasks:** many-to-many teams, tasks assigned to teams and people
4. **Time tracking:** staff log hours against projects to show the true labour cost
5. **Revenue & expenses:** collected revenue, categorised expenses, profit
6. **Business-health dashboard:** KPIs, cash position, at-risk projects, utilisation

> **Status:** Planning. This repository holds the architecture and documentation only. No application code yet.

## Tech stack

| Layer | Choice |
| --- | --- |
| Frontend / server | [Next.js](https://nextjs.org) (App Router, React Server Components, Server Actions), TypeScript |
| UI | Tailwind CSS + [shadcn/ui](https://ui.shadcn.com), Recharts for charts |
| Backend | [Supabase](https://supabase.com): Postgres, Auth, Storage, Row Level Security |
| Validation | Zod (shared between forms and Server Actions) |
| PDFs / email | `@react-pdf/renderer` for invoice PDFs, Resend for transactional email |
| Hosting | [Vercel](https://vercel.com) (app) + Supabase Cloud (database) |
| Testing | Vitest (unit), Playwright (end-to-end), pgTAP (database / RLS policies) |

## Documentation

| Doc | What's in it |
| --- | --- |
| [Architecture](docs/ARCHITECTURE.md) | System design, request flow, folder structure, environments, key decisions |
| [Data model](docs/DATA_MODEL.md) | Database schema, relationships, derived metrics, RLS access matrix |
| [Security](docs/SECURITY.md) | Threat model, authentication, authorisation, secrets, operational controls |
| [Contributing](CONTRIBUTING.md) | Workflow, conventions, migration rules, PR checklist |
| [CLAUDE.md](CLAUDE.md) | Guidance for AI coding agents working in this repo |
| Roadmap | *To be agreed after these docs are reviewed* |

## Roles at a glance

| Role | Can do |
| --- | --- |
| **Admin** | Everything: finances, invoices, expenses, staff cost rates, user management, settings |
| **Manager** | Clients, projects (incl. budgets and burn), teams, tasks, everyone's time entries |
| **Member** | Their teams' tasks, their own time entries, read-only project directory |

Full detail: [Data model, access matrix](docs/DATA_MODEL.md#access-matrix).

## Getting started

*Not yet available.* Once the app is scaffolded, local development will need Node.js (LTS), pnpm, Docker, and the Supabase CLI:

```bash
pnpm install
supabase start          # local Postgres, Auth and Storage in Docker
cp .env.example .env.local
pnpm db:reset           # apply migrations and seed data
pnpm dev
```
