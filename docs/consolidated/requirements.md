# Consolidated Requirements: Student Task Tracker

**Status**: Consolidated (Lab 7, step 7). Authoritative over `specs/001-student-task-tracker/spec.md`
for the selected architecture.
**Date**: 2026-11-05
**Decision**: [docs/decision.md](../decision.md): alternative (serverless) architecture accepted with conditions
**Instructions followed**: [docs/consolidation-instructions.md](../consolidation-instructions.md)
**Companion documents**: [architecture.md](./architecture.md) · [design.md](./design.md)

**Sources**: [spec.md](../../specs/001-student-task-tracker/spec.md) ·
[constitution](../../.specify/memory/constitution.md) ·
[alternative architecture](../architecture-alternative.md) ·
[base review](../reviews/review-base.md) · [alternative review](../reviews/review-alternative.md)

> **Production launch is gated.** Two blocking conditions from the decision are open:
> **G-1** Principle IV third-party processor approval (see §6) and **G-2** measured evidence for
> SC-003 reminder timing (see §5). Until both are closed, only synthetic data and local email
> capture may be used.

---

## 1. Scope

A web application where university students track courses, assignments, due dates and grades, see
what is due soon, and receive email reminders before deadlines.

**In scope (v1)**: email/password accounts; courses; assignments; upcoming/overdue/done lists and
filters; email reminders; weighted course averages; weekly calendar; account deletion.

**Out of scope (v1)**: university SSO; push or SMS notifications; sharing with other students or
instructors; Canvas or other LMS integration; a public or mobile API (see A-6).

## 2. User Stories (unchanged in substance from spec.md)

| ID | Priority | Story | Independent test |
|----|----------|-------|------------------|
| US1 | P1 | Track assignments and due dates: add courses and assignments, see one list sorted by due date, mark done/undo, see overdue items | Add 2 courses and 3 assignments; list order is correct; marking one done moves it to Completed |
| US2 | P2 | Deadline reminders: receive an email at a chosen lead time before each not-done assignment | Assignment due in 61 min with 1-hour lead: provider accepts one email within 5 min of `send_at`; none when marked done first |
| US3 | P3 | Record grades and see course averages | Weights 20/30/50 and scores 90/80/70 → average 77.0% |
| US4 | P4 | Weekly calendar view colored by course | Assignments due Monday and Friday appear on those days in course color |

The acceptance scenarios in [spec.md](../../specs/001-student-task-tracker/spec.md) §User Scenarios
remain in force and are implemented as Playwright end-to-end tests (see §8).

## 3. Functional Requirements

FR-001 to FR-013 are kept from spec.md, with clarifications **in bold** where the review or
instructions required them. FR-014 to FR-020 are new.

| ID | Requirement |
|----|-------------|
| FR-001 | Students MUST be able to register with email and password, sign in and sign out. **Registration and sign-in errors MUST NOT reveal whether an email is already registered (no account enumeration).** |
| FR-002 | Students MUST be able to reset a forgotten password by email. **The reset request MUST return the same response whether or not the account exists.** |
| FR-003 | Students MUST be able to create, rename and delete courses. **Deleting a course with assignments requires confirmation and deletes its assignments and their reminders.** |
| FR-004 | Students MUST be able to create, edit and delete assignments with title (1–200 chars), course, due date and time, optional notes (≤ 2,000 chars), optional weight (0–100 %) and optional score (0–100 %). **Past due dates are accepted and shown as overdue.** |
| FR-005 | The system MUST show all not-done assignments in one list sorted by due time ascending, with overdue items marked. |
| FR-006 | Students MUST be able to mark an assignment done and undo that. |
| FR-007 | Students MUST be able to filter by course and by status. **Status filter values are `upcoming`, `overdue`, `done`. Only `upcoming` and `done` are stored; `overdue` is derived as "stored status is upcoming AND due time < now" and is never persisted.** |
| FR-008 | The system MUST send one email reminder per not-done assignment at `send_at = due_at − lead time`, where lead time is chosen by the student between 1 hour and 7 days (default 24 hours). **If `send_at` is already in the past when the assignment is created or edited, and the due time is still in the future, the reminder is sent at the next dispatch run; if the due time has passed, no reminder is created.** |
| FR-009 | Students MUST be able to turn reminders on or off. Turning them off cancels all pending reminders; turning them on schedules reminders for all upcoming assignments whose due time is in the future. |
| FR-010 | The system MUST compute each course's weighted average as **Σ(score × weight) / Σ(weight) over assignments that have both a score and a weight greater than 0. If no such assignment exists the average is null and shown as "No grades yet". The result is shown to one decimal place and labeled "based on graded assignments only". If the course's weights sum to more than 100 %, the average is still computed from the given weights and a warning is shown; weights summing to less than 100 % are normal (not everything is graded yet).** |
| FR-011 | The system MUST show a weekly calendar (Monday–Sunday in the student's time zone) of assignments, colored by course, with previous/next week navigation. |
| FR-012 | Each student MUST see and change only their own data. **Enforced at the database by Row-Level Security AND checked in server code (see FR-016).** |
| FR-013 | Students MUST be able to delete their account; this MUST delete their profile, courses, assignments, reminders and authentication record. |
| FR-014 *(new)* | **Reminder reliability**: a reminder MUST NOT be sent if, at the moment of sending, the assignment is done or deleted or the student has turned reminders off. Delivery is attempted at most **3** times with backoff; after the 3rd failure the reminder is marked `failed` and visible to operators. Overlapping dispatch runs MUST NOT send the same reminder twice in the normal case (see design for the residual risk). Reminders missed because a dispatch run did not happen MUST be sent by the next successful run if the due time has not passed. |
| FR-015 *(new)* | **Reminder rescheduling**: creating an assignment, editing its due time, undoing done, changing lead time, or turning reminders on MUST (re)compute `send_at` (a time-zone change does not alter `send_at`; see §4); marking done, deleting, or turning reminders off MUST cancel the pending reminder. |
| FR-016 *(new)* | **Authorization at two layers**: every data operation MUST be authorized by RLS policies in PostgreSQL and MUST also be run with the signed-in student's identity from server code. No table holding student data may be readable or writable without an RLS policy. |
| FR-017 *(new)* | **Privileged job isolation**: the reminder dispatch job is the only component allowed to use the database service-role credential. It MUST run only server-side, MUST be invoked only by the scheduler with a secret that is checked on every call, and MUST be limited to the reminder tables and functions. The credential MUST never reach the browser, client bundles, logs or source control. |
| FR-018 *(new)* | **Privacy and data minimization**: reminder emails MUST contain only the recipient address, assignment title, course name or code, and due time. They MUST NOT contain notes, scores, weights or other courses. Student data MUST NOT be sold or used for advertising or any purpose other than operating the tracker. |
| FR-019 *(new)* | **Validation and errors**: all input MUST be validated on the server against shared schemas and by database constraints. Errors use one structured shape (`code`, `message`, optional `fieldErrors`) and never include secrets, stack traces or other students' data. Form errors MUST be announced accessibly (associated labels and `aria-live` region). |
| FR-020 *(new)* | **Time zones**: due times and `send_at` are stored in UTC. Each student has an IANA time zone (default from the browser at sign-up). Due times are entered and shown in that zone, and daylight-saving transitions are handled by the time-zone database, not fixed offsets. |

## 4. Edge Cases (all kept from spec.md, clarified)

| Case | Required behavior |
|------|-------------------|
| Due date in the past | Accepted; shown as overdue immediately; no reminder created (FR-008). |
| Delete course with assignments | Confirmation dialog listing the number of assignments; on confirm, course, assignments and reminders are deleted in one transaction. |
| Weights total more than 100 % | Warning on the course page; average still computed (FR-010). |
| Reminder email fails | Retried up to 3 total attempts with backoff; then `failed` (FR-014). |
| Student changes time zone | Due times shown in the new zone; stored UTC instants do not change; `send_at` is recomputed only if the lead time is stored relative to local time (it is not; it is a duration, so `send_at` stays the same). |
| Two tabs edit the same assignment | Last saved write wins; the other tab shows the saved value on its next refresh or navigation. |
| Mark done while the reminder is being sent | Final pre-send check and claim logic make a send after completion unlikely; the remaining race window is documented in design §5.4. |
| Dispatch run missed or timed out | Next run picks up all due `pending` reminders and expired leases (FR-014). |

## 5. Success Criteria and how they are measured

| ID | Criterion | Measurement method |
|----|-----------|--------------------|
| SC-001 | A new student signs up and adds a first course and assignment in under 3 minutes. | Moderated usability session, at least 5 participants who have not seen the app; median and maximum times recorded. |
| SC-002 | The assignment list and calendar return in **under 2 seconds at p95** for a student with 500 assignments across 8 courses. | Load test against a preview deployment seeded with synthetic data, measuring server response plus render time in Playwright traces; 200 samples. |
| SC-003 | **99 % of reminders are accepted by the email provider within 5 minutes of `send_at`**, over a measurement window of at least 7 days (or a compressed load test with at least 2,000 reminders), at the expected peak of 1,000 reminders per hour. **Start**: `send_at`. **End**: the time the provider API returns success (`provider_accepted_at`). Inbox receipt is not claimed because the provider cannot verify it. Reminders that end in `failed`, or that are accepted after 5 minutes, count as misses. Reminders cancelled because the assignment was done do not count. | Automated query over `reminders` (`provider_accepted_at − send_at`) plus a load/failure test (design §8.4). **Gate G-2**: production launch requires this evidence. |
| SC-004 | Course averages match a hand calculation to one decimal place in 100 % of test cases. | Unit and database tests including the 90/80/70 × 20/30/50 → 77.0 % case, null case and over-100 % case. |
| SC-005 | 90 % of test users complete "add an assignment and mark it done" on their first try without help. | Usability study, at least 10 participants (students), task success recorded by an observer. **Automated Playwright tests check that the flow works but do not count as evidence for SC-005.** |
| SC-006 *(new)* | Zero cross-student data access in the security test suite. | pgTAP RLS tests and Server Action authorization tests (design §8) pass in CI on every push. |

## 6. Privacy, Security and Governance Requirements

| ID | Requirement |
|----|-------------|
| PR-1 | **Processor inventory.** Supabase processes: student email, auth user ID, hashed password and auth metadata (sign-in times, IP addresses in auth logs), display name, time zone, reminder settings, course names, codes, colors and terms, assignment titles, notes, due times, statuses, weights and scores, and reminder records. The email provider processes the recipient email address and the reminder subject and body (assignment title, course name/code, due time) plus delivery metadata. Vercel processes HTTP request metadata (IP address, user agent) and server function logs. |
| PR-2 | **Data minimization**: only the reminder fields in FR-018 are sent to the email provider. Logs MUST NOT contain email bodies, notes, scores or credentials. |
| PR-3 | **Retention**: account data is kept until the student deletes the account. Reminder rows are deleted 30 days after reaching `sent`, `failed` or `cancelled`. Application logs are kept no longer than 30 days. Backups expire according to the provider's backup retention, which must be documented before launch. |
| PR-4 | **Purpose limitation**: no sale, advertising, profiling or secondary use of student data. |
| PR-5 | **Processor review before production**: each provider's data-processing terms, subprocessors, hosting region, security certifications, breach notification terms, and deletion and backup behavior MUST be reviewed and the result recorded in `docs/`. |
| PR-6 | **Governance gate G-1**: the constitution's Principle IV says "No personal data is shared with third parties." Using Supabase, Vercel and an email provider is third-party processing. **This conflict is not resolved by this document.** Before any real student data is processed, the project owner MUST either (a) approve a narrowly scoped processor exception, or (b) merge a constitution amendment PR to Principle IV that allows contracted processors under PR-1 to PR-5, following the constitution's governance section (PR with rationale and impact, MINOR version bump). **Status: OPEN – BLOCKING.** |
| PR-7 | **Development data**: local development, CI and preview environments use synthetic data and local email capture (Mailpit) or the provider's test mode, never real student data. |
| SR-1 | Passwords are handled only by Supabase Auth; the application never stores or logs passwords. |
| SR-2 | All traffic is HTTPS; session cookies are `HttpOnly`, `Secure`, `SameSite=Lax`. |
| SR-3 | Mutations happen only through Server Actions (POST, with Next.js origin checking for CSRF protection) and redirects after sign-in accept only same-site relative paths. |
| SR-4 | Sign-in, sign-up and password-reset requests are rate limited (Supabase Auth limits plus CAPTCHA if abuse is detected). |
| SR-5 | Secrets are stored in Vercel and Supabase secret managers, separate per environment (development, preview, production), and rotated at least each term and immediately after any suspected exposure. |
| SR-6 | Database backups exist and a restore is tested before launch and once per term. |

## 7. Non-Functional Requirements

| ID | Requirement |
|----|-------------|
| NFR-1 | **Scale (design target, not proven capacity)**: 10,000 registered students and 1,000 concurrently active. Capacity is considered proven only after the load test in design §8.4. |
| NFR-2 | **Accessibility**: WCAG 2.1 AA: keyboard navigation, labeled controls, 4.5:1 text contrast, course colors never the only way to tell courses apart (course code shown too). |
| NFR-3 | **Browsers**: latest two versions of Chrome, Edge, Firefox and Safari; mobile Safari and Chrome; layouts from 360 px wide. |
| NFR-4 | **Availability**: relies on Vercel and Supabase availability; no custom SLA in v1. Reminder backlog recovers automatically after an outage (FR-014). |
| NFR-5 | **Process**: test-first (Red-Green-Refactor), feature branches, pull requests with one review, and CI must pass before merge (constitution III and V). |

## 8. Requirement Traceability

Component names are defined in [architecture.md §3](./architecture.md#3-components). Test IDs are
defined in [design.md §8](./design.md#8-test-plan).

| Req | User-visible behavior / control | Responsible component(s) | Verification |
|-----|---------------------------------|--------------------------|--------------|
| FR-001 | Sign-up, sign-in, sign-out pages | Auth pages, Supabase Auth, `profiles` trigger | E2E-AUTH-1; ACT-AUTH-1..3 |
| FR-002 | "Forgot password" flow | Supabase Auth reset email, reset page | E2E-AUTH-2; ACT-AUTH-4 (same response for unknown email) |
| FR-003 | Course create/rename/delete | `createCourse`, `updateCourse`, `deleteCourse`, `courses` RLS | ACT-COURSE-1..4; DB-RLS-COURSES; E2E-US1 |
| FR-004 | Assignment form | `createAssignment`, `updateAssignment`, `deleteAssignment`, DB checks | ACT-ASG-1..6; DB-CHK-1 |
| FR-005 | Dashboard list | `listAssignments`, `assignments_view`, index `assignments_student_due` | E2E-US1; DB-VIEW-1; PERF-1 |
| FR-006 | Done / Undo buttons | `setAssignmentStatus` | ACT-ASG-7; E2E-US1 |
| FR-007 | Course and status filters | `listAssignments` filters, derived overdue | DB-VIEW-2 (overdue derivation); E2E-US1 |
| FR-008 | Reminder email at lead time | `reminders` table, schedule function, dispatch function, email adapter | REM-1..3; E2E-US2 |
| FR-009 | Reminders on/off toggle | `updateReminderSettings`, `reschedule_student_reminders()` | REM-4; ACT-PROFILE-2 |
| FR-010 | Course average display | `course_averages` view | DB-AVG-1..4; UNIT-AVG-1 |
| FR-011 | Weekly calendar page | `listAssignments` with `from`/`to` in student zone | E2E-US4; UNIT-TZ-1 |
| FR-012 | Only own data visible | RLS on all tables; server actions run as user | DB-RLS-* (owner, non-owner, anon); ACT-SEC-1..3 |
| FR-013 | Delete account button | `deleteAccount` action → `delete_my_account()` SECURITY DEFINER function, FK cascade | ACT-ACCT-1; DB-CASCADE-1 |
| FR-014 | Reliable, non-duplicated reminders | `claim_due_reminders()`, lease, attempts, Resend idempotency key | REM-5..10 |
| FR-015 | Reminders follow edits | Trigger `assignments_reminder_sync`, `reschedule_student_reminders()` | REM-11..16 |
| FR-016 | Two-layer authorization | RLS + user-scoped Supabase client in actions | DB-RLS-*; ACT-SEC-1..3 |
| FR-017 | Service key isolation | Edge Function `dispatch-reminders`, scheduler secret | SEC-KEY-1 (bundle scan), SEC-CRON-1 (rejects missing/bad secret) |
| FR-018 | Minimal email content | Email template + adapter | UNIT-EMAIL-1 (snapshot excludes notes/scores) |
| FR-019 | Structured, accessible errors | Shared Zod schemas, `ActionResult` type, form components | ACT-VAL-1; E2E-A11Y-1 (axe) |
| FR-020 | Local-time display, UTC storage | `toUtc`/`fromUtc` helpers, `profiles.time_zone` | UNIT-TZ-1..3 (DST transitions) |
| SC-001 | Fast onboarding | Whole app | Usability session (§5) |
| SC-002 | List < 2 s p95 | Indexes, view, server rendering | PERF-1 |
| SC-003 | 99 % within 5 min | Supabase Cron 1-min schedule, dispatch, provider | REM-LOAD-1; production metric `reminder_delay_p99` |
| SC-004 | Correct averages | `course_averages` view | DB-AVG-1..4 |
| SC-005 | 90 % first-try success | UI design | Usability study (§5) |
| SC-006 | No cross-student access | RLS + actions | DB-RLS-*; ACT-SEC-* |

## 9. Assumptions

- A-1: Users are university students using modern browsers with internet access.
- A-2: Email/password auth is enough for v1.
- A-3: Peak reminder volume is about 1,000 per hour (evening deadlines); average far lower.
- A-4: Supabase Cron supports a one-minute schedule on the chosen Supabase plan (verified in the
  Supabase Cron documentation, which describes schedules from every second to yearly; plan
  availability must be confirmed before launch).
- A-5: The email provider is Resend (supports `Idempotency-Key`, kept for 24 hours). Another
  provider can be swapped in behind the adapter.
- A-6: Server Actions are internal web interfaces, not a public API. A versioned public API would be
  a separate future decision.

## 10. Discrepancies Found Between Sources

| # | Discrepancy | Resolution |
|---|-------------|------------|
| D-1 | The alternative architecture used **Vercel Cron every 5 minutes**. Vercel's current docs state Hobby cron jobs may run **only once per day** with ±59 min precision; once-per-minute requires Pro. | Reminder scheduling moved to **Supabase Cron (pg_cron) every minute** invoking a Supabase Edge Function (architecture §6). |
| D-2 | The alternative's Constitution Check marked all principles PASS. | Corrected: Principle IV is **OPEN – BLOCKING (G-1)**; Principle III is **PASS with interpretation** (Server Action contract tests replace REST contract tests, design §4). |
| D-3 | spec.md says "reminder email arrives"; the provider can only confirm acceptance. | SC-003 now measures to provider acceptance. |
| D-4 | spec.md edge case "time-zone change" implied re-scheduling. | Lead time is a duration, so the UTC `send_at` doesn't change; only display changes (§4). |
