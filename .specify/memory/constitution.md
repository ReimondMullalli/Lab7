# Student Task Tracker Constitution

## Core Principles

### I. Student-First Simplicity

Every feature MUST serve a student managing coursework: courses, assignments, due dates and grades.
Features that do not directly support that goal are out of scope. The simplest design that meets a
requirement MUST be chosen (YAGNI); added complexity requires a written justification in the plan.

### II. Spec-Driven Development

No code is written before a specification exists. Every feature MUST flow through
spec → plan → tasks → implementation, and every requirement in a spec MUST be testable and
traceable to at least one task.

### III. Test-First (NON-NEGOTIABLE)

Automated tests MUST be written before the code they verify, and MUST fail before implementation
begins (Red-Green-Refactor). Every API endpoint MUST have a contract test; every user story MUST have
at least one end-to-end acceptance test.

### IV. Data Privacy and Security

Each student's data MUST be visible only to that student. Passwords MUST be stored as salted
hashes, never in plain text. All traffic MUST use HTTPS. No personal data is shared with third
parties.

### V. Continuous Integration and Version Control

All work MUST live in Git on GitHub, use feature branches and be merged through pull requests.
Every push MUST trigger a CI pipeline that builds the project and runs the full test suite; a red
build blocks merging.

## Technology and Quality Constraints

- The system is a web application usable from desktop and mobile browsers.
- Pages MUST load in under 2 seconds for a student with up to 500 assignments.
- The UI MUST meet WCAG 2.1 AA accessibility basics (keyboard navigation, labels, contrast).
- Dependencies MUST be actively maintained, open-source and pinned to exact versions.

## Development Workflow

- Each feature starts with `/speckit-specify`, then `/speckit-plan`, `/speckit-tasks` and
  `/speckit-implement`.
- Pull requests require one review and a passing CI build before merge.
- Architecture decisions that change a principle or constraint are recorded in `docs/`.

## Governance

This constitution supersedes all other project practices. Amendments require a pull request that
explains the change, its rationale and its impact on existing specs and plans. Versioning follows
semantic versioning: MAJOR for removed or redefined principles, MINOR for new principles or
sections, PATCH for wording fixes. Every plan MUST include a Constitution Check against these
principles.

**Version**: 1.0.0 | **Ratified**: 2026-11-05 | **Last Amended**: 2026-11-05
