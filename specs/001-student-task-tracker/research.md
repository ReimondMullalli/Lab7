# Research: Student Task Tracker (Phase 0)

## 1. Application style

- **Decision**: Single-page app (React) + REST API (Express) + PostgreSQL.
- **Rationale**: A well-known, well-documented stack. The frontend and backend can be built and
  tested separately, and the REST contract makes contract testing straightforward.
- **Alternatives considered**: Server-rendered pages (fewer moving parts but a less interactive
  calendar); Backend-as-a-Service such as Supabase or Firebase (faster to build, more vendor
  lock-in; see the alternative architecture document).

## 2. Database

- **Decision**: PostgreSQL 16 with the Prisma ORM.
- **Rationale**: The data is relational (student → course → assignment), needs foreign keys and
  cascading deletes, and needs date queries. Prisma provides typed queries and migrations.
- **Alternatives considered**: MongoDB (weaker fit for relational data); SQLite (no concurrent
  writers across several instances).

## 3. Scheduled reminders

- **Decision**: BullMQ delayed jobs on Redis, run by a separate worker process.
- **Rationale**: Jobs survive restarts, retry automatically with backoff, and can be cancelled when
  an assignment is marked done.
- **Alternatives considered**: A cron job polling the database every minute (simpler, but up to
  60 s late and puts more load on the database); in-process timers (lost on restart).

## 4. Authentication

- **Decision**: Email and password with server sessions stored in PostgreSQL; Argon2id hashing.
- **Rationale**: Meets the constitution's security rules without third-party identity services.
  HTTP-only cookies are safer against XSS than storing a token in the browser.
- **Alternatives considered**: JWT in localStorage (XSS risk); university SSO (out of scope for v1).

## 5. Email delivery

- **Decision**: Nodemailer over SMTP to a transactional email provider.
- **Rationale**: The provider can be swapped by changing configuration only.
- **Alternatives considered**: A provider-specific SDK (more lock-in).

## 6. Testing and CI

- **Decision**: Vitest for unit tests, Supertest for API contract tests, Playwright for E2E tests,
  all run in GitHub Actions on every push.
- **Rationale**: Meets constitution principles III and V; all tools work with TypeScript.
- **Alternatives considered**: Jest (slower with ESM); Cypress (Playwright has better
  multi-browser support).

## 7. Deployment

- **Decision**: Docker containers, with managed PostgreSQL and Redis.
- **Rationale**: The same images run locally, in CI and in production.
- **Alternatives considered**: Serverless functions (cheaper when idle; see the alternative
  architecture document).
