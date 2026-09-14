# How Deployment Airlock works

← Back to [Deployment Airlock](../../README.md)

The name is load-bearing: in an airlock both doors are never open at once. Nothing moves until the
plan is built and fixed — the `plan` / `execute` split — and one server runs one operation at a time.

What each guarantee means in practice, and where it stops. For every setting mentioned here — its
default, range and how the IDE and project levels merge — see [Settings](settings.md); for
connecting an agent, see [Setup](setup.md).

## Protected servers

Under the default trust mode every upload is confirmed, to any server. The protected-server settings
decide *where*: in a form in the agent's terminal, or in a dialog inside the IDE.

The two are not equal. A form in the terminal proves the answer came from the client, not that a
person gave it — Claude Code, for one, has a hook (`ElicitationResult`) that can answer the form in
the user's place, and Airlock cannot tell the difference. A dialog in the IDE bypasses the agent
entirely: an agent cannot press a button in PhpStorm. So for servers where a human hand is required
— production above all — the confirmation goes to the IDE.

**Every server is protected unless you allowed its host.** The IDE-level page keeps a list of
**Hosts confirmed in the agent's terminal**, empty by default. An upload to a server whose host is
not on it is confirmed in the IDE. The list is filled from the IDE dialog itself: when the host is
the only reason a server is protected, the dialog offers a checkbox to confirm uploads to that host
in the terminal from now on. Tick it for staging, never for production. Hosts are removed on the
IDE-level page.

Why hosts, and why a list of allowed ones rather than protected ones. A project-level server's
name, host and SSH configuration live in files under `.idea`, which an agent can edit. Protection by
name is lifted by renaming the server; protection by a remembered host is lifted by writing the IP
address or another DNS name of the same machine. An allowed list is not: an unfamiliar address goes
to the dialog, and a familiar one leads to the host you allowed yourself. The list lives only at the
IDE level, outside the project.

The list holds hosts, not servers: port, user and directory are not part of it, because an agent
can change those too. If staging and production live on the same machine, allowing that host allows
both. Keep production on a host of its own, or tick it in **Protected servers**, which only keeps it
in the dialog while the project settings file is untouched.

A server is also protected, even when its host is allowed, when either holds:

- it is ticked in **Protected servers** on the project's settings page;
- it is the project's default deployment server and **Always protect the project default deployment
  server** is on (off by default). This is a rule, not a stored name: it follows the default server
  when that changes.

Both live in files an agent can edit when set only in the project, so they tighten, and the host
list is what guarantees. A plan also records which server it was built for: if the server behind the
name changes before the upload, the plan is refused.

Where an upload is confirmed under the default trust mode, **Always ask**:

| Server                                    | Always confirm protected servers in the IDE | Where the upload is confirmed                                                        |
| ----------------------------------------- | ------------------------------------------- | ------------------------------------------------------------------------------------ |
| protected: host not allowed, ticked, or the default server under its rule | on (the default) | a dialog in the IDE |
| protected: host not allowed, ticked, or the default server under its rule | off | a form in the agent's terminal, with the server's name in red |
| host allowed and not otherwise protected | either | a form in the agent's terminal; if the client cannot show forms, a dialog in the IDE |

![Upload confirmation dialog in the IDE for the protected server production: four files with their remote paths, the plan's expiry time and a banner saying the server is protected](img/confirm-upload.png)

Turning **Always confirm protected servers in the IDE** off sends every server whose host is not
allowed to the terminal, and the status widget shows that as a weakened barrier. Trust modes come
before all of this: **Never show the IDE dialog** sends protected servers to
the terminal too, **Auto-approve, never ask** asks nobody, and session trust covers its server until the
project closes.

### Which server an agent can deploy to

The target is the agent's choice, not the IDE's:

- **The agent names the server.** `deployment_plan` takes a server name. The project's default
  deployment server — the one chosen in the IDE's own Deployment settings — is used only when the
  agent names none.
- **Any server the IDE knows can be named.** IDE-level servers are shared by all projects, so that
  includes the servers of your other sites.
- **But the server has to take the files.** It needs mappings in this project that cover the planned
  files; otherwise the plan is refused with `NO_MAPPING`. In practice another site's server, with no
  mapping here, is out of reach — while production, mapped next to a default staging server, is
  not. That is the case protection exists for.
- **An unknown name, or the name of a server group, is `SERVER_NOT_FOUND`.**
- **A server outside *Servers the agent may use* is `SERVER_NOT_ALLOWED`.**

**Servers the agent may use**, in the *Agent scope* group of the project's settings page, narrows
the choice: tick the servers the agent may upload to and download from, and every other one is
refused with `SERVER_NOT_ALLOWED`, the message listing what is allowed. The first entry, *Project
default server*, is a rule rather than a name: it follows the default chosen in Deployment settings
and survives a rename. Nothing ticked means any server. Set in the project, the list guards against
the agent picking the wrong server, not against an agent set on getting around it — the default
server and the project's own servers live in `.idea` files it can edit. The same two levers reach an
IDE-level restriction too: the *Project default server* entry allows whatever `.idea/deployment.xml`
names at the time, and a project server can be renamed into a ticked name — only a list of names set
at the IDE level, naming servers also defined at the IDE level, is proof against the agent. Protection
is separate: it decides who confirms an upload, not where it may go.

This is why **Protected servers** takes several servers rather than one. A project has one default
server, but the agent is not bound to it: protecting only the default would drop the dialog exactly
when the target is another server — you work against staging, the agent names production. So
protection is per server, any number of servers can be ticked, and a server can be ticked before it
is even mapped. The dropdown lists the default server first, then the servers mapped in this
project, then the rest.

The agent is told as well: `deployment_servers` marks a protected server with `protected: true`, the
plan says the confirmation will be requested in the IDE, and a ticked name that no longer matches
any server — a renamed one, usually — is reported both on the settings page and to the agent.

**What protection does not do.** It does not block an upload; it chooses who confirms it. And a list
kept only at the project level is a convenience, not a guarantee: it lives in
`.idea/deployment-airlock.xml`, which an agent can rewrite. The guarantee is what the IDE level
holds — keep **Always protect the project default deployment server** on there.

## Trust modes

Out of the box every upload is confirmed: a form in the agent's terminal, or a modal dialog in
the IDE for protected servers. Confirming twenty uploads an hour is how a barrier gets removed
altogether, so there are three ways to ask less often. All are off by default, all live at the
**IDE level only** — a project's own settings file sits inside the project, where an agent can
edit it — and each one writes its own channel into the audit log.

| Mode                                 | What it does                                                                                                                                                                                                                                       |
| ------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Always ask                           | The default. Every upload is confirmed.                                                                                                                                                                                                            |
| Allow "don't ask again this session" | Adds a third button to the confirmation dialog. Pressing it covers **one server** until the project closes, any Airlock setting changes, or you run **Tools \| Deployment Airlock: Revoke Session Trust**. The plain Upload button grants nothing. |
| Never show the IDE dialog            | Every confirmation goes to the agent's terminal, protected servers included. If that channel is unavailable the upload is **refused**, never sent silently.                                                                                        |
| Auto-approve, never ask              | No confirmation at all. The plan, the allowed paths, the file limit, the content fingerprints and the audit log still apply; only the human leaves the chain.                                                                                      |

Session trust is granted by the dialog button and nothing else — an elicitation form proves the
answer came from the client, not that a person typed it, and the cost of being wrong there is
not one upload but every upload after it.

## The agent's path zone

By default the agent may propose any file inside the project. The **Agent scope** list on the
project's settings page narrows that to the directories a feature actually lives in: add them, and
everything else is out of reach. It is an extra restriction — absent by default, switched on by
adding a path and off by removing the last one — and a mistake here fails in the safe direction,
because an extra path narrows what can be sent rather than widening it.

The check runs where the project boundary already runs, and for the same reason: lexically, before
directories are expanded; by **real** path afterwards, since a symlink such as
`feature/link.php -> ../../../.env` passes only the first check; and once more immediately before
the upload, because the zone can be narrowed while a person is still reading the plan.

| `scope`         | A path outside the zone                                                                                                                                                                      |
| --------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `files`         | The plan is **refused** with `PATH_NOT_ALLOWED`, and the message lists the allowed paths so the agent can correct itself. It named that file: it asked for something forbidden.              |
| `changed_files` | The file is **dropped with a warning**, next to `excluded` and `no mapping`. This scope collects everyone's edits, so refusing would break the feature in exactly the project it exists for. |

Paths are stored relative to the project root — the settings file travels through VCS to machines
where the root is somewhere else. Nothing is pre-filled: a working zone cannot be guessed, and a
guessed one is worse than none.

## Downloading from the server

Some things exist only on the server: a code generator that runs on the site writes its output
there and nowhere else. Airlock can bring that back — `deployment_remote_changes` to see what
differs, `deployment_pull` to download it — and the guarantees are the mirror image of the upload
ones, because the risk is the mirror image too. An upload can overwrite a file on the server; a
download can overwrite work in the project that was never committed anywhere.

So downloading is **off by default**, and the switch that turns it on lives at the IDE level only
— the same reasoning as the trust modes: it widens what the agent may write, and a project's own
settings file sits inside the project.

- **The plan is the comparison.** `deployment_remote_changes` walks the server once and returns
  what it found; `deployment_pull` takes that plan by id. There is no second walk to build a plan
  from, because a second walk would let the person confirm one list while a different one is
  downloaded.
- **Compare cheaply or exactly, and never pretend.** By default files are compared by size, which
  cannot see a change that keeps a file's length — and the answer says so in its warnings. Pass
  `verify: true` for SHA-256 on both sides.
- **Uncommitted work is asked about.** Every local file the plan would overwrite is classified
  first: absent, clean (a copy exists in VCS, so the overwrite is reversible), or dirty. A plan
  that would overwrite anything dirty stops at a modal dialog with three answers — download
  everything, skip the modified files, or cancel — and the dialog lists what is at stake. A file
  with unsaved editor changes counts as dirty, for the same reason a file outside VCS does.

  ![Download dialog for production listing a file with uncommitted changes that would be overwritten, with Cancel, Skip modified and Pull buttons](img/pull-overwrite.png)

- **Or ask about every download, not only that one.** By default a plan of files the project does
  not have yet arrives silently — nothing is overwritten, so nothing is asked. **Confirm every
  download** turns that into a question like any other. It is the one download setting that lives
  at both levels, because it is the one that tightens: a project may switch it on, and cannot
  switch off what the IDE level switched on. On a plan with nothing dirty the dialog offers two
  answers rather than three — "skip the modified files" would skip nothing there.
- **Some names are never a download target**, whatever the server's mapping says: `.idea`, `.git`,
  `.claude`, and agent instruction files — `CLAUDE.md`, `AGENTS.md`, `GEMINI.md`, `.cursor`, `.codex`,
  `.gemini` — at any depth, because a server's copy would become the agent's instructions. The check
  is on the resolved path, so a symlink does not get around it.
- **The path zone applies in this direction too**, and the project boundary is absolute.

## Excluding paths from deployment

![Project view context menu on cart.twig with Exclude from Deployment and Exclude from Deployment on All Servers](img/exclude-menu.png)

Right-click a file or folder in the Project view for two context menu items — **Exclude from
Deployment** and **Exclude from Deployment on All Servers** — that add the selection to the IDE's
own **Deployment | Excluded Paths**, the list that actually stops it from being sent, automatic
upload by ⌘S included. Marking a directory **Excluded** through **Mark Directory As** does nothing
here: the upload path never looks at the project model, only at that list.

The first item writes to the project's default deployment server (every member, if the default is a
group); the second writes to every server configured in the project — servers shared at the IDE
level, with other projects, are left alone. The second item appears only when it would do something
the first does not: with a single project server that is also the default, both would write to the
same list, so only the first is offered. Clicking again removes the entry; when a path is covered
only by an exclusion on a parent folder, the item is disabled and names the parent. This is the one
place Airlock writes to the native deployment settings, and it writes exactly one field: no MCP tool
exposes it, so an agent cannot use it either — it is a human action from the Project view menu.

Every click says what it did, and a write that did not happen says so: if a server's name turns out
to be taken by a group, or the server is gone, the notification is a warning naming the reason
rather than a cheerful count. The routine confirmation and the warnings sit in separate notification
groups — **Deployment Airlock: Activity** and **Deployment Airlock** — so the IDE's own **Settings |
Appearance & Behavior | Notifications** can mute the confirmation after every click without
silencing the warning that nothing was written.

## Seeing the state without opening anything

A status bar widget shows one icon for the current state and spells the whole of it out in the
tooltip; clicking it opens the settings page. Three things it makes visible that nothing else did:

- **the barrier is weaker than default right now** — a trust mode other than "Always ask", session
  trust granted to a server, the IDE's own automatic upload running, downloads allowed without a
  question on each (or overwriting modified files without asking), protected servers confirmed
  outside the IDE, or a run configuration that uploads before it starts;
- **a transfer is in flight**, and in which direction;
- **a server is still locked after a transfer timed out.** Airlock keeps it reserved because it
  cannot confirm the transfer stopped, and the lock clears only when the IDE restarts. Before the
  widget the only way to learn this was to be refused on the next attempt.

![Status bar with the Airlock widget during an upload to production and its tooltip: Uploading to 'production'. Click to open settings.](img/status-widget.png)

The icon carries two independent colours; the tooltip lists everything that is true. The surround —
the rocket's outline on a standard screen, the letter on Retina — says which state won; the rocket
and its flame say which direction is moving. On a standard screen the flame is a droplet that
appears only while a transfer runs; on Retina it is always there, tinted by the state colour when
nothing is transferring. The two used to share one colour, so an upload in flight used to hide a
weakened barrier — that no longer happens, because the colours no longer take turns.

| Icon                                                                                      | Colour    | State                                                                                                                              |
| ----------------------------------------------------------------------------------------- | --------- | ---------------------------------------------------------------------------------------------------------------------------------- |
| <img src="../../icons/airlock@2x.svg" width="18" height="18" alt="">         | Grey      | nothing else below applies                                                                                                         |
| <img src="../../icons/airlockWeakened@2x.svg" width="18" height="18" alt=""> | Orange    | the barrier is weaker than default: a trust mode other than "Always ask", session trust granted, the IDE's own automatic upload, unconfirmed downloads, protected servers confirmed outside the IDE, or a run configuration that uploads before it starts |
| <img src="../../icons/airlockReserved@2x.svg" width="18" height="18" alt=""> | Red       | a server is still locked after a transfer timed out; only an IDE restart clears it                                                 |
| <img src="../../icons/airlockDisabled@2x.svg" width="18" height="18" alt=""> | Pale grey | Airlock is switched off and the four tools refuse                                                                                  |

One state shows at a time, and each row beats the ones above it: a weakened barrier beats nothing
else applying, a locked server beats a weakened barrier, and switched off beats all of it.

| Icon                                                                                      | Colour | Direction                                                                                    |
| ----------------------------------------------------------------------------------------- | ------ | --------------------------------------------------------------------------------------------- |
| <img src="../../icons/airlock@2x.svg" width="18" height="18" alt="">         | —      | nothing is running                                                                             |
| <img src="../../icons/airlockUpload@2x.svg" width="18" height="18" alt="">   | Blue   | an upload is running                                                                           |
| <img src="../../icons/airlockPull@2x.svg" width="18" height="18" alt="">     | Green  | a download is running                                                                          |
| <img src="../../icons/airlockBoth@2x.svg" width="18" height="18" alt="">     | Purple | both directions at once — one server runs one operation, but there is more than one server     |

Switched off shows no direction: a disabled Airlock never starts a new transfer, so there is
nothing to colour — the icon is the same pale grey regardless of what was running a moment ago.

## What Airlock does not control

The IDE has an upload path of its own: **Upload changed files automatically to the default
server**. With it on and external changes allowed, edits made outside the IDE — which is exactly
how an agent makes them — leave for the default server by themselves: no plan, no confirmation, no
audit entry. Neither the path zone nor server protection covers this; they bound Airlock's channel,
not the IDE's.

Airlock cannot stop that channel, and it never learns what left through it — all it can see is
whether the upload listener is running. So it says that, using the platform's own condition rather
than a similar-looking one of its own, which would lie in both directions:

- **to the agent** — a line in the plan's warnings and in `deployment_servers`, whenever automatic
  upload is running;
- **to the person** — the same warning, in the confirmation dialog's warning banner, which renders
  the plan's warnings in full. It shows whenever automatic upload is running, not only when
  external edits are the ones leaving — that is when the dialog's promise that nothing moves until
  you click is false in the literal sense, but there is no separate line reserved for just that
  case; one warning, one place it renders.

The IDE's own MCP server offers more ways to reach a server, and Airlock bounds none of them:

| Channel                                                                                             | What it can do                                                                                                                   | What stops it                                                                                                                     |
| --------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------- |
| `invoke_ide_action` (PhpStorm: added by the bundled PHP plugin; WebStorm 2026.2.2 does not have it) | runs any IDE action by id on files the agent names — the IDE's own upload to the default server (`PublishGroup.Upload`) included | nothing: neither the tool nor the upload action asks. Switch the tool off in **Settings \| Tools \| MCP Server \| Exposed Tools** |
| `execute_terminal_command`, `execute_run_configuration`                                             | a shell command, or a run configuration, that uploads                                                                            | a confirmation in the IDE — as long as brave mode is off                                                                          |
| the agent's own shell                                                                               | `scp`, `rsync` or `ssh` with the keys in your home directory                                                                     | only the client's own permission rules                                                                                            |

The agent instructions template rules out every one of these for deployment. That is a rule, not a
barrier; the barrier is switching the channels off.

Two more paths need no tool call at all, because the agent only edits project files and you do
the uploading:

- **An Upload step before a run configuration starts.** An agent can add one to
  `.idea/workspace.xml` or a file in `.run`; your next press of **Run** uploads without a plan or a
  confirmation. Airlock names every such run configuration in `deployment_servers`, in the plan's
  warnings and in the status widget, which turns orange.
- **A changed server address.** The host of a project-level server lives in `.idea/webServers.xml`,
  and for SFTP often in a project SSH configuration under `.idea`. If an agent points it somewhere
  else, your own uploads from the IDE go there. Airlock checks where its own plans go, not where the
  IDE's uploads go. Session trust is bound to the server's address, so it does not carry over to a
  changed one. Keep production servers at the IDE level, where the agent cannot edit them.
