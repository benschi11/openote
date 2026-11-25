# 15 · School Server Protocol

> **Protocol version 1.0 — draft.** Nothing here is built yet; it may change
> without notice until a first server and client ship against it.
> **Licence: CC0-1.0** ([`LICENSE`](LICENSE)). Implement it freely, no
> attribution required.
> **Audience:** anyone writing a **School Server** or a client for one. Read the
> [file format spec](10-file-format-spec.md) §2–§3 first: this protocol moves
> exactly those files and adds no second format for notebook content.
> **Normativity:** MUST, SHOULD and MAY are used in the RFC 2119 sense. Anything
> marked *(informative)* is guidance, not a conformance requirement.
> **Related:** [ADR-0009](../adr/ADR-0009-optional-school-server.md) (the server) · [ADR-0010](../adr/ADR-0010-permissions-per-notebook.md) (permissions per notebook) · [ADR-0011](../adr/ADR-0011-assignments-and-submission-notebooks.md) (assignments) · [Glossary](../../GLOSSARY.md)

---

## 1. Goals and non-goals

1. **The server is a transport.** It stores and serves the files of a
   `.onotebook` directory — op logs, blobs, media — and nothing else about a
   notebook's content. Clients stay local-first; the protocol only moves bytes
   they already have or are missing.
2. **The server never parses an op.** Every access decision is made from the URL,
   the caller's identity and the server's own records. A log line is opaque bytes
   to it. This is what lets the server be written once, in any language, and
   never chase the file format (ADR-0010).
3. **Permissions are enforced here, not in the client.** A client is never sent a
   byte its user may not read and never succeeds in storing one its user may not
   write (ADR-0009 §2).
4. **Retries are always safe.** Every write is idempotent, because school Wi-Fi
   drops requests mid-flight.

**Non-goals for 1.0:** live push (a later minor version adds it, §11),
per-section permissions (ADR-0010), grades, browser clients — though nothing
here assumes a filesystem or prevents one.

## 2. Conventions

- **Transport: HTTPS only.** A client MUST refuse an `http://` server address,
  except `http://localhost` and `http://127.0.0.1` for development.
- **Base path: `/api/v1`.** Every path below is relative to it unless it starts
  with `/.well-known`.
- **JSON** bodies are UTF-8, `Content-Type: application/json`. Field names are
  `camelCase`. Unknown fields MUST be ignored by both sides.
- **Identifiers** are UUIDv7 in canonical lowercase form, as in the file format
  (§4). **Timestamps** are Unix epoch milliseconds, UTC.
- **Errors** are RFC 9457 problem details, `Content-Type:
  application/problem+json`, with `type` set to one of the codes in §10.
- **Authentication:** `Authorization: Bearer <access token>` on every request
  except `GET /server` and the OIDC endpoints. Tokens MUST NOT appear in URLs.
- **CORS:** a server SHOULD allow configured origins, so a browser client is
  possible later without a protocol change.

## 3. Discovery

### `GET /api/v1/server` — unauthenticated

```json
{
  "name": "BG Musterstadt",
  "protocol": {"major": 1, "minor": 0},
  "issuer": "https://notes.bg-musterstadt.at",
  "features": ["courses", "assignments"],
  "limits": {"maxRequestBytes": 8388608, "maxMediaBytes": 2147483648}
}
```

- A client MUST refuse a server whose `protocol.major` it does not implement, and
  MUST tolerate a higher `minor`.
- `features` names optional parts of this spec the server implements. A client
  MUST hide what is not listed instead of calling and failing.
- `issuer` is the OIDC issuer for §4. It is usually the server itself.

## 4. Sign-in

The server is the **only identity provider the client knows** (ADR-0009 §13).
Whether a person signs in with a local password, Microsoft Entra ID or LDAPS is
the server's business, behind its own login page.

- The server MUST be an OpenID Connect provider with discovery at
  `{issuer}/.well-known/openid-configuration`.
- The desktop client is a **public client**: client id `openote-desktop`,
  authorisation code flow with **PKCE (S256)**, and a **loopback redirect**
  (`http://127.0.0.1:{port}/callback`, any port) in the **system browser**, per
  RFC 8252. No client secret exists.
- Scopes: `openid profile offline_access openote`.
- Access tokens SHOULD live one hour. Refresh tokens SHOULD be sliding, 30 days
  (ADR-0009 §24). Revoking a person's sessions invalidates their refresh tokens.

*(Informative)* Working offline needs no token. An expired refresh token only
means signing in again before the next sync; nothing local is touched.

### `GET /me`

```json
{
  "user": {"id": "0199…", "displayName": "Anna Berger", "role": "student"},
  "groups": [{"id": "0199…", "name": "3B"}],
  "settings": {"studentsMayShare": false}
}
```

`role` is one of `teacher`, `student`, `admin`. An admin may also be a teacher.
In that case `role` is `teacher` and `"admin": true` is present.

## 5. Devices

Every op log is named for a device id (file format §3.2). On this server **each
device id belongs to exactly one user**, and only that user can append to logs
under it. This is what stops a student uploading a log under a teacher's id.

- **A client MUST use a separate device id for each (installation, server,
  account)**, generated as a fresh UUIDv7 the first time that account signs in on
  that installation. It MUST NOT reuse the installation's local device id. Two
  students who share a lab PC are then two writers, not one identity that keeps
  forking (ADR-0009 §12).

### `POST /devices`

```json
{"id": "019a…", "label": "Windows computer"}
```

- `201` when the id is new and now bound to the caller. `200` when it is already
  bound to the caller, which makes the call idempotent.
- `409 device-taken` when it is bound to someone else. The client MUST generate a
  new id and register again, exactly as for a fork (ADR-0006 §6a.2).
- A client MUST register a device before it appends under it.

### `GET /devices` · `DELETE /devices/{id}`

`GET` lists the caller's own devices. `DELETE` retires one: nothing more may be
appended under it, and its logs remain. An admin may do either for any user.
Retiring is how a lost laptop is cut off without losing what it already synced.

## 6. Notebooks

### 6.1 Grants

A **Grant** is `{"principal": {"kind": "user" | "group", "id": "…"}, "level":
"read" | "write" | "owner" | "submit"}`. A person's **effective level** on a
notebook is the highest of every grant that names them or one of their groups:
`read < submit < write < owner`.

| Level | May read | May append | May change grants |
|---|---|---|---|
| `read` | every log, blob and media file | nothing | no |
| `submit` | **only what their own devices uploaded** | under their own devices | no |
| `write` | everything | under their own devices | no |
| `owner` | everything | under their own devices | yes |

`submit` is valid only on a notebook whose `kind` is `submission` (§9). Nobody,
whatever their level, ever appends to a log under another user's device.

### 6.2 Listing and reading

#### `GET /notebooks`

Lists every notebook the caller holds a grant on:

```json
[{
  "id": "0198f3c2-…", "title": "Mathe 3B – Stoffsammlung", "kind": "notebook",
  "level": "read", "courseId": "0199…", "archived": false,
  "changeCursor": 18342
}]
```

`kind` is `notebook` or `submission`. `changeCursor` is explained in §8.

#### `GET /notebooks/{id}/manifest`

Returns a `manifest.json` (file format §3.1) **generated by the server**:
`notebookId`, `format`, `title`, `createdAt` and `scope: {"kind": "notebook",
...}`. `devices` is empty. Clients MUST NOT rely on it here, and the server does
not reveal which devices exist. A client writes this file into its local
`.onotebook` as-is.

### 6.3 Publishing

#### `POST /notebooks`

```json
{"id": "0198f3c2-…", "title": "Mathe 3B – Stoffsammlung", "courseId": null}
```

Creates an empty server notebook with the caller as `owner`. The caller must be a
teacher, or a student when `studentsMayShare` is on.

- `id` SHOULD be the local notebook's existing id, so the published notebook is
  the same notebook rather than a copy. `409 notebook-exists` if that id is
  already on the server.
- The client then uploads the notebook's existing logs, blobs and media (§7).

**Existing logs at publish time.** A local notebook usually already has logs
under device ids that are not registered here, written by this person's other
installations before the server existed. The publishing client uploads them with
`POST /notebooks/{id}/logs/{device}:import`. That call is identical to `append`
(§7.1), except that:

- it is allowed only to an `owner`, and only while the notebook has no log the
  caller did not import;
- the device id is bound to the caller and marked **closed**: nothing more may
  ever be appended under it, by anyone. `409 device-taken` if the id belongs to
  someone else.

From then on, each of the caller's installations writes under its own registered
per-account id (§5). *(Informative)* Closing imported logs keeps the rule
"exactly one live writer per log" true without asking anything of the logs
themselves.

### 6.4 Grants and lifecycle

| Call | Who | Effect |
|---|---|---|
| `GET /notebooks/{id}/grants` | `owner` | the notebook's grants |
| `PUT /notebooks/{id}/grants` | `owner` | replaces them all; MUST leave at least one `owner` |
| `PATCH /notebooks/{id}` `{"title"}` | `owner` | renames it on the server; the title inside the notebook is a `notebook.meta` op, as always |
| `DELETE /notebooks/{id}` | admin | marks it deleted; data goes after the configured retention (ADR-0009 §26) |

A grant change takes effect on the **next request**. There is no grace period
and nothing is pushed.

## 7. Moving the files

The three kinds of file in a `.onotebook` directory map to three resources. All
three are stored on the server **in the `.onotebook` layout**, so a server's data
directory is a set of notebooks Openote can open directly (ADR-0009 §18).

### 7.1 Logs

A log is addressed by **byte offset**, never by `seq`. The server keeps each log
as an exact byte copy of the device's file and never looks inside it.

#### `GET /notebooks/{id}/logs`

```json
{"logs": [{"device": "019a…", "length": 48211, "closed": false}],
 "changeCursor": 18342}
```

For a `submit` holder the list contains **only their own devices' logs**. Supports
`ETag` / `If-None-Match`, answering `304` when nothing changed.

#### `GET /notebooks/{id}/logs/{device}`

The log's bytes. A client SHOULD send `Range: bytes={from}-` with the length it
already holds. The server answers `206` with the bytes from that offset onward,
or `200` with the whole log. The response always carries `Openote-Log-Length` (the
total length). `404` for a device the caller may not read, so a log's existence is
not revealed.

#### `POST /notebooks/{id}/logs/{device}:append`

Header `Openote-Offset: {n}`, body `application/octet-stream`: bytes to add at
offset `n`.

- The caller MUST own `device` (§5), the device MUST NOT be closed or retired,
  and the caller's level MUST be `submit`, `write` or `owner`.
- The body MUST be non-empty and MUST end with `\n`. A client uploads **complete
  lines only** and never a torn tail (file format §3). The server checks the
  final byte and nothing else.
- If `n` equals the current length, the bytes are appended: `200
  {"length": …}`.
- If `n` is less than the current length and the stored bytes from `n` onward
  begin with exactly the body, the request is a **retry** and changes nothing:
  `200` with the current length.
- Otherwise `409 offset-mismatch` with `{"length": …}`. The client re-reads its
  position and resends from there.

A client with more than `maxRequestBytes` to send splits it into several
appends, each ending on a line boundary.

*(Informative)* The offset check makes every append idempotent and makes a
second writer under one id impossible to miss, without the server parsing `seq`.

### 7.2 Blobs

`GET | HEAD | PUT /notebooks/{id}/blobs/{sha256}`, where `{sha256}` is lowercase
hex.

- `PUT`: the server MUST verify that the body's SHA-256 equals `{sha256}`, or
  answer `422 hash-mismatch`. A `PUT` of bytes the server already holds succeeds
  with `200` and records the caller as an uploader. Requires `submit`, `write`
  or `owner`.
- `GET` / `HEAD`: `404` unless the blob was uploaded to **this** notebook. For a
  `submit` holder, also `404` unless one of their own devices uploaded it.
- Blobs are immutable. There is no `DELETE` (ADR-0007).
- The client stores the bytes as `blobs/{sha256}.blob`.

*(Informative)* Scoping blobs per notebook stops a hash being used to ask
whether some file exists anywhere on the server (ADR-0009 §19). The server may
still keep one copy of identical bytes internally.

### 7.3 Media

`GET | HEAD | PUT /notebooks/{id}/media/{name}`, where `{name}` is a media file's
bare filename (file format §3.7) and matches `^[0-9a-f-]{36}\.[a-z0-9]{1,10}$`.

- Media files are **write-once**. A `PUT` to an existing name succeeds only if
  the body is byte-identical, which makes retries safe. Otherwise it fails with
  `409 media-exists`.
- `GET` supports `Range`, so a client can stream video or resume a download.
- The same read rules as blobs apply, including the `submit` restriction.
- Bodies may be up to `limits.maxMediaBytes`. *(Informative)* Resumable uploads
  for very large media are a candidate for a later minor version.

### 7.4 What a client does on each sync *(informative)*

1. `GET /notebooks/{id}/logs`. If the answer is `304`, the client is done.
2. For each log it holds less of, a ranged `GET`. The client appends the bytes
   to its local copy of that log verbatim.
3. Fetch blobs and media lazily, when a page needs them.
4. For its own device, `append` everything past the server's `length`. It first
   uploads any blob or media file those ops reference, so no reader ever sees a
   reference before the bytes it points to.

## 8. Change feed

Polling every notebook separately does not scale to a student with fifteen
courses, so the server keeps one **change cursor**: a number that rises every
time any notebook changes, whether through an append, an import, a grant change
or a course change.

#### `GET /changes?since={cursor}`

```json
{"cursor": 18350,
 "notebooks": [{"id": "0198f3c2-…", "reason": "logs"},
               {"id": "0199…", "reason": "access"}]}
```

- It lists only notebooks the caller can see now, **plus** any the caller could
  see at `since` but can no longer see. Those carry `"reason": "access"`.
- `reason` is `logs`, `access` (grants, archive or deletion) or `meta`.
- `since` omitted or `0`: answered with just the current cursor, as a starting
  point.
- Clients SHOULD poll no more often than every 15 seconds while active and
  SHOULD back off when idle.

**When access is lost.** If a client gets `403 no-grant` or `404` for a notebook
it holds, its user's access was revoked. The client SHOULD then remove its local
copy, after telling the user and offering an export of anything unsent (ADR-0009
§16). If it gets `403 level-insufficient` on an append, the user's level was
lowered (for example Write → Read). The client MUST keep the refused ops, open
the notebook read-only, and offer to copy the affected pages into a local
notebook (ADR-0009 §23).

## 9. Courses and assignments

Present when `features` lists `courses` and `assignments` respectively.

### 9.1 Courses

```json
{"id": "0199…", "name": "Mathematik 3B", "groupIds": ["…"], "teacherIds": ["…"],
 "notebookIds": ["…"], "submissionNotebookId": "…", "archived": false}
```

| Call | Who |
|---|---|
| `GET /courses` | anyone: the courses they are a teacher or group member of |
| `POST /courses`, `PATCH /courses/{id}` | teachers; then the course's teachers |
| `POST /courses/{id}:archive` | the course's teachers: every student `write` / `submit` grant on its notebooks becomes `read` (ADR-0009 §26) |
| `POST /courses/{id}/student-notebooks` | the course's teachers. Body `{"title": "Mathe 3B – {displayName}"}`. Creates one empty notebook per student in the course's groups, with the student at `write` and the caller at `owner` (ADR-0009 §17). Idempotent per student |

Creating a course also creates its **Submission Notebook**: `kind: submission`,
the course's teachers at `owner`, and the course's groups at `submit`.

### 9.2 Assignments

```json
{"id": "019b…", "courseId": "0199…", "title": "Aufgabe 3 – Vektoren",
 "dueAt": 1764000000000, "sectionId": "019b…",
 "template": {"notebookId": "…", "pageId": "…"}, "closed": false}
```

`sectionId` is the section that holds this assignment's submissions inside the
course's submission notebook. The teacher's client creates it as an ordinary
`node.upsert` in its own log before creating the assignment.

| Call | Who |
|---|---|
| `GET /courses/{id}/assignments` | the course's teachers and students |
| `POST /courses/{id}/assignments`, `PATCH /assignments/{id}` | the course's teachers. Setting `closed: true` stops further submissions |
| `GET /assignments/{id}/submissions` | teachers: all of them. Students: their own |

*(Informative)* Distributing an assignment is done by the teacher's client. It
copies the template page into each student's notebook, where the teacher is
`owner`, as ordinary ops. The server only records the assignment.

### 9.3 Handing in

Handing in a page is done in two steps (ADR-0011 §3):

1. The student's client copies the page into the submission notebook, as a new
   page with fresh ids whose `parentId` is the assignment's `sectionId`. It
   writes the copy to its own log there and uploads it with `append` (§7.1),
   blobs first.
2. `POST /assignments/{id}/submissions` with
   `{"device": "019a…", "pageId": "019c…"}`.

The server records:

```json
{"id": "019d…", "assignmentId": "019b…", "studentId": "0199…", "version": 2,
 "device": "019a…", "pageId": "019c…", "logLength": 48211,
 "submittedAt": 1763990000000, "late": false, "returnedAt": null}
```

- `logLength` is **the server's own length of that log at the moment of the
  call**, and `submittedAt` is the server's own clock. Neither is taken from the
  client.
- `late` is `submittedAt > dueAt`. Late submissions are accepted and marked
  (ADR-0011 §6). `409 assignment-closed` when the assignment is closed.
- Each call creates the next `version`. Earlier versions are never changed.
- The caller MUST own `device`, and the device's log in that notebook MUST be
  non-empty.

**A submission is pinned to `logLength`.** To show submission version *v*, a
teacher's client MUST read the student's log `device` **only up to
`logLength` bytes**. It MUST apply only ops that create or change `pageId`, or
blocks within it, and MUST ignore any op in that log that targets a page the log
did not create. Ops after `logLength`, and ops aimed at other students' pages,
therefore never show.

*(Informative)* This is why submissions are pinned to a byte offset recorded by
the server rather than to a `seq` the client reports: a tampering client could
later write a line with a smaller `seq`, but it cannot move bytes to before an
offset the server already recorded.

### 9.4 Returning

`POST /submissions/{id}:return`, called by the course's teachers, sets
`returnedAt`. The copy itself is done by the teacher's client: it copies the
annotated page into the student's notebook, where the teacher is `owner`, as
ordinary ops (ADR-0011 §7).

## 10. Error codes

`type` is `https://openote.org/problems/{code}`. Clients MUST branch on the code,
not on the text.

| Status | Code | Meaning |
|---|---|---|
| 400 | `bad-request` | malformed request |
| 401 | `unauthenticated` | no token, or it expired. Refresh and retry |
| 403 | `no-grant` | the caller holds no grant on this notebook (any more) |
| 403 | `level-insufficient` | the caller has a grant, but not a high enough level |
| 403 | `device-not-owned` | the device is not bound to the caller |
| 403 | `device-closed` | the device is imported-and-closed or retired |
| 404 | `not-found` | it does not exist, or the caller may not know it does |
| 409 | `device-taken` | the device id is bound to someone else. Generate a new one |
| 409 | `notebook-exists` | publishing an id that is already on the server |
| 409 | `offset-mismatch` | append at the wrong offset. The body carries the current `length` |
| 409 | `media-exists` | a different file already has this media name |
| 409 | `assignment-closed` | no more submissions are accepted |
| 413 | `too-large` | over `maxRequestBytes` or `maxMediaBytes` |
| 422 | `hash-mismatch` | blob bytes do not match the hash in the URL |
| 426 | `protocol-unsupported` | the client's protocol major is too old |
| 507 | `quota-exceeded` | the notebook or user is over its storage quota |

## 11. Versioning

- **Minor versions are additive**: new endpoints, new fields, new `features`. A
  1.0 client works against any 1.x server.
- **A major version** changes the base path (`/api/v2`). A server MAY serve
  several majors side by side.
- Planned for a later 1.x minor version: a push channel (SignalR or WebSocket)
  that sends the same events as §8; group sync from Entra ID and LDAP (a server
  feature, invisible to the protocol); resumable media uploads.

## 12. Client requirements beyond the wire *(normative)*

These rules do not show up in requests, but a client that skips them is unsafe
against this server.

1. **An op whose `dev` differs from its log's filename MUST be ignored.** The
   file format already says `dev` is always the log's own filename. Against a
   server this rule carries weight: the server binds the filename to a person,
   so `dev` is what "who wrote this" is shown from, and it must not be forgeable
   inside a log.
2. **A notebook held at `read` MUST be opened read-only**, with no local log for
   it. An op written anyway would be refused, and it would stay in the user's own
   log.
3. **A server notebook MUST NOT also be attached to a folder or git transport**
   (ADR-0009 §11).
4. **Shared computers:** signing out SHOULD remove that account's server notebooks
   from the machine when the installation is configured as shared.

## Changelog

- **1.0 draft (2026-11-25)**: first draft, written after the design session
  recorded in ADR-0009 to ADR-0011.
