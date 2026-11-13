---
spec_id: 001-persist-tasks-in-postgres-and-manage
title: Persist tasks in Postgres and manage them in the web UI
status: draft
priority: P1
shape: gitops-app
---

# 001 — Persist tasks in Postgres and manage them in the web UI

## Problem

The taskboard12 Task API can create and list tasks (`POST /tasks`, `GET /tasks`), but keeps them in its memory only: every restart or new deployment of the API loses every task, so nobody can rely on the board. And the only way to use it is to call the API directly; the taskboard12 web UI shows nothing about tasks yet. This feature keeps tasks in a Postgres database, so that they survive restarts and deployments, and gives people a page in the web UI to see their tasks and create new ones.

## Stories

As a taskboard12 user, I want the tasks I created to still be there after the system was restarted or updated, so that I can rely on the board as my list of things to do.
**Repos:** taskboard12-api, taskboard12-gitops

As a taskboard12 user, I want to open the web UI and see all tasks with their due dates, so that I know what is open and when it is due without calling the API.
**Repos:** taskboard12-web, taskboard12-api, taskboard12-gitops

As a taskboard12 user, I want to create a task with a title and an optional due date in the web UI, so that I can write down what I have to do without any tool besides the browser.
**Repos:** taskboard12-web, taskboard12-api

## Acceptance criteria

- **AC-001.1** Given tasks were created through `POST /tasks`, when the API is restarted or a new version of it is deployed, then `GET /tasks` returns `200` with the same tasks, each with the same `id`, `title`, `due_date` and `created_at` as returned on creation, in creation order (oldest first).
- **AC-001.2** Given the database cannot be reached by the API, when a client sends `GET /tasks` or `POST /tasks`, then the response is `503` with a JSON body `{"detail": "Task storage is unavailable"}`, no task is created, and the API's readiness check `/ready` answers `503` until the database is reachable again.
- **AC-001.3** Given tasks exist (created through the web UI or the API), when a user opens the web UI's start page, then it shows one entry per task with its title and its due date as `YYYY-MM-DD` (or `No due date` when it has none), oldest first; with no tasks it shows `No tasks yet`.
- **AC-001.4** Given a user is on the web UI's start page, when they enter a title (and optionally pick a due date) and submit the form, then the new task appears at the end of the list without leaving the page, the form is cleared, and `GET /tasks` on the API returns it with the same title and due date.
- **AC-001.5** Given a user is on the web UI's start page, when they submit the form with a title that is empty or whitespace only, or longer than 200 characters after trimming, then no task is created, the form keeps their input and shows `Title is required` or `Title must be at most 200 characters` respectively.
- **AC-001.6** Given the API is unreachable or answers with an error status, when a user opens the start page, then the page still loads and shows `Tasks could not be loaded. Please try again later.` in place of the list.
- **AC-001.7** Given the API is unreachable or answers with an error status, when a user submits the create form, then the form keeps their input, shows `The task could not be saved. Please try again.`, and the list is unchanged.

## Non-goals

- Editing, completing or deleting tasks, in the API or the web UI.
- Filtering, sorting options, search and pagination of the list.
- Users, accounts, per-user task lists, authentication and access control: as in the API so far, anyone who can reach the system may see and create tasks.
- Live updates: tasks created elsewhere appear after the page is reloaded.
- Running the system in staging or prod, database backups and restore.
- Keeping tasks that were created in the API's memory before this feature is deployed (they are lost on that deployment as on every restart today).

## Assumptions

- **A1** The system gets one Postgres database used only by taskboard12-api; the web UI never talks to the database, only to the API (other option: the web UI reads the database directly; affects AC-001.1, AC-001.3).
- **A2** The API contract of `POST /tasks` and `GET /tasks` (fields, validation, `422` errors, creation order) stays as it is today; only where tasks are kept changes (other option: extend the contract in this spec; affects AC-001.1, AC-001.4).
- **A3** A database outage answers `503` with `{"detail": "Task storage is unavailable"}` and makes `/ready` fail (other option: `500`, or `/ready` independent of the database; affects AC-001.2).
- **A4** The task list and the create form are on the web UI's start page `/` (other option: a separate `/tasks` page; affects AC-001.3 to AC-001.7).
- **A5** Due dates are shown as stored, `YYYY-MM-DD`, and tasks without one show `No due date` (other option: a localized date format; affects AC-001.3).
- **A6** The web form checks the title (required, at most 200 characters after trimming) before sending, with the same limits as the API; past due dates are accepted (other option: rely on the API's `422` only; affects AC-001.5).
- **A7** The user-facing texts are English and exactly as written in the criteria (other option: other wording or language; affects AC-001.3, AC-001.5 to AC-001.7).
- **A8** The list shows all tasks, oldest first, without a limit, matching the API's order (other option: newest first; affects AC-001.3, AC-001.4).

## Open questions

## Changelog

- 2026-11-13 created
