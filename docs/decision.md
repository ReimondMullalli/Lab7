# Architecture Decision: Adopt the Alternative Architecture

**Date:** 2026-11-05  
**Status:** Accepted, subject to the conditions below  
**Decision owner:** Project owner  
**Chosen option:** Serverless full-stack architecture described in `docs/architecture-alternative.md`

## Decision

The project will use the alternative architecture: Next.js on Vercel, Supabase Auth and managed PostgreSQL with Row-Level Security (RLS), scheduled reminder processing, and a transactional email provider.

This option is preferred because the Student Task Tracker is a small student project with a bounded initial scale (up to 10,000 registered students and 1,000 active users). It has lower expected cost, fewer services to deploy, and less infrastructure to operate than the base React/Express/Redis/BullMQ architecture.

## Conditions of acceptance

The alternative is accepted only after the consolidated design addresses both material issues identified in the architecture reviews:

1. **Reminder timing and reliability:** The implementation must demonstrate the specification's SC-003 target—99% of reminder emails delivered within five minutes of their scheduled time—with measured margin for scheduler delay, function startup, database work, retries, and email-provider latency. A five-minute polling interval alone is not sufficient evidence. The design must define bounded batches, concurrency/claiming, retry and recovery behavior, duplicate prevention, and monitoring.
2. **Third-party data processing:** Supabase and the email-delivery provider process personal data. This must not be treated as compliant with the current constitution's literal prohibition on sharing personal data with third parties. Before real student data is processed, document and obtain approval for a narrowly scoped processor exception or formally amend Principle IV through the constitution's required governance process. Require appropriate contractual/privacy safeguards, data minimization, retention/deletion rules, and no advertising or sale of student data. Until approval is in place, use synthetic data and local email capture for development/testing.

## What this decision does not authorize

- It does not authorize weakening student-to-student data isolation or relying on UI checks instead of database-enforced RLS.
- It does not authorize exposing Supabase service-role credentials to browsers or client bundles.
- It does not authorize claiming SC-003 is met without an end-to-end measurement.
- It does not silently amend the constitution. Any amendment must follow the constitution's PR and rationale requirements.
- It does not add a separate Express API, Redis, or BullMQ unless later evidence shows the selected design cannot meet requirements.

## Basis

The comparative reviews are recorded in:

- `docs/reviews/review-base.md`
- `docs/reviews/review-alternative.md`

The alternative scored 8.0/10 overall versus 7.5/10 for the base architecture. Its cost and operational simplicity are decisive for the current project, while its reminder SLO and third-party processing concerns are explicit acceptance conditions rather than accepted residual risks.
