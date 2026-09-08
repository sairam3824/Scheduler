# Running, configuring and testing

## Requirements

- Python 3.10–3.13 (3.14 is not supported — pandas/numpy have no wheels for it)
- Node.js 20+
- Optionally Docker, for the PostgreSQL setup

## Option A — scripted setup (macOS/Linux, Windows)

```bash
./setup.sh          # setup.bat on Windows
```

Creates the virtualenv, installs backend and frontend dependencies, copies
`frontend/.env.local`, and runs `python -m scripts.seed --reset` — which imports
the sample workbook and runs a real scheduling pass, so the dashboard is
populated on first sign-in. Then, in two terminals:

```bash
./run-backend.sh    # http://localhost:8000  (docs at /docs)
./run-frontend.sh   # http://localhost:3000
```

## Option B — Docker Compose (PostgreSQL)

```bash
docker compose up --build
docker compose exec backend python -m scripts.seed --reset
```

| Service       | URL                        |
| ------------- | -------------------------- |
| Web dashboard | http://localhost:3000      |
| API docs      | http://localhost:8000/docs |
| PostgreSQL    | localhost:5432             |

## Option C — manual local development

The backend defaults to SQLite, so it runs with no infrastructure at all.

```bash
cd backend
python3 -m venv .venv && source .venv/bin/activate
pip install -r requirements-dev.txt
cp .env.example .env                 # optional; switch DATABASE_URL for PostgreSQL
python -m scripts.seed --reset
uvicorn app.main:app --reload --port 8000
```

```bash
cd frontend
npm install
cp .env.local.example .env.local
npm run dev
```

`NEXT_PUBLIC_API_URL=/api/v1` (the default) proxies through the Next.js rewrite
in `next.config.ts`, which forwards `/api/v1/*` to `BACKEND_URL`
(`http://127.0.0.1:8000` unless set). Point `NEXT_PUBLIC_API_URL` straight at the
backend instead if you would rather bypass the proxy — the backend's
`CORS_ORIGINS` already allows `localhost:3000`.

## Sign-in

| Role | Account | Source |
| ---- | ------- | ------ |
| ADMIN | `admin@example.com` / `admin123` | bootstrapped on first start |
| FACULTY | `faculty.demo@institute.edu` / `demo1234` | created by `scripts/seed.py` |
| STUDENT | `student.demo@example.com` / `demo1234` | created by `scripts/seed.py` |

Each demo account below ADMIN is restricted to its own data — useful for seeing
the scoping behaviour immediately. **Change `FIRST_ADMIN_PASSWORD` and
`SECRET_KEY` before deploying anywhere real.**

## Seeding

```bash
python -m scripts.seed [--reset] [--no-schedule] [--no-evaluations]
```

The seed runs the real workflow rather than inserting fixtures: create tables →
bootstrap → import `samples/interview_data.xlsx` → recalculate free slots →
generate a schedule → confirm it → import `samples/evaluations.xlsx` → create the
demo logins. `--reset` drops every table first. `python -m scripts.generate_samples`
regenerates the sample files.

## Configuration

Every setting is in `app/core/config.py` and overridable by environment variable
or `.env`. Nothing a deployment might want to change is hardcoded deeper.

### Application and database

| Variable | Default | Purpose |
| -------- | ------- | ------- |
| `ENVIRONMENT`, `DEBUG`, `LOG_LEVEL` | `development`, `true`, `INFO` | runtime mode |
| `API_V1_PREFIX` | `/api/v1` | API mount point |
| `DATABASE_URL` | `sqlite:///./academisync.db` | SQLite or `postgresql+psycopg://…` |
| `SQL_ECHO` | `false` | log every statement |
| `DB_POOL_SIZE`, `DB_MAX_OVERFLOW` | 10, 20 | PostgreSQL pooling |

### Security

| Variable | Default | Purpose |
| -------- | ------- | ------- |
| `SECRET_KEY` | placeholder | JWT signing key — **must** be replaced in production |
| `JWT_ALGORITHM` | `HS256` | |
| `ACCESS_TOKEN_EXPIRE_MINUTES` | 720 | token lifetime |
| `CORS_ORIGINS` | localhost 3000/3001 | comma-separated or JSON list |
| `FIRST_ADMIN_EMAIL` / `_PASSWORD` / `_NAME` | `admin@example.com` / `admin123` | bootstrap account |

### Uploads

| Variable | Default |
| -------- | ------- |
| `UPLOAD_DIR` | `./uploads` |
| `MAX_UPLOAD_BYTES` | 26214400 (25 MB) |
| `ALLOWED_UPLOAD_EXTENSIONS` | `.xlsx,.xls,.csv` |

### Scheduling defaults

These seed the `interview_settings` row on first start; afterwards the row (and
the Settings page) is authoritative.

| Variable | Default |
| -------- | ------- |
| `DEFAULT_INTERVIEW_DURATION_MIN` | 30 |
| `DEFAULT_BREAK_DURATION_MIN` | 10 |
| `DEFAULT_DAY_START` / `DEFAULT_DAY_END` | `09:00` / `17:00` |
| `DEFAULT_SLOT_GRANULARITY_MIN` | 15 |
| `DEFAULT_MIN_PANEL_SIZE` / `DEFAULT_MAX_PANEL_SIZE` | 2 / 4 |
| `DEFAULT_MAX_INTERVIEWS_PER_FACULTY_PER_DAY` | 12 |
| `DEFAULT_ALGORITHM` | `optimized` |

`PRIORITY_WEIGHTS` (HARD 1000, HIGH 100, MEDIUM 50, LOW 20, FLEXIBLE 5) and
`DEFAULT_EVALUATION_METRICS` are structured settings in the same file.

## Testing

```bash
cd backend
pytest                      # everything
pytest tests/test_scheduler.py -v      # engine guarantees, per algorithm
pytest tests/test_api.py -v            # full HTTP workflow
pytest tests/test_timeutils.py -v      # interval algebra and parsing
```

Three suites:

- **`test_scheduler.py`** builds problem instances in memory — no database — and
  asserts the engine's guarantees, most of them parametrised across `greedy`,
  `backtracking` and `optimized`.
- **`test_api.py`** drives the real HTTP surface end to end: sign in, import a
  workbook, calculate free slots, generate and confirm a schedule, reschedule,
  lock, cancel, import evaluations, reweight metrics, read analytics.
- **`test_timeutils.py`** covers interval algebra and spreadsheet-tolerant
  date/time parsing.

Tests default to a SQLite file (`DATABASE_URL` is set in `tests/conftest.py`), so
they need no infrastructure.

Frontend: `npm run typecheck` (`tsc --noEmit`) and `npm run build`.

## Database changes

There is no migration tool. `Base.metadata.create_all()` at startup creates
missing tables but never alters existing ones, so a change to a model needs
either a recreated database (`python -m scripts.seed --reset`, or deleting the
SQLite file) or a hand-written migration.

## Troubleshooting

**"Could not reach the API" on the login page** — the backend is not running, or
`NEXT_PUBLIC_API_URL` points somewhere else. Check `curl http://localhost:8000/health`.

**401 immediately after signing in** — `SECRET_KEY` changed between issuing and
verifying the token (for example, a container restarted without a fixed
`SECRET_KEY`). Sign in again.

**"This faculty account is not linked to a faculty record"** — a `FACULTY` or
`STUDENT` account exists without its link. An administrator sets it on the Users
page; the system refuses rather than silently treating it as unrestricted.

**Everything imported but nothing schedules** — check, in order: faculty
availability exists for the window; free slots were calculated (`/free-slots`,
or recalculate); at least one active panel has active members; the scheduling
window in Settings covers the availability dates. Each unscheduled candidate in
the preview carries the exact reason its domain was empty.

**Candidates unscheduled with "Outside candidate availability"** — a candidate
with a non-empty `availability` list is hard-filtered to those windows. Clear the
column to mean "available whenever".

**Free slots look stale** — they are derived. `POST /free-slots/recalculate` (or
the button on the Free Slots page) rebuilds them; every availability, busy-slot
and interview write already triggers this automatically.

**Import rejected as an unsupported file type** — only `.xlsx`, `.xls` and `.csv`
are accepted (`ALLOWED_UPLOAD_EXTENSIONS`).

**pandas/numpy fail to install** — you are probably on Python 3.14. Use 3.12 or
3.13.
