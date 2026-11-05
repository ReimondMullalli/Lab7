# Data Model: Student Task Tracker (Phase 1)

```text
Student 1 ──── * Course 1 ──── * Assignment 1 ──── 0..1 Reminder
```

## Student

| Field | Type | Rules |
|-------|------|-------|
| id | UUID | primary key |
| email | text | unique, valid email, stored lowercase |
| passwordHash | text | Argon2id; never returned by the API |
| displayName | text | 1–60 characters |
| timeZone | text | IANA zone, for example `America/Detroit` |
| remindersEnabled | boolean | default `true` (FR-009) |
| reminderLeadMinutes | integer | 60–10,080 (1 hour to 7 days); default 1,440 (FR-008) |
| createdAt | timestamp | set by the system |

## Course

| Field | Type | Rules |
|-------|------|-------|
| id | UUID | primary key |
| studentId | UUID | foreign key → Student; cascade delete |
| name | text | 1–100 characters; required |
| code | text | optional, for example `CIS 285` |
| color | text | hex color, for example `#3366CC` |
| term | text | for example `Fall 2026` |

## Assignment

| Field | Type | Rules |
|-------|------|-------|
| id | UUID | primary key |
| courseId | UUID | foreign key → Course; cascade delete |
| title | text | 1–200 characters; required |
| dueAt | timestamp (UTC) | required; past dates allowed (shown as overdue) |
| notes | text | optional, up to 2,000 characters |
| status | enum | `upcoming` or `done` |
| weight | decimal | optional, 0–100 (%) |
| score | decimal | optional, 0–100 (%) |
| completedAt | timestamp | set when status becomes `done`; cleared on undo |

**Derived values**: *overdue* = `status = upcoming` and `dueAt < now`. *Course average* =
Σ(score × weight) / Σ(weight) over assignments with both score and weight (FR-010).

**State transitions**: `upcoming → done` (mark done, cancels the reminder);
`done → upcoming` (undo, reschedules the reminder if its send time is still in the future).

## Reminder

| Field | Type | Rules |
|-------|------|-------|
| id | UUID | primary key |
| assignmentId | UUID | foreign key → Assignment; unique; cascade delete |
| sendAt | timestamp (UTC) | `dueAt − reminderLeadMinutes` |
| status | enum | `pending`, `sent`, `failed`, `cancelled` |
| attempts | integer | 0–3 |

**State transitions**: `pending → sent`; `pending → failed` (after 3 attempts);
`pending → cancelled` (assignment done, deleted or reminders turned off).
