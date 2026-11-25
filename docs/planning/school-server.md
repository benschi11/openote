# School server — design summary

A summary of a design session (2026-11-25) on an **optional, self-hosted server**
that lets a school store notebooks centrally, with roles and access rights —
roughly what OneNote Class Notebooks do, but built on Openote's own sync model.
Nothing here is built yet. The decisions are recorded in
[ADR-0009](../adr/ADR-0009-optional-school-server.md),
[ADR-0010](../adr/ADR-0010-permissions-per-notebook.md) and
[ADR-0011](../adr/ADR-0011-assignments-and-submission-notebooks.md); the terms
are defined in [GLOSSARY.md](../../GLOSSARY.md).

## The core idea

**The server is another sync transport, not a new source of truth.** It moves the
same per-device op logs that ADR-0006 already defines — alongside folders and
git — so clients stay local-first and fully usable offline. Without a server,
Openote is unchanged.

What a server adds that a shared folder never can is **withholding data**. So
**permissions are enforced only on the server**: a client is never sent what its
user may not read, and uploads its user may not make are refused. Anything the
client merely hides would be readable in its plaintext SQLite (the lesson of
ADR-0008).

This deliberately departs from ADR-0006 §8 ("Openote never talks to a cloud
API"). That rule's reasons — vendor OAuth secrets in an open binary, four vendor
APIs — don't apply to a server the school runs itself, and ADR-0006 §3 already
anticipated it: *"live collaboration is a transport swap, not a redesign."*

## Permissions: whole notebooks, opaque logs

- **In v1 a permission attaches to a whole notebook**, never a section or page.
  Levels: **Read**, **Write**, **Owner** (Owner may change grants), plus
  **Submit** for assignments (below). Grants go to users or groups.
- The server **never parses ops**. It only answers "may this user read/write this
  notebook". Read-only is simple: the client keeps no log of its own there, and
  the server refuses uploads.
- Why not per page? Each device writes one gapless, never-rewritten log per
  notebook (file format §3.2). A server can't serve that log with someone else's
  private pages cut out, and can't refuse one forbidden op without stalling every
  allowed op behind it.
- **Per-section "share scopes" are deferred, not rejected.** The manifest's
  `scope` object exists so they can be added later as a new value, without a
  layout change.
- What per-page rights would have done is done with **more notebooks**: a
  **Course** groups a subject's notebooks; each student gets their own notebook
  (created in bulk per group); making a page private means moving it to another
  notebook — which requires **copying between notebooks**, not supported today.
- **Revocation stops the flow; it doesn't reach back.** Nothing new is sent or
  accepted from then on; what a device already holds stays there. The client
  removes its local copy as a courtesy (offering an export first), and the UI is
  honest that this is not security.

## Assignments

Teachers write feedback directly on students' pages (teacher is Owner of each
per-student notebook, and the client shows who wrote what), but a student with
Write could change that feedback. So there is also a hand-in workflow:

- **One submission notebook per Course**, with a section per assignment.
- The course group holds **Submit**: a write-only grant, enforced **per log
  file**. A student may upload their own device log and is served only that —
  never other students' logs. The server still reads no content.
- **Handing in** = the client copies the page (fresh ids) into its own log there,
  then calls `submit(assignment, page, seq)`. The server records its own time and
  that `seq`.
- **A submission is pinned to that `seq`**: the teacher's client shows the
  student's ops only up to it, and only on pages that student created. Later
  edits or tampering never show. Handing in again creates a new version.
- **Late is marked, not refused** (server clock); a teacher can close an
  assignment. **Returning** copies the annotated page back into the student's
  notebook; the teacher keeps the original.
- Assignments are server entities (due date, course, template, per-student
  status). Distributing one copies the template into each student's notebook.
- **No grades in v1.**

## Accounts and identity

- Roles live on the account: **Teacher**, **Student**, **Admin**. By default only
  teachers create server notebooks; letting students share is an admin setting,
  off by default.
- **Local accounts and Microsoft Entra ID from the start, LDAPS planned.** The
  server is the only identity provider the client knows: the school registers
  *its server* with Entra once; the client signs in to the server (system
  browser, OAuth + PKCE, loopback) and gets the server's own tokens. Openote
  ships no vendor client id or secret.
- **Groups (classes) are maintained on the server** (by hand or CSV); syncing
  them from Entra/LDAP is a later add-on.
- **Each device id is bound to one user** on first upload, so nobody can upload a
  log under someone else's device id. The client uses **one device id per
  (install, server account)**, so two students on one lab PC don't collide;
  signing out on a shared PC removes that account's server notebooks.
- Sessions: 1-hour access tokens, 30-day sliding refresh tokens; offline work
  needs none. Admins can end all sessions of an account.
- **Nothing on the server is private from teachers** — truly private notes stay
  in local notebooks.

## Operations

- **One self-hosted instance per school**, one container image, one data volume.
  No multi-tenancy (keeps GDPR simple).
- Op logs and blobs are stored **in the `.onotebook` layout on disk**, so a backup
  is a folder Openote can open without the server. Metadata (accounts, groups,
  grants, assignments, device bindings) in EF Core — SQLite by default,
  PostgreSQL optional.
- **Blobs are scoped per notebook**, so a hash can't be used to probe what exists.
- **HTTPS only** (plain http only for `localhost`). v1 transport is
  request/response; SignalR push later. The protocol stays browser-compatible,
  but a browser/iPad/Chromebook client is not part of v1.
- A server notebook syncs **only** through the server (no parallel folder/git).
- Offline edits refused after a grant narrowed are **kept**: the notebook opens
  read-only and offers to copy the affected pages into a local notebook.
- School year: archiving a course turns students' Write/Submit into Read; leaving
  students are deactivated (a revocation); owner grants can be transferred;
  deletion only by an admin after a retention period.
- Admin web UI on the server (accounts, groups, CSV import, bulk per-student
  notebooks); grants and courses are edited in the Openote client's Share dialog.

## Code and licensing

- Server in **C# / .NET 10 LTS**: ASP.NET Core minimal APIs, EF Core, ASP.NET Core
  Identity + OpenIddict, Blazor admin UI. AGPL-3.0-or-later like the app.
- Proposed as **its own repository** (working name `openote-server`) — to be
  agreed with the maintainer. The client half (server transport, sign-in, Share
  dialog, cross-notebook copy) has to live in this repo regardless.
- The contract is a spec, **`docs/specs/15-server-protocol.md`**, next to the file
  format, so neither side's code is the authority.

## Implementation order

Each step is usable on its own.

1. Protocol spec.
2. Server core (local accounts, groups, notebooks with Read/Write/Owner, HTTPS
   sync) + client (server transport, sign-in, publish/add, read-only open).
   *This alone covers handing material to a class.*
3. Microsoft Entra ID sign-in.
4. Copy between notebooks, courses, per-student notebooks.
5. Assignments: Submit, hand in, return.
6. LDAPS, group sync, SignalR push.

## What it asks of the Openote client

- A server sync transport next to folder and git, plus sign-in.
- Opening a notebook read-only when the user holds only Read.
- A device id per (install, server account).
- Copying pages between notebooks.
- A Share dialog, courses in the sidebar, and showing who wrote what.
