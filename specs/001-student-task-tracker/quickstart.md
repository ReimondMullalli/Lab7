# Quickstart: Validate the Student Task Tracker

## Prerequisites

- Docker and Docker Compose
- Node.js 22 LTS (for running tests outside containers)
- A test SMTP inbox (for example Mailpit, included in `docker-compose.yml` for local use)

## Run

```bash
docker compose up --build        # web on :5173, api on :3000, mail inbox on :8025
npm --prefix backend run migrate # create database tables
```

## Validation scenarios

| # | Story | Steps | Expected result |
|---|-------|-------|-----------------|
| 1 | US1 | Register, add course "CIS 285", add 3 assignments with different due dates | Upcoming list shows them sorted by due date |
| 2 | US1 | Mark the earliest assignment done | It moves to Completed |
| 3 | US1 | Add an assignment due yesterday | It shows as overdue |
| 4 | US2 | Set lead time to 1 hour; add an assignment due in 61 minutes | One email in the Mailpit inbox about 1 minute later |
| 5 | US2 | Repeat 4, but mark it done first | No email arrives |
| 6 | US3 | Weights 20/30/50, scores 90/80/70 | Course average shows 77.0% |
| 7 | US4 | Open the calendar for a week with assignments on Monday and Friday | Each appears on its due day in its course color |
| 8 | FR-012 | Sign in as a second student and request the first student's course by ID | 404 Not Found |

## Automated checks

```bash
npm --prefix backend test         # unit + contract tests (see contracts/openapi.yaml)
npm --prefix frontend run test:e2e  # Playwright tests for scenarios 1–7
```

See [data-model.md](./data-model.md) for field rules and [contracts/openapi.yaml](./contracts/openapi.yaml)
for the endpoints these scenarios exercise.
