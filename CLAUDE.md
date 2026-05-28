# CLAUDE.md — Learn to Cloud App

## Project Overview

**Learn to Cloud** is a structured cloud-learning platform delivered as a web app. Users progress through phases of cloud engineering curriculum, complete hands-on exercises, and have their work verified automatically (GitHub, AI-powered code analysis, token-based labs, etc.).

**Stack:** FastAPI (Python 3.13) · PostgreSQL · Jinja2/HTMX · Tailwind CSS v4 · Azure Container Apps

---

## Repository Layout

```
learn-to-cloud-app/
├── api/                  # FastAPI backend (all Python source code)
├── content/              # Curriculum as YAML (phases, topics, steps)
├── infra/                # Terraform/Azure infrastructure as code
├── docs/                 # Developer documentation
├── scripts/              # Utility/maintenance scripts
├── .github/workflows/    # GitHub Actions CI/CD
├── docker-compose.yml    # Local PostgreSQL service
└── .pre-commit-config.yaml
```

### `api/` Internal Structure

```
api/
├── main.py               # App entry point — OTel, middleware, route registration
├── models.py             # SQLAlchemy ORM models
├── schemas.py            # Pydantic request/response schemas
├── core/                 # Cross-cutting infrastructure
│   ├── config.py         # Settings via pydantic-settings
│   ├── database.py       # Engine, sessions, connection pool
│   ├── auth.py           # GitHub OAuth (Authlib)
│   ├── azure_auth.py     # Azure managed identity
│   ├── cache.py          # In-memory caching utilities
│   ├── csrf.py           # CSRF middleware
│   ├── github_client.py  # GitHub API client
│   ├── llm_client.py     # Azure OpenAI client
│   ├── logger.py         # Logging configuration
│   ├── middleware.py     # Security headers, user tracking
│   ├── observability.py  # OpenTelemetry setup
│   ├── ratelimit.py      # Rate limiting (slowapi)
│   └── templates.py      # Jinja2 environment
├── routes/               # HTTP layer (thin — delegate to services)
│   ├── pages_routes.py   # Full page renders
│   ├── htmx_routes.py    # Partial HTML responses + SSE streaming
│   ├── auth_routes.py    # GitHub OAuth login/logout
│   ├── users_routes.py   # User API endpoints
│   ├── analytics_routes.py
│   ├── health_routes.py
│   └── legacy_redirects.py
├── services/             # Business logic and orchestration
├── repositories/         # Data access layer (DB queries only)
├── rendering/            # Template context builders
│   ├── context.py        # Page/FAQ context assembly
│   └── steps.py          # Step rendering helpers
├── templates/            # Jinja2 HTML templates
│   ├── base.html
│   ├── layouts/
│   ├── pages/
│   └── partials/         # HTMX fragments
├── static/
│   ├── css/input.css     # Tailwind source
│   └── css/styles.css    # Generated (built during Docker build)
├── alembic/              # Database migrations
│   └── versions/         # Sequential migration files (0001_baseline → …)
└── tests/                # pytest test suite
    ├── conftest.py
    ├── core/
    ├── routes/
    ├── services/
    ├── repositories/
    └── rendering/
```

---

## Local Development Setup

### Prerequisites

- Python 3.13 (via `uv`)
- Docker (for PostgreSQL)
- Node.js (only needed if rebuilding Tailwind CSS)

### 1. Start the database

```bash
docker compose up db -d
```

PostgreSQL is exposed at `localhost:5432` with user/pass `postgres`/`postgres` and database `learn_to_cloud`.

### 2. Install dependencies

```bash
cd api
uv sync
```

### 3. Configure environment

```bash
cp api/.env.example api/.env
# Edit .env — minimum required for local dev:
# DATABASE_URL (already set to the docker-compose DB)
# DEBUG=true
# SESSION_SECRET_KEY (generate one — see .env.example comment)
```

### 4. Run database migrations

```bash
cd api
uv run alembic upgrade head
```

### 5. Start the API

```bash
cd api
uv run uvicorn main:app --reload
```

App is available at `http://localhost:8000`. With `DEBUG=true`, OpenAPI docs are at `/docs`.

### Tailwind CSS (only if editing styles)

```bash
cd api
npm install
npm run dev     # watch mode
npm run build   # one-shot build
```

Generated output goes to `api/static/css/styles.css`. This file is rebuilt automatically during Docker builds — do not commit manually generated CSS unless intentional.

---

## Key Commands

All commands run from the `api/` directory with `uv run`.

| Task | Command |
|------|---------|
| Run API (dev) | `uv run uvicorn main:app --reload` |
| Run tests | `uv run pytest` |
| Run tests with coverage | `uv run pytest --cov=. --cov-report=term-missing` |
| Run a single test file | `uv run pytest tests/services/test_foo.py -v` |
| Run tests by marker | `uv run pytest -m unit` |
| Lint (check only) | `uv run ruff check .` |
| Lint (auto-fix) | `uv run ruff check --fix .` |
| Format | `uv run ruff format .` |
| Type check | `uv run ty check --exclude scripts --exclude tests .` |
| Create migration | `uv run alembic revision --autogenerate -m "describe_change"` |
| Apply migrations | `uv run alembic upgrade head` |
| Downgrade migration | `uv run alembic downgrade -1` |

---

## Architecture Patterns

### Request Flow

```
HTTP Request
  → Middleware (CSRF, session, security headers, rate limiting)
  → Route handler (validate, extract, call service)
  → Service (business logic, orchestration)
  → Repository (database queries via SQLAlchemy)
  → Response (Jinja2 HTML or JSON)
```

Routes must stay thin. Business logic belongs in services; database queries belong in repositories.

### Frontend Pattern (HTMX + Jinja2)

- Full-page loads use `pages_routes.py` + templates in `templates/pages/`
- Interactive updates use `htmx_routes.py` returning HTML fragments from `templates/partials/`
- No separate JS framework — Alpine.js handles small interactive bits inline
- Tailwind CSS v4 for all styling

### Async Database Access

Sessions are async (`asyncpg` driver). Always use `async with` and `await` for DB operations. The session is injected via FastAPI dependency injection from `core/database.py`.

```python
# Correct pattern
async def get_user(session: AsyncSession, user_id: int) -> User | None:
    result = await session.execute(select(User).where(User.id == user_id))
    return result.scalar_one_or_none()
```

### Verification System

Hands-on exercises route through `services/hands_on_verification_service.py`, which dispatches to specialized verifiers based on `SubmissionType`:

| Type | Verifier |
|------|---------|
| `GITHUB_PROFILE`, `PROFILE_README`, `REPO_FORK`, `PR_REVIEW` | `github_hands_on_verification_service.py` |
| `CODE_ANALYSIS` | `code_verification_service.py` (Azure OpenAI) |
| `DEVOPS_ANALYSIS` | `devops_verification_service.py` |
| `DEPLOYED_API` | `deployed_api_verification_service.py` |
| `SECURITY_ANALYSIS` | `security_verification_service.py` |
| `CTF_TOKEN` | `ctf_service.py` |
| `NETWORKING_TOKEN` | `networking_lab_service.py` |
| `JOURNAL_ENTRY`, `EVIDENCE_URL` | direct pass-through |

---

## Database

### ORM Models (`api/models.py`)

- `User` — GitHub-authenticated user (PK is the GitHub user ID, a `BigInteger`)
- `Submission` — a single verification attempt with type, status, and result payload
- `StepProgress` — completion record for one step per user
- `UserPhaseProgress` — denormalized phase progress (updated on step completion)
- `AnalyticsSnapshot` — aggregated metrics for dashboard

### Migrations (Alembic)

See `docs/migrations.md` for full strategy. Key rules:

- Name files `NNNN_short_description.py` (sequential numbering)
- Always include both `upgrade()` and `downgrade()`
- Use `IF NOT EXISTS` / `IF EXISTS` guards for idempotency where Alembic doesn't handle it
- Advisory locking is used to prevent migration races in multi-instance deployments

---

## Content System

Course content lives in `content/phases/` as YAML files — **not** in the database.

```
content/phases/
├── phase0/
│   ├── _phase.yaml         # Phase metadata (title, description, order)
│   └── topic-name.yaml     # Topic definition with steps array
├── phase1/
...
```

The `services/content_service.py` loads and caches this YAML at startup. To add new curriculum content, edit or add YAML files under `content/phases/` — no database changes required.

---

## Environment Variables

See `api/.env.example` for all variables. Key ones:

| Variable | Required | Purpose |
|----------|----------|---------|
| `DATABASE_URL` | Yes | PostgreSQL connection (`postgresql+asyncpg://...`) |
| `DEBUG` | No | Enables `/docs`, CORS from localhost, relaxes validation |
| `SESSION_SECRET_KEY` | Yes (prod) | Cookie signing key |
| `GITHUB_CLIENT_ID` / `GITHUB_CLIENT_SECRET` | Yes (prod) | GitHub OAuth app |
| `GITHUB_TOKEN` | No | Higher GitHub API rate limits |
| `LABS_VERIFICATION_SECRET` | No | CTF/networking lab token validation |
| `LLM_BASE_URL` / `LLM_API_KEY` / `LLM_MODEL` | No | Azure OpenAI for code verification |
| `APPLICATIONINSIGHTS_CONNECTION_STRING` | No | Azure Monitor telemetry |
| `OTLP_ENDPOINT` / `OTEL_SERVICE_NAME` | No | Local Aspire Dashboard observability |

---

## Testing

**Framework:** pytest with `asyncio_mode = auto` (all tests can be async).

**Test markers:**

| Marker | Meaning |
|--------|---------|
| `unit` | Fast, no I/O — mock all external dependencies |
| `integration` | Requires a live database |
| `slow` | Takes > 1 second |
| `smoke` | Minimal subset for quick validation |
| `skip_until_fixed` | Temporarily skipped (must reference an issue) |

**Coverage:** Minimum 40% enforced in CI (`fail_under = 40` in `pyproject.toml`).

**Fixtures** are in `tests/conftest.py` — check there before creating new ones.

Run only unit tests locally to stay fast: `uv run pytest -m unit`

---

## Code Quality

**Linter/formatter:** Ruff (`line-length = 88`, rules `E`, `F`, `I`, `UP`)
**Type checker:** `ty` (Astral) — excludes `scripts/` and `tests/`
**Pre-commit hooks** (using `prek`):

```bash
prek install      # install hooks
prek run --all-files  # run manually
```

Hooks run: `ruff lint --fix`, `ruff format`, `ty check`, `trailing-whitespace`, `end-of-file-fixer`, `check-yaml`, `check-json`, `check-added-large-files` (max 500 KB).

**Style rules:**
- No type annotations in `tests/` (excluded from ty)
- `invalid-argument-type` is `warn` not `error` due to a known ty upstream issue with `Protocol[ParamSpec]`
- Line length: 88 characters

---

## CI/CD

**Workflow:** `.github/workflows/deploy.yml`

**Triggers:** push to `main`, PRs to `main`, manual `workflow_dispatch`

**Jobs:**

1. **changes** — path-filter; skips deploy jobs when only docs/scripts changed
2. **ci** — lint → type-check → test (with coverage) → build Docker image; runs on every push and PR
3. **deploy** — push image to Azure Container Registry → `terraform apply` → trigger Azure Container Apps revision; only runs on `main`

Deployment requires these GitHub secrets/vars: `AZURE_CREDENTIALS` (JSON service principal), `AZURE_ENV_NAME`, `AZURE_LOCATION`, `AZURE_SUBSCRIPTION_ID`.

---

## Docker

The `api/Dockerfile` uses a multi-stage build:

1. **builder** — installs Python dependencies with `uv`
2. **tailwind** — builds CSS from `static/css/input.css`
3. **runtime** — lean final image, copies built assets

For multi-worker testing, use: `docker compose up db api-multiworker --build`

---

## Common Gotchas

- **OTel must be configured before `fastapi.FastAPI()` instantiates** — `configure_observability()` is called at the top of `main.py` before the app object is created.
- **Tailwind CSS is built during Docker build**, not committed as source. The `static/css/styles.css` file in the repo is a pre-built artifact used for local dev without Docker.
- **GitHub user IDs are `BigInteger`** (`int64`) — use `BigInteger` not `Integer` in any model referencing `User.id`.
- **All DB sessions are async** — never use synchronous SQLAlchemy patterns; always `await` queries.
- **Content is loaded from YAML at startup** — changes to `content/` require an app restart (or redeploy); the database is not involved.
- **Advisory locking in migrations** — the Alembic env uses PostgreSQL advisory locks to prevent race conditions when multiple container instances start simultaneously.
