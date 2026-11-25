# ADR-0011: Assignments, a Submit grant, and one submission notebook per course

> **Status:** Accepted (2026-11-25, grilling session). **Not built.**
> **Refines** [ADR-0009](ADR-0009-optional-school-server.md) and [ADR-0010](ADR-0010-permissions-per-notebook.md). Wire format: [protocol spec §9](../specs/15-server-protocol.md).
>
> **Amended 2026-11-25 (protocol spec).** First drafted as pinning to a `seq` the
> client reports. Changed to a **byte offset the server records**: a tampering
> client could later append a line with a smaller `seq`, but cannot move bytes
> before an offset the server already holds — and the server still reads no op.

## Context

A teacher who writes feedback straight onto a student's page cannot stop the
student editing it: under ADR-0010 the student holds Write on the whole notebook.
The wanted workflow is the familiar one: a student **hands in** a page, a copy
goes to the teacher, the teacher annotates the copy and **returns** it. Each
student must see only their own submissions, and thirty assignments must not
mean thirty notebooks.

## Decision

1. **One submission notebook per Course**, owned by the teacher, with a section
   per assignment. The Course's group holds a fourth grant level, **Submit**.
2. **Submit is write-only, enforced per log file.** A Submit holder may upload
   its own device logs and is served **only those** — never another device's.
   The server still never reads an op: it already knows which log belongs to
   whom (ADR-0009 §12). This is what keeps one shared notebook private per
   student.
3. **Handing in is a copy plus an explicit call.** The client copies the page,
   with fresh ids, into its own log in the submission notebook (the
   cross-notebook copy ADR-0010 needs anyway), uploads, then calls
   `submit(assignment, page)`. The server records its own clock time and **its own
   length of that log, in bytes**.
4. **A submission is pinned to that byte length.** The teacher's client applies a
   student's ops to a submitted page only up to the recorded offset, and only to
   pages that student's log created. A later edit — or a tampering client
   writing to someone else's page id — never shows. Handing in again creates a
   new **version**; the teacher sees all of them.
5. **Assignments are server entities** — title, due date, Course, template page,
   per-student status (open, submitted, late, returned). The content stays in
   notebooks. Distributing an assignment copies the template into each
   student's notebook, where the teacher is Owner.
6. **Late is marked, not refused**, by the server's clock. A teacher may close
   an assignment to further submissions; the server then refuses the
   `submit` call, and an unregistered page is never shown as a submission.
7. **Returning copies back.** The teacher annotates the submitted copy and
   returns it: their client copies the annotated page into the student's
   notebook. The student may edit that copy; the original stays with the
   teacher and is the record.

## Considered options

- **A submission notebook per assignment.** First proposed: it let the server
  enforce a deadline by refusing uploads to the notebook, no API call needed.
  Rejected as unmanageable at thirty assignments a year per class; the explicit
  `submit` call recovers the deadline and adds tamper-proof pinning besides.
- **The server copies the page.** Rejected: it would have to parse ops,
  contradicting ADR-0010.
- **Feedback only on the student's own page.** Kept as well (ADR-0009 §17), but it
  cannot protect the feedback from the student.
