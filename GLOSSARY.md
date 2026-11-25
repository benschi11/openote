# Openote

Openote is an open-source, local-first notebook app. This glossary also covers the optional
**School Server** that lets a school store and share notebooks centrally with role-based access.

## School

**School Server**:
An optional, self-hosted server run by one school, through which **Teachers** and **Students** share notebooks with enforced access rights.
_Avoid_: Cloud, backend, sync server

**Teacher**:
A person on a **School Server** who creates and manages class material and can see the work of their **Students**.
_Avoid_: Admin, owner, Lehrer (in code and docs)

**Student**:
A person on a **School Server** who reads class material and works in the areas a **Teacher** grants them.
_Avoid_: Pupil, user, Schüler (in code and docs)

**Admin**:
A person on a **School Server** who manages accounts, **Groups** and server settings; not a role in any notebook.
_Avoid_: Superuser, root

**Group**:
A named set of **Teachers** and/or **Students** on a **School Server**, typically a class, to which **Grants** can be given.
_Avoid_: Class (in code), team, role

**Course**:
A set of notebooks on a **School Server** that belong together for one subject and **Group**, shown together by the client.
_Avoid_: Class notebook, Kursnotizbuch, workspace

## Access

**Grant**:
The access one person or **Group** holds on one notebook: **Read**, **Write** or **Owner**.
_Avoid_: Permission, ACL entry, share

**Read**:
A **Grant** that lets its holder see a notebook and receive its changes, and nothing more.
_Avoid_: View, read-only access

**Write**:
A **Grant** that lets its holder change a notebook as well as read it.
_Avoid_: Edit, contribute

**Owner**:
A **Grant** that adds, to **Write**, the right to change who holds which **Grant** on the notebook.
_Avoid_: Admin, manager

**Revocation**:
Taking a **Grant** away: from then on nothing new flows to or from its former holder, while what they already received stays with them.
_Avoid_: Unshare, delete access

**Submit**:
A **Grant** on a **Submission Notebook** that lets its holder hand in work and see only what they themselves handed in.
_Avoid_: Write-only, drop box access

## Assignments

**Assignment**:
A task a **Teacher** sets for a **Course**, with a due date and a template page each **Student** works on in their own notebook.
_Avoid_: Task, homework, Aufgabe (in code)

**Submission Notebook**:
The one notebook per **Course** into which **Students** hand in their work for every **Assignment**.
_Avoid_: Drop box, Abgabeordner, inbox

**Submission**:
A frozen copy of a **Student**'s page, handed in for an **Assignment**; handing in again makes a new version, never changes an old one.
_Avoid_: Hand-in, upload, Abgabe (in code)

**Return**:
Sending a **Teacher**'s annotated copy of a **Submission** back into the **Student**'s own notebook.
_Avoid_: Feedback (as a verb), grade
