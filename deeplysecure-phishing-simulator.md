# Phisher

Multi-tenant phishing simulation platform for security awareness training.
Organizations can create realistic phishing campaigns, track employee
interactions, and generate reports -- all isolated per tenant via PostgreSQL
Row-Level Security.

## Architecture

```
                     +------------+
                     |  Next.js   |  (ui/)
                     |  Frontend  |
                     +-----+------+
                           | /api/* proxy
                           v
+----------+     +---------+--------+     +----------+
|          |     |   FastAPI API    |     |          |
| Postgres +<----+  (src/phisher/)  +---->+  Redis   |
|  (RLS)   |     +--------+--------+      +----+-----+
+----------+              |                    |
                          v                    v
                   +------+------+      +------+------+
                   |  Tracking   |      | ARQ Worker  |
                   | /t/ public  |      | (cron+jobs) |
                   +-------------+      +-------------+
```

**Backend** -- Python 3.12+, FastAPI, async SQLAlchemy 2, asyncpg, ARQ (Redis
job queue), aiosmtplib.

**Frontend** -- Next.js 16, React 19, TypeScript, Tailwind CSS v4, shadcn/ui,
TanStack Query + Table, Zustand.

**Infrastructure** -- PostgreSQL 16 (with RLS), Redis 7, Mailpit (dev SMTP
catcher).

## Key Features

- **Multi-tenancy** -- full data isolation through PostgreSQL Row-Level Security
  policies; every table is scoped to a tenant via `set_config()`.
- **Admin tenant impersonation** -- platform admins can issue short-lived
  tenant impersonation tokens from provisioning endpoints to "log in as tenant"
  in the UI. Impersonated requests still run through app-role DB sessions with
  `set_config('app.current_tenant_id', ..., true)`, so PostgreSQL RLS remains
  the enforcement layer.
- **Campaign lifecycle** -- state machine (draft -> generating -> review ->
  scheduled -> running -> paused/completed/cancelled) with optimistic locking.
- **LLM email generation** -- OpenRouter integration generates realistic
  phishing emails from scenario templates; batch processing with retry and
  concurrency control.
- **Tracking** -- open pixel, link click (302 redirect), landing page view,
  form submission capture, and phishing report (via Outlook add-in) at
  unauthenticated `/t/` endpoints.
- **Reporting** -- per-campaign results with aggregation, timeline, CSV export;
  cross-campaign overview and per-department breakdown.
- **Data retention** -- configurable per-tenant IP anonymization and event
  cleanup via cron jobs.
- **Rate limiting** -- sliding-window rate limiter, separate thresholds for API
  and tracking endpoints.
- **Encrypted credentials** -- SMTP passwords stored with Fernet symmetric
  encryption.
- **Global domains & SMTP** -- platform-wide domain registry with per-tenant
  assignment (auto-assign for new tenants); global SMTP servers managed by
  admins. Sender identity (from address/name) is configured on email
  templates, not on SMTP servers. The from-address domain and tracking link
  domain are selected from global domains (not freetext), ensuring only
  registered domains are used.
- **Template assignments** -- system email templates can be assigned to
  specific tenants (similar to domain assignments), controlling which
  templates each tenant can use for campaigns.
- **Template variables** -- email templates support `{first_name}`,
  `{last_name}`, `{email}`, `{department}`, `{position}` for target
  personalization, and `{link}` which resolves to the tracking URL at send
  time (landing page or click-redirect depending on campaign configuration).
  The `{link}` domain is determined by the template's link domain setting.
- **Message correlation headers** -- every outbound phishing email includes
  `X-DeeplySecure-ID` with the message's tracking UUID,
  `X-DeeplySecure-Host` with the configured app domain (`APP_DOMAIN` when set,
  otherwise the tracking-link domain), and `X-DeeplySecure-Tenant` with the
  tenant slug, allowing inbox-level correlation with tracking events.
- **Default phishing redirect** -- emails without a landing page redirect
  targets to a configurable awareness page after click tracking.

## Project Layout

```
src/phisher/
  api/v1/        # FastAPI routers (provisioning, campaigns, reports, ...)
  api/tracking.py# public tracking endpoints (/t/)
  auth/          # API key + master JWT authentication
  db/            # async session factories, RLS context helper
  models/        # SQLAlchemy models (all prefixed phisher_)
  schemas/       # Pydantic request/response models
  services/      # business logic (LLM, SMTP, encryption, retention, ...)
  middleware/     # rate limiter, error handler, access logger
  tasks/         # ARQ worker, campaign sender, retention cron jobs
  config.py      # Pydantic Settings (PHISHER_ env prefix)
  main.py        # create_app() factory

ui/              # Next.js frontend
alembic/         # database migrations (001-015)
scripts/         # seed_data.py, gen_jwt.py
tests/           # 489 tests (API, auth, DB, services, middleware, RLS isolation)
docker/          # init-db.sql (test database creation)
```

## Prerequisites

- Docker and Docker Compose v2
- Python 3.12+ and [uv](https://docs.astral.sh/uv/) (for local dev without
  Docker)
- Node.js 20+ and npm (for the frontend)

## Quick Start (Development)

### 1. Configure environment

```sh
cp .env.example .env
```

Edit `.env` and set the two required secrets:

```sh
# Any random string:
PHISHER_MASTER_APP_JWT_SECRET=some-secret-string

# Generate a valid Fernet key:
python -c "from cryptography.fernet import Fernet; print(Fernet.generate_key().decode())"
# Paste the output as:
PHISHER_CREDENTIAL_ENCRYPTION_KEY=<generated-key>
```

Optionally set `PHISHER_OPENROUTER_API_KEY` for LLM-powered scenario
generation.

### 2. Start all services

```sh
docker compose up --build -d
```

This brings up PostgreSQL, Redis, Mailpit, runs Alembic migrations, then starts
the API (with hot-reload) and ARQ worker.

| Service   | URL                           | Purpose                       |
|-----------|-------------------------------|-------------------------------|
| API       | http://localhost:8822          | FastAPI backend               |
| Mailpit   | http://localhost:8026          | SMTP web UI (captured emails) |
| Postgres  | localhost:5435                | database (user: phisher)      |
| Redis     | 127.0.0.1:6381               | job queue                     |

The API mounts source directories as read-only volumes, so code changes
trigger uvicorn's auto-reload without rebuilding the image.

### 3. Seed demo data (optional)

```sh
docker compose exec api python /app/scripts/seed_data.py
```

Creates a demo tenant ("Acme Corporation") with admins, users, SMTP config,
targets, landing pages, and sample campaigns.

### 4. Generate an admin JWT

From inside the container (where `.env` is loaded):

```sh
docker compose exec api python /app/scripts/gen_jwt.py
```

Or locally (requires the package installed):

```sh
python scripts/gen_jwt.py --sub admin@platform.com --hours 24
```

Use the printed token as `Authorization: Bearer <token>` for provisioning
endpoints.

For admin "login as tenant" flows, call:

```http
POST /api/v1/provisioning/tenants/{tenant_id}/impersonate
Authorization: Bearer <master-jwt>
Content-Type: application/json

{"expires_in_seconds": 900}
```

The response contains a short-lived bearer token (60-3600 seconds) that can be
used on tenant endpoints only. It is intentionally rejected by provisioning
admin endpoints.

### 5. Start the frontend

```sh
cd ui
npm install
npm run dev
```

The frontend dev server runs at http://localhost:3001 and proxies `/api/*`
requests to the backend at `localhost:8822`.

### 6. Explore the API

- OpenAPI docs: http://localhost:8822/docs
- Health check: http://localhost:8822/health

## Production Deployment

The Compose file includes a `prod` profile that runs Traefik (reverse proxy
with automatic Let's Encrypt TLS), the FastAPI backend, Next.js frontend, and
ARQ worker -- all behind two configurable domains.

### 1. Prepare environment

Create a `.env` with production values:

```sh
# Required secrets
PHISHER_MASTER_APP_JWT_SECRET=<strong-random-secret>
PHISHER_CREDENTIAL_ENCRYPTION_KEY=<fernet-key>
PHISHER_DEBUG=false

# Domains (used by Traefik for routing and TLS certificates)
APP_DOMAIN=campaign-02.deeply.cloud
TRACKING_DOMAIN=noreply-mail.online
# Optional: comma-separated allow-list for cross-origin requests to /t/*
# Default: https://report.deeply.cloud
# TRACKING_CORS_ORIGINS=https://report.deeply.cloud
ACME_EMAIL=admin@example.com
TRAEFIK_BASIC_AUTH_USERS=admin:$apr1$H6uskkkW$IgXLP6ewTrSuBkTrqE8wj/

# Optional: bind API to a specific IP (default 0.0.0.0)
# PHISHER_API_HOST=127.0.0.1
```

Generate `TRAEFIK_BASIC_AUTH_USERS` with `htpasswd` (Apache utilities):

```sh
htpasswd -nb admin 'change-this-password'
```

Use the full `user:hash` output as the `.env` value. In production profile,
Traefik applies this basic auth middleware to private routes on `APP_DOMAIN`
(frontend UI, `/docs`, `/openapi.json`). `/api/*` remains protected by
application auth (API keys / bearer JWT), and `/t/*` plus `/health` remain
publicly reachable.

`PHISHER_TRACKING_BASE_URL` is automatically derived from `TRACKING_DOMAIN` in
the Compose file (`https://${TRACKING_DOMAIN}`), so you don't need to set it
separately.

### 2. Build and start

```sh
docker compose --profile prod up --build -d
```

This starts:

| Service | Description |
|---|---|
| `traefik` | Reverse proxy (ports 80/443), automatic Let's Encrypt |
| `api-prod` | FastAPI backend (2 uvicorn workers), host port 8000 |
| `frontend-prod` | Next.js standalone server |
| `worker-prod` | ARQ background worker |
| `postgres` | Database |
| `redis` | Job queue |

To run only production services (skip dev):

```sh
docker compose --profile prod up --build -d postgres redis migrate traefik api-prod frontend-prod worker-prod
```

### Routing

Traefik routes requests based on domain and path:

| Domain | Path | Target |
|---|---|---|
| `APP_DOMAIN` | `/api/*` | api-prod (app auth: API key / JWT) |
| `APP_DOMAIN` | `/docs`, `/openapi.json` | api-prod (basic auth) |
| `APP_DOMAIN` | `/t/*`, `/health` | api-prod (public) |
| `APP_DOMAIN` | everything else | frontend-prod (basic auth) |
| `TRACKING_DOMAIN` | `/t/*`, `/health` | api-prod |
| `TRACKING_DOMAIN` | everything else | blocked (no route) |

All HTTP traffic is automatically redirected to HTTPS. TLS certificates are
obtained and renewed via Let's Encrypt HTTP challenge.

### Production considerations

- **Same secrets as dev when sharing a database**: If prod uses the same
  PostgreSQL data as dev, `PHISHER_MASTER_APP_JWT_SECRET` and
  `PHISHER_CREDENTIAL_ENCRYPTION_KEY` must be identical. SMTP passwords and
  other encrypted data were created with the encryption key; a different key
  causes "Unsupported state or unable to authenticate data" (decryption
  failure). Avoid trailing newlines or spaces in these values in `.env` (the
  app strips whitespace from these two settings).
- Make sure DNS for both `APP_DOMAIN` and `TRACKING_DOMAIN` points to the
  server before starting -- Traefik needs this for the ACME HTTP challenge.
- Scale `worker-prod` instances for higher email throughput (each runs
  independent ARQ cron cycles).
- The PostgreSQL container in docker-compose uses local volumes. For production,
  use a managed database or configure proper backup/replication.
- Review rate limit settings (`PHISHER_RATE_LIMIT_API_RPM`,
  `PHISHER_RATE_LIMIT_TRACKING_RPM`) for your expected load.
- Traefik stores certificates in the `traefik_certs` Docker volume. Back this
  up or use a DNS challenge provider if you need wildcard certificates.
- **Frontend API proxy**: The Next.js rewrite destination is baked at build
  time via the `NEXT_PUBLIC_API_URL` build arg. docker-compose sets this to
  `http://api-prod:8000` for the prod profile. In normal operation Traefik
  routes `/api/*` directly to `api-prod` (priority 100), so the rewrite is
  only a fallback.

## Local Development (without Docker)

For faster iteration on the Python backend:

```sh
# Install with dev dependencies
uv pip install -e ".[dev]"

# Run migrations (adjust URL to your local Postgres)
PHISHER_DATABASE_ADMIN_URL="postgresql+asyncpg://phisher:phisher@127.0.0.1:5432/phisher" \
  python -m alembic upgrade head

# Start the API
uvicorn phisher.main:create_app --factory --reload --port 8000

# Start the worker (separate terminal)
arq phisher.tasks.worker.WorkerSettings
```

Requires PostgreSQL and Redis running locally (or via `docker compose up
postgres redis`).

## Running Tests

Tests use a dedicated `phisher_test` database, created by
`docker/init-db.sql` during first Postgres startup.

```sh
# With infrastructure running (postgres + redis)
pytest

# With coverage
pytest --cov=phisher --cov-report=term-missing

# Specific test module
pytest tests/test_api/test_campaigns.py -v
```

The suite auto-truncates all tenant tables between tests for isolation.

## Database Roles

| Role            | Purpose                                   | RLS      |
|-----------------|-------------------------------------------|----------|
| `phisher`       | Schema owner, runs migrations             | bypasses |
| `phisher_admin` | Background tasks, tracking event recorder | bypasses |
| `phisher_app`   | Application queries (API requests)        | enforced |

## Environment Variables

Application variables are prefixed with `PHISHER_`. Most infrastructure
variables (`TRACKING_DOMAIN`, `TRACKING_CORS_ORIGINS`, `ACME_EMAIL`) are used
by docker-compose only when running the `prod` profile. `APP_DOMAIN` is also
read by the application to populate the outbound `X-DeeplySecure-Host` header
when present; if it is unset, that header falls back to the tracking-link
domain.

| Variable                              | Required | Default                          | Description                              |
|---------------------------------------|----------|----------------------------------|------------------------------------------|
| `PHISHER_MASTER_APP_JWT_SECRET`       | yes      | --                               | HS256 secret for provisioning JWTs       |
| `PHISHER_CREDENTIAL_ENCRYPTION_KEY`   | yes      | --                               | Fernet key for SMTP password encryption  |
| `PHISHER_OPENROUTER_API_KEY`          | no       | --                               | OpenRouter key for LLM generation        |
| `PHISHER_DATABASE_URL`                | no       | `...phisher_app@localhost/phisher`| asyncpg connection string (app role)     |
| `PHISHER_DATABASE_ADMIN_URL`          | no       | `...phisher_admin@localhost/phisher`| asyncpg connection string (admin role) |
| `PHISHER_REDIS_URL`                   | no       | `redis://127.0.0.1:6379/0`      | Redis connection                         |
| `PHISHER_DEBUG`                       | no       | `false`                          | Enable debug mode                        |
| `PHISHER_TRACKING_BASE_URL`           | no       | `http://localhost:8000`          | Public base URL for tracking links       |
| `PHISHER_DEFAULT_PHISHING_REDIRECT_URL`   | no       | `https://learn.deeplysecure.com/phishing-landing` | Redirect URL for emails without a landing page |
| `PHISHER_SENDING_DEFAULT_RATE_PER_MINUTE` | no  | `60`                             | Default email send rate                  |
| `PHISHER_RATE_LIMIT_API_RPM`          | no       | `300`                            | API rate limit (requests/minute)         |
| `PHISHER_RATE_LIMIT_TRACKING_RPM`     | no       | `1000`                           | Tracking rate limit (requests/minute)    |
| `PHISHER_API_HOST`                    | no       | `0.0.0.0`                        | IP to bind prod API (prod profile)       |
| `APP_DOMAIN`                          | no       | --                               | Main app domain (prod profile). Traefik falls back to `campaign-02.deeply.cloud`; the app uses it for outbound `X-DeeplySecure-Host` when set |
| `TRACKING_DOMAIN`                     | no       | `noreply-mail.online`            | Tracking domain (prod profile)           |
| `TRACKING_CORS_ORIGINS`               | no       | `https://report.deeply.cloud`    | Comma-separated CORS allow-list for tracking routes (`/t/*`) |
| `ACME_EMAIL`                          | no       | `admin@example.com`              | Let's Encrypt registration email         |
| `TRAEFIK_BASIC_AUTH_USERS`            | yes (prod) | --                             | Traefik basic auth `user:hash` list for frontend + docs on `APP_DOMAIN` |

## License

Proprietary. All rights reserved.
