# Architecture Review: Base Student Task Tracker

**Reviewed:** 2026-11-05  
**Architecture source:** `specs/001-student-task-tracker/plan.md`  
**Evidence reviewed:** `spec.md`, `research.md`, `data-model.md`, `contracts/openapi.yaml`, and `.specify/memory/constitution.md`

## Executive summary

The base design is a conventional, portable three-tier application: React/Vite, an Express/Prisma REST API, PostgreSQL, and a Redis/BullMQ reminder worker. It is a credible fit for the stated 10,000-student / 1,000-concurrent-user target and offers a clear public API, strong separation of concerns, and good testing seams. Its main weakness is disproportionate operational and infrastructure complexity for a student-scale product, plus several important security and reliability controls that are asserted but not specified.

**Overall assessment: 7.5/10.** A sound, maintainable baseline if portability, a documented API, and precise reminder scheduling matter enough to justify operating multiple services. Before implementation, close the gaps around session/CSRF protection, rate limiting, reminder idempotency and transactional scheduling, API schema completeness, and deployment/monitoring.

## Scores

Scores use 10 as best. For **Operational complexity**, 10 means simplest to operate. For **Main risks**, 10 means risks are well identified and mitigated.

| Category | Score | Assessment |
|---|---:|---|
| Functional requirements and success criteria | 8/10 | All 13 functional requirements have a plausible component or endpoint. The queue-based reminder design can meet SC-003, and the React/API/database stack can meet the UI and performance goals. Some behavior and API details remain underspecified. |
| Constitution compliance | 7/10 | Strong alignment with simplicity rationale, spec-first planning, test layers, hashing, HTTPS, and GitHub Actions. However, test-first/red-green sequencing and task traceability are not demonstrated; the constitution's PR review and red-build merge gate are not explicit. Third-party email processing also needs reconciliation with the no-third-party-sharing rule. |
| Scalability | 8/10 | PostgreSQL, stateless API containers, and a separately scalable worker are appropriate for the stated scale. Indexes, pagination, connection pooling, queue throughput, and horizontal-scaling limits are not documented. |
| Security | 7/10 | Argon2id, secure HTTP-only cookies, HTTPS, and student-scoped queries are good foundations. Missing design details include CSRF defenses, session rotation/expiry/revocation, login and reset rate limits, reset-token lifecycle, input validation, secrets management, audit logging, and defense-in-depth database constraints. |
| Cost | 6/10 | A VM/container service plus managed PostgreSQL and Redis incurs always-on infrastructure and operational overhead. It is likely affordable at the stated scale, but unnecessarily expensive for an early class-project workload compared with a managed/serverless option. No workload-based estimate is supplied. |
| Operational complexity | 5/10 | API, web, worker, PostgreSQL, Redis, reverse proxy/TLS, migrations, backups, monitoring, and deployment coordination create several failure and maintenance points. Docker Compose helps local development but does not remove production operations. |
| Testability | 9/10 | Layered services, an explicit OpenAPI contract, Supertest, Vitest, and Playwright give clear unit, contract, integration, and user-journey boundaries. The plan should still require tests to be written and shown failing before implementation, as the constitution mandates. |
| Vendor lock-in | 9/10 | Docker, Node/TypeScript, PostgreSQL, Redis, SMTP, and a REST API are broadly portable. Prisma and BullMQ create some ecosystem coupling, but neither ties the system to one hosting provider. |
| Main risks (risk control) | 6/10 | The main risks are recognizable, but critical failure modes—duplicate or lost reminders, partial writes during rescheduling, queue recovery, email-provider outages, and session/CSRF abuse—lack explicit controls and acceptance tests. |

## Functional fit against the specification

### Strong coverage

- **FR-001 / FR-002:** Registration, sign-in/out, and password reset are represented in the auth layer and API contract.
- **FR-003 / FR-004 / FR-006:** Course and assignment create/edit/delete operations, completion, and undo are represented by REST routes and the data model.
- **FR-005 / FR-007 / FR-011:** The assignments endpoint supports course/status/date filtering; the client includes list and calendar screens. Due-date ordering and overdue status are defined.
- **FR-008 / FR-009:** BullMQ delayed jobs, reminder preferences, cancellation, retry, and worker-side state re-check provide a credible reminder workflow.
- **FR-010:** Weighted averages are assigned to the service layer and the formula is defined in the data model.
- **FR-012 / FR-013:** Session-derived student scoping and cascading ownership data support isolation and account deletion.

The architecture is also directionally compatible with the measurable criteria: a focused onboarding flow can satisfy SC-001; indexed PostgreSQL queries and a bounded result set can satisfy SC-002; a durable queue can support SC-003; deterministic grade logic can satisfy SC-004; and a simple assignment flow can be usability-tested for SC-005.

### Gaps and ambiguities to resolve

1. **Reminder delivery is not exactly-once by design.** A worker can send an email and crash before marking the reminder sent, causing a retry and duplicate email. Define an idempotency strategy and document the intended at-least-once semantics.
2. **Scheduling updates need atomicity.** Assignment writes and queue changes span PostgreSQL and Redis. A database commit followed by a failed enqueue (or the reverse) can leave a reminder missing or stale. Use a transactional outbox or a recoverable reconciliation process.
3. **Reminder settings changes need complete behavior.** Turning reminders off must cancel pending jobs; turning them on, changing lead time, editing a due date, undoing completion, and changing time zone must all have defined rescheduling behavior.
4. **The OpenAPI contract is incomplete.** Several write operations have no request-body schema (course create/update, assignment patch, profile patch, password reset), and responses often omit response schemas, validation/error formats, and security declarations. The contract also does not specify cookie attributes or CSRF behavior.
5. **Pagination and query limits are absent.** The stated 500-assignment target is modest, but an unbounded assignments response and missing pagination strategy are avoidable performance risks.
6. **SC-005 is a usability outcome, not an architecture property.** It needs a defined usability test protocol and success measurement, not just Playwright coverage.

The core weighted-average example is consistent with the documented formula: (90×20 + 80×30 + 70×50) / 100 = 77.0%.

## Constitution review

- **I. Student-First Simplicity — mostly compliant.** A separate worker and Redis are justified by durable delayed reminders and multiple API instances. However, the plan should quantify why the 10,000-user scope needs Redis/BullMQ rather than a simpler database-backed scheduler, and include the total operational cost in the trade-off.
- **II. Spec-Driven Development — partially evidenced.** The plan references the specification and maps requirements to endpoints, but the required spec → plan → tasks → implementation chain and requirement-to-task traceability are not evidenced in the reviewed plan. Ensure a task artifact maps every FR to implementation and tests.
- **III. Test-First — good test structure, incomplete process evidence.** Vitest, Supertest, and Playwright are appropriate, but naming tools does not establish Red-Green-Refactor. Make failing tests a required pre-implementation checkpoint; include at least one E2E acceptance test per user story and a contract test for every endpoint.
- **IV. Data Privacy and Security — partial compliance pending details.** Password hashing, HTTPS, and student-scoped queries are positive. The plan must explain how SMTP delivery fits the prohibition on sharing personal data with third parties, or record and approve a narrowly scoped processor exception through the constitution's amendment process. Define security controls listed above.
- **V. CI and Version Control — partially evidenced.** GitHub Actions on every push is stated. Feature branches, PR review by one reviewer, and a passing CI build that blocks merge should be explicit pipeline/repository requirements.

The plan's blanket “PASS” should therefore be changed to **conditional / follow-up required** until these items are documented.

## Quality attributes and design observations

### Scalability and performance

The design can scale horizontally at the API and worker tiers, while PostgreSQL remains the system of record. For the target load, this is likely sufficient without microservices. Add indexes for ownership and common list access paths (for example, student/course plus due time and status), pagination, connection pooling, and load tests at the 1,000-concurrent-user target. Verify the under-two-second list target with 500 assignments using production-like data and query plans.

### Security and privacy

Keep the session cookie HTTP-only, Secure, and appropriately SameSite; add CSRF protection for state-changing requests where needed. Rotate session identifiers at authentication and privilege changes, expire and revoke sessions, and rate-limit login/reset endpoints. Password-reset tokens should be single-use, short-lived, stored hashed, and responses should not reveal whether an email exists. Validate ownership on every resource operation, not just list queries. The worker should have least-privilege database access and secrets should be managed outside images and source control.

### Cost and operations

The design has predictable infrastructure but a relatively high minimum footprint: API, web, worker, Redis, PostgreSQL, TLS/reverse proxy, and monitoring. Document a monthly estimate and recovery objectives. Production operations need health checks, structured logs, queue-depth/age alerts, dead-letter or failed-job handling, database backups/restore drills, migrations, and a process to reconcile reminders after outages.

### Testability and lock-in

This is the strongest aspect of the base design. The REST contract enables independent clients and contract testing, and standard infrastructure keeps migration options open. Preserve the OpenAPI document as the source of truth and validate implementation drift in CI.

## Prioritized actions before implementation

1. Complete OpenAPI request/response schemas, standard error responses, cookie/security definitions, and endpoint-by-endpoint contract-test mapping.
2. Specify reminder correctness: transactional outbox or reconciliation, idempotent send handling, retry/backoff, concurrency control, and all preference/state-change transitions.
3. Add a security design for CSRF, session lifecycle, rate limits, reset tokens, secrets, and least privilege.
4. Reconcile third-party email delivery with Principle IV and document any approved exception.
5. Make red-green test-first, task traceability, PR review, and merge-blocking CI explicit.
6. Add query/index/pagination decisions, load-test criteria, observability, backup/restore, and cost estimates.

## Recommendation

Retain the base architecture when a stable REST API for future clients, portability across hosting providers, and explicit control over reminder processing are strategic priorities. It is technically conventional and highly testable, but its infrastructure is heavier than the current product scope demands. Do not treat the current Constitution Check as fully passed until the security, third-party data processing, test-first workflow, and reminder reliability gaps are closed.
