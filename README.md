# AcademiSync

**Automated interview scheduling and evaluation management.**

AcademiSync turns a spreadsheet of candidates, faculty and availability into a
conflict-free interview schedule, then manages the evaluation that follows.
A constraint-satisfaction engine does the scheduling; a Next.js dashboard exposes
the whole workflow to five different kinds of user.

The pipeline is real end to end — there is no mock data anywhere in the UI:

```
Excel/CSV upload → validation → database → faculty availability
   → free-slot calculation → panel assignment → constraint-based scheduling
   → conflict detection → schedule views → manual override
   → evaluation → compiled results → analytics & reports
```

| | |
| --- | --- |
| **Backend** | FastAPI · Pydantic v2 · SQLAlchemy 2 · Pandas · OpenPyXL · Python 3.10–3.13 |
| **Frontend** | Next.js 16 · React 19 · TypeScript · Tailwind v4 · TanStack Query · Recharts · FullCalendar |
| **Database** | SQLite out of the box, PostgreSQL 16 in Docker/production |
| **Tests** | 103 backend tests (`pytest`) |
| **Licence** | MIT |

---

## Contents

- [Quick start](#quick-start)
- [Signing in](#signing-in)
- [Documentation](#documentation)
- [What the system does](#what-the-system-does)
- [Roles](#roles)
- [Architecture](#architecture)
- [The scheduling engine](#the-scheduling-engine)
- [Free-slot calculation](#free-slot-calculation)
- [Flexible scheduling priorities](#flexible-scheduling-priorities)
- [Evaluation metrics](#evaluation-metrics)
- [Excel input formats](#excel-input-formats)
- [Sample data](#sample-data)
- [API reference](#api-reference)
- [Configuration](#configuration)
- [Testing](#testing)
- [Project structure](#project-structure)
- [Troubleshooting](#troubleshooting)

---

## Quick start

### Option A — the setup scripts (recommended)

They check your toolchain, create the virtual environment, install both
dependency sets and load the sample data. Run once:

```bash
./setup.sh          # macOS / Linux        (Windows: setup.bat)
```

Then start the two servers, each in its own terminal:

```bash
./run-backend.sh    # http://localhost:8000  (API docs at /docs)
./run-frontend.sh   # http://localhost:3000
```

Requires Python 3.10–3.13 (**not** 3.14 — pandas/numpy have no wheels for it yet)
and Node.js 20+.

### Option B — Docker Compose (PostgreSQL, closest to production)

```bash
docker compose up --build
docker compose exec backend python -m scripts.seed --reset   # sample data + a first schedule
```

| Service | URL |
| ------- | --- |
| Web dashboard | http://localhost:3000 |
| API docs | http://localhost:8000/docs |
| PostgreSQL | localhost:5432 |

### Option C — manual local development

The backend defaults to SQLite, so it runs with zero infrastructure.

```bash
# backend
cd backend
python3 -m venv .venv && source .venv/bin/activate
pip install -r requirements-dev.txt
cp .env.example .env                 # optional: switch DATABASE_URL to PostgreSQL
python -m scripts.seed --reset       # sample data + a real scheduling run
uvicorn app.main:app --reload --port 8000
```

```bash
# frontend, in a second terminal
cd frontend
npm install
cp .env.local.example .env.local     # NEXT_PUBLIC_API_URL=http://localhost:8000/api/v1
npm run dev
```

The schema is created from the SQLAlchemy metadata on startup, so a fresh
database needs no migration step. To use PostgreSQL locally instead:

```bash
createdb interview_scheduler
export DATABASE_URL="postgresql+psycopg://$USER@localhost:5432/interview_scheduler"
python -m scripts.seed --reset
```

---

## Signing in

Open http://localhost:3000. The bootstrap administrator exists on first start;
`scripts.seed` additionally creates one demo account per restricted role:

| Role | Email | Password | Lands on |
| ---- | ----- | -------- | -------- |
| `ADMIN` | `admin@example.com` | `admin123` | Dashboard |
| `FACULTY` | `faculty.demo@institute.edu` | `demo1234` | My Schedule |
| `STUDENT` | `student.demo@example.com` | `demo1234` | My Interview |

Each demo account below `ADMIN` sees only its own data — that is the point of
signing in as one.

> Change `FIRST_ADMIN_PASSWORD` and `SECRET_KEY` before deploying anywhere real.

**Account lifecycle.** Administrators create accounts on the *User Accounts*
page, or bulk-provision one login per imported faculty/candidate
(`POST /uploads/provision-accounts`) with generated passwords. A person the
institute already has on file can self-claim a login at `/claim`. Generated and
reset passwords are marked as such, and the holder is sent to `/change-password`
on first sign-in.

---

## Documentation

This README is the tour. The reference lives in [`docs/`](docs/README.md), and as
a single Word document —
[`docs/AcademiSync-Documentation.docx`](docs/AcademiSync-Documentation.docx),
rebuilt from these files with `python3 docs/build-docx.py` (needs `pandoc`):

| Document | Covers |
| -------- | ------ |
| [architecture.md](docs/architecture.md) | layers, bootstrap, error model, derived data, request lifecycle |
| [data-model.md](docs/data-model.md) | every table, column and enum |
| [roles-and-permissions.md](docs/roles-and-permissions.md) | the five roles, gating vs scoping, account lifecycle |
| [scheduling-engine.md](docs/scheduling-engine.md) | free slots, domains, constraints, the three algorithms, overrides |
| [evaluation-and-analytics.md](docs/evaluation-and-analytics.md) | configurable metrics, multi-evaluator compiling, dashboards |
| [data-import.md](docs/data-import.md) | accepted sheets, column aliases, validation |
| [api-reference.md](docs/api-reference.md) | every endpoint with the roles that may call it |
| [frontend.md](docs/frontend.md) | routes, pages, data layer, guards |
| [operations.md](docs/operations.md) | setup, configuration, testing, troubleshooting |

`project.md` holds the original requirement specification and `torun.txt` a
manual walkthrough script.

---

## What the system does

### 1. Data ingestion
Upload one multi-sheet workbook or several single-purpose files. Sheets are
recognised by their **columns**, not their position or name, and each field
accepts aliases (`faculty_id`, `Faculty ID`, `faculty code`, `employee_id`…).
Validation reports the sheet, row number, column and value for every problem, and
**nothing is imported unless every sheet passes**.

### 2. Free-slot calculation
Availability minus busy time minus booked interviews, recomputed automatically
whenever any of those change.

### 3. Automated scheduling
A constraint-satisfaction engine assigns *candidate × panel × time slot*,
honouring hard constraints absolutely and optimising the soft ones by score. A
run is stored as a `PREVIEW` and becomes real interviews only when confirmed.

### 4. Manual override
Administrators can move interviews, swap panels or members, lock, cancel,
complete, and mark faculty unavailable. Every change re-checks conflicts,
recalculates the affected free slots, surfaces warnings and is written to
history. Faculty and students can raise a change request instead of editing.

### 5. Evaluation and analytics
Seven configurable metrics with weights and ranges, marks from several evaluators
averaged per metric, weighted and normalised scores, rankings,
strongest/weakest metric detection, and dashboards covering workload,
utilisation, scheduling efficiency and score distribution.

---

## Roles

Five roles share one dashboard. Policy lives in `backend/app/core/permissions.py`
(no FastAPI imports, so it is testable on its own); `frontend/lib/access.ts`
mirrors it to decide what to *render* — the server remains the authority.

| Role | Can do |
| ---- | ------ |
| `ADMIN` | Everything, including user administration |
| `COORDINATOR` | Admin-lite: runs the whole scheduling workflow, but may not manage accounts |
| `FACULTY` | Own availability, own panels' interviews, own evaluations |
| `STUDENT` | Own interview only (a candidate's login) |
| `VIEWER` | Read-only institute-wide reporting |

Two mechanisms, because two different things need protecting:

- **Gating** — endpoints a role may not touch at all (uploads, the scheduler,
  settings, user administration). A blocked call returns 403 naming the caller's
  role and the allowed roles.
- **Scoping** — endpoints everyone may call, where the *rows* narrow to the
  caller. A faculty member listing interviews sees their own panels, not a 403,
  so one shared page keeps working for every role. A `FACULTY` or `STUDENT`
  account with no linked person is refused rather than treated as unrestricted.

Ownership checks on mutations (`assert_faculty_owns`, `assert_candidate_owns`)
are called from the service layer, not only the routers, so alternative entry
points — the Excel importer, for instance — cannot bypass them.

Full matrix: [docs/roles-and-permissions.md](docs/roles-and-permissions.md).

---

## Architecture

```
frontend/                Next.js 16 · TypeScript · Tailwind v4 · TanStack Query · Recharts · FullCalendar
  app/(app)/…            15 dashboard pages behind an auth + role guard
  app/login, claim,      standalone auth pages
      change-password
  components/ui/         shadcn-style primitives (Radix + CVA)
  lib/api.ts             typed fetch client with JWT handling
  lib/queries.ts         React Query hooks and cache invalidation
  lib/access.ts          route → roles map shared by the sidebar and the guard

backend/                 FastAPI · Pydantic v2 · SQLAlchemy 2 · Pandas · OpenPyXL
  app/core/              config, database, security, permissions, logging, exceptions
  app/models/            19 SQLAlchemy tables with indexes and foreign keys
  app/schemas/           request/response validation
  app/repositories/      all database access lives here
  app/services/          business logic (Excel, free slots, panels, scheduling,
                         interviews, evaluation, analytics, accounts, history)
  app/scheduling/        the engine — no database or web imports at all
  app/api/v1/routers/    REST endpoints
  tests/                 103 tests
```

**Layering rule:** routers → services → repositories → models. The scheduling
engine sits beside them and depends on none of them, which is what makes it
directly unit-testable and replaceable.

### Database tables

`users`, `candidates`, `faculty`, `faculty_availability`, `faculty_busy_slots`,
`faculty_free_slots`, `panel_groups`, `panel_members`, `interviews`,
`interview_panel_members`, `interview_schedule_history`,
`interview_change_requests`, `scheduling_runs`, `scheduling_constraints`,
`interview_settings`, `evaluation_metrics`, `evaluations`, `evaluation_scores`,
`uploaded_files`.

---

## The scheduling engine

`backend/app/scheduling/` implements the documented process:

| Step | Module | What happens |
| ---- | ------ | ------------ |
| 1–4 | `services/scheduling_service.py` | Load candidates, faculty free slots, panels and constraints into a plain-Python `SchedulingContext` |
| 5 | `domain.py` | Generate every candidate × panel × time-slot combination |
| 6 | `constraints.py` | Drop combinations violating a **HARD** rule |
| 7 | `constraints.py` | Score survivors from the soft-constraint outcomes |
| 8 | `algorithms/` | Assign the best slots |
| 9 | `state.py` | Re-check overlap, break and cap rules at placement time |
| 10 | `algorithms/backtracking.py` | Try alternative slots and panels on conflict |
| 11–12 | `engine.py` | Return scheduled + unscheduled candidates, each with a reason |

### Guarantees (each one has a test)

1. A candidate never has overlapping interviews.
2. A faculty member never has overlapping interviews — across *all* panels.
3. A panel is never double-booked.
4. Interviews land only inside faculty free slots.
5. The interview duration is respected exactly.
6. The configured break is preserved between consecutive interviews.
7. Existing (locked/completed) bookings are respected.
8. Candidate and faculty availability are respected.
9. Conflicts are verified independently after solving.
10. Soft constraints are maximised, not left to chance.

### Pluggable algorithms

Strategies register themselves and are chosen per run. All three work on the same
domain and the same objective — `(candidates scheduled, total score)`, in that
order:

| Name | Description |
| ---- | ----------- |
| `greedy` | One pass. Most-constrained candidate first, best-scoring free slot first. |
| `backtracking` | CSP search with most-constrained-variable ordering, a greedy incumbent as a lower bound, bounded node budget, and "leave this candidate out" as an explicit branch. |
| `optimized` | **Default.** Backtracking, then a local-search pass (insertion, relocation and swap moves) that keeps only strict gains. |

Adding integer programming or full constraint programming means writing one class:

```python
@register_algorithm("my-solver")
class MySolver(SchedulerAlgorithm):
    description = "..."

    def solve(self, ctx, options, blocked) -> Solution:
        ...
```

Nothing else changes — data loading, constraint evaluation, conflict detection and
persistence are all outside the algorithm.

### Adding a scheduling rule

```python
@register(ConstraintType.MY_RULE)
def _my_rule(ctx, constraint, candidate, panel, day, slot, faculty_ids):
    ok = ...                      # your condition
    return _outcome(constraint, ctx, ok, "why this passed or failed")
```

Set its priority to `HARD` and it filters; set it to anything else and it scores.
The solver itself never changes.

---

## Free-slot calculation

```
free slots = declared availability − (busy slots + booked interviews)
```

The worked example from the specification, which is also a test:

```
Available window   09:00–17:00
Busy               10:00–11:00, 13:00–14:00
Calculated free    09:00–10:00, 11:00–13:00, 14:00–17:00
```

Recalculation is triggered automatically when faculty availability changes, and
when an interview is scheduled, rescheduled or cancelled — cancelling an
interview removes its busy rows and hands the time back.

---

## Flexible scheduling priorities

Eleven rules are seeded into the `scheduling_constraints` table, each with a
priority band, and the band can be changed at runtime from the **Automated
Scheduler** page.

| Priority | Weight | Behaviour | Seeded examples |
| -------- | ------ | --------- | --------------- |
| `HARD` | 1000 | Filters combinations; never violated | Faculty free, candidate available, panel size, break between interviews, daily cap |
| `HIGH` | 100 | Strongly preferred | Candidate preferred date |
| `MEDIUM` | 50 | Preferred, decays with distance | Candidate preferred time |
| `LOW` | 20 | Nice to have | Preferred panel, department match |
| `FLEXIBLE` | 5 | Tie-breakers | Compact schedule, faculty load balancing |

Raising *candidate preferred date* from `HIGH` to `HARD` turns a preference into a
filter — the candidate is reported unscheduled rather than moved to another day.
Every scheduled interview stores the per-rule outcome, so the UI can show exactly
why a slot was chosen, and every unscheduled candidate carries the reason its
domain came out empty.

---

## Evaluation metrics

Seven metrics are seeded, but **no metric name appears anywhere in the code**.
Names, weights, ranges and the number of metrics are all editable at runtime and
drive both scoring and Excel column matching.

```
normalised(metric) = (raw − min) / (max − min) × 100
overall_score      = Σ (raw × weight)
normalised_score   = Σ (normalised × weight) / Σ weight
```

Marks from several evaluators are averaged per metric before compiling. Changing
a weight re-scores every stored evaluation immediately; rankings, the
strongest/weakest metric per candidate and the radar profiles all follow.

---

## Excel input formats

Column names are matched case-insensitively and ignore spaces, hyphens and
underscores. Required columns are **bold**; everything else is optional. The
**Data / Excel Upload** page lists the live mapping, including aliases.

| Sheet | Columns |
| ----- | ------- |
| Candidates | **candidate_id**, **candidate_name**, email, phone, department, position, preferred_date, preferred_time, preferred_panel, availability, priority, constraints, notes |
| Faculty | **faculty_id**, **faculty_name**, department, email, phone, designation, max_interviews_per_day |
| Faculty Availability | **faculty_id**, **date**, **start_time**, **end_time**, availability_status, note |
| Faculty Busy Slots | **faculty_id**, **date**, **start_time**, **end_time**, reason |
| Panel Groups | **panel_id**, **panel_name**, **faculty_members**, minimum_panel_size, maximum_panel_size, department, description, mandatory_members |
| Interview Settings | **interview_duration**, break_duration, start_time, end_time, scheduling_date_range, slot_granularity, min/max_panel_size, max_interviews_per_faculty_per_day, allow_weekends |
| Evaluation | **candidate_id**, *one column per configured metric*, evaluator_id, panel_id, evaluation_date, recommendation, remarks |

**Accepted value formats**

- Dates: `2026-03-02`, `02-03-2026`, `02/03/2026`, `2 Mar 2026`, native Excel dates
- Times: `09:00`, `9:00 AM`, `14:30:00`, `0930`, native Excel times
- Faculty lists: `FAC001, FAC002; FAC003` (comma, semicolon, pipe or slash)
- Candidate availability: `2026-03-02 09:00-12:00; 2026-03-03 14:00 to 17:00`, or
  just `09:00-12:00` to reuse the preferred date

---

## Sample data

`samples/` is regenerated (with dates in the coming week) by:

```bash
cd backend && python -m scripts.generate_samples
```

| File | Contents |
| ---- | -------- |
| `interview_data.xlsx` | 6 sheets: 30 candidates, 12 faculty, 103 availability windows, 7 busy slots, 4 panels, 1 settings row |
| `evaluations.xlsx` | 7-metric scores for 20 candidates |
| `candidates.csv` | The CSV single-dataset path |
| `invalid_candidates.xlsx` | Deliberately broken — missing column, bad date, bad time — to demonstrate validation |

`python -m scripts.seed --reset` imports the workbook, calculates free slots, runs
the scheduler, confirms the result, imports the evaluations and creates the demo
accounts. On the sample data it schedules **30/30 candidates with 0 conflicts in
about half a second**.

Useful flags: `--no-schedule`, `--no-evaluations`.

---

## API reference

Interactive documentation: **http://localhost:8000/docs**. All endpoints below
are prefixed with `/api/v1` and require `Authorization: Bearer <token>` except
login and claim. Which roles may call what is listed in
[docs/api-reference.md](docs/api-reference.md).

| Area | Endpoints |
| ---- | --------- |
| Auth | `POST /auth/login`, `GET /auth/me`, `POST /auth/claim`, `POST /auth/change-password` |
| Users | `GET\|POST /auth/users`, `PUT\|DELETE /auth/users/{id}`, `POST /auth/users/{id}/reset-password` |
| Upload | `POST /uploads/validate`, `POST /uploads/import`, `POST /uploads/{id}/import`, `POST /uploads/provision-accounts`, `GET /uploads`, `GET /uploads/column-mappings`, `DELETE /uploads/{id}` |
| Candidates | `GET\|POST /candidates`, `GET\|PUT\|DELETE /candidates/{id}` |
| Faculty | `GET\|POST /faculty`, `GET\|PUT\|DELETE /faculty/{id}`, `GET /faculty/departments`, `GET /faculty/{id}/availability` |
| Availability | `GET\|POST /faculty-availability`, `PUT\|DELETE /faculty-availability/{id}`, `GET\|POST /faculty-busy-slots`, `DELETE /faculty-busy-slots/{id}` |
| Free slots | `GET /free-slots`, `GET /free-slots/grouped`, `GET /free-slots/timeline`, `POST /free-slots/recalculate` |
| Panels | `GET\|POST /panels`, `GET\|PUT\|DELETE /panels/{id}`, `GET /panels/{id}/availability`, `POST /panels/alternatives`, `GET /panels/conflicts` |
| Scheduling | `POST /scheduling/generate`, `POST /scheduling/generate-and-confirm`, `GET /scheduling/runs`, `GET /scheduling/runs/{id}`, `POST /scheduling/runs/{id}/confirm`, `POST /scheduling/runs/{id}/discard`, `GET /scheduling/algorithms` |
| Constraints | `GET\|POST /constraints`, `PUT\|DELETE /constraints/{id}` |
| Interviews | `GET\|POST /interviews`, `GET\|DELETE /interviews/{id}`, `PUT /interviews/{id}/reschedule`, `PUT /interviews/{id}/status`, `PUT /interviews/{id}/lock`, `POST /interviews/{id}/cancel`, `POST /interviews/{id}/confirm`, `GET /interviews/{id}/history`, `GET /interviews/calendar`, `GET /interviews/conflicts`, `GET /interviews/available-slots`, `POST /interviews/faculty-unavailable` |
| Change requests | `POST /interviews/{id}/change-request`, `GET /interview-requests`, `POST /interview-requests/{id}/decide` |
| Evaluation | `GET\|POST /evaluations`, `GET\|PUT\|DELETE /evaluations/{id}`, `GET /evaluations/rankings`, `GET /evaluations/my-result`, `GET /evaluations/candidate/{id}/profile`, `GET\|POST /evaluation-metrics`, `PUT /evaluation-metrics`, `PUT\|DELETE /evaluation-metrics/{id}` |
| Analytics | `GET /analytics/dashboard`, `/analytics/summary`, `/analytics/scheduling`, `/analytics/evaluation`, `/analytics/report` |
| Settings | `GET\|PUT /settings` |

### Example: generate and confirm a schedule

```bash
TOKEN=$(curl -s -X POST http://localhost:8000/api/v1/auth/login \
  -H 'Content-Type: application/json' \
  -d '{"email":"admin@example.com","password":"admin123"}' | jq -r .access_token)

RUN=$(curl -s -X POST http://localhost:8000/api/v1/scheduling/generate \
  -H "Authorization: Bearer $TOKEN" -H 'Content-Type: application/json' \
  -d '{"algorithm":"optimized"}')

echo "$RUN" | jq '{scheduled: (.scheduled|length), unscheduled: (.unscheduled|length), rate: .success_rate}'

curl -s -X POST "http://localhost:8000/api/v1/scheduling/runs/$(echo "$RUN" | jq .run_id)/confirm" \
  -H "Authorization: Bearer $TOKEN" | jq
```

Errors always come back in the same shape:

```json
{ "error": "conflict",
  "message": "The requested change conflicts with the schedule",
  "details": ["Faculty 1 on 2026-09-11: 09:00-09:30 and 09:30-10:00 leave less than 10 min between interviews"] }
```

---

## Configuration

Everything tunable lives in `backend/app/core/config.py` and can be overridden by
environment variables or `backend/.env` (see `.env.example`).

| Variable | Default | Purpose |
| -------- | ------- | ------- |
| `DATABASE_URL` | `sqlite:///./academisync.db` | PostgreSQL in production |
| `SECRET_KEY` | dev placeholder | JWT signing key — **change it** |
| `ACCESS_TOKEN_EXPIRE_MINUTES` | `720` | Token lifetime |
| `CORS_ORIGINS` | `http://localhost:3000` (+127.0.0.1, :3001) | Comma-separated allowed origins |
| `FIRST_ADMIN_EMAIL` / `_PASSWORD` | `admin@example.com` / `admin123` | Bootstrap account |
| `MAX_UPLOAD_BYTES` | `26214400` | 25 MB upload cap |
| `ALLOWED_UPLOAD_EXTENSIONS` | `.xlsx,.xls,.csv` | Accepted upload types |
| `DEFAULT_INTERVIEW_DURATION_MIN` | `30` | Seed value for settings |
| `DEFAULT_BREAK_DURATION_MIN` | `10` | Break between interviews |
| `DEFAULT_SLOT_GRANULARITY_MIN` | `15` | Slot grid |
| `DEFAULT_ALGORITHM` | `optimized` | Default strategy |

Runtime settings (duration, break, working day, date range, panel sizes, daily
caps, weekends, algorithm) are stored in `interview_settings` and edited on the
**Settings** page. The frontend reads `NEXT_PUBLIC_API_URL` — baked in at build
time, so rebuild after changing it.

---

## Testing

```bash
cd backend
.venv/bin/python -m pytest -q                      # 103 tests
.venv/bin/python -m pytest tests/test_scheduler.py -v
```

| File | Covers |
| ---- | ------ |
| `tests/test_timeutils.py` | Interval algebra, free-slot subtraction, spreadsheet date/time parsing |
| `tests/test_scheduler.py` | Every scheduling guarantee, all three algorithms, priority handling, unscheduled reasons, a 40-candidate instance |
| `tests/test_api.py` | The full workflow over HTTP: upload → validate → import → free slots → schedule → confirm → override → evaluate → analytics, plus role gating and scoping |

Frontend checks:

```bash
cd frontend
npm run typecheck
npm run build
```

---

## Project structure

```
.
├── backend/
│   ├── app/
│   │   ├── api/v1/routers/     auth, uploads, candidates, faculty, free_slots,
│   │   │                       panels, scheduling, interviews, evaluations,
│   │   │                       analytics, settings
│   │   ├── core/               config · database · security · permissions ·
│   │   │                       logging · exceptions
│   │   ├── models/             19 tables
│   │   ├── repositories/       data access
│   │   ├── scheduling/         engine (types, constraints, domain, state,
│   │   │                       placement, conflicts, algorithms/, engine)
│   │   ├── schemas/            Pydantic models
│   │   ├── services/           excel · free slots · panels · scheduling ·
│   │   │                       interviews · evaluation · analytics ·
│   │   │                       accounts · history · bootstrap
│   │   └── utils/timeutils.py  interval arithmetic
│   ├── scripts/                generate_samples.py · seed.py
│   └── tests/                  103 tests
├── frontend/
│   ├── app/(app)/              dashboard, upload, users, candidates, faculty,
│   │                           availability, free-slots, panels, scheduler,
│   │                           schedule, evaluations, analytics, settings,
│   │                           my-schedule, my-interview
│   ├── app/                    login, claim, change-password
│   ├── components/             ui primitives, layout, charts, calendar
│   └── lib/                    api client, query hooks, types, access, utils
├── docs/                       the full reference (+ generated .docx)
├── samples/                    generated Excel/CSV input files
├── setup.sh / setup.bat        one-time setup
├── run-backend.sh / .bat       start the API
├── run-frontend.sh / .bat      start the dashboard
└── docker-compose.yml
```

---

## Troubleshooting

**"Could not reach the API" on the login page**
The backend is not running or `NEXT_PUBLIC_API_URL` is wrong. Check
`curl http://localhost:8000/health` and `frontend/.env.local`. The value is baked
in at build time, so rebuild the frontend after changing it.

**Setup fails on Python 3.14**
pandas and numpy have no wheels for it yet. Install Python 3.12 or 3.13;
`setup.sh` looks for those first.

**The scheduler returns "No active panel group has available faculty members"**
Free slots are empty. Import faculty availability, then use **Recalculate** on the
Free Slots page. Confirm the scheduling date range on the Settings page overlaps
the availability dates.

**Candidates come back unscheduled**
The reason is shown per candidate on the scheduler results and on the dashboard —
typically no free faculty in the window, an availability window that does not
overlap any panel's free time, or a daily interview cap that has been reached.
Widen the date range, lower the panel minimum size, or relax the constraint
priority.

**A faculty or student sign-in is refused with 403**
The account is not linked to a faculty/candidate record. Link it on the **User
Accounts** page; the system fails closed rather than showing everything.

**A reschedule is rejected with 409**
That is the conflict check doing its job; `details` lists exactly what clashes.
Choose another slot, or resubmit with `force: true` (the UI's "Apply anyway"),
which saves the change and flags the interview as `CONFLICT`.

**Uploads fail validation**
The response names the sheet, row, column and value. Compare the headers against
**Data / Excel Upload → Accepted columns**, which is generated from the live
mapping.

---

## License

MIT — see [LICENSE](LICENSE).
