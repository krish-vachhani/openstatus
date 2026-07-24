# Docker Setup Guide

Complete guide for running OpenStatus with Docker

## Quick Start

```bash
# 1. Copy environment file
cp .env.docker.example .env.docker

# 2. Configure required variables (see Configuration section)
vim .env.docker

# 3. Build and start services (migrations will run automatically)
export DOCKER_BUILDKIT=1
docker compose up -d

# 4. Check service health
docker compose ps

# 5. (Optional) Seed database with test data
docker compose --profile seed run --rm seed

# 6. (Optional) Deploy Tinybird local - requires tb CLI
cd packages/tinybird
tb --local deploy

# 7. Access the application
open http://localhost:3002  # Dashboard
open http://localhost:3003  # Status Page Theme Explorer
# Note: Status pages are accessed via subdomain/slug (e.g., http://localhost:3003/status)
```

## Cleanup

```bash
# Remove stopped containers
docker compose down

# Remove volumes
docker compose down -v

# Clean build cache
docker builder prune
```

## Services

| Service | Port | Purpose |
|---------|------|---------|
| workflows | 3000 | Background jobs |
| server | 3001 | API backend (tRPC) |
| dashboard | 3002 | Admin interface |
| status-page | 3003 | Public status pages |
| private-location | 8081 | Monitoring agent |
| checker | 8082 | HTTP/TCP/DNS probe (on-demand tests + check runs) |
| libsql | 8080 | Database (HTTP) |
| libsql | 5001 | Database (gRPC) |
| tinybird-local | 7181 | Analytics (opt-in, `--profile analytics`, ~6GB RAM) |


## Architecture

```
┌─────────────┐     ┌─────────────┐
│  Dashboard  │────▶│   Server    │
│  (Next.js)  │     │   (Bun)     │
└─────────────┘     └─────────────┘
      │                    │
      ▼                    ▼
┌─────────────┐     ┌─────────────┐
│ Status Page │     │  Workflows  │
│  (Next.js)  │     │   (Bun)     │
└─────────────┘     └─────────────┘
      │                    │
      └────────┬───────────┘
               ▼
        ┌─────────────┐
        │   LibSQL    │
        │  (Database) │
        └─────────────┘
```

## Database Setup

### Automatic Migrations

Migrations run **automatically** when you start the stack with `docker compose up -d`,
via the one-shot `migrate` service. It runs the Drizzle migrator once (after `libsql`
is healthy, before the app services start) and exits; the app services gate on it with
`depends_on: { migrate: { condition: service_completed_successfully } }`.

> **Why a separate service?** The `workflows` **runtime** image is distroless — it
> contains only the compiled binary and `curl`, with no shell, no Deno, and no source
> tree. It therefore cannot migrate itself. The `migrate` service reuses the workflows
> Dockerfile's `build` stage, which *does* include Deno, the source, and the drizzle
> SQL files. Do **not** try `docker compose exec workflows …` — there is no shell there.

**Verifying migrations:**
```bash
# Check the migrate service logs
docker compose logs migrate

# Should show:
# openstatus-migrate  | Running migrations
# openstatus-migrate  | Migrated successfully
```

**Manual / re-run migration:**

```bash
# Re-run the one-shot migrate service (idempotent — applied migrations are skipped)
docker compose up migrate

# Or run the migrator ad-hoc from the build stage
docker compose run --rm --entrypoint sh migrate \
  -c "cd /app/packages/db && deno run -A src/migrate.mts"
```

### Seeding Test Data (Optional)

**Note:** Migrations run automatically, but seeding does **not**. You must manually seed the database if you want test data.

After migrations complete, seed the database with sample data using the
profile-gated `seed` service (it waits for `migrate` to finish, then seeds):

```bash
docker compose --profile seed run --rm seed
```

This creates:
- 3 workspaces (`love-openstatus`, `test2`, `test3`)
- 5 sample monitors and 1 status page with slug `status`
- Test user account: `ping@openstatus.dev`
- Sample incidents, status reports, and maintenance windows

**Verifying seeded data:**
```bash
# Check table counts via libsql HTTP API
curl -s http://localhost:8080/ -H "Content-Type: application/json" \
  -d '{"statements":["SELECT COUNT(*) FROM page"]}' | jq -r '.[0].results.rows[0][0]'

# Should output: 1
```

**Accessing Seeded Data:**

After seeding, you can access the test data:

**Dashboard:**
1. Navigate to http://localhost:3002/login
2. Use magic link authentication with email: `ping@openstatus.dev`
3. Check your console/logs for the magic link (with `SELF_HOST=true` in `.env.docker`)
4. After logging in, you'll see the `love-openstatus` workspace with all seeded monitors and status page

**Status Page:**
- The seeded status page has slug `status`
- Access it via subdomain routing: http://status.localhost:3003
- Or view theme explorer at: http://localhost:3003

**If you use a different email address**, the system will create a new empty workspace for you instead of showing the seeded data. To access seeded data with a different account, you must add your user to the seeded workspace using SQL:

  ```bash
  # First, find your user_id
  curl -X POST http://localhost:8080/ -H "Content-Type: application/json" \
    -d '{"statements":["SELECT id, email FROM user"]}'

  # Then add association (replace USER_ID with your id)
  curl -X POST http://localhost:8080/ -H "Content-Type: application/json" \
    -d '{"statements":["INSERT INTO users_to_workspaces (user_id, workspace_id, role) VALUES (USER_ID, 1, '\''owner'\'')"]}'
  ```


## Local Checker (monitor tests)

On-demand monitor tests ("Test" button) and check triggers are served by the
local `checker` service (`apps/checker`). The dashboard/server route to it via
`OPENSTATUS_CHECKER_URL=http://checker:8080` (set in `.env.docker`); when unset,
the hosted checker is used instead.

The checker runs with `CLOUD_PROVIDER=koyeb` / `KOYEB_REGION=fra` so it performs
checks locally for **any** requested region (the `fly` provider would otherwise
try to fly-replay to a remote region) and reports results tagged `koyeb_fra`.
Auth uses the shared `CRON_SECRET`.

```bash
# Health
curl http://localhost:8082/health   # {"message":"pong","region":"koyeb_fra",...}

# Direct probe (mirrors what the dashboard sends)
curl -X POST http://localhost:8082/ping/ams \
  -H "Authorization: Basic $CRON_SECRET" -H 'Content-Type: application/json' \
  -d '{"url":"https://example.com","method":"GET"}'
```

## Tinybird Setup (Optional)

Tinybird powers analytics and monitoring charts. The application works without
it — monitors, checks, the Test button, status pages, and auth are all
unaffected; only charts/metrics are unavailable.

**It is off by default.** The `tinybird-local` image needs ~6GB RAM, which
starves the stack on an 8GB Docker Desktop (the dashboard fails to start). It is
gated behind the `analytics` compose profile. Enable it only if Docker Desktop
has enough memory (bump to ~12-16GB in Settings → Resources):

```bash
docker compose --profile analytics up -d   # starts tinybird-local too
```

Alternatively, use Tinybird Cloud and set `TINY_BIRD_API_KEY` in `.env.docker`
(leave `tinybird-local` disabled).

## Configuration

### Required Environment Variables

Edit `.env.docker` and set:

```bash
# Authentication
AUTH_SECRET=your-secret-here

# Database
DATABASE_URL=http://libsql:8080
DATABASE_AUTH_TOKEN=basic:token

# Email
RESEND_API_KEY=test
```

### Optional Services

Configure these for full functionality:

```bash
# Redis
UPSTASH_REDIS_REST_URL=
UPSTASH_REDIS_REST_TOKEN=

# Analytics
TINY_BIRD_API_KEY=

# OAuth providers
AUTH_GITHUB_ID=
AUTH_GITHUB_SECRET=
AUTH_GOOGLE_ID=
AUTH_GOOGLE_SECRET=
```

See [.env.docker.example](.env.docker.example) for complete list.

## Development Workflow

### Common Commands

```bash
# View logs
docker compose logs -f [service-name]

# Restart service
docker compose restart [service-name]

# Rebuild after code changes
docker compose up -d --build [service-name]

# Stop all services
docker compose down

# Reset database (removes all data)
docker compose down -v
docker compose up -d
# Migrations run automatically on startup
```

### Authentication

**Magic Link**:

Set `SELF_HOST=true` in `.env.docker` to enable email-based magic link authentication. This allows users to sign in without configuring OAuth providers.

**OAuth Providers**:

Configure GitHub/Google OAuth credentials in `.env.docker` and set up callback URLs:
  - GitHub: `http://localhost:3002/api/auth/callback/github`
  - Google: `http://localhost:3002/api/auth/callback/google`

### Creating Status Pages

**Via Dashboard (Recommended)**:
1. Login to http://localhost:3002
2. Create a workspace
3. Create a status page with a slug
4. Access at http://localhost:3003/[slug]

**Via Database (Testing)**:
```bash
# Insert test data
curl -s http://localhost:8080/v2/pipeline \
  -H 'Content-Type: application/json' \
  --data-raw '{
    "requests":[{
      "type":"execute",
      "stmt":{
        "sql":"INSERT INTO workspace (id, slug, name) VALUES (1, '\''test'\'', '\''Test Workspace'\'');"
      }
    }]
  }'
```

### Resource Limits

Add to `docker-compose.yaml`:

```yaml
services:
  dashboard:
    deploy:
      resources:
        limits:
          cpus: '1.0'
          memory: 1G
        reservations:
          cpus: '0.5'
          memory: 512M
```

## Monitoring

### Health Checks

All services have automated health checks:

```bash
# View health status
docker compose ps

# Inspect specific service
docker inspect openstatus-dashboard --format='{{.State.Health.Status}}'
```

## Getting Help

- **Documentation**: [docs.openstatus.dev](https://www.openstatus.dev/docs)
- **Discord**: [openstatus.dev/discord](https://www.openstatus.dev/discord)
- **GitHub Issues**: [github.com/openstatusHQ/openstatus/issues](https://github.com/openstatusHQ/openstatus/issues)
