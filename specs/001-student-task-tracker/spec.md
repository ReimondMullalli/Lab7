# Feature Specification: Student Task Tracker

**Feature Branch**: `001-student-task-tracker`

**Created**: 2026-11-05

**Status**: Draft

**Input**: User description: "A web app where students track their courses, assignments, due dates
and grades in one place, see what is due soon, and get reminders before deadlines."

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Track assignments and due dates (Priority: P1)

A student signs up, adds the courses they are taking this semester, and adds assignments to each
course with a title, due date and time, and optional notes. They see one list of all upcoming
assignments sorted by due date and can mark an assignment as done.

**Why this priority**: This is the core value of the product. Without it, nothing else matters.

**Independent Test**: Create an account, add two courses and three assignments, and confirm the
upcoming list shows them in due-date order and that marking one done removes it from "upcoming".

**Acceptance Scenarios**:

1. **Given** a signed-in student with no courses, **When** they add a course "CIS 285", **Then**
   the course appears in their course list.
2. **Given** a course exists, **When** the student adds an assignment due 11/12 at 11:59 pm,
   **Then** it appears in the upcoming list in the correct date order.
3. **Given** an upcoming assignment, **When** the student marks it done, **Then** it moves to a
   "completed" list and no longer appears as upcoming.
4. **Given** an assignment whose due date has passed and is not done, **When** the student views
   the list, **Then** it is shown as overdue.

---

### User Story 2 - Deadline reminders (Priority: P2)

A student chooses to receive an email reminder a set time before each assignment is due (default:
24 hours), so they don't miss deadlines.

**Why this priority**: Reminders add a lot of value, but the tracker is still useful without them.

**Independent Test**: Create an assignment due in 25 hours with a 24-hour reminder and confirm an
email arrives about one hour later; mark another done and confirm no reminder is sent.

**Acceptance Scenarios**:

1. **Given** reminders are on with a 24-hour lead time, **When** an assignment is 24 hours from its
   due time, **Then** the student receives one reminder email naming the assignment and course.
2. **Given** an assignment is marked done, **When** its reminder time arrives, **Then** no
   reminder is sent.
3. **Given** a student turns reminders off, **When** any reminder time arrives, **Then** no
   reminder is sent.

---

### User Story 3 - Record grades and see course averages (Priority: P3)

A student enters the score and weight for each graded assignment, and sees their current weighted
average for each course.

**Why this priority**: Useful for planning study effort, but secondary to not missing deadlines.

**Independent Test**: Enter scores for three weighted assignments in one course and confirm the
displayed average matches a hand calculation.

**Acceptance Scenarios**:

1. **Given** a course with assignments weighted 20%, 30% and 50%, **When** the student enters
   scores of 90, 80 and 70, **Then** the course average shows 77.0%.
2. **Given** a course where only some assignments are graded, **When** the student views it,
   **Then** the average is calculated from graded assignments only, and this is labeled clearly.

---

### User Story 4 - Weekly calendar view (Priority: P4)

A student views a week-at-a-glance calendar showing all assignments due that week, colored by
course.

**Why this priority**: A convenience view of data that already exists.

**Independent Test**: With assignments spread across a week, open the calendar and confirm each
appears on the correct day in its course color.

**Acceptance Scenarios**:

1. **Given** assignments due Monday and Friday, **When** the student opens this week's calendar,
   **Then** each assignment appears on its due day.
2. **Given** the calendar is open, **When** the student moves to the next week, **Then** that
   week's assignments are shown.

### Edge Cases

- A due date is entered in the past: the system accepts it and shows it as overdue at once.
- A course is deleted that still has assignments: the student is warned, and on confirmation the
  course and its assignments are deleted together.
- Grade weights in a course add up to more than 100%: the student is warned, and the average is
  still calculated from the weights given.
- A reminder email fails to send: the system retries, and stops after three failed attempts.
- The student changes time zones: due times are shown in the student's chosen time zone.
- Two browser tabs edit the same assignment: the last saved change wins, and the other tab shows the
  updated value when it refreshes.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: System MUST let a student create an account with an email and password, sign in and
  sign out.
- **FR-002**: System MUST let a student reset a forgotten password by email.
- **FR-003**: Students MUST be able to create, rename and delete courses.
- **FR-004**: Students MUST be able to create, edit and delete assignments with a title, course,
  due date and time, optional notes, optional weight (%) and optional score.
- **FR-005**: System MUST show all not-done assignments in one list sorted by due date, with overdue
  items marked.
- **FR-006**: Students MUST be able to mark an assignment done and undo that.
- **FR-007**: Students MUST be able to filter the assignment list by course and by status
  (upcoming, overdue, done).
- **FR-008**: System MUST send an email reminder at the lead time the student chooses (1 hour to 7
  days; default 24 hours), and MUST NOT send one for an assignment that is done.
- **FR-009**: Students MUST be able to turn reminders on or off.
- **FR-010**: System MUST compute each course's weighted average from graded assignments.
- **FR-011**: System MUST show a weekly calendar of assignments, colored by course.
- **FR-012**: System MUST show each student only their own data.
- **FR-013**: Students MUST be able to delete their account and all their data.

### Key Entities

- **Student**: a person with an account; has an email, a display name, a time zone and reminder
  settings. Owns courses.
- **Course**: a class the student is taking; has a name, an optional code (for example "CIS 285"),
  a color and a term. Belongs to one student; has many assignments.
- **Assignment**: a piece of coursework; has a title, a due date and time, notes, a status
  (upcoming/done), an optional weight and an optional score. Belongs to one course.
- **Reminder**: a scheduled notification for one assignment; has a send time and a delivery status
  (pending/sent/failed/cancelled).

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: A new student can sign up and add their first course and assignment in under 3
  minutes.
- **SC-002**: The upcoming list appears in under 2 seconds for a student with 500 assignments.
- **SC-003**: 99% of reminder emails are delivered within 5 minutes of their scheduled time.
- **SC-004**: Course averages match a hand calculation to one decimal place in 100% of test cases.
- **SC-005**: 90% of test users complete "add an assignment and mark it done" on their first try
  without help.

## Assumptions

- Users are university students using a desktop or mobile web browser with internet access.
- Email and password sign-in is enough for version 1; university single sign-on is out of scope.
- Reminders are sent by email only; push notifications and SMS are out of scope for version 1.
- No sharing between students or with instructors in version 1.
- There is no integration with Canvas or other learning systems in version 1. Students enter
  assignments by hand.
- The expected scale is up to 10,000 registered students and 1,000 active at the same time.
