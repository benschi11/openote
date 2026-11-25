# ADR-0010: Permissions attach to whole notebooks

> **Status:** Accepted for v1 (2026-11-25, grilling session). **Not built.**
> **Refines** [ADR-0009](ADR-0009-optional-school-server.md).
> **Related:** [File format §3](../specs/10-file-format-spec.md) (op log, `manifest.json` `scope`)

## Context

Teachers want some material read-only for students and some visible to one
student only — and, ideally, to set this per section or page. The op log makes
sub-notebook permissions expensive: each device writes **one** gapless,
never-rewritten log per notebook (file format §3.2). A server can neither serve
a reader that log with someone else's private pages cut out, nor refuse one
forbidden op from a student's log without stalling every allowed op behind it.

## Decision

**In v1, a notebook is the only thing a permission attaches to.** A notebook on
the **School Server** carries one access list of users and groups at Read, Write
or Owner. The server treats op logs as opaque bytes: it checks "may this user
read / write this notebook", never what an op contains. Read-only is a client
with no log of its own in that notebook, and a server that refuses its uploads.

What sub-notebook permissions would have done is done with more notebooks:

- a **Course** groups several notebooks, shown together by the client;
- a student's private area is a notebook per student, created in bulk for a
  group;
- making a page private means **moving it to another notebook** — which Openote
  cannot do yet (`app/lib/api/mcp_tools.dart`: "moving between notebooks is not
  supported") and must learn.

## Considered options

- **Share scopes** — a section or page with its own op logs and access list, the
  client routing each op to its nearest scope. Deferred, not rejected: the
  manifest's `scope` object exists precisely so a section scope can be added as
  a new value without a layout change, and a notebook then simply becomes the
  outermost scope. Revisit when teachers using v1 show per-section rights are
  needed rather than imagined.
- **A server that parses ops and builds per-user filtered streams.** Rejected:
  it needs gaps in `seq` (a format change), a C# reimplementation of the tree
  logic kept in lockstep with Dart and Rust, and an answer to what happens to a
  refused op already written to an append-only log.
