# Architecture Review: Alternative Student Task Tracker

**Reviewed:** 2026-11-05  
**Architecture source:** `docs/architecture-alternative.md`  
**Evidence reviewed:** `specs/001-student-task-tracker/spec.md`, the base plan's research and data model, and `.specify/memory/constitution.md`

## Executive summary

The alternative combines Next.js on Vercel with Supabase Auth and PostgreSQL/RLS, using a five-minute scheduled function to send reminders through Resend. It removes the separately operated Express API, Redis, and worker, which is a strong simplicity and time-to-first-version advantage for a product capped at 10,000 students. Database-enforced row-level security is a meaningful defense-in-depth improvement over relying exclusively on application query scoping.

The trade-off is greater dependence on managed-platform behavior and weaker separation between the web application and a reusable public API. The largest technical concern is reminder timeliness: a five-minute polling interval leaves little margin against the 99%-within-five-minutes success criterion, especially when cron execution, cold starts, retries, and provider latency are included.

**Overall assessment: 8.0/10.** This is the better default for a small team prioritizing delivery speed and low operational burden, provided reminder delivery is redesigned or measured to prove SC-003, RLS is exhaustively tested, and the constitution's third-party data-processing rule is addressed.

## Scores

Scores use 10 as best. For **Operational complexity**, 10 means simplest to operate. For **Main risks**, 10 means risks are well identified and mitigated.

| Category | Score | Assessment |
|---|---:|---|
| Functional requirements and success criteria | 7/10 | All FRs have a plausible implementation path. However, five-minute polling does not provide adequate timing margin for a 99%-within-five-minutes target, and some behaviors are only described at a high level. |
| Constitution compliance | 6/10 | Strong simplicity, spec alignment, CI, HTTPS-by-platform, and database-level isolation direction. The no-third-party-sharing principle is a major unresolved issue; test-first red-green and PR workflow are not demonstrated, and replacing endpoint contract tests needs explicit constitutional approval/interpretation. |
| Scalability | 8/10 | Managed Postgres, server-rendered pages, and serverless execution can support the stated scale with less infrastructure management. Cron fan-out, function duration/concurrency limits, database connection management, and plan quotas need validation. |
| Security | 8/10 | Supabase Auth plus RLS can enforce tenant isolation below the application layer. But RLS policy correctness, service-role bypass, cron endpoint authentication, secrets, rate limiting, CSRF, and auth configuration are not specified sufficiently to award a higher score. |
| Cost | 9/10 | Free/low-cost managed tiers make this attractive for a class project and light usage. Costs can rise with database size, function invocations, email volume, preview environments, and paid cron requirements; free-tier assumptions are not a production cost model. |
| Operational complexity | 9/10 | No long-running API, worker, or Redis to patch and monitor. The team still owns schema migrations, RLS, scheduled execution, observability, provider configuration, backups, and platform quota management. |
| Testability | 7/10 | Vitest, pgTAP, and Playwright form a reasonable suite, and database tests are essential for RLS. The loss of an explicit REST contract weakens consumer-facing contract testing; tests for every Server Action and every authorization boundary need a clear inventory. |
| Vendor lock-in | 5/10 | PostgreSQL data is portable, but Supabase Auth/session semantics, RLS conventions, generated client APIs, Vercel deployment/cron, and Server Actions create meaningful platform coupling. Resend is another provider dependency. |
| Main risks (risk control) | 6/10 | The document identifies free-tier limits, RLS mistakes, and cold starts. It does not fully address the narrow reminder SLO margin, service-role bypass, cron overlap/replay, email idempotency, or platform outages and migration paths. |

## Functional fit against the specification

### Strong coverage

- **FR-001 / FR-002:** Supabase Auth supplies email/password registration, sessions, and reset flows.
- **FR-003 – FR-007:** Next.js Server Actions and PostgreSQL operations cover course/assignment management, completion/undo, and filters.
- **FR-008 / FR-009:** A reminders table and scheduled query can find due reminders, re-check assignment status, and honor reminder preferences.
- **FR-010:** A SQL view/function can compute the documented weighted average.
- **FR-011:** A weekly calendar can query assignments within the selected week.
- **FR-012:** RLS can make the database enforce student-level isolation even if application code accidentally omits a filter.
- **FR-013:** Cascading foreign keys from the auth user to profile and owned records can support account deletion.

The managed stack is a good fit for the stated scale, and preview deployments plus database tests can help the team validate changes early.

### Gaps and ambiguities to resolve

1. **SC-003 is not safely demonstrated.** A five-minute polling schedule can consume the entire five-minute delivery allowance before function startup, query execution, retries, and Resend delivery are counted. A cron trigger is not equivalent to an email being delivered within five minutes. Define the measured SLO from scheduled send time to provider acceptance/delivery, then prove it under load and failure conditions. Consider a more frequent supported schedule or a durable event/queue mechanism.
2. **The cron job's execution contract is vague.** Define batch size, pagination, execution timeout, overlap prevention, retry behavior, and recovery after missed invocations. A single long-running batch may exceed function limits.
3. **Email idempotency is missing.** If the function sends an email and fails before marking the row sent, the next run may send it again. Use an atomic claim/lease and idempotency key where supported, with explicit at-least-once semantics.
4. **RLS policy details are absent.** “Every table” is not a policy specification. Define policies for SELECT/INSERT/UPDATE/DELETE on every table, including child-table ownership through course/assignment relationships. Test both allowed and denied cases using non-owner identities.
5. **The service key bypasses RLS.** The cron route's service-role credential is effectively privileged. Isolate it to the server, authenticate cron invocations, restrict its operations, validate all selected rows, and never expose it to client bundles or logs.
6. **The data model changes are incomplete.** Adding `student_id` to every table is useful for simple RLS, but ownership consistency must be enforced (for example, prevent an assignment's student_id from disagreeing with its course's student_id). Define the profile row creation and auth-user deletion flow.
7. **Server Actions are not a complete public API strategy.** They suit the current web app but do not provide the same stable, documented contract for a future mobile client or third-party integration. The architecture notes this trade-off but should decide whether a versioned API is a foreseeable requirement.
8. **SC-005 still needs usability evidence.** Playwright verifies scripted behavior, not whether 90% of test users can complete the task unaided. Add a usability test protocol and measure first-try completion.

The alternative should also specify behavior for turning reminders off/on, changing lead time or due time, undoing completion, and changing time zones. These state transitions must not leave stale pending rows.

## Constitution review

- **I. Student-First Simplicity — compliant in direction.** The managed stack removes several components and is proportionate to the product scope. Avoid adding platform services unless tied to a requirement.
- **II. Spec-Driven Development — partially evidenced.** The document maps functional requirements to implementation components, but does not demonstrate the required spec → plan → tasks → implementation workflow or trace every requirement to a task and test.
- **III. Test-First — partial / interpretation required.** Vitest, pgTAP, and Playwright are sensible. However, the constitution explicitly requires every API endpoint to have a contract test. The alternative has no Express REST API, but Supabase exposes database access APIs and Server Actions are externally callable mutation boundaries. Define a test for every action and data-access policy, and obtain a documented interpretation or constitution amendment rather than silently treating the requirement as inapplicable. Red-Green-Refactor must be explicit.
- **IV. Data Privacy and Security — unresolved conflict.** Supabase stores student account and coursework data; Resend processes email addresses and reminder contents. This is third-party processing and appears to conflict with the literal rule that no personal data is shared with third parties. The design needs an approved, narrowly scoped exception or a constitution change, plus a data-processing/privacy review. RLS and password hashing do not resolve this governance issue.
- **V. CI and Version Control — partially evidenced.** GitHub Actions and preview deployments are described, but feature-branch/PR review requirements and a passing CI build that blocks merge should be explicit. Ensure database/RLS tests run on every push and before merge.

The alternative's Constitution Check overstates the evidence by marking all principles PASS. It should be recorded as **conditional / unresolved** until the third-party processing and test-first/contract-test questions are settled.

## Quality attributes and design observations

### Scalability and performance

Managed PostgreSQL is a sensible core for this relational workload. Server rendering may improve first-load experience, but SC-002 is specifically about the upcoming list for 500 assignments; benchmark the actual query and rendered page rather than assuming server rendering guarantees the target. Add indexes for student ownership, course, due time, and reminder status/send time. Validate connection pooling and function concurrency against the target of 1,000 active users. Ensure scheduled jobs process bounded batches.

### Security and privacy

RLS is a strong architectural advantage only if policies are complete, tested, and enabled on every exposed table. Use deny-by-default policies and test cross-student reads and writes, including attempts to forge student IDs or attach assignments to another student's course. Keep the service-role key server-only, authenticate the cron route, and use least privilege for all other credentials. Add rate limits and abuse controls for authentication and password reset. Configure secure session/cookie behavior and CSRF protections for state-changing actions. Establish how Supabase and Resend handle, retain, and protect student data.

### Cost and operations

This option has the lowest initial operating burden and likely lowest cost at class-project scale. However, the “free tier covers a class project” claim is conditional: inactivity pauses, cron frequency, database branching, usage quotas, and email volume can force upgrades or workarounds. Forecast the cost at expected active-user and reminder volumes, including production and preview environments. Add alerts for failed cron invocations, old pending reminders, function errors, email failures, database saturation, and quota exhaustion. Define backup/restore and incident recovery responsibilities.

### Testability and lock-in

Database-level RLS tests are mandatory, not optional. Build a test matrix for every table and operation with owner, non-owner, unauthenticated, and privileged-service contexts. Add Server Action tests for validation and business rules, and E2E tests for each user story. Preserve SQL migrations and avoid burying business logic in platform-specific UI actions. To reduce lock-in, isolate Supabase Auth and Resend behind small adapters, keep schema/migrations portable PostgreSQL, and document an exit path from Vercel Cron (for example, an external scheduler invoking a protected endpoint).

## Prioritized actions before implementation

1. Rework and load-test reminder scheduling so the measured SC-003 target has real margin; specify batching, retries, idempotency, overlap protection, and missed-run recovery.
2. Write explicit RLS policies for every table and operation; add pgTAP tests proving both permitted and denied access across multiple student identities.
3. Harden the cron route and service-role handling: authenticated invocation, server-only secret, least privilege, and audit/operational logging.
4. Resolve Principle IV's prohibition on third-party personal-data sharing for Supabase and Resend through the constitution's formal governance process.
5. Define the test-first workflow and a complete action/policy/E2E test inventory; document how the no-REST design satisfies or changes the endpoint contract-test rule.
6. Complete ownership constraints, profile provisioning, deletion behavior, and reminder state transitions in SQL migrations.
7. Validate platform quotas, cost at expected usage, connection pooling, observability, backups, and a provider exit plan.

## Recommendation

Prefer the alternative for the current product scope if rapid delivery and low operational burden are the primary goals. It is simpler to deploy and operate, and RLS offers a strong isolation boundary. However, do not ship the five-minute reminder design without evidence that SC-003 is met with margin, and do not accept the stated Constitution Check as passed while third-party processing conflicts with Principle IV. If a portable public API, precise reminder scheduling, or reduced platform dependence becomes a priority, the base architecture is the stronger foundation.
