# DeeplySecure

Enterprise Security Awareness Training Platform — a multi-tenant SaaS product for organizations to train employees on cybersecurity through interactive learning modules and phishing simulations.

## Table of Contents

- [Architecture Overview](#architecture-overview)
- [Monorepo Structure](#monorepo-structure)
- [Tech Stack](#tech-stack)
- [Prerequisites](#prerequisites)
- [Environment Setup](#environment-setup)
- [Supabase (Shared Database)](#supabase-shared-database)
- [Admin Dashboard (`admin/`)](#admin-dashboard-admin)
- [User Portal (`user/`)](#user-portal-user)
- [Super Admin Panel (`superadmin/`)](#super-admin-panel-superadmin)
- [Report API (`api/`)](#report-api-api)
- [Add-ons](#add-ons)
- [Port Allocation](#port-allocation)
- [Design System](#design-system)
- [Content Guidelines](#content-guidelines)
- [Testing](#testing)
- [AI Development Notes](#ai-development-notes)

---

## Architecture Overview

DeeplySecure is a monorepo with three full-stack TypeScript applications plus a standalone public report API service, all sharing a single Supabase backend (PostgreSQL + Auth + RLS):

| Application | Purpose | Audience |
|-------------|---------|----------|
| **admin/** | Organization admin dashboard | Security leaders, compliance managers, IT admins |
| **user/** | End-user learning portal | Employees completing security training |
| **superadmin/** | Platform management panel | DeeplySecure platform operators |
| **api/** | Shared public report API for Outlook and Gmail report-button flows | Add-ins and public clients |

All three apps connect to the same Supabase project, share the same database schema and migrations, and follow identical architectural patterns (Express + React + Node.js).

---

## Monorepo Structure

```
deeplysecure/
├── admin/                  # Admin Dashboard (port 3000, Dockerfile)
│   ├── client/src/         # React frontend
│   │   ├── pages/          # Route pages
│   │   ├── components/     # UI components + layout
│   │   ├── lib/            # Utilities, dummy data, navigation config
│   │   └── hooks/          # Custom React hooks
│   ├── server/             # Express backend
│   │   ├── routes/         # API route handlers
│   │   ├── config/         # Template mappings, etc.
│   │   └── lib/            # Server utilities
│   ├── shared/             # Shared types + utilities
│   │   ├── types.ts        # TypeScript type definitions (mirrors DB schema)
│   │   ├── database.types.ts  # Generated Supabase types
│   │   └── password-validation.ts
│   └── test/               # Unit + integration tests
│
├── user/                   # User Portal (port 4000, Dockerfile)
│   ├── client/src/         # React frontend
│   │   ├── pages/          # Route pages
│   │   ├── components/     # UI components + layout
│   │   ├── context/        # React context (auth)
│   │   ├── hooks/          # Custom React hooks
│   │   ├── lib/            # Utilities, helpers
│   │   └── content/        # Learning module content (markdown, etc.)
│   ├── server/             # Express backend
│   │   ├── routes/         # API route handlers
│   │   ├── middleware/     # Auth middleware
│   │   └── lib/            # Server utilities (Supabase client)
│   ├── shared/             # Shared types + utilities
│   │   ├── database.types.ts  # Generated Supabase types
│   │   └── password-validation.ts
│   └── test/               # Tests
│
├── superadmin/             # Super Admin Panel (port 5002, Dockerfile)
│   ├── client/src/         # React frontend
│   │   ├── pages/          # Route pages
│   │   ├── components/     # UI components + layout
│   │   ├── context/        # React context (auth, password setup)
│   │   ├── hooks/          # Custom React hooks
│   │   └── lib/            # Utilities (supabase client, query client)
│   ├── server/             # Express backend
│   │   ├── routes/         # API route handlers
│   │   ├── middleware/     # Auth middleware (super admin validation)
│   │   └── lib/            # Server utilities
│   ├── shared/             # Shared types + utilities
│   │   ├── database.types.ts  # Generated Supabase types
│   │   ├── types.ts        # Shared type definitions
│   │   └── password-validation.ts
│   ├── script/             # Build + utility scripts
│   └── test/               # Auth + unit tests
│
├── api/                    # Shared report API (port 3200)
│   ├── src/                # NestJS application source
│   ├── test/               # Unit + e2e tests
│   ├── Dockerfile
│   └── docker-compose.yml
│
├── addons/
│   ├── google/             # Gmail add-on Apps Script source
│   │   ├── Code.gs         # Gmail add-on logic
│   │   ├── appsscript.json # Apps Script manifest + scopes
│   │   ├── Code.test.js    # Local helper tests
│   │   └── README.md
│   └── outlook/            # Outlook add-in static file server (port 3100, Docker)
│       ├── public/         # Add-in HTML/JS/CSS/icons
│       ├── src/            # Express server
│       ├── Dockerfile
│       └── docker-compose.yml
│
└── supabase/               # Shared Supabase configuration
    ├── config.toml         # Supabase local dev config
    ├── email-templates/    # Shared invite/magic-link/recovery auth emails
    ├── seed.sql            # Seed data (learning modules catalog)
    └── migrations/         # SQL migration files (shared by all apps)
```

---

## Tech Stack

All three applications share the same core stack:

### Runtime & Tooling
- **Node.js 22** — runtime for all application services
- **pnpm** — package manager (monorepo workspace at repo root)
- **tsx** — TypeScript execution for dev and build scripts
- **Vitest** — test runner
- **TypeScript** — language across all projects
- **Vite** — client bundling and dev server
- **esbuild** — server bundling for production

### Frontend (all apps)
- **React 19** + TypeScript
- **Wouter** — routing
- **TanStack React Query** — data fetching and caching
- **React Hook Form** + **Zod** — form handling and validation
- **Tailwind CSS 4.1** — styling
- **Radix UI** + **shadcn/ui** — component primitives
- **Lucide React** — icons
- **Framer Motion** — animations

### Additional Frontend Libraries (per app)
- **admin**: Recharts (charts)
- **user**: React Markdown (content rendering), next-themes, Recharts
- **superadmin**: Recharts

### Backend
- **Express.js** + TypeScript — `admin/`, `user/`, `superadmin/`
- **NestJS** — `api/` shared report API
- **Supabase** (PostgreSQL + Auth + Row Level Security)
- **WebSocket** support via `ws`

### Database
- **PostgreSQL 17** via Supabase
- **Row Level Security (RLS)** for multi-tenant data isolation
- **Supabase Auth** for authentication (password-based plus app-owned email-link confirmation)

---

## Prerequisites

- **Node.js 22** (latest LTS) — use `.nvmrc` or your version manager
- **pnpm** — enabled via Corepack: `corepack enable`
- **Supabase CLI** — `brew install supabase/tap/supabase` then `supabase login`
- **Docker** — required for local Supabase stack
- Access to the Supabase project keys

### Important: Use pnpm and Node.js

The four application/services (`admin/`, `user/`, `superadmin/`, and `api/`) use Node.js as the runtime with pnpm for dependency management. Install dependencies once at the repo root with `pnpm install`. The add-on helper harnesses in `addons/google/` and `addons/outlook/` use `node --test` for local verification.

For application code, follow these conventions:
- `tsx <file>` or `node <file>` instead of `ts-node <file>`
- `pnpm run test` (Vitest) instead of `jest`
- `pnpm install` at repo root instead of per-app installs
- `pnpm run <script>` instead of `npm run <script>`
- `pnpm exec <package>` instead of `npx <package>`
- Environment variables are loaded via `dotenv` in `server/env.ts` preload scripts

---

## Environment Setup

Each app needs its own `.env` file (or `.env.development` for local development). Environment files are git-ignored.

### Shared Environment Variables (all apps)

```bash
# Supabase Configuration
SUPABASE_URL=http://127.0.0.1:54321          # Supabase API URL
SUPABASE_ANON_KEY=<your-anon-key>             # Public/anon key
SUPABASE_SERVICE_ROLE_KEY=<your-service-key>  # Service role key (server-side only, bypasses RLS)

# Vite Frontend (prefix with VITE_)
VITE_SUPABASE_URL=http://127.0.0.1:54321
VITE_SUPABASE_ANON_KEY=<your-anon-key>

# Database (direct connection)
DATABASE_URL=postgresql://postgres:postgres@127.0.0.1:54322/postgres
```

### Per-App Variables

| Variable | admin | user | superadmin |
|----------|-------|------|------------|
| `PORT` | `3000` | `4000` | `5002` |
| `HOST` | Optional — bind address (default `0.0.0.0`) | Optional — bind address (default `0.0.0.0`) | Optional — bind address (default `0.0.0.0`) |
| `NODE_ENV` | `development` | `development` | `development` |
| `PHISHER_ENCRYPTION_KEY` | Required (phishing) | Required (phishing landing) | Required (Phisher provisioning) |
| `PHISHER_URL` | — | — | Required (Phisher instance base URL for auto-provisioning) |
| `PHISHER_MASTER_APP_JWT_SECRET` | — | — | Required (shared secret for Phisher provisioning JWT auth) |
| `SUPERADMIN_URL` | — | — | `http://localhost:5002` |
| `MAGIC_LINK_REDIRECT_URL` | `http://localhost:4000` | — | — |
| `MAGIC_LINK_REDIRECT_URL_ADMIN` | `http://localhost:3000` | — | `http://localhost:3000` |
| `FRAMER_WEBHOOK_SECRET` | — | — | Shared secret for Framer webhook signature verification (min 32 chars) |

### Report Button Variables

| Variable | Location | Description |
|----------|----------|-------------|
| `PHISHING_REPORT_ADDIN_URL` | `admin/.env` | Public HTTPS URL of the Outlook add-in static host |
| `PHISHING_REPORT_API_URL` | `admin/.env` | Public HTTPS URL of the shared report API |

### Report API Variables (`api/`)

| Variable | Description |
|----------|-------------|
| `PORT` | API server port (default `3200`) |
| `SUPABASE_URL` | Supabase project URL |
| `SUPABASE_ANON_KEY` | Supabase anon/public key |
| `SUPABASE_SERVICE_ROLE_KEY` | Supabase service-role key for server-side config lookups |
| `SMTP_HOST` | SMTP server hostname used for forwarded suspicious-email reports |
| `SMTP_PORT` | SMTP server port (default `587`) |
| `SMTP_USER` | SMTP username |
| `SMTP_PASS` | SMTP password |
| `SMTP_FROM` | Sender address for report emails |
| `CORS_ADDITIONAL_ORIGINS` | Optional comma-separated list of extra allowed origins (e.g. `https://report.example.com`). The API already allows Outlook/Office origins, `https://report.deeply.cloud`, and `http://localhost:*` for the report taskpane. |

### Supabase SMTP (for email sending in config.toml)

```bash
SUPABASE_SMTP_HOST=
SUPABASE_SMTP_USER=
SUPABASE_SMTP_PASS=
SUPABASE_SMTP_ADMIN_EMAIL=
SUPABASE_SMTP_SENDER_NAME=
```

---

## Supabase (Shared Database)

All three applications share the same Supabase project. Migrations, schema, and seed data are centralized in `supabase/`.

### Local Development Stack

```bash
# Start local Supabase (PostgreSQL, Auth, Studio, Inbucket)
supabase start --workdir supabase

# Stop local Supabase
supabase stop --workdir supabase
```

Local service ports:
- **API**: `54321`
- **Database**: `54322`
- **Studio**: `54323` (web UI at http://127.0.0.1:54323)
- **Inbucket** (email testing): `54324`

### Auth Email Flow

Supabase email-auth flows now use an app-owned interstitial confirmation page before the token is consumed:

- Invite, magic-link/sign-in, and password-reset emails land on `/auth/confirm` in the target app first.
- The email templates use `{{ .RedirectTo }}` plus `{{ .TokenHash }}` (`type=invite`, `type=email`, `type=recovery`) instead of linking directly to `{{ .ConfirmationURL }}`.
- The browser only calls `supabase.auth.verifyOtp(...)` after the user clicks the confirmation button, which prevents Outlook/Safe Links style prefetchers from consuming the token first.
- Already-sent legacy `?code=` and hash-based recovery links are still accepted during the rollout window.
- Local `supabase/config.toml` now allow-lists admin (`3000`), user (`4000`), and superadmin (`5002`) callback origins and points to the shared templates in `supabase/email-templates/`.

### Local Auth Email Testing

Use Inbucket to verify the new flow locally:

1. Start Supabase: `supabase start --workdir supabase`
2. Start the relevant frontend(s): `cd admin && pnpm run dev`, `cd user && pnpm run dev`, `cd superadmin && pnpm run dev`
3. Trigger one of the migrated flows:
   - admin managed-user invite / resend
   - admin forgot-password
   - superadmin simple/full admin invite
   - superadmin invite
4. Open Inbucket at `http://127.0.0.1:54324`
5. Confirm the email link points to the target app’s `/auth/confirm?next=...` route and includes `token_hash=...` plus the expected `type=...`
6. Open the link and verify that no session is created until the button on the interstitial page is clicked
7. For migration safety, also test one legacy `?code=` or hash-based recovery link and confirm it still initializes correctly

### Local MFA Testing

TOTP MFA is enabled in local development through `supabase/config.toml`.
For hosted environments, you must also enable TOTP MFA in the Supabase project settings before the admin and superadmin apps can enroll or verify factors.

Use this flow to verify optional staff MFA locally:

1. Start Supabase: `supabase start --workdir supabase`
2. Start the staff frontends you need: `cd admin && pnpm run dev`, `cd superadmin && pnpm run dev`
3. Sign in as an admin or superadmin and open `Settings > Security`
4. Start TOTP enrollment, scan the QR code with an authenticator app (or enter the setup secret manually), and verify with a 6-digit code
5. Sign out and sign back in to confirm enrolled accounts are prompted for a TOTP code before the dashboard loads
6. Verify a wrong code is rejected, a correct code upgrades the session, and disabling MFA from Settings removes the extra login step

### Migration Workflow

```bash
# Create a new migration
supabase migration new <name> --workdir supabase

# Apply migrations locally (recreates DB + runs seed)
supabase db reset --workdir supabase

# Push migrations to remote (production/staging)
supabase db push --workdir supabase
```

### Type Generation (keep all apps in sync)

After any migration change, regenerate types in **all three** apps:

```bash
cd admin && pnpm run db:types       # → admin/shared/database.types.ts
cd user && pnpm run db:types        # → user/shared/database.types.ts
cd superadmin && pnpm run db:types  # → superadmin/shared/database.types.ts
```

### Schema Change Workflow

1. Create/edit migration in `supabase/migrations/`
2. Apply locally: `supabase db reset --workdir supabase`
3. Regenerate types in **all three** apps: `pnpm run db:types`
4. Commit migration files + regenerated type files

### Learning Catalog Revision Workflow

Use this workflow for the English-key / English-base catalog revision driven by `modules-list.csv`:

1. Verify the matching learner content folders exist under `user/client/src/content/modules-v2/` for every target key referenced by `modules-list.csv`
2. If `modules-list.csv` or `superadmin/script/module-key-remap.ts` changes, regenerate the SQL payload:

```bash
cd superadmin && pnpm run generate:learning-catalog-revision-sql
```

3. Preview the live database before applying anything:

```bash
cd superadmin && pnpm run preview:learning-catalog-revision
```

4. Apply the pending migration to the target database:

```bash
supabase db push --workdir supabase
```

5. Re-run the preview command and check the reported `Catalog state`:
   - Before applying the migration, a healthy catalog reports `pre-migration` with zero missing remap sources.
   - After applying the migration, a healthy catalog reports `post-migration` with zero missing revised target rows, zero duplicate assignment collisions, and zero missing retired-legacy content files. The old remap source rows are expected to be absent at that point because the migration deletes them after reassignment.
6. The preview may still list unpublished revised modules with missing content files. Those rows can safely remain in the DB as long as they stay unpublished and out of normal assignment flows.

### Key Database Tables

| Table | Description |
|-------|-------------|
| `organizations` | Tenant organizations (name, domain, subscription, seat limits, `default_language`, `branding_logo_url`, `branding_primary_color`) |
| `admin_users` | Organization administrators (linked to Supabase Auth) |
| `managed_users` | End users managed by organizations |
| `departments` | Organization departments |
| `tags` / `managed_user_tags` | Tagging system for managed users |
| `learning_modules` | Training module catalog (`key` + `version` canonical row, `is_retired` hides legacy rows, `is_published` hides unfinished revised rows from normal assignment flows) |
| `learning_module_translations` | Localized module metadata (`title`, `description`, `category`, block titles) per locale |
| `learning_module_assignments` | Module assignments to users (resolved by exact `module_key` + `module_version`) |
| `learning_progress` / `learning_quiz_tracking` | User progress tracking |
| `phisher_connections` | Phisher integration config (`api_endpoint`, `api_key_encrypted`, `phisher_tenant_id`, per org) |
| `report_button_public_configs` | Public per-org IDs used by Outlook/Gmail report-button clients |
| `super_admins` | Platform-level super administrators |
| `super_admin_audit_log` | Audit trail for super admin actions |

Organization branding assets are stored in the private Supabase Storage bucket `organization-assets` with RLS policies scoped to each organization.
Admin profile avatars are stored in the private Supabase Storage bucket `admin-avatars` at `{org_id}/{admin_user_id}/avatar`, with RLS policies restricting read/write access to the authenticated admin's own path.
Both `admin/` and `user/` dashboards consume `organizations.branding_logo_url` and `organizations.branding_primary_color`, with safe fallbacks to the default DeeplySecure logo and default primary color when values are unset/invalid.
In Admin Settings, resetting the organization primary color clears the custom value and persists `organizations.branding_primary_color` as `NULL`.
Dashboard theming derives hover/active button colors and a soft secondary accent from the organization primary color to keep interactive states visually consistent.

### Seed Data

`supabase/seed.sql` still seeds the legacy **28 learning modules** across categories:
- Grundlagen (Basics), Verwaltung (Administration), IT, Management
- Fachbereich (Department), Governance, Homeoffice, Reisen (Travel), Smartphone

The follow-up migration `20260401010000_learning_catalog_revision.sql` expands that baseline into the revised English-key catalog from `modules-list.csv`, remaps the 30 approved legacy assignments in place, and keeps 5 unmatched manager modules as retired legacy rows.
It also marks only content-complete revised rows as published, so unfinished modules can exist in the database without appearing in normal admin/user catalog flows.
The follow-up migration `20260404130000_backfill_missing_de_learning_module_translations.sql` backfills German metadata rows for revised modules that previously only had English DB metadata, while modules without localized body assets still rely on the existing runtime English-content fallback.
The follow-up migration `20260416120000_backfill_fr_it_learning_module_translations.sql` backfills French and Italian metadata rows for revised modules with locale Vimeo IDs defined in `modules-list.csv`, using English DB metadata as the source baseline for block mappings.
The follow-up migration `20260416183000_allow_italian_language_guardrails.sql` widens the persisted language guardrails for admins, managed users, organizations, and `update_my_managed_user_language(...)` from `en`/`de` to `en`/`de`/`it`.
The follow-up migration `20260417100000_allow_french_language_guardrails.sql` extends those persisted language guardrails and the managed-user language RPC again so the supported stored set becomes `en`/`de`/`fr`/`it`.
The follow-up migration `20260504200000_backfill_fr_it_learning_module_vimeo_ids.sql` backfills newly available French and Italian Vimeo overrides from `modules-list.csv` for revised modules, preserving existing translated metadata while inserting English-baseline rows where a locale row was still missing.

### Row Level Security (RLS)

Multi-tenant data isolation is enforced at the database level via RLS policies. Key patterns:
- Organizations can only see their own data
- Admin users can manage their organization's users
- Managed users can read their own data and organization-level data
- Super admins bypass organization-scoped policies
- Seat limits enforced via triggers

Migration files for RLS policies are in `supabase/migrations/` (files named `*_rls_*.sql`).

---

## Admin Dashboard (`admin/`)

**Product:** DeeplySecure Admin Dashboard
**Target Users:** Security leaders, compliance managers, IT administrators
**Design Philosophy:** Minimalist, glassy morphism, elegant simplicity inspired by Linear and Attio

### Commands

```bash
cd admin
pnpm install          # Install dependencies (from repo root)
pnpm run dev          # Start full-stack dev server (port 3000)
pnpm run build        # Build for production
pnpm run start        # Run production server
pnpm run db:types     # Generate Supabase TypeScript types
pnpm run test         # Run tests
```

### Implemented Pages

| Route | Page | Status |
|-------|------|--------|
| `/` | Dashboard | Complete UI with dummy data |
| `/users` | Users & Groups | **Fully functional** — real data with pagination, filtering, CRUD |
| `/users/new` | User Create Wizard | **Fully functional** — creates users with Supabase auth invite emails |
| `/users/import` | User Import Wizard | **Fully functional** — CSV/Excel import with validation |
| `/training` | Training Catalog | Module grid with filters and department-grouped assignment manager |
| `/phishing/simulation` | Phishing Simulation | Campaign list with tabs (active/completed) |
| `/phishing/templates` | Template Library | Browsable template grid with filters and preview |
| `/phishing/template-studio` | Template Studio | Create/edit templates with WYSIWYG editor, domain-picker sender address; `?edit=<id>` for edit mode; import from email (coming soon) |
| `/phishing/create` | Campaign Creator | 5-step wizard UI |
| `/campaigns/:id` | Campaign Details | Results and analytics view |
| `/analytics` | Analytics | KPI cards, charts, reports |
| `/settings` | Settings | Account (profile + language), Security (password change + TOTP 2FA), Phishing Report Button, Billing |
| `/support` | Support Center | Markdown-driven FAQ/help articles with search, category filters, locale-aware content |

### Backend API Routes

**Managed Users (`/api/managed-users`):**
- `GET /` — List with pagination, filtering (status, department, search)
- `GET /:id` — Single user with department and tags
- `POST /` — Create user with Supabase auth invite email
- `PUT /:id` — Update user details and tags
- `DELETE /:id` — Delete user (updates seat count)
- `POST /:id/resend-invite` — Resend invite / sign-in email
- `POST /import` — Bulk import from CSV/Excel (no invite emails)

### Key Features

**Managed Users (Fully Implemented):**
- Server-side pagination (20 items/page default, max 100)
- Search by name or email, filter by status and department
- Role-based access control (admin / member roles)
- Seat limit enforcement (validates against subscription)
- Domain restriction (emails must match organization domain)
- Supabase auth emails now use app-owned `/auth/confirm` interstitial pages before token verification
- New managed users inherit `organizations.default_language`; invite metadata includes `language` for localized Supabase templates
- Cross-tenant security via RLS

**Onboarding Flow (3-step wizard):**
1. Profile Information (name, email, job title)
2. Organization Creation (name, default language `en`/`de`/`fr`/`it`, auto-sets domain from admin email, 10 trial seats)
3. Password Setup (OWASP/NIST compliant requirements)

**Password Requirements (OWASP/NIST):**
- Minimum 12 characters
- At least one uppercase, one lowercase, one number, one special character

**User Import:**
- CSV/Excel upload with validation
- Auto-creates departments, skips duplicate emails
- Seat limit and domain validation
- Does NOT send invite emails (users remain "pending")

**Password Change (Settings > Security):**
- Current password verification before change
- Same OWASP/NIST requirements as onboarding
- Session refresh after change

**Admin Language Preference (Settings > Account):**
- Source of truth: `admin_users.language` (`en` / `de` / `fr` / `it`)
- UI displays full labels (`English`, `Deutsch`, `Français`, `Italiano`) while persisting shortcodes
- Admin app applies language from DB on auth/membership load

**Phishing (sub-navigation: Simulation, Templates, Template Studio):**
- Sidebar expands with sub-items like Analytics; routes: `/phishing/simulation`, `/phishing/templates`, `/phishing/template-studio`
- **Simulation** — Campaign list (active/completed tabs), sort, pagination, create/complete/delete actions
- **Templates** — Browsable library grid with search, category & difficulty filters, scaled email preview thumbnails, full-size preview dialog
- **Template Studio** — Unified create/edit email-composer layout (`?edit=<id>` for edit mode) with Tiptap WYSIWYG rich text editor (formatting toolbar: bold/italic/underline/strikethrough, headings, lists, alignment, links, undo/redo), inline placeholder variable chips (`{first_name}` etc. rendered as styled badges), `{link}` tracking-URL variable (required, prompts for visible link text on insert, renders as `<a href="{link}">text</a>` in storage/preview, shown as chip in editor), difficulty pill selector, domain-picker sender address (local-part text input + domain dropdown populated from Phisher `/api/v1/domains`), subject line, collapsible description/plain-text section, preview dialog, and delete action in edit mode. Edit button on template cards navigates to studio. Email template import via `POST /api/phisher/templates/import` (EML, HTML, plain text formats).
- **Scenarios** — LLM-based scenario templates for AI-generated phishing emails. CRUD via `/api/phisher/scenarios` with preview support. Scenarios define system/user prompts for per-target email generation.
- Phisher API integration for campaign management
- Per-organization connection with encrypted API key stored in `phisher_connections`
- Campaign creation wizard (5 steps: Basics, Type, Template, Audience, Launch) with campaign type selection (Basic / Enhanced — enhanced coming soon), email template selection, department-grouped target selection with individual user checkboxes (including users without departments), and auto-launch
- Campaign-scoped email template management: `GET/PATCH /api/phisher/campaigns/:id/template`, preview via `POST /api/phisher/campaigns/:id/template/preview`, reset to source via `POST /api/phisher/campaigns/:id/template/reset`
- Individual campaign email review: `GET/PATCH /api/phisher/campaigns/:id/emails/:emailId`, approve/reject/regenerate actions for LLM-generated emails in review status
- Target sync: managed users are synced to Phisher as individual targets
- Target difficulty: adaptive difficulty evaluation (`POST /api/phisher/targets/difficulty/evaluate`), auto-grouping by difficulty (`POST /api/phisher/targets/difficulty/auto-group`), and summary (`GET /api/phisher/targets/difficulty/summary`)
- Campaign detail page: KPI cards for open rate, click rate, submission rate, and report rate; activity chart with report events; CSV export button
- CSV export: download per-target campaign results via `GET /api/phisher/campaigns/:id/results/export`
- Email templates now support `from_address_domain_id` and `link_domain_id` for domain-based sender address and tracking link URL configuration. Template Studio automatically sets `link_domain_id` to match `from_address_domain_id` (selected sender domain) on create and update.

**Phishing Reports & Analytics (Awareness Score view on `/analytics`):**
- Tenant overview: aggregated open/click/submission/report rates from `GET /api/phisher/reports/overview`
- Per-campaign summary table showing individual campaign performance
- Department report: horizontal bar chart of click/report rates by department from `GET /api/phisher/reports/departments`
- Target report: per-target cross-campaign interaction history from `GET /api/phisher/reports/targets/:id`
- Backend proxy routes: `GET /api/phisher/reports/overview`, `GET /api/phisher/reports/departments`, `GET /api/phisher/reports/targets/:targetId`
- Phishing rates metric (`GET /api/metrics/phishing-rates`) now uses single `getTenantOverview()` call instead of N+1 campaign loop; response includes `openRate` and `submissionRate`

**E-Learning Analytics (`/analytics/e-learning`):**
- Dedicated analytics tab accessible via sidebar sub-navigation under Analytics
- Summary cards: users with assignments, started rate, completion rate, average quiz score
- Horizontal bar chart: per-department e-learning participation (started vs total users)
- Score gauge: circular SVG arc displaying average score across all quiz results
- Paginated user table with search, department filter, and sortable columns (first name, last name, email, department, registered, all completed, modules passed, score)
- Module scores dialog: click a user's score to view individual module results (0-100)
- Backend endpoints: `GET /api/metrics/elearning-summary`, `GET /api/metrics/elearning-departments`, `GET /api/metrics/elearning-users`
- Localized in English, German, French, and Italian

### Implementation Files Reference

| Feature | Frontend | Backend | Shared |
|---------|----------|---------|--------|
| Managed Users | `client/src/hooks/use-managed-users.ts`, `client/src/pages/users.tsx` | `server/routes/managed-users.ts` | — |
| Onboarding | `client/src/pages/onboarding.tsx` | `server/routes/onboarding.ts` | `shared/password-validation.ts` |
| Password Change | `client/src/components/settings/change-password-form.tsx`, `client/src/hooks/use-change-password.ts` | — (uses Supabase Auth directly) | `shared/password-validation.ts` |
| User Import | `client/src/pages/user-import.tsx` | `server/routes/managed-users.ts` (import endpoint) | — |
| Settings | `client/src/pages/settings.tsx` | — | — |
| E-Learning Analytics | `client/src/components/analytics/elearning-analytics.tsx`, `client/src/hooks/use-metrics.ts` | `server/routes/metrics.ts` (elearning-* endpoints) | — |
| Phishing Reports | `client/src/pages/analytics.tsx` (awareness view), `client/src/pages/campaign-detail.tsx`, `client/src/hooks/use-phisher.ts` (report hooks) | `server/routes/phisher.ts` (report proxy routes), `server/routes/metrics.ts` (phishing-rates), `server/lib/phisher.ts` (report types + methods) | — |
| Phishing Templates | `client/src/components/phishing/phishing-templates.tsx`, `client/src/components/phishing/template-form-shared.tsx`, `client/src/hooks/use-phisher.ts` | `server/routes/phisher.ts` (templates CRUD endpoints) | — |
| Template Studio | `client/src/components/phishing/phishing-template-studio.tsx`, `client/src/components/phishing/rich-text-editor.tsx`, `client/src/components/phishing/placeholder-extension.ts`, `client/src/components/phishing/template-form-shared.tsx`, `client/src/hooks/use-phisher.ts` | `server/routes/phisher.ts` (POST templates endpoint) | — |
| Support Center | `client/src/pages/support.tsx`, `client/src/components/support/support-markdown.tsx`, `client/src/lib/support-content.ts`, `client/src/content/support/` | — (static content, no backend) | — |

### Current Limitations (Admin)

- Most pages beyond Users use **dummy data** from `client/src/lib/dummy-data.ts`
- No email sending for phishing simulations yet

### UI Component Notes

- Use `PopupDialogContent` from `client/src/components/ui/dialog.tsx` for all popup dialogs
- Avoid using `DialogContent` directly for popup dialogs

---

## User Portal (`user/`)

**Product:** DeeplySecure User Portal
**Target Users:** Employees completing security awareness training
**Runtime Notes:** Uses Node.js with pnpm. Environment variables are loaded via `server/env.ts` preload.

### Commands

```bash
cd user
pnpm install          # Install dependencies (from repo root)
pnpm run dev          # Start full-stack dev server (port 4000)
pnpm run build        # Build for production
pnpm run start        # Run production server
pnpm run db:types     # Generate Supabase TypeScript types
pnpm run test         # Run tests
pnpm run check        # TypeScript type checking
```

### Pages & Routes

**Public Routes:**
| Route | Page |
|-------|------|
| `/login` | User login |
| `/forgot-password` | Password reset request |
| `/reset-password` | Password reset form |
| `/phishing-landing` | Phishing simulation landing page |

**Protected Routes (require auth + managed user profile):**
| Route | Page |
|-------|------|
| `/` | Dashboard — daily progress, weekly streak, recent activity |
| `/modules` | Learning modules catalog with category filters and search |
| `/module/:id` | Full-screen module player (video, markdown, quizzes) |
| `/achievements` | Achievements trophy cabinet with streak, progress, mastery, and momentum milestones |
| `/settings` | User settings |
| `/onboarding` | First-time user onboarding |

### Key Features

**Learning Management System:**
- Module assignments from admin
- Block-level progress tracking
- Completion tracking with quiz results
- Weekly learning streaks
- Daily goals (3 blocks/day)
- Achievements trophy cabinet derived from streaks, module completions, quiz results, and focused learning days

**Content Delivery:**
- Markdown rendering for text content
- Vimeo video integration
- Interactive components (quizzes, spot-the-fake exercises)
- Module scheduling and sequencing

**Authentication:**
- Supabase Auth (magic links + password)
- Managed user profile requirement (users must be created by admin)
- Onboarding flow for first-time users
- Session management API (`/api/sessions`)

**Embedded assistant (optional, local dev):**
- Main app shell and full-screen module player load the same `<ds-chatbot>` web component as the admin dashboard (`user/client/index.html`, `user/client/src/components/layout/Layout.tsx`, `user/client/src/components/chatbot-widget.tsx`, `user/client/src/pages/ModulePlayer.tsx`), passing the Supabase access token and `lang` from `managed_users.language`. In the module player, `module-id` is set to the current module key so RAG can scope to that module. Widget script and API base URL point at `https://chatbot.deeplysecure.com` (admin uses the same host via `admin/client/index.html` and `admin/client/src/components/chatbot-widget.tsx`).

### Content Structure

Learning module content lives in `client/src/content/modules-v2/`. Modules are structured with blocks:
- Intro blocks (text/markdown)
- Video blocks (Vimeo integration)
- Terminology blocks (Fachbegriffe)
- Interactive blocks (native React exercises resolved from `interactive/`)
- Knowledge-check quiz blocks

### UI Internationalization (User App Pilot)

The `user/` app includes UI i18n support for `en`, `de`, `fr`, and `it` using `i18next` + `react-i18next`.

- **Bootstrap:** `client/src/i18n/index.ts` initialized from `client/src/main.tsx`
- **Locale resources:** `client/src/i18n/locales/en.ts`, `client/src/i18n/locales/de.ts`, `client/src/i18n/locales/fr.ts`, and `client/src/i18n/locales/it.ts`
- **Language persistence:** `managed_users.language` (`en` / `de` / `fr` / `it`) via `update_my_managed_user_language(p_language text)`
- **Migrations:** `supabase/migrations/20260210000000_managed_users_language.sql`, widened by `supabase/migrations/20260416183000_allow_italian_language_guardrails.sql` and `supabase/migrations/20260417100000_allow_french_language_guardrails.sql`
- **Supported locales:** configured in `client/src/i18n/config.ts`

#### Key conventions

- Use semantic keys, not literal text, e.g. `auth.loginFailedTitle`
- Group keys by namespace: `common`, `navigation`, `auth`, `dashboard`, `modules`, `settings`, `errors`, `layout`
- Keep module player chrome/controls in the `modulePlayer.*` namespace (for example quiz nav, step labels, CTA buttons) to avoid hardcoded copy in `ModulePlayer.tsx`
- Prefer `t("namespace.key")` in components for user-visible strings

#### Adding new UI translations

1. Add keys in `en.ts`, `de.ts`, `fr.ts`, and `it.ts`
2. Use `useTranslation()` and replace hardcoded text with `t("...")`
3. For pluralized text, use i18next plural forms (`_one`, `_other`)
4. Keep module markdown/quiz content outside this UI i18n layer (handled separately)
5. Onboarding questionnaire text in `client/src/content/onboarding/questions.ts` is sourced from locale keys under `onboarding.questions.*`
6. Onboarding page/chrome copy in `client/src/pages/Onboarding.tsx` is sourced from locale keys under `onboarding.page.*`
7. Keep UI language shortcodes in DB (`en`, `de`, `fr`, `it`) and render full language labels in Settings UI

#### Localizing module content (`modules-v2`)

Module body content is localized in module files, not in `client/src/i18n/locales/*.ts`.

- **Localized file convention (current standard):**
  - Markdown blocks: `client/src/content/modules-v2/<module>/blocks/<lang>/<file>.md`
  - Quiz blocks: `client/src/content/modules-v2/<module>/quiz/<lang>/<file>.json`
- **Rollout status:** all current `modules-v2` modules are migrated to locale folder structure with both `de/` and `en/` files across all modules. Italian and French content use the same folder structure (`it/`, `fr/`) where available.
- **Legacy source compatibility:** existing manifest/DB `source` values like `blocks/intro.md` and `quiz/knowledge-check.json` are treated as backward-compatibility fallback paths.
- **Runtime fallback order for content loading:**
  1. requested locale (for example `en`)
  2. default locale (`en`)
  3. other supported locale folders (for example `de`, `fr`, `it`)
  4. legacy non-localized source path
- **Strict locale mode (`it`, `fr`):** when the active user locale resolves to Italian or French, module body/quiz content and module metadata must exist in that locale or the module is omitted instead of falling back to English/German.

#### Native interactive blocks (`modules-v2`)

Interactive exercises are implemented as isolated React modules inside each module folder.

- **Preferred file convention:** `client/src/content/modules-v2/<module>/interactive/<component-key>/index.tsx`
- **Legacy compatibility:** the runtime still supports `client/src/content/modules-v2/<module>/interactive/<component-key>.tsx`, but new work should use the folder-based pattern.
- **Block metadata contract:** keep the module block entry as `{ "type": "interactive", "component": "<component-key>" }` in the `learning_modules.blocks` JSON.
- **Isolation rule:** keep interactive-specific subcomponents, assets, and locale/content files inside the same `interactive/<component-key>/` folder so the module can be added or removed cleanly.
- **Localization rule:** keep interactive copy next to the interactive implementation rather than in app-wide `client/src/i18n/locales/*.ts`; the module player passes the active locale into the interactive component at runtime.
- **Loading behavior:** interactive modules are lazy-loaded in the user module player, with folder entrypoints resolved before legacy single-file entrypoints.
- **Migration workflow:** after copying the required code/assets into `user/client/src/content/modules-v2/...`, the temporary source upload folder can be deleted.

##### Add translated module content checklist

1. Keep existing German file as baseline source (`de`)
2. Add translated file under `blocks/<locale>/` or `quiz/<locale>/` using the same filename (for example `en`, `fr`, or `it`)
3. Keep question IDs and option IDs stable in translated quiz JSON
4. Verify module in UI by switching language in Settings and opening the module

##### Localizing module metadata in DB (`learning_module_translations`)

Module metadata is localized in `learning_module_translations` (not in markdown/quiz files):

- **Source table:** `learning_modules` remains the canonical module/version catalog.
- **Translation table key:** `(module_key, module_version, locale)` where `locale` is a lowercase shortcode such as `en`, `de`, `fr`, `it`, or `es`.
- **Localized fields:** `title`, `description`, `category`, `block_titles` JSONB (`{ "<block-id>": "<localized title>" }`), and `block_vimeo_ids` JSONB (`{ "<block-id>": "<localized-vimeo-id>" }`).
- **Runtime behavior:** apps load `learning_modules` plus locale-filtered `learning_module_translations` and overlay translated metadata.
- **Fallback behavior:** for `en`/`de` (and other non-strict locales), missing translation rows or fields fall back to base values from `learning_modules`. For `it` and `fr`, the translation row must be complete or the module is omitted.
- **Retired rows:** `learning_modules.is_retired = true` keeps legacy modules resolvable for already-assigned learners while admin training/onboarding flows default to active rows only.
- **Published rows:** `learning_modules.is_published = false` keeps unfinished revised modules in the catalog data model without surfacing them in normal assignment and learner discovery flows.

##### Add or update module metadata translations checklist

1. Add/update rows in `learning_module_translations` for the target locale and module version
2. Keep `module_key`/`module_version` aligned with the source row in `learning_modules`
3. Keep `block_titles` keys aligned with block IDs in the module `blocks` JSON
4. Regenerate DB types in all apps after schema changes (`pnpm run db:types`)
5. New metadata locales can be added from the superadmin `/modules/:key/:version` translation editor using lowercase shortcodes
6. Verify localized module card title/description/category and module-player block titles in both `user/` and `admin/`

### UI Internationalization (Admin App)

The `admin/` app includes broad UI i18n coverage for `en`, `de`, `fr`, and `it` using `i18next` + `react-i18next`.

- **Bootstrap:** `admin/client/src/i18n/index.ts` initialized from `admin/client/src/main.tsx`
- **Locale resources:** `admin/client/src/i18n/locales/en.ts`, `admin/client/src/i18n/locales/de.ts`, `admin/client/src/i18n/locales/fr.ts`, and `admin/client/src/i18n/locales/it.ts`
- **Language persistence:** `admin_users.language` (`en` / `de` / `fr` / `it`)
- **Migrations:** `supabase/migrations/20260210000001_admin_users_language_guardrails.sql`, widened by `supabase/migrations/20260416183000_allow_italian_language_guardrails.sql` and `supabase/migrations/20260417100000_allow_french_language_guardrails.sql`
- **Settings behavior:** full names in UI (`English`, `Deutsch`, `Français`, `Italiano`) mapped to DB shortcodes (`en`, `de`, `fr`, `it`)

#### Namespace conventions

- Use semantic keys and keep copy grouped by domain namespace.
- Current admin namespaces include: `common`, `nav`, `auth`, `settings`, `dashboard`, `users`, `phishing`, `campaign`, `campaignDetail`, `analytics`, `training`, `onboarding`, `notFound`, `support`.
- Keep action/utility labels in `common` (for example `common.cancel`, `common.loading`) and page-specific copy in feature namespaces.
- User onboarding/import wizard copy in the users area should be nested under `users.create.*` and `users.import.*` for consistent grouping.

#### Adding or changing admin UI text

1. Add translation keys to `admin/client/src/i18n/locales/en.ts`, `admin/client/src/i18n/locales/de.ts`, `admin/client/src/i18n/locales/fr.ts`, and `admin/client/src/i18n/locales/it.ts`.
2. Replace hardcoded user-visible text in components/pages with `t("namespace.key")`.
3. Include accessibility and assistive text (`aria-label`, `sr-only`, placeholders, dialog copy) in i18n.
4. Reuse existing keys for shared semantics instead of adding near-duplicates.
5. Keep DB values as shortcodes (`en`, `de`, `fr`, `it`) and display full labels in UI.

---

## Super Admin Panel (`superadmin/`)

**Product:** DeeplySecure Super Admin Panel
**Target Users:** Platform operators managing the multi-tenant infrastructure

### Commands

```bash
cd superadmin
pnpm install          # Install dependencies (from repo root)
pnpm run dev          # Start full-stack dev server (port 5002)
pnpm run build        # Build for production
pnpm run start        # Run production server
pnpm run db:types     # Generate Supabase TypeScript types
pnpm run test         # Run tests
pnpm run check        # TypeScript type checking
pnpm run set-user-password -- --email "user@example.com" --password "StrongPassword123!"
```

### Demo Org Seeder (Local Fixtures)

Use the demo seeder script to (re)populate a dedicated demo organization with realistic analytics data:

```bash
cd superadmin
pnpm run seed:demo-org -- --admin-email "demo.owner@example.com" --reset true --seed 37 --users 50
```

Alternative selector:

```bash
pnpm run seed:demo-org -- --admin-id "<admin-user-uuid>" --reset true --seed 37 --users 50
```

Guardrails:
- The script is demo-only. If the selected admin belongs to a non-demo org (`organizations.is_demo = false`), the run aborts.
- `--reset` defaults to `true` and clears/reseeds org-scoped demo data to keep runs idempotent.
- Reset cleanup batches large `IN (...)` deletes to avoid PostgREST URL length limits (`URI too long`) on larger demo datasets.
- Managed user fixtures are distributed across current/previous/older weekly cohorts (including a small pending-invite slice) so Dashboard "This Week's Overview" metrics show realistic week-over-week movement.
- Demo phishing simulation/reporting data is stored in local fixture tables (`demo_phishing_*`).
- Template Library and Template Studio for demo orgs use the real Phisher tenant API; the seeder auto-provisions a Phisher tenant/connection for demo orgs if missing.
- Deleting the demo org removes demo fixtures via FK cascade (`ON DELETE CASCADE`).

### Pages & Routes

| Route | Page |
|-------|------|
| `/` | Dashboard — organization overview |
| `/modules` | Modules overview — search, filter, create, and inspect DB-backed learning modules |
| `/modules/:key/:version` | Module detail — edit metadata, guided blocks, translations, and video preview |
| `/organizations` | List, search, sort, paginate organizations |
| `/organizations/:id` | Organization detail — edit, view admins and managed users |
| `/super-admins` | Manage super admin users, send invites |
| `/settings` | User settings (password change + TOTP 2FA) |
| `/login` | Authentication |
| `/set-password` | Password setup for new super admin users |

### Backend API Routes

**Auth (`/api/auth`):**
- `GET /me` — Current super admin profile

**Organizations (`/api/organizations`):**
- `GET /` — List with pagination, search, sorting
- `GET /export` — Export the current organizations result set as CSV (respects search and sorting)
- `GET /:id` — Organization details with users
- `PUT /:id` — Update organization
- `DELETE /:id` — Delete organization
- `POST /invite-simple` — Email-only invite (includes `preferredLanguage` for Supabase template metadata)
- `POST /invite-demo` — Create a panel demo org (`organizations.is_demo = true`) with the default demo owner profile and send the setup invite
- `POST /invite-full` — Create organization + invite admin (includes `preferredLanguage` for email/template + initial admin profile)
- `POST /:id/convert-demo` — Activate a panel demo org by updating the owner name, company name, seat count, and clearing `is_demo`

**Modules (`/api/modules`):**
- `GET /` — List learning modules with pagination, search, filters, sorting, summary metrics, translation coverage, and assignment counts
- `POST /` — Create a new learning module catalog row
- `GET /:key/:version` — Module detail with blocks, translation rows, and assignment usage count
- `PUT /:key/:version` — Update base module metadata and block definitions
- `DELETE /:key/:version` — Delete an unused module (blocked when assignments exist)
- `GET /:key/:version/translations` — List locale overrides for a module version
- `PUT /:key/:version/translations/:locale` — Upsert localized metadata, block titles, and Vimeo overrides for any lowercase locale shortcode (for example `en`, `de`, `fr`, `es`)
- `DELETE /:key/:version/translations/:locale` — Remove a locale translation row

**Super Admins (`/api/super-admins`):**
- `GET /` — List all super admins
- `POST /invite` — Invite new super admin

**Sessions (`/api/sessions`):**
- `GET /` — Current user's active sessions
- `DELETE /:sessionId` — Revoke a session

**Webhooks (`/api/webhooks`):**
- `POST /framer-signup` — Framer form webhook: creates a trial organization + invites admin (no auth required, secured via HMAC-SHA256 signature)

### Framer Webhook Integration

The `/api/webhooks/framer-signup` endpoint allows a Framer website form to automatically create trial organizations and invite admin users. This remains separate from the superadmin panel's `Create Demo` action: panel demos are stored with `organizations.is_demo = true` and can later be activated into normal orgs from the org detail page, while the webhook keeps creating standard trial orgs (`is_demo = false`).

**How it works:**
1. A visitor fills out a signup form on the Framer website
2. Framer sends the form data as a JSON POST to the webhook URL
3. The endpoint verifies the Framer HMAC-SHA256 signature
4. A trial organization (10 seats) is created and the admin receives an invite email

**Framer form field mapping:**

| Framer Input Name | Required | Description |
|-------------------|----------|-------------|
| `email` | Yes | Admin email address |
| `organizationName` | Yes | Organization name |
| `firstName` | No | Admin first name (default: "Demo") |
| `lastName` | No | Admin last name (default: "User") |
| `language` | No | Preferred language `en`, `de`, `fr`, or `it` (default: `de`) |

**Setup:**
1. Add `FRAMER_WEBHOOK_SECRET` to the superadmin `.env` (minimum 32 characters)
2. In Framer, set the form webhook URL to `https://<superadmin-host>/api/webhooks/framer-signup`
3. In Framer, enable signature verification and paste the same secret

**Security:**
- Requests are authenticated via Framer's `Framer-Signature` header (HMAC-SHA256)
- Duplicate email submissions return 200 (to stop Framer's retry mechanism) with `{ duplicate: true }`
- All webhook-triggered creations are logged to `super_admin_audit_log` with action `webhook.framer_signup`

**Implementation files:** `server/routes/webhooks.ts`, `server/lib/framer-webhook.ts`, `server/lib/create-organization.ts`

### Key Features

- **Organization CRUD** — Create, read, update, delete organizations
- **Organizations CSV Export** — The `/organizations` list includes an export action that downloads all currently filtered/sorted organizations as CSV
- **Invite System** — Three panel flows: simple invite (existing org), panel demo create (`is_demo = true`), and full invite (create org + admin)
- **Language-Aware Invites** — Simple, panel demo, and full invite flows accept `preferredLanguage` (`en`/`de`/`fr`/`it`) and send `auth.admin.inviteUserByEmail(..., { data: { language } })` for Supabase template branching (`{{ .Data.language }}`). Demo and full invite flows set `organizations.default_language` + `admin_users.language` immediately; simple invite applies language during onboarding completion.
- **Demo Conversion** — Panel-created demo orgs show an `Activate Demo` action on the organization detail page. Conversion updates the owner name in both `admin_users` and the owner's shadow `managed_users` row, updates the company name and seat count, sets the org to `active`, and clears `is_demo` so it behaves like a normal organization afterward. The user portal's `ensure_admin_user_learning_access()` RPC also refreshes that shadow name on sign-in if it ever drifts. This does not change the separate scripted demo-data seeder flow.
- **Localized Auth Emails** — Supabase templates in `supabase/email-templates/` branch copy using Go templates (for example `{{ if eq .Data.language "de" }}...{{ else }}...{{ end }}`) with English fallback when `language` metadata is missing, and now send users to app-owned `/auth/confirm` pages via `{{ .RedirectTo }}` + `{{ .TokenHash }}`.
- **Framer Webhook** — Public endpoint for Framer forms to create trial organizations (HMAC-SHA256 secured)
- **Phisher Auto-Provisioning** — When an organization is created (via invite-full or Framer webhook), a Phisher tenant and API key are automatically provisioned and stored in `phisher_connections` for the admin app to use. After tenant creation, all system email templates and active global domains are automatically assigned to the new tenant (the Phisher API now requires explicit assignment for system resources to appear in a tenant's lists). Non-blocking: org creation succeeds even if Phisher is unavailable. Requires `PHISHER_URL`, `PHISHER_MASTER_APP_JWT_SECRET`, and `PHISHER_ENCRYPTION_KEY` env vars. Provisioning client also supports tenant admin deletion via `deleteTenantAdmin()`.
- **Phisher Resource Assignment** — Super admins can assign all global templates and domains to a specific org via `POST /api/organizations/:id/assign-phisher-resources` or to all tenants at once via `POST /api/organizations/assign-phisher-resources-all` (useful for backfilling after the Phisher API's assignment model change).
- **Modules Management** — Super admins can manage the DB-backed learning-module catalog from `/modules`, including create/edit flows, guided block editing, dynamic translation-language management via the detail UI, translation-specific Vimeo overrides, module summary metrics, and inline video preview. The module detail preview also includes a preview-only locale selector so operators can switch languages without leaving the current translation tab and immediately see whether the active Vimeo ID comes from a locale override or the base module. Delete is blocked whenever assignments already exist so historical learner data is not orphaned.
- **Super Admin Management** — Invite/manage platform operators
- **Audit Logging** — All actions logged to `super_admin_audit_log` table (supports system/webhook entries with nullable `super_admin_id`)
- **Session Management** — View and revoke active sessions
- **Auth Middleware** — Validates Bearer tokens + `is_super_admin` metadata in `server/middleware/super-auth.ts`

---

## Report API (`api/`)

**Product:** Shared public report API for Outlook and Gmail
**Target Users:** Outlook add-ins, Gmail add-ons, and public report-button clients
**Framework:** NestJS + Swagger/OpenAPI

### Commands

```bash
cd api
pnpm install
pnpm run dev
pnpm run test
pnpm run build
docker compose up -d
docker compose down
```

### API Docs

- Swagger UI: `/docs`
- OpenAPI JSON: `/docs-json`
- Health check: `/health`

### Public Contract

- `POST /v1/report/attempt`
  - Outlook and Gmail call this first with `channel`, `publicConfigId`, and `internetHeaders`
  - Response statuses include `simulation_recorded`, `simulation_already_recorded`, `requires_forward`, `manual_forward`, and `not_reportable`
- `POST /v1/report/forward`
  - Outlook-only fallback for suspicious emails that need SMTP forwarding
  - Requires full message metadata plus `publicConfigId`
- `POST /v1/report/telemetry`
  - Best-effort telemetry endpoint used by the Outlook taskpane

### Docker

Build context is the monorepo root (`api/Dockerfile` copies workspace manifests from the parent directory).

```bash
cd api
docker compose up -d
docker compose down
docker compose build
```

Or from the repo root:

```bash
docker build -f api/Dockerfile .
```

## Add-ons

### Outlook (`addons/outlook/`)

**Product:** Outlook Add-in for Phishing Email Reporting
**Target Users:** Employees using Outlook to report suspicious emails
**Deployment:** Docker container serving static add-in files

#### Overview

An Outlook add-in that adds a "Report Phishing" button to Outlook. All platforms (desktop, web, and mobile) show the same branded taskpane UI. When the user reports an email:

1. The manifest now embeds `apiBaseUrl` and `publicConfigId` instead of `reportEmail` and `trackingOrigin`.
2. The taskpane calls `POST /v1/report/attempt` on the shared `api/` service with the current message headers.
3. If the API returns `simulation_recorded` or `simulation_already_recorded`, the taskpane shows the phishing-simulation success state without reading the full body HTML.
4. If the API returns `requires_forward`, the taskpane reads the message metadata/body and calls `POST /v1/report/forward`.
5. If the API returns `manual_forward`, the taskpane shows manual-forward guidance and does not read the body.

Header-based simulation detection still depends on Outlook exposing `getAllInternetHeadersAsync()` in the current `MessageRead` client context. When that API is unavailable, the taskpane still calls the shared `attempt` endpoint with an empty header string and relies on the API response to decide the next step.

The add-in uses an add-in-only XML manifest (not the unified manifest) for compatibility with internal deployment via Microsoft 365 Admin Center and Outlook Mobile support.

#### Architecture

- **All platforms (Desktop/Web/Mobile):** Uses `MessageReadCommandSurface` (desktop) / `MobileMessageReadCommandSurface` (mobile) with `ShowTaskpane` action opening `taskpane.html`. Consistent branded UI everywhere.
- **Backend:** Static file host for the Outlook add-in assets plus `/health`. SMTP forwarding and telemetry now live behind the shared `api/` service instead of the Outlook host.

#### Commands

```bash
cd addons/outlook
docker compose up -d          # Start the add-in server
docker compose down           # Stop the server
docker compose build          # Rebuild after changes
```

#### Configuration

| Variable | Location | Description |
|----------|----------|-------------|
| `PORT` | `addons/outlook/.env` | Server port (default `3100`) |
| `PHISHING_REPORT_ADDIN_URL` | `admin/.env` | Public HTTPS URL of the add-in server (e.g., `https://report-addin.yourdomain.com`) |
| `PHISHING_REPORT_API_URL` | `admin/.env` | Public HTTPS URL of the shared report API (e.g. `https://api.deeply.cloud`) |

#### Admin Settings Integration

The admin dashboard has a **"Phishing Report Button"** tab under Settings > Organization:
- **Report email address** — the security team email used by the shared `api/` forwarding flow for non-simulation reports. If this is unset, Outlook users receive manual-forward guidance.
- **Download Manifest** — generates a per-org XML manifest with add-in URL, shared API URL, public config ID, and branding values embedded

When unsaved report-button changes exist, the manifest action switches to **"Save & Download Manifest"** and persists the report-button settings before generating the file.

#### Deployment Workflow

1. Set `PHISHING_REPORT_ADDIN_URL` and `PHISHING_REPORT_API_URL` in the admin app's environment
2. Deploy `api/` and `addons/outlook/` behind HTTPS
3. In Admin Settings > Phishing Report Button, configure the report email
4. Click "Download Manifest" to get the XML file
5. Upload the manifest to **Microsoft 365 Admin Center > Integrated Apps**
6. The add-in will appear in users' Outlook clients (Web, Windows, Mac, iOS, Android)

#### Add-in Branding

**Add-in branding:** Org logo and primary colour are baked into the manifest at download time. Logo and colour come from the Organisation branding settings. The ribbon button label is hardcoded as "Report Phishing" (EN) / "Phishing Melden" (DE). Re-download and re-deploy the manifest after any branding change. All clients (desktop, web, mobile) show the same branded taskpane popup.

#### Key Database Table

| Column | Table | Description |
|--------|-------|-------------|
| `phishing_report_email` | `organizations` | Per-org report mailbox used by the shared `api/` forwarding flow |
| `id` / `org_id` | `report_button_public_configs` | Public per-org identifier embedded in Outlook manifests and Gmail Script Properties |

#### Implementation Files

| Component | Files |
|-----------|-------|
| Static server | `addons/outlook/src/server.js` |
| Header parsing + validation | `addons/outlook/public/deeplysecure-headers.js` |
| Taskpane (all platforms) | `addons/outlook/public/taskpane.html`, `taskpane.js`, `taskpane.css` |
| Manifest generation API | `admin/server/routes/report-button.ts` |
| Shared report endpoints | `api/src/report/*` |
| Admin settings tab | `admin/client/src/pages/settings.tsx` (ReportButtonTab component) |
| Icon assets | `addons/outlook/public/assets/icon-{16,32,80}.png` |

### Google (`addons/google/`)

**Product:** Gmail add-on for DeeplySecure simulation reporting
**Target Users:** Employees using Gmail to report DeeplySecure-tagged simulation emails
**Deployment:** Google Apps Script project deployed as a Gmail add-on

#### Overview

The Google add-on now always shows a screenshot-inspired confirmation card for the opened Gmail message, with the Outlook shield icon, subject/name/email detail rows, and a bottom horizontal action row. When the user clicks `Report Email`, the add-on rereads `X-DeeplySecure-ID` and `X-DeeplySecure-Host` from the current message and posts to the hardcoded shared endpoint `https://api.deeply.cloud/v1/report/attempt`. `PUBLIC_CONFIG_ID` is still read from Apps Script Script Properties. `simulation_recorded` and `simulation_already_recorded` responses show the phishing-recognized success state and attempt to move the currently opened message to the bin; if Gmail rejects the trash step after a successful report, the add-on keeps the success state and tells the user to delete the email manually. Missing / invalid headers or `not_reportable` show the "not part of the campaign" / contact-IT guidance state. Gmail still shows its native action spinner while the request runs.

Unlike the Outlook add-in, the Google add-on still does **not** inspect the full email body, does **not** forward arbitrary suspicious emails through SMTP, and does **not** use the admin manifest download flow. It remains a manual Apps Script deployment stored in-repo.

#### Files

- `addons/google/Code.gs` — Gmail add-on logic and branded card builders
- `addons/google/appsscript.json` — Apps Script manifest, scopes, Gmail trigger configuration, and DeeplySecure branding metadata
- `addons/google/Code.test.js` — local helper tests
- `addons/google/README.md` — setup notes for Apps Script deployment

#### Deployment Workflow

1. Open `scripts.google.com` and create a new Apps Script project
2. Copy `addons/google/Code.gs` into the project
3. Replace the generated manifest with `addons/google/appsscript.json`
4. Set the `PUBLIC_CONFIG_ID` Script Property
5. If your reporting host differs from the repo default, update the hardcoded request URL in `addons/google/Code.gs` and the `urlFetchWhitelist` entry in `addons/google/appsscript.json`
6. Use the Gmail add-on test deployment flow in Apps Script

#### Local Verification

```bash
cd addons/google
node --test
```

#### Add-on Branding

**Add-on branding:** The Gmail add-on keeps the committed Apps Script manifest branding (`DeeplySecure Report Button`, hosted DeeplySecure logo URL, and `#133D3A` primary/secondary colors), while the in-card UI reuses the Outlook shield icon from `addons/outlook/public/assets/icon-80.png` as an embedded image for the screenshot-style layout. Runtime copy follows the Gmail locale (`en` / `de`). Google Workspace add-ons do not currently expose host-theme detection for `CardService`, so the Gmail add-on does not switch between separate light/dark icons. Because Gmail add-ons are `CardService`-based, this remains an Outlook-style approximation rather than a custom HTML/CSS taskpane clone.

---

## Port Allocation

| Service | Port |
|---------|------|
| Admin Dashboard | `3000` |
| User Portal | `4000` |
| Super Admin Panel | `5002` |
| Report API | `3200` |
| Outlook Add-in (Docker) | `3100` |
| Supabase API | `54321` |
| Supabase Database (PostgreSQL) | `54322` |
| Supabase Studio | `54323` |
| Supabase Inbucket (email testing) | `54324` |
| Supabase Analytics | `54327` |

---

## Favicon Assets

Favicon assets are app-local static files in each Vite client public directory:

- `admin/client/public/`
- `user/client/public/`
- `superadmin/client/public/`

Each app includes the same three favicon files:

- `favicon-light.png` (light mode)
- `favicon-dark.png` (dark mode)
- `favicon.png` (default fallback)

The root files `icon_light.png` and `icon_dark.png` are source assets used for copying into app public directories and can be removed after copying.

---

## Design System

The design system applies primarily to the Admin Dashboard but establishes patterns used across all apps.

### Core Principles

- **Minimalism:** Every element serves a purpose, no decorative UI
- **Glassy Morphism:** Frosted glass effect on cards with `backdrop-filter: blur`
- **Thin Typography:** Inter font with light weights (300-500)
- **Generous Whitespace:** 32px+ padding, breathing room between sections

### Color Palette

**Primary:**
| Token | Value | Usage |
|-------|-------|-------|
| Background | `#FAFAFA` | Page background (off-white) |
| Foreground | `#FFFFFF` | Cards/surfaces |
| Text Primary | `#1A1A1A` | Main text |
| Text Secondary | `#666666` | Supporting text |
| Text Tertiary | `#999999` | Muted text |

**Accent:**
| Token | Value | Usage |
|-------|-------|-------|
| Primary Accent | `#133D3A` | Dark teal — primary actions |
| Accent Hover | `#2F4F4A` | Hover states |
| Success | `#10B981` | Success states |
| Warning | `#F59E0B` | Warning states |
| Error | `#EF4444` | Error states |
| Info | `#3B82F6` | Informational |

**Glass Effects:**
| Token | Value |
|-------|-------|
| Glass Background | `rgba(255, 255, 255, 0.4)` |
| Glass Border | `rgba(0, 0, 0, 0.08)` |
| Divider | `rgba(0, 0, 0, 0.06)` |

### Status Color Tokens (Tailwind)

For results, timelines, and activity charts — premium, subtle look:

| Status | Background | Border | Text/Stroke |
|--------|-----------|--------|-------------|
| Sent/Opened | `gray-50` | `gray-200` | `gray-700` |
| Clicked | `amber-50` | `amber-200` | `amber-700` |
| Submitted | `red-50` | `red-200` | `red-700` |
| Reported | `emerald-50` | `emerald-200` | `emerald-700` |

### Chart Color Palette

All Recharts area charts, bar charts, and data visualisations must use this four-color palette (derived from the Campaign Activity chart). Apply colors by semantic role, not by fixed series position.

| Role | Hex | Tailwind Equivalent | Semantic Meaning |
|------|-----|---------------------|------------------|
| Neutral / baseline | `#374151` | `gray-700` | First step, opens, registrations |
| Engagement / action | `#B45309` | `amber-700` | Clicks, started actions |
| Danger / negative | `#B91C1C` | `red-700` | Credential submissions, failures |
| Positive / success | `#047857` | `emerald-700` | Reports, completions, correctness |

Usage across views:

| View | Series → Color |
|------|---------------|
| Campaign Activity | open → `#374151`, click → `#B45309`, credential_submit → `#B91C1C`, report → `#047857` |
| Awareness Score | clickRate → `#B45309`, reportRate → `#047857` |
| Activity Score | registrationRate → `#374151`, startedRate → `#B45309`, completionRate → `#047857` |
| Knowledge Score | firstTryCorrectAvg → `#047857` |
| Department Report (bars) | click_rate → `#B45309`, report_rate → `#047857` |

### Typography

**Font:** Inter (weights 300, 400, 500, 600 only — no bold 700+)

| Element | Size | Weight | Line Height |
|---------|------|--------|-------------|
| H1 | 32px | 400 | 1.8 |
| H2 | 24px | 400 | 1.7 |
| H3 | 18px | 500 | 1.7 |
| H4 | 16px | 500 | 1.6 |
| Body | 14px | 400 | 1.6 |
| Caption | 12px | 400 | 1.5 |
| Label | 12px | 500 | 1.5 |

Letter spacing: `-0.01em` default, `0.05em` for all-caps

### Component Specifications

**Cards:** `border-radius: 8px`, `border: 1px solid rgba(0,0,0,0.08)`, padding `24px`/`32px`, optional `backdrop-filter: blur(10px)`

**Buttons:** height `36px`, `border-radius: 6px`, primary = solid teal bg + white text, secondary = transparent + teal border, `transition: all 200ms ease`, hover = opacity +10% + scale 1.02

**Inputs:** height `36px`, `border: 1px solid rgba(0,0,0,0.08)`, `border-radius: 6px`, focus = accent border color, no outline

**Badges:** height `24px`, padding `4px 10px`, `border-radius: 4px`, font 12px weight 400

### Spacing Scale

Base unit: `4px`. Scale: 4, 8, 12, 16, 20, 24, 28, 32, 40, 48, 56, 64px

- Padding inside cards: 16-32px
- Gap between elements: 8-24px
- Margin around sections: 24-48px

### Animations

| Type | Specification |
|------|--------------|
| Default transition | `200ms ease-out` |
| Modal entrance | Scale `0.95 → 1.0`, fade-in `200ms` |
| Dropdown open | Scale `0.98 → 1.0`, fade-in `200ms` |
| Loading states | Skeleton screens (not spinners), shimmer left-to-right 2s loop |
| Progress bars | 3px height, accent color |
| Line charts | Draw animation `600ms ease-out` |
| Bar charts | Grow from bottom `400ms ease-out`, staggered 50ms |
| Number counters | Animate 0 → value `600ms ease-out` |

### Layout

- **Sidebar:** 280px width, collapses to icons at 1024px
- **Grid:** 12-column, max-width 1440px
- **Breakpoints:** 1440px, 1024px, 768px, 375px

### Accessibility

- Color contrast: 4.5:1 for normal text
- Focus indicators: 2px accent border
- Keyboard navigation for all interactive elements
- ARIA labels on icons and buttons

---

## Support Content Authoring (Admin)

The admin `/support` page renders FAQ and help articles from markdown files at build time. No backend or CMS is required — content updates are a code change.

### Folder Structure

```
admin/client/src/content/support/
├── types.ts                  # SupportArticleMeta / SupportArticle types
├── en/
│   ├── manifest.ts           # Metadata array (title, slug, category, order, tags, lastUpdated)
│   ├── getting-started.md
│   ├── whitelisting-overview.md
│   └── ...
└── de/
    ├── manifest.ts           # German metadata (same slugs, translated titles/summaries)
    ├── getting-started.md
    ├── whitelisting-overview.md
    └── ...
```

### Adding a New Article

1. Create a markdown file in `en/<slug>.md` and `de/<slug>.md`
2. Add a metadata entry to both `en/manifest.ts` and `de/manifest.ts` with matching `slug`
3. Set `order` within the article's `category` to control sort position
4. Set `lastUpdated` to the current date (`YYYY-MM-DD`)
5. The article will automatically appear on the `/support` page — no route changes needed

### Manifest Fields

| Field | Type | Description |
|-------|------|-------------|
| `slug` | `string` | URL-safe identifier, must match the `.md` filename |
| `title` | `string` | Localized article title |
| `summary` | `string` | Short description shown in the article list |
| `category` | `string` | Grouping label (e.g. `"Whitelisting"`, `"Getting Started"`) |
| `order` | `number` | Sort position within the category |
| `tags` | `string[]` | Search keywords |
| `lastUpdated` | `string` | ISO date string (`YYYY-MM-DD`) |
| `sourceRef` | `string?` | Internal note for content traceability (not displayed) |

### Locale Fallback

If a markdown file exists in one locale but not the other, the system falls back to the available locale. This allows incremental translation — publish in one language first, translate later.

### Markdown Features

Articles support standard markdown plus GitHub Flavored Markdown (tables, task lists, strikethrough) via `remark-gfm`. Raw HTML is stripped (`skipHtml: true`) for security. External links automatically open in a new tab.

### Content Guidelines for Support Articles

- Write original content adapted for DeeplySecure's product context
- Do **not** copy external help documentation verbatim
- Use `sourceRef` in the manifest to note inspiration sources for traceability
- Keep articles focused on a single topic; split long guides into multiple articles
- Use checklists (`- [ ]`) for verification steps

---

## Content Guidelines

### Tone

- Professional but approachable
- Clear and concise, avoid jargon
- Action-oriented, use active voice
- Positive framing

### Terminology

Use consistently across all apps:
- **"Campaign"** not "Test"
- **"Phishing Simulation"** not "Phishing Test"
- **"Learning Path"** not "Course"
- **"User"** not "Employee" (technical contexts)
- **"Risk Score"** not "Risk Rating"

### Microcopy Patterns

- Success: `"Campaign sent! View results"`
- Loading: `"Generating insights..."`
- Empty state: `"No campaigns yet. Create your first to get started."`
- Error: `"Something went wrong. Try again or contact support."`

---

## Testing

The three application projects use Vitest via their package `test` scripts (`pnpm run test`). The Google and Outlook add-on helper harnesses use `node --test`.

### Auth Session Invalidation

- `admin/` and `user/` now revalidate cached browser sessions with `supabase.auth.getUser()` during auth bootstrap before trusting `supabase.auth.getSession()`.
- If the backing Supabase Auth user has been deleted (for example after organization deletion), the client signs out locally on reload instead of falling through into onboarding or a stale authenticated state.
- Protected client-side API helpers also sign out on `401` responses as a secondary safeguard.
- Superadmin organization deletion now explicitly removes `auth.sessions` rows for collected org auth users before calling `auth.admin.deleteUser(...)`; access JWTs can still remain valid until expiry, so the client-side revalidation remains the primary UX safeguard.

### Running Tests

```bash
# Whole repo quick check
pnpm test
node --test

# Per project
cd admin && pnpm run test
cd user && pnpm run test
cd superadmin && pnpm run test
cd api && pnpm run test
cd addons/google && node --test
cd addons/outlook && node --test

# Watch mode (per project)
pnpm run test:watch
```

### Test Structure

Tests live in each project's `test/` directory:

**Admin tests:**
- `test/unit/` — Unit tests (no Supabase required): CSV parsing, Excel parsing, seat count validation, domain validation, password change
- `test/managed-users/` — Integration tests: CRUD, import, domain restrictions
- `test/settings/` — Integration tests: password change flow

**Superadmin tests:**
- `test/auth/` — Auth flow tests
- `test/unit/` — Unit tests

**User tests:**
- `test/` — Tests for user portal functionality

### Test Conventions

```typescript
import { test, expect } from "vitest";

test("description", () => {
  expect(1).toBe(1);
});
```

### Continuous Integration

GitHub Actions workflow [`.github/workflows/ci.yml`](.github/workflows/ci.yml) runs on every pull request and on pushes to `main` and version tags (`v*`).

**Pull requests:** run the full test suite, start a local Supabase stack for integration tests, and build all Docker images (no registry push).

**`main` and `v*` tags:** same checks, then push images to GitHub Container Registry (GHCR).

| Service | Image |
|---------|-------|
| Admin | `ghcr.io/<owner>/deeplysecure-admin` |
| User | `ghcr.io/<owner>/deeplysecure-user` |
| Super Admin | `ghcr.io/<owner>/deeplysecure-superadmin` |
| Report API | `ghcr.io/<owner>/deeplysecure-api` |
| Outlook add-in | `ghcr.io/<owner>/deeplysecure-outlook` |

Replace `<owner>` with the GitHub org or user that owns the repository. Tags include the commit SHA; `latest` is published from `main`; semver tags are published from `v*` git tags.

**Repository setting:** under **Settings → Actions → General → Workflow permissions**, enable **Read and write permissions** so `GITHUB_TOKEN` can push packages to GHCR.

**Local parity (tests):**

```bash
supabase start
# Map CLI names to app env vars (same as CI; local .env files usually already use SUPABASE_*).
eval "$(supabase status -o env \
  --override-name api.url=SUPABASE_URL \
  --override-name auth.anon_key=SUPABASE_ANON_KEY \
  --override-name auth.service_role_key=SUPABASE_SERVICE_ROLE_KEY)"
export VITE_SUPABASE_URL="$SUPABASE_URL"
export VITE_SUPABASE_ANON_KEY="$SUPABASE_ANON_KEY"
pnpm install --frozen-lockfile
pnpm --filter deeplysecure-admin test
pnpm --filter deeplysecure-user test
pnpm --filter deeplysecure-superadmin test
pnpm --filter deeplysecure-report-api test
cd addons/outlook && npm ci && npm test
```

**Local parity (Docker builds from repo root):**

```bash
docker build -f admin/Dockerfile .
docker build -f user/Dockerfile .
docker build -f superadmin/Dockerfile .
docker build -f api/Dockerfile .
docker build -f addons/outlook/Dockerfile addons/outlook
```

Example pull after a `main` build:

```bash
docker pull ghcr.io/<owner>/deeplysecure-admin:latest
```

Container images expect runtime configuration (Supabase URL/keys and other secrets) via environment variables at deploy time; they are not baked into the image.

**Docker Compose (production host):** see [`compose/README.md`](compose/README.md) for pulling GHCR images, per-service `.env` files, and bringing up all five services on ports 3000, 4000, 5002, 3200, and 3100.

---

## AI Development Notes

This section provides context for AI coding assistants (Claude Code, Cursor, etc.).

### General Rules

1. **Use pnpm at repo root** — install once with `pnpm install`; run scripts with `pnpm run` or `pnpm -r`
2. **Supabase is the database** — no separate ORM; use Supabase client + RLS
3. **Shared database** — all three apps connect to the same Supabase project
4. **Type generation** — after schema changes, run `pnpm run db:types` in all three apps
5. **RLS is critical** — all data access must respect row-level security policies
6. **Password validation** — shared via `shared/password-validation.ts` in each project

### Node.js Conventions

| Use | Instead of |
|-----|-----------|
| `express` | custom HTTP servers for full-stack apps |
| `pg` | direct SQL without a client library |
| `WebSocket` via `ws` | ad-hoc socket implementations |
| `node:fs/promises` | sync file I/O in server code |
| `tsx` | `ts-node` for dev scripts |

> **Note:** The admin and superadmin apps use Express.js. This is intentional for their existing codebase.

### Architecture Patterns

- **Frontend state:** TanStack React Query for server state, React context for auth
- **Routing:** Wouter (lightweight, hooks-based)
- **Forms:** React Hook Form + Zod validation
- **Components:** Radix UI primitives wrapped in shadcn/ui components
- **API calls:** Fetch-based, going through Express proxy to Supabase
- **Auth flow:** Supabase Auth (passwords + app-owned email-link confirmation pages, with legacy link compatibility during rollout), verified via middleware
- **Multi-tenancy:** RLS policies on every table, organization-scoped data

### File Naming Conventions

- React components: `PascalCase.tsx` (e.g., `DashboardLayout.tsx`)
- Hooks: `use-kebab-case.ts` (e.g., `use-managed-users.ts`)
- Utilities: `kebab-case.ts` (e.g., `password-validation.ts`)
- Pages: `kebab-case.tsx` (e.g., `user-import.tsx`)
- Test files: `kebab-case.test.ts`

### Quick Start for Development

```bash
# 1. Start Supabase local stack
supabase start --workdir supabase

# 2. Install dependencies at repo root
pnpm install

# 3. Start the dev server for the app you're working on
cd admin && pnpm run dev   # or user/ or superadmin/

# 4. Access the apps
# Admin:      http://localhost:3000
# User:       http://localhost:4000
# Superadmin: http://localhost:5002
# Supabase Studio: http://localhost:54323
# Inbucket (test emails): http://localhost:54324
```

### Working with Multiple Apps Simultaneously

Since all three apps share one database, you can run them all at once:

```bash
# Terminal 1: Supabase
supabase start --workdir supabase

# Terminal 2: Admin
cd admin && pnpm run dev

# Terminal 3: User
cd user && pnpm run dev

# Terminal 4: Superadmin
cd superadmin && pnpm run dev
```

### Root Utility Script

The repo includes a helper script at `run_dev.sh` for common multi-app workflows:

```bash
# Start all app dev servers (admin, user, superadmin)
./run_dev.sh
# or explicitly:
./run_dev.sh dev

# Install dependencies in all app directories
./run_dev.sh install

# Generate Supabase DB types in all app directories
./run_dev.sh gen-types

# Apply pending local Supabase migrations
./run_dev.sh migration
```

`gen-types` runs `pnpm run db:types` in `admin/`, `user/`, and `superadmin/`, and expects a local `supabase` CLI available via PATH (Homebrew installs in `/opt/homebrew/bin` or `/usr/local/bin` are supported).
`migration` runs `supabase migration up` from the `supabase/` directory (matching the standard local workflow).
