---
plan_id: 001-persist-tasks-in-postgres-and-manage
spec: 001-persist-tasks-in-postgres-and-manage
feature: Persist tasks in Postgres and manage them in the web UI
gitops_app: ika100-demo/taskboard12-gitops
status: draft
repos:
  - id: taskboard12-gitops
    shape: gitops-app
    summary: Declare the Postgres database for taskboard12-api and wire the API address into taskboard12-web
    acs: [AC-001.1, AC-001.3]
    order: before-merge
    gitops:
      - {resource: postgres, type: database}
      - {uses: postgres, component: taskboard12-api}
      - {env: {TASKBOARD12_API_URL: 'http://taskboard12-api'}, component: taskboard12-web}
    depends_on: []
    done: false
  - id: taskboard12-api
    shape: service-python
    summary: Keep tasks in Postgres and answer 503 while the database is unreachable
    acs: [AC-001.1, AC-001.2]
    criteria:
      - "Given tasks A, B and C were created through `POST /tasks` in that order, when the service is restarted (a new process with the same `DATABASE_URL`), then `GET /tasks` returns `200` with A, B and C in creation order (oldest first), each with the same `id`, `title`, `due_date` and `created_at` as returned on creation (system AC-001.1) (replaces AC-001.7)"
      - "Given the service starts against an empty Postgres database (no tables), when a client sends `GET /tasks`, then the response is `200` with `[]`, and a following `POST /tasks` with `{\"title\": \"Buy milk\", \"due_date\": \"2026-12-01\"}` answers `201` and is listed by `GET /tasks` (system AC-001.1)"
      - "Given tasks are kept in Postgres, when a client sends `POST /tasks` with a title of 200 characters surrounded by whitespace, one with a due date and one without, then `GET /tasks` returns them with the trimmed title, `due_date` as `YYYY-MM-DD` or `null`, and `created_at` as an ISO 8601 UTC timestamp, unchanged after a restart (system AC-001.1)"
      - "Given the database cannot be reached, when a client sends `GET /tasks`, then the response is `503` with exactly the JSON body `{\"detail\": \"Task storage is unavailable\"}` within 5 seconds (system AC-001.2)"
      - "Given the database cannot be reached, when a client sends a valid `POST /tasks`, then the response is `503` with `{\"detail\": \"Task storage is unavailable\"}`, and once the database is reachable again `GET /tasks` does not contain that task; an invalid body still answers `422` as before (system AC-001.2)"
      - "Given the database cannot be reached, when the readiness check `GET /ready` is called, then it answers `503` while `GET /health` keeps answering `200`; once the database is reachable again, `/ready` answers `200` and `GET /tasks` answers `200` without a restart of the service (system AC-001.2)"
    arguments: |
      Connection from DATABASE_URL (injected by the gitops entry with PG*); the service creates its table itself at
      startup if missing, and a database that is down at startup must not crash the process (/health stays 200,
      /ready 503). Creation order = ORDER BY created_at, then an insertion sequence as tie-break.
    depends_on: []
    done: false
  - id: taskboard12-web
    shape: web-nextjs
    summary: Task list and create form on the start page, backed by the Task API
    acs: [AC-001.3, AC-001.4, AC-001.5, AC-001.6, AC-001.7]
    criteria:
      - "Given the Task API's `GET /tasks` returns tasks, when a user opens `/`, then the page shows one entry per task in the order the API returns them (oldest first), each with its `title` and its `due_date` as `YYYY-MM-DD`, or `No due date` when `due_date` is `null` (system AC-001.3)"
      - "Given the Task API's `GET /tasks` returns `[]`, when a user opens `/`, then the page shows `No tasks yet` (system AC-001.3)"
      - "Given a user is on `/`, when they enter a title, optionally pick a due date and submit the form, then the web sends `POST /tasks` with `{\"title\": <trimmed title>, \"due_date\": <YYYY-MM-DD or null>}` to the Task API, the returned task appears at the end of the list without leaving the page, and the form is cleared (system AC-001.4)"
      - "Given a user is on `/`, when they submit the form with an empty or whitespace-only title, then no request is sent to the Task API, the form keeps their input and shows `Title is required` (system AC-001.5)"
      - "Given a user is on `/`, when they submit a title longer than 200 characters after trimming, then no request is sent to the Task API, the form keeps their input and shows `Title must be at most 200 characters`; a title of exactly 200 characters after trimming is sent; a past due date is sent as well (system AC-001.5)"
      - "Given the Task API is unreachable, slower than 5 seconds, or answers `GET /tasks` with any status other than `200` (including `503`), when a user opens `/`, then the page loads with status `200` and shows `Tasks could not be loaded. Please try again later.` in place of the list (system AC-001.6)"
      - "Given the Task API is unreachable, slower than 5 seconds, or answers `POST /tasks` with any status other than `201` (including `422` and `503`), when a user submits the form, then the form keeps their input, shows `The task could not be saved. Please try again.`, and the list is unchanged (system AC-001.7)"
    arguments: |
      The API base URL comes from the server-side env var TASKBOARD12_API_URL (set by the gitops entry to
      http://taskboard12-api; document it in docs/env-vars.md). Only the Next.js server calls the API (Server
      Component for the list, Server Action or route handler for the create); the browser never calls the API
      directly, so no CORS and no NEXT_PUBLIC_ variable. /api/ready stays independent of the API.
      Tests mock the Task API per the plan's Contract.
    depends_on: []
    done: false
gitops_pin: []
---

# Plan 001 — Persist tasks in Postgres and manage them in the web UI

Three entries in two levels:

| Level | Repo | Criteria |
|---|---|---|
| 0 (before merge) | taskboard12-gitops | AC-001.1, AC-001.3 |
| 1 (in parallel) | taskboard12-api | AC-001.1, AC-001.2 |
| 1 (in parallel) | taskboard12-web | AC-001.3, AC-001.4, AC-001.5, AC-001.6, AC-001.7 |

The API and the web share only the HTTP contract below, which already exists in the API (spec A2 keeps it unchanged) and is extended by the `503` answer only; so the web needs no `depends_on` on the API and both build in parallel against the contract. The web never touches the database (spec A1).

## taskboard12-gitops — Declare the Postgres database for taskboard12-api and wire the API address into taskboard12-web

**Shape:** `gitops-app` · **Criteria:** AC-001.1, AC-001.3 · **Depends on:** []

Tasks can only survive a restart (AC-001.1) when the system has a database and the API gets its credentials; the web can only show tasks (AC-001.3) when it knows the API's address. Operations, in order:

1. `{resource: postgres, type: database}`: the system's first resource, a CloudNativePG cluster (the platform renders the operator into `operators/`; `/system:ship` applies `operators/root.yaml` once on the local cluster).
2. `{uses: postgres, component: taskboard12-api}`: injects `DATABASE_URL` and `PG*` into the API.
3. `{env: {TASKBOARD12_API_URL: 'http://taskboard12-api'}, component: taskboard12-web}`: the API's in-namespace Service on port 80.

No replicas or resources change: the spec states no instance setting.

## taskboard12-api — Keep tasks in Postgres and answer 503 while the database is unreachable

**Shape:** `service-python` · **Criteria:** AC-001.1, AC-001.2 · **Depends on:** []

The API replaces its in-memory store with a Postgres-backed one (AC-001.1) and maps every storage failure to `503` with the fixed body, failing `/ready` until the database is back (AC-001.2). Its first criterion supersedes AC-001.7 of its spec `001-create-and-list-tasks-in-memory` (a restart empties the list). The rest of that spec (fields, `201`, `422` validation, trimming, creation order) stays as is and keeps its tests.

## taskboard12-web — Task list and create form on the start page, backed by the Task API

**Shape:** `web-nextjs` · **Criteria:** AC-001.3 to AC-001.7 · **Depends on:** []

The start page `/` (spec A4) replaces the template's content with the task list and the create form. The list is rendered server-side from `GET /tasks`; the form validates the title on its own (spec A6) and then creates through the Next.js server. All user-facing texts are exactly those of the spec (A7).

## Contract

Caller: the taskboard12-web server (never the browser). Base URL: env `TASKBOARD12_API_URL` (dev: `http://taskboard12-api`). JSON, UTF-8, no authentication.

**Task object** (unchanged):

| Field | Type | Notes |
|---|---|---|
| `id` | string | opaque, server-generated, non-empty |
| `title` | string | trimmed, 1 to 200 characters |
| `due_date` | string `YYYY-MM-DD` or `null` | calendar date, past dates allowed |
| `created_at` | string | ISO 8601 UTC timestamp |

**`GET /tasks`**
- `200`: JSON array of all task objects, creation order (oldest first), no limit, no pagination; `[]` when none.
- `503`: `{"detail": "Task storage is unavailable"}` when the database cannot be reached.

**`POST /tasks`**
- Request: `{"title": string, "due_date": "YYYY-MM-DD" | null}`; `due_date` may be omitted; unknown fields are ignored.
- `201`: the created task object.
- `422`: FastAPI validation error (`{"detail": [{"loc": ["body", "title"|"due_date"], "msg": ..., "type": ...}]}`) for a title missing, null, empty or whitespace only, longer than 200 characters after trimming, or an invalid `due_date`. Validation happens before storage: an invalid body answers `422` even while the database is down.
- `503`: `{"detail": "Task storage is unavailable"}`; no task is created.

**`GET /ready`** (Kubernetes readiness probe): `200` `{"status": "ready"}` when the database answers, else `503`. `GET /health` stays `200` regardless of the database.

No events.

### Errors

- Error format: FastAPI's `{"detail": ...}`; `detail` is a string for `503` (exactly `Task storage is unavailable`) and a list of validation entries for `422`.
- The API answers `503` for every storage failure (connection refused, timeout, lost connection) and never `500` for one; an unexpected exception elsewhere is `500` with FastAPI's default body.
- The web treats any response it does not expect as a failure, without parsing the body: on `GET /tasks` anything but `200` with a JSON array (including `422`, `500`, `503`, a non-JSON body, a connection error) shows `Tasks could not be loaded. Please try again later.`; on `POST /tasks` anything but `201` with a task object shows `The task could not be saved. Please try again.` and keeps the input. A `422` should not occur because the web checks the same limits first; if it does, it is the same save failure. The web does not retry.

### Timeouts

- Web to API, each call (`GET /tasks`, `POST /tasks`): 5 seconds total (`AbortSignal.timeout(5000)`); on timeout or unreachable API, the failure behaviour of `### Errors` (page still loads, form keeps input). A `POST` that timed out may still have created the task; it then appears on the next page load (no live updates, per non-goals).
- API to Postgres: connect timeout 3 seconds, statement timeout 2 seconds, so a `GET`/`POST /tasks` with the database down answers `503` within 5 seconds, inside the web's budget. The connection is re-established on the next request after an outage, without a restart.
- `/ready`'s database check: at most 800 ms (the probe's own timeout is Kubernetes' default 1 second); no answer in time counts as unreachable (`503`).

## gitops-app PR

The gitops entry (database, `uses`, web env) lands before the component PRs merge; `/system:ship` pins both builds in dev on `compose/001-persist-tasks-in-postgres-and-manage`, in one PR. No `gitops_pin` for staging or prod: running the system in staging or prod is a non-goal of the spec, and both components have only a `dev` instance.

Verify after Argo reconciles dev:
- the CloudNativePG cluster `postgres` is healthy and the API pod has `DATABASE_URL`; `GET http://taskboard12-api.taskboard12-dev.localhost:8088/ready` answers `200`;
- a task created with `POST /tasks` is still returned by `GET /tasks` after the API pod is deleted and recreated;
- `http://taskboard12-web.taskboard12-dev.localhost:8088/` lists the tasks and creates one through the form; both repos' smoke tests pass.
