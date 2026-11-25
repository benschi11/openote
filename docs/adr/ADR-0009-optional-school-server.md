# ADR-0009: An optional school server, as a sync transport that enforces permissions

> **Status:** Accepted as a direction (2026-11-25, grilling session). **Not built.**
> Open questions — the permission model, the unit a permission attaches to,
> accounts — are still being decided and will be recorded here or in follow-up ADRs.
> **Related:** [ADR-0006](ADR-0006-sync-transport-and-text-model.md) (op log; §8 "never talks to a cloud API") · [ADR-0008](ADR-0008-page-protection.md) (why a client-side gate is theatre) · [ADR-0005](ADR-0005-licensing.md) (licensing)

## Context

The ask: store pages centrally for a school, with roles **Teacher** and
**Student**, where some pages are read-only for students and some are visible only
to one student — the shape of a OneNote class notebook. The server is to be
written in C#, and must be **optional**: Openote without it stays exactly what it
is today.

This deliberately departs from ADR-0006 §8, which says Openote never talks to a
network service. That rule was argued against *cloud vendor APIs* (OAuth secrets
in an open binary, four vendor auth flows, it buys nothing). A school server is
different in the one way that matters: it buys something a folder cannot —
**withholding data from a reader**. A shared folder can only give everyone
everything.

## Decision

1. **The server is a transport, not the source of truth.** It receives and
   redistributes the same per-device op logs ADR-0006 §3 defines. Clients stay
   local-first and work offline; the server is another place logs travel through,
   alongside folders and git. ADR-0006 §3 already anticipated this: *"live
   collaboration is a transport swap, not a redesign."*
2. **Permissions are enforced only on the server**, in both directions: a client
   is never sent data its user may not read, and writes its user may not make are
   refused on upload. What the client shows (hidden tools, a read-only badge) is
   convenience, never protection — the lesson of ADR-0008 §1, where anything that
   reaches the client's plaintext SQLite is readable by its user.
3. **One self-hosted instance per school** (or per teacher), shipped as a
   container. No multi-tenancy: pupils' data stays on the school's own
   infrastructure, which keeps the data-protection (GDPR/DSGVO) story simple.
4. **A separate project**, licensed AGPL-3.0-or-later like the app, with client
   changes kept small and optional so they could be offered upstream. Whether it
   is a separate repository is still open, pending a conversation with the
   upstream maintainer.
5. **Sign-in from the start: local accounts and Microsoft Entra ID (OIDC)**, with
   LDAPS designed for but not necessarily built first. Most Austrian schools
   already run Microsoft 365, so a school without Entra is the exception, not
   the rule — but a school without any directory must still work.
6. **HTTPS only.** The client refuses a plain-`http` server address, except for
   `localhost` during development.
7. **Access is granted per user or group, as Read or Write.** Many people can hold
   either on the same material. Read-only is enforced where it matters: the
   server refuses the upload.
8. **Revoking access stops the flow; it does not reach back.** From the moment
   someone loses access, the server sends them nothing new and accepts nothing
   from them. What their device already holds — including the full history of
   the material up to that point — stays there; the server cannot delete it and
   does not pretend to. The UI says so when access is narrowed.

## Considered options

- **Server as source of truth (REST CRUD on pages).** Rejected: it contradicts
  the data model, makes the server mandatory for any notebook it touches, and
  throws away offline editing.
- **Permissions enforced in the client.** Rejected: see ADR-0008 §1.
- **Multi-tenant hosted service.** Rejected for now: a data-protection and
  operations burden a school project does not need.
