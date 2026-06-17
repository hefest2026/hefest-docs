# Getting Started

This guide gets a local Hefest development environment running in a single command.

## Prerequisites

- **Docker** with the Compose plugin — verify with `docker compose version`
- **Git** — to clone repos that aren't present yet

No other tools are required. Python, uv, and all API dependencies run inside containers.

## Quick start

From the repository root:

```bash
./hefest-compose/setup.sh
```

The script handles everything automatically:

1. Clones `hefest2026/hefest-api` into `../hefest-api` if it isn't present
2. Creates `hefest-compose/.env` from `.env.example`
3. Builds the API Docker image and starts all services
4. Waits for the API to pass its health check, then runs database migrations

Once it finishes, the API is available at **`http://localhost:8000`**.

## Services

| Service | Image | Port |
|---------|-------|------|
| PostgreSQL 16 | `postgres:16-alpine` | `5432` |
| Hefest API | built from `hefest-api/` | `8000` |

## Useful endpoints

| URL | Description |
|-----|-------------|
| `http://localhost:8000/docs` | Swagger UI — interactive API explorer |
| `http://localhost:8000/health` | Liveness probe (`{"status":"ok","version":"..."}`) |
| `http://localhost:8000/ready` | Readiness probe — checks Postgres and Redis |

## Day-to-day commands

Run these from `hefest-compose/`:

```bash
# Start
docker compose up -d

# Watch API logs
docker compose logs -f api

# Apply pending migrations
docker compose exec api uv run tortoise -c hefest.config.TORTOISE_ORM migrate

# Generate a migration after editing models
docker compose exec api uv run tortoise -c hefest.config.TORTOISE_ORM makemigrations

# Roll back last migration
docker compose exec api uv run tortoise -c hefest.config.TORTOISE_ORM downgrade -v -1

# Shell into the API container
docker compose exec api bash

# Stop
docker compose down

# Full reset (wipes all data volumes)
docker compose down -v && docker compose up -d
```

## Environment variables

`hefest-compose/.env` is created automatically by `setup.sh`. The only variable that needs attention before sharing an environment:

| Variable | Default | Notes |
|----------|---------|-------|
| `HEFEST_JWT_SECRET` | `change-me-for-local-dev` | Change for any non-local environment |

Database and Redis connection strings are set in `compose.yml` directly and do not need to appear in `.env`.

## Repository layout

```
HefestProject/
├── hefest-api/        # FastAPI backend (cloned by setup.sh if missing)
├── hefest-compose/    # Docker Compose for local dev
│   ├── compose.yml
│   ├── setup.sh       # one-shot bootstrap
│   ├── .env.example
│   └── README.md
└── hefest-docs/       # this documentation
```
