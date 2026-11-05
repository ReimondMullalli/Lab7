# Implementation Plan: Student Task Tracker

**Branch**: `001-student-task-tracker` | **Date**: 2026-11-05 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/001-student-task-tracker/spec.md`

**Note**: This is the **base architecture** generated with Spec Kit (`/speckit-plan`). An
alternative architecture is in [docs/architecture-alternative.md](../../docs/architecture-alternative.md).

## Summary

Students track courses, assignments, due dates and grades, and receive email reminders before
deadlines. The base architecture is a classic **three-tier web application**: a React single-page
app (frontend) talks to a REST API (backend), which stores data in PostgreSQL. A separate
background worker sends reminder emails from a job queue. Each part is packaged as a Docker
container.

## Technical Context

**Language/Version**: TypeScript 5.x on Node.js 22 LTS (frontend and backend)

**Primary Dependencies**: React 19 + Vite (frontend); Express 5 + Prisma ORM (backend); BullMQ +
Redis (reminder job queue); Nodemailer with an SMTP provider (email)

**Storage**: PostgreSQL 16 (all application data); Redis 7 (job queue only)

**Testing**: Vitest (unit), Supertest (API contract), Playwright (end-to-end)

**Target Platform**: Linux containers; modern desktop and mobile browsers

**Project Type**: Web application (frontend + backend + worker)

**Performance Goals**: Upcoming list in under 2 s with 500 assignments; 1,000 concurrent users

**Constraints**: HTTPS only; reminders sent within 5 minutes of schedule; WCAG 2.1 AA

**Scale/Scope**: Up to 10,000 students; about 12 screens; about 20 API endpoints

## Architecture Overview

```text
 Browser (React SPA)
        │  HTTPS + JSON (REST), session cookie
        ▼
 ┌─────────────────────┐      ┌──────────────┐
 │  API server         │─────▶│ PostgreSQL   │
 │  Express + Prisma   │      │ (all data)   │
 └─────────┬───────────┘      └──────▲───────┘
           │ enqueue reminder jobs   │ read assignment state
           ▼                         │
 ┌─────────────────────┐      ┌──────┴───────┐      ┌──────────────┐
 │  Redis (BullMQ)     │─────▶│ Reminder     │─────▶│ SMTP email   │
 │  delayed job queue  │      │ worker       │      │ provider     │
 └─────────────────────┘      └──────────────┘      └──────────────┘
```

### Components

| Component | Responsibility |
|-----------|----------------|
| **Web client** | React SPA: sign-in, course and assignment screens, list, calendar, grades |
| **API server** | REST endpoints, validation, authentication, authorization, grade math |
| **PostgreSQL** | Source of truth for students, courses, assignments and reminders |
| **Redis + BullMQ** | Delayed jobs: one job per reminder, scheduled at its send time |
| **Reminder worker** | Runs due jobs, re-checks the assignment is still not done, sends email, retries up to 3 times |
| **Email provider** | Delivers reminder and password-reset emails over SMTP |

### Key design decisions

- **Layered backend**: routes → services → repositories (Prisma). Business rules (grade averages,
  overdue status) live in the service layer so they can be unit-tested.
- **Session authentication**: HTTP-only, secure session cookies; passwords hashed with Argon2id.
- **Authorization**: every query is scoped by `studentId` taken from the session (FR-012).
- **Reminders**: when an assignment is created or its due date changes, the API (re)schedules a
  delayed job. Marking it done cancels the job. The worker re-checks state before sending (FR-008).
- **Time zones**: due times are stored in UTC and shown in the student's time zone.

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principle | Status | How the plan satisfies it |
|-----------|--------|---------------------------|
| I. Student-First Simplicity | PASS (justified) | One API, one database. The worker and Redis are justified in Complexity Tracking. |
| II. Spec-Driven Development | PASS | Plan derived from spec.md; every FR maps to an endpoint in contracts/ |
| III. Test-First | PASS | Contract tests per endpoint (Supertest), E2E per user story (Playwright) |
| IV. Data Privacy and Security | PASS | Argon2id hashes, HTTPS, per-student query scoping, account deletion |
| V. CI and Version Control | PASS | GitHub Actions runs lint, unit, contract and E2E tests on every push |

Post-design re-check: **PASS** (no new violations introduced by the data model or contracts).

## Project Structure

### Documentation (this feature)

```text
specs/001-student-task-tracker/
├── plan.md              # This file
├── research.md          # Phase 0 output
├── data-model.md        # Phase 1 output
├── quickstart.md        # Phase 1 output
├── contracts/
│   └── openapi.yaml     # Phase 1 output (REST API contract)
└── checklists/
    └── requirements.md  # Spec quality checklist
```

### Source Code (repository root)

```text
backend/
├── src/
│   ├── api/            # Express routes and request validation
│   ├── services/       # Business rules (assignments, grades, reminders)
│   ├── repositories/   # Prisma data access
│   ├── worker/         # BullMQ reminder worker
│   └── auth/           # Sessions, password hashing
├── prisma/schema.prisma
└── tests/
    ├── contract/
    ├── integration/
    └── unit/

frontend/
├── src/
│   ├── pages/          # Dashboard, Course, Calendar, Settings, Auth
│   ├── components/
│   └── services/       # API client
└── tests/e2e/

docker-compose.yml      # api, worker, web, postgres, redis (+ mailpit locally)
.github/workflows/ci.yml
```

**Structure Decision**: Web application layout (frontend + backend), with the reminder worker in
the backend package so it shares models and services with the API.

## Deployment

- **Local and CI**: `docker compose up` starts all five services.
- **Production**: containers on a single cloud VM or a container service (for example Render or
  AWS ECS), managed PostgreSQL and Redis, and a reverse proxy that terminates TLS.

## Complexity Tracking

| Violation | Why Needed | Simpler Alternative Rejected Because |
|-----------|------------|-------------------------------------|
| Separate worker process + Redis | Reminders must fire on time and retry (SC-003, edge cases) | A timer inside the API process loses jobs on restart and doesn't scale to several API instances |
