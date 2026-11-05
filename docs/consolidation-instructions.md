# Consolidation Instructions for Claude Code

**Date:** 2026-11-05  
**Repository:** `ReimondMullalli/lab7`  
**Decision:** Adopt the alternative serverless architecture, subject to the conditions in `docs/decision.md`.

## Mission

Produce a coherent, implementation-ready consolidated specification and design for the Student Task Tracker. Treat the alternative architecture in `docs/architecture-alternative.md` as the architectural baseline. Use the base Spec Kit documents and both architecture reviews as source material. Do not merely concatenate the documents: resolve contradictions, close the identified gaps, and make every requirement, design decision, interface, and test traceable.

Create exactly these deliverables under `docs/consolidated/`:

1. `docs/consolidated/requirements.md`
2. `docs/consolidated/architecture.md`
3. `docs/consolidated/design.md`

Also update cross-references in the new documents so they point to the source documents and decision. Do not overwrite the original source documents or the two review reports. Do not implement application code in this task.

## Source documents to read

Read the complete current versions of:

- `docs/decision.md`
- `docs/architecture-alternative.md`
- `docs/reviews/review-alternative.md`
- `docs/reviews/review-base.md`
- `specs/001-student-task-tracker/spec.md`
- `specs/001-student-task-tracker/plan.md`
- `specs/001-student-task-tracker/research.md`
- `specs/001-student-task-tracker/data-model.md`
- `specs/001-student-task-tracker/contracts/openapi.yaml`
- `.specify/memory/constitution.md`

If any source differs from the review's description, use the repository's current source and document the discrepancy.

## 1. What to keep from the chosen alternative

Keep these choices as the default architecture:

- **Next.js App Router** as the single full-stack web application, using server-rendered pages where useful and client components only for interactive UI.
- **Vercel** as the initial application hosting/deployment platform.
- **Supabase Auth** for email/password registration, sign-in/out, sessions, and password-reset workflows.
- **Supabase-managed PostgreSQL** as the system of record.
- **PostgreSQL Row-Level Security (RLS)** as the mandatory database enforcement boundary for student data isolation. Application-level ownership checks remain useful defense in depth, not a replacement for RLS.
- **SQL migrations, constraints, indexes, and database views/functions** for schema evolution, integrity, overdue/list queries, and weighted averages where these are simpler and testable.
- **Typed Next.js Server Actions** for web mutations and server-side application workflows. Keep the UI and database access in clear modules rather than scattering SQL or business rules through page components.
- **Scheduled reminder processing** rather than introducing a continuously running worker and Redis by default.
- **A transactional email API/provider** behind a small provider adapter, so provider-specific details do not spread through business logic.
- **Vitest, Playwright, GitHub Actions, and database/RLS tests** as the core quality toolchain.
- **Preview deployments** where supported and cost-appropriate.

Do not reintroduce Express, a separate API container, Redis, BullMQ, or a dedicated worker just because they exist in the base design. They are alternatives, not defaults.

## 2. Improvements to fold in from both reviews and the base architecture

### Reminder correctness (highest priority)

The alternative's five-minute polling interval is not acceptable as the sole design justification for SC-003. A five-minute schedule consumes the entire five-minute delivery allowance before execution and provider latency are considered.

Design a reminder mechanism that has demonstrable timing margin. Prefer a supported scheduler/trigger that runs more frequently than every five minutes (target one-minute dispatch intervals where the chosen plan/platform supports it), or another low-cost durable scheduling mechanism compatible with Supabase. Verify current platform/plan limits rather than assuming a feature or cadence is available. If the required cadence needs a paid tier, state the expected plan and cost; do not claim the free tier meets the SLO without evidence.

The design must include:

- Store the intended `send_at` timestamp in UTC and calculate it from the assignment due time and student's configured lead time.
- A bounded, indexed query for due pending reminders, with batch size and pagination/continuation behavior.
- Safe concurrent execution: atomically claim rows or lease them so overlapping scheduled invocations cannot process the same reminder concurrently.
- A retry policy with a maximum of three delivery attempts, explicit backoff, and clear terminal failure state.
- Duplicate prevention/idempotency. Explain the unavoidable boundary between an external email send and a database commit; use provider idempotency if available, otherwise define at-least-once semantics, stable message identifiers, claim/lease recovery, and how duplicate risk is minimized. Do not promise exactly-once delivery.
- Recovery after missed invocations, function timeouts, provider outage, and expired processing leases.
- Re-check that the assignment is not done and reminders are enabled immediately before sending.
- Correct handling of create, due-date edit, course/assignment deletion, mark done, undo, reminders on/off, lead-time changes, and time-zone changes.
- A reconciliation process that can find stale or missing reminder rows.
- Observability: oldest pending reminder age, scheduled-versus-actual send delay, attempts, failures, queue/backlog size, cron invocation failures, and provider errors.
- A load/failure test proving SC-003 at the target workload. Measure from scheduled send time through provider acceptance or delivery; define which timestamp is observable and avoid claiming inbox receipt if the provider cannot verify it.

Use a transactional database operation for reminder state transitions. Because the database and email API cannot share one atomic transaction, document the delivery state machine and failure semantics explicitly. Do not rely on an in-memory timer.

### Security and privacy

Fold in the strongest controls from the base review while retaining RLS:

- Enable RLS on every table exposed through Supabase APIs; use deny-by-default policies.
- Specify SELECT, INSERT, UPDATE, and DELETE policies as applicable for each table. Derive child-row ownership through trusted foreign-key relationships and/or a consistent `student_id` design with database constraints preventing ownership mismatches.
- Test owner, non-owner, unauthenticated, and privileged server contexts. Include negative tests for forged IDs, cross-student course/assignment references, and unauthorized updates/deletes.
- Keep the Supabase service-role key server-only, out of client bundles, browser responses, logs, and source control. Use it only for the minimum privileged scheduled task path. Authenticate scheduled invocations and restrict their permitted operation.
- Add authentication/reset abuse controls and rate limits where supported. Avoid account-enumeration responses. Specify session and cookie security settings, CSRF protections for mutation paths, and secure handling of redirects and return URLs.
- Validate all inputs on the server, enforce database constraints, and return safe structured errors without leaking secrets or other students' data.
- Store secrets in platform secret management, rotate them, and separate development, preview, and production credentials.
- Define backup/restore, deletion, retention, and incident response expectations appropriate to the project.

### Third-party processor issue — do not gloss over it

The constitution currently says no personal data is shared with third parties. The chosen design necessarily uses third-party hosting/authentication/database services and an email processor. RLS and password hashing do not resolve that governance conflict.

The consolidated documents must:

1. Identify the data Supabase processes (including email, account identifiers, course/assignment content, timestamps, and authentication metadata) and the data the email provider processes (recipient address and message content/metadata).
2. Minimize what is sent to the email provider. Send only the recipient address and the minimum reminder content needed; do not include grades, notes, or unrelated course data in email.
3. Require review/approval of the providers' data-processing terms, retention, subprocessors, region/residency, security controls, breach terms, and deletion behavior before production use.
4. Prohibit sale, advertising use, or unrelated secondary use of student data; document purpose limitation and retention/deletion periods.
5. Record an explicit, narrowly scoped processor exception or propose a formal amendment to Principle IV through the constitution's PR governance process. Do not edit the constitution silently and do not label the architecture fully compliant until that approval is recorded.
6. Use synthetic data and local email capture (such as Mailpit) for local development and tests. Production processing must remain gated on approval.

If the project owner has not formally approved the constitution amendment/exception, mark it as a blocking governance decision in the documents, not as resolved.

### Test-first and traceability

Adopt the base architecture's clear testing boundaries, adapted to the serverless stack:

- Follow Red-Green-Refactor: write a test, observe it fail, then implement.
- Every user story must have at least one Playwright end-to-end acceptance test.
- Every Server Action/mutation boundary must have validation, authorization, and behavior tests. Since the chosen architecture does not define a custom REST API, explain how these action tests satisfy the intent of the interface-contract requirement; do not silently claim there are REST endpoints.
- Every RLS policy/table operation must have database tests (pgTAP or an equivalent repeatable SQL test suite), including negative cross-tenant tests.
- Test SQL views/functions and grade calculations, including the 90/80/70 weighted example yielding 77.0%.
- Include reminder timing, retries, duplicate/replay, concurrent invocation, missed-run recovery, and preference/state-transition tests.
- CI on every push must run lint, type checks, unit tests, database/migration/RLS tests, and relevant E2E tests. A failing required check blocks merge.
- Require feature branches, pull requests, at least one review, and passing CI before merge, consistent with the constitution.
- Distinguish automated UI-flow tests from the SC-005 usability study: scripted Playwright tests do not prove that 90% of real test users succeed unaided.

### Improvements from the base architecture to retain selectively

Use these ideas without importing the base stack wholesale:

- The base's explicit REST/OpenAPI contract is valuable documentation. Keep the selected architecture's Server Actions, but define typed, versionable input/output contracts for every action in `design.md`. Use shared validation schemas (for example, Zod) and stable error shapes. Document the authentication and authorization preconditions for each action.
- Do not create a public REST API solely to imitate the base architecture. If a mobile client or external integration becomes an actual requirement, define a versioned API as a separate future decision. Record that Server Actions are web-app interfaces, not a stable public API.
- Keep the base's separation of presentation, application/business logic, and persistence. Put grade calculation, reminder scheduling rules, and ownership checks in testable server-side modules/database functions rather than UI components.
- Keep the base's UTC storage and student-local display approach for due dates and reminders.
- Keep the base's portable PostgreSQL schema and migrations, and isolate email-provider-specific behavior behind an adapter.
- Keep its explicit operational practices (health/cron monitoring, backups, restore drills, secrets, migrations, and load testing), but tailor them to managed services.

## 3. Required changes to the requirements

Create `docs/consolidated/requirements.md` as the authoritative consolidated requirements document. Start from `specs/001-student-task-tracker/spec.md`, preserving its student-focused scope and priorities unless a change below is required.

### Preserve the core functional scope

Retain all 13 functional requirements and the four user stories:

- Accounts: registration, sign-in/out, password reset.
- Course create/rename/delete.
- Assignment create/edit/delete, title, course, due date/time, notes, optional weight and score.
- Upcoming list sorted by due date, overdue marking, completion and undo, course/status filtering.
- Configurable email reminders (1 hour to 7 days, default 24 hours), and no reminder for done assignments or when reminders are disabled.
- Weighted course averages from graded assignments only.
- Weekly calendar colored by course.
- Strict per-student data isolation.
- Account deletion and cascading deletion of all associated data.

Retain the existing edge cases: past due dates, course deletion with assignments, weights over 100%, reminder failures, time-zone changes, and concurrent edits/last-save behavior. Clarify ambiguous behavior rather than dropping it.

### Clarify and strengthen requirements

- Make reminder timing measurable: define SC-003 as 99% of reminders accepted by the email provider (or delivered, if provider telemetry reliably supports it) within five minutes of `send_at`, over a stated measurement window and under a stated expected load. State the exact start/end timestamps and how failed provider attempts are counted. The architecture must target a dispatch interval shorter than five minutes to leave execution/provider margin.
- Add explicit reminder reliability requirements for no send after completion/disable, bounded retries (maximum three attempts), idempotent processing/replay protection, missed-run recovery, and visibility of terminal failures.
- Add an authorization requirement that all data operations enforce student ownership at the database RLS layer as well as the server action layer.
- Add requirements for safe scheduled-job invocation and service-role secret isolation.
- Add privacy requirements covering processor approval, data minimization, retention/deletion, and no advertising/sale/secondary use. Mark production use blocked pending the Principle IV governance decision.
- Add requirements for structured validation/errors and accessible feedback.
- Preserve the page/list performance target: under two seconds for 500 assignments. Define the test environment, representative data, and percentile (recommend p95) so it can be verified.
- Preserve the onboarding target (first course and assignment in under three minutes) and completion usability target (90% first-try success). Define the test protocol/sample and measure the usability target with participants, not just automated tests.
- Preserve the 10,000 registered / 1,000 concurrent active-user planning scope. Clarify that it is a design/test target, not a claim of proven capacity until load-tested.
- Keep WCAG 2.1 AA as the accessibility target.
- Specify supported browser/mobile viewport assumptions and time-zone handling (IANA zone, UTC persistence, daylight-saving transitions).
- Clarify the weighted-average rule when weights total more or less than 100%, and when no assignments are graded. Keep the documented weighted formula and label the average as based only on graded assignments.
- Define status/filter semantics so overdue is derived from an upcoming assignment whose due time has passed; it is not a separate persisted assignment status unless the design justifies otherwise.

### Requirement traceability

Include a table mapping every FR and SC to:
- the user-visible behavior or system control,
- the responsible component (page/action/database/RLS/scheduler/provider),
- and at least one automated test or measurable validation method.

Do not claim a requirement is satisfied merely because a technology has a feature that could support it.

## 4. Required architecture document

Create `docs/consolidated/architecture.md`.

It must explain:

- Context, goals, non-goals, assumptions, and scale.
- A clear component diagram and end-to-end flows for sign-up, assignment CRUD, list/calendar/grade reads, reminder scheduling/delivery, password reset, and account deletion.
- Why Next.js + Vercel + Supabase + scheduled processing + email provider is the selected design.
- Which decisions are intentionally not adopted from the base architecture (Express, Redis, BullMQ, separately operated worker) and why.
- Clear trust boundaries: browser, Next.js server, Supabase Auth, PostgreSQL/RLS, scheduled invocation, email provider.
- Authentication, authorization, RLS, service-role handling, secrets, privacy/processor governance.
- Reminder SLO design, timing margin, claim/lease, retries, idempotency semantics, recovery, and monitoring.
- Scalability and performance plan for 1,000 concurrent users and 500-assignment lists, including query/index/pooling/batch considerations.
- Cost model with assumptions and expected paid-tier dependencies. Do not invent exact current vendor prices; identify usage drivers and require verification against current plan/pricing documentation.
- Operational responsibilities despite managed hosting: migrations, monitoring, backups, restore tests, incident handling, quotas, provider failures, and environment separation.
- Test strategy, CI gates, deployment/rollback approach, and accessibility.
- Vendor lock-in and exit strategy: portable PostgreSQL migrations/data export, provider adapter, auth migration considerations, scheduler replacement, and Server Actions not being a public API.
- Risks, mitigations, and unresolved decisions, including the third-party processor governance gate.

Use Mermaid if helpful, but ensure diagrams are valid and labels are readable.

## 5. Required design document

Create `docs/consolidated/design.md` with implementation-level detail sufficient for a coding agent to implement consistently.

### Data model and database integrity

Start with the original Student/Course/Assignment/Reminder entities. Decide and document whether `student_id` is stored on each tenant-owned table for simple RLS. If it is, enforce consistency with composite foreign keys or equivalent database constraints so an assignment cannot reference a course owned by another student. Include:

- Student profile linked to Supabase `auth.users.id`; no local password hash.
- Course ownership, name/code/color/term constraints.
- Assignment title, due timestamp UTC, notes, status, weight/score constraints, completion timestamp.
- Reminder schedule timestamp, status, attempt count, next retry/lease fields, provider message/idempotency identifier where applicable, last error safe summary, sent timestamp.
- Indexes for owner/course/due-time/status list queries and reminder due scans.
- Cascading deletion behavior and cleanup of auth-linked data.
- Unique/foreign-key/check constraints and migration order.
- Whether reminder preferences live on profile or a dedicated settings row.
- Any audit/event data needed for reminder reliability, without collecting unnecessary personal data.

Define the weighted-average calculation precisely, including null when no graded weighted assignments exist, and behavior for weights that sum above/below 100%.

### RLS policy matrix

Provide a matrix for each table and each operation (SELECT/INSERT/UPDATE/DELETE):
- authenticated owner,
- authenticated non-owner,
- unauthenticated,
- trusted scheduled server context.

State the actual policy predicate/relationship logic or precise pseudocode, including how child ownership is verified. Explicitly explain the service-role bypass and ensure only the trusted server path can use it. Include tests for every policy and negative case.

### Server Action / interface contracts

Document every action needed for the functional requirements. For each, include:
- stable action name,
- authenticated/unauthenticated precondition,
- input schema and validation,
- successful output shape,
- expected validation/not-found/unauthorized/conflict error shape,
- side effects and transaction boundaries,
- required unit/integration/security tests.

At minimum cover registration/sign-in/out/password reset via Supabase Auth; profile and reminder settings; course CRUD; assignment CRUD and completion/undo; upcoming/filter/calendar reads; course average; account deletion; and protected scheduled reminder dispatch/reconciliation.

Use shared runtime schemas and typed TypeScript contracts. Do not expose secrets or password material in action responses. Explicitly state that Server Actions are internal web interfaces, not a versioned public API. Include an OpenAPI contract only if a genuine external API requirement is identified; otherwise record the base OpenAPI file as useful reference material, not an endpoint set to implement.

### Reminder state machine and workflows

Specify valid state transitions, including pending → processing/leased → sent/failed/cancelled and lease expiry/retry recovery. Explain race handling when a student marks an assignment done while a reminder is being claimed or sent. State the strongest achievable guarantee and residual duplicate risk.

### Deployment and environments

Specify:
- Vercel app deployment and Supabase project/database setup.
- Separate development, preview, and production projects/credentials; avoid sharing production data with preview.
- Supabase migrations as the source of truth for schema and RLS.
- Scheduled trigger configuration, authentication, frequency, batch limits, timeout, retry, and monitoring. Validate platform plan constraints before committing to a schedule.
- Email provider configuration and local Mailpit/sandbox usage.
- CI checks, preview workflow, migration safety, rollback/forward-fix policy.
- Secret inventory and rotation.
- Backups, restore drills, alert thresholds, and operational ownership.
- Production launch gate for processor approval and reminder SLO evidence.

## 6. Cross-document consistency requirements

Before finishing, verify:

- Every FR and SC in requirements has a corresponding design element and test/measurement.
- The architecture's diagram matches the design's actual components and data flow.
- All reminder timing values agree across requirements, architecture, and design.
- No document claims a five-minute polling interval by itself guarantees SC-003.
- RLS is enabled and specified for every exposed tenant table.
- Service-role credentials never enter browser/client code.
- The third-party processing conflict is clearly marked as a governance gate unless formally approved.
- No document claims full constitution compliance while Principle IV remains unresolved.
- The selected design does not accidentally reintroduce the base architecture's Redis/worker stack.
- Cost and plan claims are qualified and must be checked against current provider limits/pricing.
- Original source documents and review reports remain unchanged.

## 7. Completion checklist and agent response

After writing the three files:

1. Read all three back from the repository.
2. Check links, headings, diagrams, tables, terminology, and internal consistency.
3. Confirm each requested output path exists.
4. Summarize key decisions, unresolved blockers, and any assumptions made.
5. Report the exact paths created and the commit(s) made.

Do not mark the third-party data issue or SC-003 as solved by documentation alone: the documents must define the controls and evidence required, and production launch remains gated until those conditions are met.
