# Hivemind — Product and Technology Spec

Oct 21, 2026 · refined from `hivemind.md` with @Michael Michailidis

## 1. Purpose, scope and conventions

Hivemind is a port of **orchestra** (v0.9.1, `github.com/noesis-sol/orchestra`) to a desktop app with a headless engine. It works through a backlog of tickets with several Pi coding agents side by side, each in its own git worktree, and merges each finished ticket, checked, into the branch the run started on. The engine runs on the user's machine or on a remote server; the Electron app and a command line attach to it.

This spec fixes the product's behaviour, its architecture and its technology stack. Its reader is an AI agent that will turn it into an implementation plan. It supersedes `hivemind.md` (the technology spec it was refined from), which is kept unchanged as the original; §27 lists every `hivemind.md` `[DECISION]` this spec changes, with the reason.

In scope: behaviour (as a port of orchestra), process boundaries, data stores, contracts, remote operation, packaging, security and the development toolchain. Out of scope: pricing and branding beyond the name.

**Markers**

- **MUST / SHOULD / MAY** follow RFC 2119.
- **\[DECISION\]** is settled. Change it only with a stated reason and a note in §27.
- **\[OPEN\]** is undecided. Propose options with trade-offs and a recommendation; §28 lists them.
- **\[VERIFY\]** is a fact read from docs on 2026-10-21. Check it against the current release or vendored source before building on it.

**Rules for the refining agent**

- Pin every dependency to an exact version. "latest" is never a valid value.
- Do not add a dependency that ships native binaries without an entry in the native inventory (§22.4).
- Every dependency of the **engine** MUST run on its four targets, and every dependency of the **app** on its five (§4). A gap on any target blocks adoption.
- Prefer the vendored source of Effect, Foldkit and Pi over web search when an API is in doubt.
- Where this spec says a behaviour is orchestra's, orchestra 0.9.1's documentation and code are the reference: `docs/running.md`, `docs/workers.md`, `docs/setup.md`, `docs/organs.md`, `docs/dashboard.md`, `docs/events.md`, `skills/orchestra/SKILL.md`, and the source under `internal/` and `cmd/orchestra/`. tutti (`github.com/noesis-sol/tutti`), an earlier TypeScript/Effect port, frozen as a reference on 2026-10-21 (H48), is a second reference: its spec (`.orchestra/specs/tutti-spec.md`), its parity matrix (`docs/parity.md`) and its list of what it still lacks from orchestra (tutti-kkb) record most traps a port meets. Hivemind is new code; nothing is copied from either.

## 2. What Hivemind is

Hivemind keeps orchestra's concepts and changes four things:

1. **Any model.** Workers are Pi agents; Pi's provider layer (`pi-ai`) reaches Anthropic, OpenAI, Google, OpenRouter, local models and others. orchestra is bound to Claude Code and stays so.
2. **An engine that outlives its clients.** One long-lived engine per user per machine holds projects and runs them; runs go on with no client attached. It runs on a laptop or a server, headless.
3. **A desktop app and a CLI as clients.** The Electron app is the rich interface; the CLI does everything but windows. Both speak one typed protocol, locally or over SSH.
4. **Its own backlog.** Tickets live in Hivemind's own work graph (SQLite in the engine, synced through git), behind a tracker interface that other backends (JIRA, Linear, GitHub Issues) may implement later. Beads is not used.

The differentiator over side-by-side agent apps (Conductor, Superset, Nimbalyst, Maestro, Paseo, Codex desktop) is orchestra's: coordination. A dependency-aware backlog, footprints that keep conflicting tickets apart, a merge queue that checks every merge, hand-backs to the worker that wrote the code, questions that block only what they must, and runs that keep going overnight.

### 2.1 Glossary

Unchanged from orchestra unless marked.

| Term | Meaning |
|---|---|
| **Engine** | *New.* The long-lived Hivemind process on a machine (§7). It holds projects, the work graph, runs, workers and organs. |
| **Client** | *New.* The desktop app or the CLI, attached to one or more engines (§8). |
| **Project** | *New as a named concept.* A git repository registered with an engine, with its settings and backlog. |
| **Run** | One pass of the engine over a project's ready tickets, from its start checks to its end line. At most one per project at a time. |
| **Base** | The branch the project's checkout is on when the run starts. Finished tickets land here. |
| **Ticket** | An item in the work graph. Epics are never dispatched; their children are the work. Questions are tickets for the maintainer. |
| **Question** | A ticket of type `question`: a decision only the maintainer can make, with 2 to 4 options and one recommended (§15). It blocks the tickets that wait on it. |
| **Worker** | A Pi agent session working on one ticket in its own worktree (`hivemind/<id>` branch). |
| **Organ** | A one-shot adviser: triage, predictor, reviewer, scout, the spec breakdown. It answers; it never changes code or tickets itself (§16). |
| **Talk** | An interactive session with the maintainer: refining a spec, talking a feature through, answering questions, directing a run (§15). |
| **Footprint** | The files, functions and area labels a ticket names or its worktree changed. Overlapping tickets don't run side by side (§13.4). |
| **Merge queue** | Finished tickets merge one at a time: rebase, setup if needed, merge check, fast-forward (§13.7). |
| **Hand-back** | A rebase conflict or a failing merge check sent back to the ticket's own worker to fix. |
| **Solo** | A ticket labelled `solo` runs alone. |
| **Held back** | A ready ticket waiting for something other than a slot: an overlap, its subtickets, a blocker not merged, a solo ticket. |
| **Set aside** | Deferred, or left for review (`CHECKS_FAILED`, `MERGE_CONFLICT`, `CLOSED_UNCOMMITTED`, …). |
| **Checks** | `check_own` (a worker's check of its change), `check_fast` (the merge check, on every merge), `check_full` (once at a run's end). |
| **Requirements** | The project's `requires` list: what the machine must have (§17). |
| **Hold** | A reason the run takes no new tickets: environment, usage limit, machine budget, a stop, a drain, a budget (§14). |
| **Attention request** | Something waiting on the maintainer right now: a question or a permission request (§15). |
| **Overnight run** | A run started after a pre-flight checklist and its questions answered (§15.6). |

## 3. Decisions

Settled with the maintainer on 2026-10-21, on top of `hivemind.md`'s. Each shapes the sections named.

| # | Decision | Chosen | Sections |
|---|---|---|---|
| H1 | Basis | A new product, **Hivemind**, in fresh code. orchestra 0.9.1 is the behavioural reference, tutti a second reference | §1, §26 |
| H2 | Agent harness | **Pi only** (`hivemind.md` §7). Model agnosticism is Pi's provider layer | §12 |
| H3 | Choosing a model | A **provider and model per project**, overridable **per ticket**, and a **fallback list used only when a provider or model fails to start** (credentials, provider down) | §12.5 |
| H4 | Engine scope | **One long-lived engine per user per machine**, holding many projects and running several at once under one machine-wide worker budget. Clients attach and detach; runs go on without them | §7 |
| H5 | Engine runtime | **Node**: an engine distribution per target, holding a pinned Node runtime, the engine bundle and Pi, the same on a laptop and a server. No Bun | §6, §7, §22 |
| H6 | Client protocol | **Effect RPC, JSON on the wire**, over a local socket, an SSH stdio bridge, or a WebSocket over TLS | §8 |
| H7 | Remote access | **SSH by default** (the engine listens on its local socket only; H30); **TLS with a revocable pairing token** as an option | §9 |
| H8 | Users | **One person per engine, any number of clients at once.** The first answer to a question or permission request wins | §8.5 |
| H9 | Adding a project | **Register a checkout on the engine's host, or clone from a git URL** | §10 |
| H10 | Pushing | A per-project **`push`** setting: `never` (default for a registered checkout), `after_run` (default for a cloned one: Base is pushed once the run's full check passes), `after_merge`. Fetch and fast-forward Base before a run; never force | §10.3 |
| H11 | Ticket backend | A **tracker interface** (`WorkGraph`); the **local tracker** is the only one built. Each adapter defines its own sync | §11 |
| H12 | Local tracker | **SQLite is the engine's live store; a dedicated git ref is the synced record**, one JSON file per ticket, merged per field, continuously | §11 |
| H13 | Coordination | **orchestra's model** (the engine dispatches each ready ticket to a fresh worker; the merge queue is the gate), **plus `hivemind.md`'s typed hand-offs and supervision policies** | §13 |
| H14 | Worker control channel | **The first-party Pi extension's tools** over a per-agent socket: the worker's ticket, filing tickets, questions (2–4 options enforced), notes, and the outcomes `finish`, `defer`, `wait_for`. A turn that ends with no outcome gets one nudge | §12.3 |
| H15 | Standing rules | **Injected by the extension into Pi's system prompt**; rules about an action live in that tool's description and schema | §12.4 |
| H16 | Isolation | **A worktree by default; a container per worker as a project option** | §12.7 |
| H17 | Permissions | **One neutral policy** (`workspace`, `ask`, `full`) enforced by the extension's gate; what it can't decide becomes an attention request; every decision is logged | §12.6 |
| H18 | Organs | Through **`pi-ai` calls**, or **short Pi sessions** where an organ needs tools; each organ may name its own model | §16 |
| H19 | Verifier | **The engine runs it**, as a separate read-only session, preferably with a different model from the worker's | §13.6 |
| H20 | Usage limits | **A usage limit holds the whole run** until it resets, within `usage_limit_wait` | §14.2 |
| H21 | Cost | **Tokens and cost recorded and shown** per ticket, run and model; **budgets optional, off by default** | §14.4 |
| H22 | Engine installation | **The app installs and updates engines over SSH**; a **versioned protocol**, checked at connect | §9.2, §8.4 |
| H23 | Headless | **A CLI in the engine distribution**, for everything but windows; no terminal dashboard | §19 |
| H24 | App, first release | **Runs and workers; backlog and review (read-only); questions, permissions and talks.** Editing the backlog in the app comes later (the CLI covers it) | §18 |
| H25 | Platforms | **Engine: macOS and Linux, x64 and arm64. App: macOS, Linux and Windows x64**; on Windows the app uses a remote engine or one in WSL2 | §4 |
| H26 | Credentials | **In the engine host's own secret store**, entered through the app, used only by the engine's model proxy (H29) | §20.3 |
| H27 | Scope | **orchestra 0.9.1's behaviour, minus what is about Herdr, tmux, cmux, screen reading or Claude Code itself**, delivered in milestones | §13–§17, §26 |
| H28 | UI stack | **Foldkit, `@foldkit/ui`, Tailwind**; Monaco, xterm.js and Cytoscape.js mounted (`hivemind.md` §11) | §18 |
| H29 | Agents' credentials | **A model proxy in the engine**: each agent's provider base URLs point at the engine with a per-agent token; the engine adds the real credential, refreshes OAuth tokens, grants permits and counts usage. No agent ever holds a credential | §12.9 |
| H30 | SSH transport | **An SSH stdio bridge**: `ssh <host> hivemind bridge`, the RPC (NDJSON) over ssh's stdin and stdout. No port opened, nothing forwarded | §8.2, §9.1 |
| H31 | WSL2 | **One more host**: the Windows app installs the Linux engine in a WSL2 distribution and connects with `wsl.exe -d <distro> -- hivemind bridge` | §7.6 |
| H32 | Ticket clocks | **A hybrid logical clock per field** | §11.3 |
| H33 | Sync cadence | **Commit within 2 s, push at once, fetch every 60 s during a run**, at its start and before each claim; every 15 min for a project with no run | §11.3, §11.4 |
| H34 | Two engines | **One engine runs a project at a time**, through a run lease on the tickets ref | §11.5 |
| H35 | Machine budget | **Half the cores, at least 2** | §7.3 |
| H36 | Containers | **Docker or Podman, whichever is present, rootless preferred**, through the Docker-compatible CLI, with limits per container | §12.7 |
| H37 | One worker's trouble | **Always set the ticket aside and go on**; nothing stops the whole run for one worker | §14.5 |
| H38 | Telemetry | **DuckDB embedded in the engine, always on and local**, feeding cost figures, reports and the app's Insights screen; SQLite stays the work graph; export (Parquet, OTLP) optional, metadata only | §21 |
| H39 | Builds | **Per-arch macOS**; no Windows arm64 or Linux armv7l yet; deb and rpm updated by hand | §4, §22.2 |
| H40 | Smaller choices | Cytoscape.js with dagre or ELK for the ticket graph; `tsgo` with `tsc` as fallback; Beads JSONL **import** in v1, export later; the run view at about 20 Hz and token batches every 50 ms | §18.1, §6, §11.6, §8.3 |
| H41 | Offline runs | **Refused unless `--offline`** when the remote can't be reached; on a lease clash after reconnecting, **the later run drains** | §11.5 |
| H42 | Design | **The Hivemind Console handoff** is the visual source: `themes.css` as the only tokens; **Lichen Bright** (default), **Classic** and **Neon Grid**; all dark, no light theme in v1; fonts bundled | §18.2 |
| H43 | Sidebar | **Projects and their runs**, across engines; "+ Add project" | §18.3 |
| H44 | The console's chat | **A run talk**: a Pi session that reads the run and acts through the engine's commands | §15.4, §18.3 |
| H45 | Progress | **Phase steps plus the worker's latest action**; no percentage | §18.3 |
| H46 | Window controls | **Native controls in the design's title bar** (traffic lights inset on macOS, the title bar overlay on Windows and Linux) | §18.3 |
| H47 | Pull requests | **After v1**: a later `push: pull_request` opens one PR per run; v1 merges into Base | §10.3 |
| H48 | Building it | **A new repository built by orchestra from this spec**, M0's spikes first; **tutti frozen** as a reference | §23, §1 |
| H49 | Licence | **Open source under MIT, public releases** on GitHub (`github.com/noesis-sol/hivemind`) | §20.5 |

Kept from `hivemind.md` as decided there: Electron; TypeScript, strict, ESM; Effect v4 at one exact version across every package; SQLite through `@effect/sql-sqlite-node` (`node:sqlite`) for the work graph; Effect's tracing and metrics (sent out only through an optional export, H38); pnpm workspaces and electron-vite; oxlint and oxfmt; electron-builder; testing against Effect test layers; the rules for developing with coding agents.

## 4. Platform targets

\[DECISION\] Each target is built, signed where applicable and smoke-tested in CI.

| Part | OS | Architectures | Formats | Notes |
|---|---|---|---|---|
| Engine | macOS | x64, arm64 | tar.gz (engine distribution) | Developer ID signed, notarised (its Node binary included). Minimum macOS follows the pinned Node's floor \[VERIFY\] |
| Engine | Linux (glibc) | x64, arm64 | tar.gz | Minimum glibc follows the pinned Node's floor \[VERIFY\]. musl (Alpine) out of scope |
| App | macOS | x64, arm64 | dmg, zip | Developer ID signed and notarised. A separate build per architecture (a universal one would carry two engine distributions). Minimum macOS follows the pinned Electron's floor \[VERIFY\] |
| App | Linux (glibc) | x64, arm64 | AppImage, deb, rpm | X11 and Wayland. The AppImage updates itself; deb and rpm are updated by hand (§22.2) |
| App | Windows | x64 | NSIS installer | Authenticode signed. Connects to a remote engine, or to one it runs in WSL2 (§7.6) |

- Code SHOULD stay portable: paths through Effect's `Path`, shell assumptions confined to the engine's process and agent layers.
- The engine is POSIX-only: process groups, `flock`, signals. A native Windows engine is out of scope.
- Windows arm64 and Linux armv7l are not built; they wait for demand (H39).

## 5. Architecture at a glance

```text
┌─────────────── machine A (laptop) ───────────────┐      ┌────────── machine B (server) ───────────┐
│  Hivemind app (Electron)                         │      │  hivemind engine (Node daemon)          │
│   ├─ Main ─ windows, lifecycle, updates          │ SSH  │   ├─ RPC server (socket, bridge, TLS)   │
│   ├─ Connection (utilityProcess) ─ transports ───┼──────┼──►├─ Projects, Runs, Supervisor         │
│   └─ Renderer (Foldkit) ◄─ MessagePort           │      │   ├─ WorkGraph (SQLite + git sync)      │
│                                                  │      │   ├─ Git, Worktrees, Checks, Organs     │
│                                                  │      │   ├─ Model proxy, Telemetry (DuckDB)    │
│  hivemind engine (local daemon) ◄─ local socket  │      │   └─ Pi agents (child processes, one    │
│   └─ Pi agents …                                 │      │      per worker, each in a worktree)    │
│  hivemind CLI ─ local socket / SSH bridge        │      └─────────────────────────────────────────┘
└──────────────────────────────────────────────────┘
```

| Process | Runs | Owns | MUST NOT |
|---|---|---|---|
| Engine | Node, the engine bundle, Effect runtime | Projects, runs, the work graph (sole writer), git and worktrees, workers' processes, organs, talks, the RPC server, the model proxy and the engine's secrets, telemetry (DuckDB) | Render UI |
| Agent (one per worker, organ session or talk) | Pi in RPC mode plus the first-party extension, on the engine's Node | One session in one worktree (or the project's checkout, read-only, for organs and talks) | Open the work-graph database; hold a model credential; talk to other agents |
| App main | Electron main | Windows, menus, app lifecycle, the app's own secrets (pairing tokens, known engines), updates, starting the local engine, spawning the connection process, brokering MessagePorts | Run orchestration logic; sit in the data path |
| App connection | Electron `utilityProcess` | Connections to engines: the local socket, the SSH and WSL2 bridges (`ssh` and `wsl.exe` as its child processes), TLS; relays RPC frames to the renderer | Render UI; touch a repository |
| App renderer | Foldkit app | UI state (the Foldkit Model) | Access Node, sockets, the file system or any database |
| CLI | The engine distribution's `hivemind` command | One command's work against an engine | Hold state between commands (except `follow` and talks while they run) |

## 6. Language, runtime and repository

\[DECISION\] One TypeScript monorepo, one exact Effect version everywhere; Foldkit's pinned Effect peer version sets that number (`hivemind.md` §5).

**Versions**

- Effect v4: every `effect` and `@effect/*` package on the same exact version, following Foldkit's pin \[VERIFY current pin\]. The SQL and RPC modules are marked unstable: pin exactly and keep them behind the project's own services.
- Electron: the latest stable major at project start, pinned.
- **Node (engine):** an LTS version pinned exactly and shipped inside the engine distribution. It MUST provide `node:sqlite`, `child_process` and Unix domain sockets \[VERIFY `node:sqlite` stability in the pinned version\]. The app's renderer and main use Electron's bundled Node, but no orchestration code runs there.
- Pi (`@earendil-works/pi-coding-agent`): exact version, upgraded deliberately with its contract tests (§24).
- Foldkit: exact version, pre-1.0; upgrade with the vendored subtree and `FOLDKIT.md` (§23).

**Toolchain**

- pnpm workspaces. electron-vite builds the app's main, preload, connection process and renderer (with `@foldkit/vite-plugin`) \[VERIFY utility-process entry support\]. The engine is bundled separately (a Node bundle plus Pi, §22).
- Type checking with `tsgo` (TypeScript's native port), which gates the build and runs after every agent's file write; `tsc` kept runnable for any check `tsgo` doesn't do yet; the Effect Language Service in editors \[VERIFY its support for `tsgo`\] (H40).
- oxlint with `@foldkit/oxlint-plugin`, and oxfmt.

**Layout**

```text
apps/desktop/            Electron main, preload, connection process, packaging config
apps/engine/             The engine's entry points: daemon (`hivemind serve`) and the CLI
packages/contracts/      Effect Schema: domain types, the client RPC protocol, extension tool payloads,
                         ticket files, settings, events (depends on effect only)
packages/engine/         Effect services: Engine, Projects, RunLoop, Supervisor, WorkGraph, GitSync,
                         Git, Worktrees, Checks, AgentPool, PiAgent, Organs, Talks, Attention,
                         UsageLimits, Telemetry, Secrets, RpcServer
packages/pi-extension/   First-party Pi package loaded into every agent: tools, gate, rules
packages/client/         The RPC client and connection management, shared by app and CLI
packages/ui/             Foldkit renderer app
repos/                   Vendored git subtrees (Foldkit, Effect, Pi), read-only reference
```

`contracts` is the shared vocabulary: every package imports it; it imports nothing but `effect`.

## 7. The engine

### 7.1 Lifecycle

- \[DECISION\] One engine per user per machine. A second `hivemind serve` finds the first through its socket and its lock (`<data>/engine.lock`, `flock`) and exits, saying so.
- Locally, the app starts the engine when it can't reach one, as a detached daemon from the engine distribution it bundles, or installs it as a login service (launchd agent, systemd user unit) when the user chooses. On a server it is a systemd user unit (with linger enabled, so it runs without a login session \[VERIFY per distribution\]) or a launchd agent.
- Closing every client leaves runs running. Stopping the engine (`hivemind engine stop`, the service manager, SIGTERM) winds its runs down as orchestra's SIGTERM does: no new tickets, running workers' sessions saved, a carry-over for the next start (§13.10).
- Everything with a lifetime is an Effect scoped resource. The engine's root scope ends its runs, then its agents, then its servers.

### 7.2 Data folder

The engine's data folder (`<data>`): `~/Library/Application Support/hivemind/` on macOS, `$XDG_DATA_HOME/hivemind/` on Linux.

```text
<data>/engine.db            Engine state: projects, hosts it pushes to, tokens (hashed), settings
<data>/engine.sock          The local RPC socket (mode 0600), or under $XDG_RUNTIME_DIR on Linux
<data>/engine.lock          The single-instance lock
<data>/projects/<id>/sessions/<ticket>/  Pi's sessions for a ticket (§12.1), for every project
<data>/projects/<id>/repo/  A cloned project's checkout (§10.1)
<data>/projects/<id>/worktrees/  A cloned project's worktrees
<data>/secrets/             The encrypted-file secret store, where no keyring exists (§20.3)
<data>/telemetry.duckdb     The telemetry store (§21)
<data>/logs/engine.log      The engine's own log
```

A project's own files are under `<repo>/.hivemind/` (§10.2).

### 7.3 Projects and runs

- The engine holds any number of projects and runs any number of them at once, one run per project.
- **The machine-wide worker budget** \[DECISION\]: `machine_workers` (engine setting, default half the CPU cores, at least 2, H35: workers mostly wait on the model, but their checks are CPU-heavy, and orchestra's runs slowed sharply past about 4 checks at once on a 10-core laptop) caps the workers live across every run on the engine. A run with a free slot waits while the budget is reached (`HOLD: MACHINE`, §14.3). This replaces orchestra's machine-wide registry: one engine sees all its runs.
- Workers share each machine's cores: each worker's environment carries its share (`HIVEMIND_CORES`, and `GOFLAGS=-p <n>` for Go projects), from the cores divided among the live workers (orchestra-nr1v.14).

### 7.4 Load

The engine samples its host's load every few seconds for the console's system stats (§18.3): each worker's process tree and the merge check's memory, the kernel's memory pressure (macOS) or pressure-stall information (Linux), swapping, and the 1-minute load against the cores (orchestra-recj, -556h).

### 7.5 Logs

- `<repo>/.hivemind/hivemind.log`: the human log per project, orchestra's line formats (dispatching, footprint, settled, merged, hold lines, warnings).
- `<repo>/.hivemind/run/events.jsonl`: the event stream per project (§13.12).
- `<data>/logs/engine.log`: the engine's own log (connections, updates, errors outside projects).

### 7.6 Windows

\[DECISION\] (H31) The app on Windows connects to a remote engine, or runs one in WSL2, which it treats as one more host:

- It looks for a WSL2 distribution, offering to install Ubuntu (`wsl --install -d Ubuntu`) when there is none.
- It installs the Linux x64 engine distribution it embeds inside that distribution, with the steps it runs over SSH (§9.2) run through `wsl.exe -d <distro> --` instead.
- It connects with `wsl.exe -d <distro> -- hivemind bridge` (§8.2), which starts the engine when it isn't running.
- Projects live in the distribution's Linux file system, where git is fast and POSIX semantics hold; the app warns about a checkout under `/mnt/c`.
- WSL2's own lifetime applies: the distribution shuts down some time after its last process ends, so the engine runs as a systemd user unit inside it where the distribution has systemd enabled \[VERIFY WSL's systemd support and idle shutdown with a running service\].

## 8. Client–engine protocol

### 8.1 Protocol

\[DECISION\] Effect RPC (`effect/unstable/rpc`: `RpcServer`, `RpcClient`, `RpcSerialization`) with JSON serialisation, framed as NDJSON on byte streams (the local socket, the SSH and WSL2 bridges). Every request, stream and error is an Effect Schema in `packages/contracts`, decoded on arrival. The schemas are also exported as JSON Schema for clients in other languages.

**Groups**

| Group | Requests | Streams |
|---|---|---|
| Engine | `version`, `status`, `settings`, `stop`, `update` | engine status, load |
| Hosts and tokens | `tokens.create`, `tokens.list`, `tokens.revoke` | |
| Secrets | `credentials.set`, `credentials.list` (names only), `credentials.remove`, `credentials.login` (a subscription's OAuth sign-in, run by the engine, §20.3) | a login's progress (the sign-in URL or device code to show) |
| Projects | `projects.add` (path or URL), `list`, `remove`, `settings.get`, `settings.set`, `init` | project status |
| Backlog | `tickets.list`, `show`, `create`, `update`, `close`, `dep.add`, `dep.remove`, `label`, `note`, `ready`, `why` (what holds a ticket back), `import` (Beads JSONL, §11.6) | backlog changes |
| Runs | `run.start` (mode, flags), `run.stop`, `run.drain`, `run.undrain`, `run.status`, `run.preflight` | the run view (§8.3), the run's events |
| Workers | `worker.message`, `worker.interrupt`, `worker.release` | a worker's transcript |
| Attention | `attention.list`, `attention.answer` | open requests |
| Talks | `talk.open` (mode, context), `talk.send`, `talk.close` | a talk's transcript |
| Review | `diff.ticket`, `diff.run`, `report.get`, `log.tail` | |
| Insights | `insights.summary`, `insights.query` (a named query with its parameters; no SQL from clients) | |

### 8.2 Transports

| Where | Transport |
|---|---|
| Same machine | Unix domain socket `<data>/engine.sock` (or `$XDG_RUNTIME_DIR/hivemind/engine.sock`), mode 0600, through Effect's `Socket` (`RpcClient.makeProtocolSocket`) |
| Remote, default (H30) | **The SSH stdio bridge.** The client runs the system's `ssh <destination> hivemind bridge` (OpenSSH; the user's config, keys and agent). On the host, `hivemind bridge` connects to the engine's local socket (starting the engine when it isn't running) and copies bytes both ways, so the RPC runs over ssh's stdin and stdout. No port opens on either machine and nothing is forwarded; it works the same with Windows' OpenSSH client. SSH keepalives (`ServerAliveInterval`) detect a dead link; the client reconnects and resumes its streams (§8.3) |
| WSL2 (H31) | The same bridge, run as `wsl.exe -d <distro> -- hivemind bridge` |
| Remote, option | WebSocket over TLS on a configured address (`RpcServer.makeProtocolWebsocket`), authenticated with a pairing token (§9.3) |

In the app, the connection process (a `utilityProcess`) owns every transport (it runs `ssh` and `wsl.exe` as its children) and relays frames to the renderer over a `MessageChannelMain` port that main brokers once; main then leaves the data path (`hivemind.md` §6) \[VERIFY M0: Effect RPC over a child process's stdio and over a MessagePort relay\]. The CLI uses the same client package directly.

### 8.3 Streams and the run view

- The engine keeps, per run, a **run view**: an immutable projection of what the console shows (§18.3) (totals, ticket rows, workers, the merge queue, holds, banners, open attention requests, the load). It is written by its own fiber, throttled to about 20 Hz (`view_rate`, an engine setting; H40), with unchanged parts keeping their identity (tutti spec §3.5).
- A stream sends a full snapshot, then patches, each with a sequence number. A client that reconnects asks for patches after its last sequence number; one too far behind gets a new snapshot.
- High-frequency events (tokens, tool output) are batched before they cross, one flush per 50 ms (`stream_flush`, an engine setting; H40; `hivemind.md` §6). Over SSH to a distant host both MAY be lowered per connection. The renderer applies updates at most once per animation frame.
- Transcripts are streamed per worker, only while a client shows that worker.

### 8.4 Versions

\[DECISION\] The protocol carries a version. At connect, client and engine exchange theirs; the engine supports the current version and the one before. A client too old for an engine, or the reverse, is told so and offered the update (§9.2). Persisted data (SQLite, ticket files, settings) is versioned through migrations.

### 8.5 Several clients

\[DECISION\] Any number of clients may attach to an engine at once (H8). They see the same state. A command is applied in the order the engine receives it; the first answer to a question or permission request wins, and the others see it answered, by whom (which client) and when.

## 9. Remote engines

### 9.1 Reaching an engine

- \[DECISION\] An engine listens only on its local socket unless TLS is configured. Clients reach remote engines through SSH: the app's "Add engine" takes an SSH destination (`user@host`, or a host from `~/.ssh/config`), and the connection process runs the SSH stdio bridge (§8.2), one `ssh` process per engine connection, every stream multiplexed over it (H30).
- Host keys are SSH's own (`known_hosts`): a first connection to an unknown host shows its fingerprint for the user to accept; a changed key refuses the connection, as `ssh` does. Interactive prompts (a key's passphrase, a password) are relayed to the app through `SSH_ASKPASS` \[VERIFY on Windows' OpenSSH\].
- The app keeps its known engines (name, SSH destination or TLS address, the engine's ID and protocol version) in its own state; pairing tokens in `safeStorage`.

### 9.2 Installing and updating

\[DECISION\] From "Add engine", the app checks the host over SSH (OS, architecture, an engine already there and its version), uploads or downloads the matching signed engine distribution, verifies its signature, unpacks it to `~/.local/share/hivemind/engine/<version>/`, links `~/.local/bin/hivemind`, installs the user service and starts it. Later versions update it the same way, with the user's consent; the engine finishes or winds down its runs before it restarts (`hivemind engine update` does the same by hand). The same steps install and update the engine in WSL2, run through `wsl.exe` (§7.6). The install script (`curl … | sh`) remains for hosts set up by hand.

### 9.3 TLS and tokens

- `hivemind serve --listen <addr>` (or the engine setting) adds a WebSocket listener over TLS, with a certificate the user provides or one the engine creates and pins (the app trusts it on first pairing, showing its fingerprint).
- `hivemind token create` prints a pairing token once; the engine stores only its hash. Tokens are listed and revoked by name. A token grants everything (H8: one person per engine).

## 10. Projects

### 10.1 Adding a project

\[DECISION\] Either:

- **Register a checkout** already on the engine's host (a path the engine can read; the app browses the host through the engine), or
- **Clone** from a git URL into `<data>/projects/<id>/repo/`, with the host's git credentials.

Either way the engine checks the project as `orchestra init` and a run's start do (§13.1), and offers `init` where `.hivemind/` is missing.

### 10.2 The project's files

```text
<repo>/.hivemind/settings.json   Committed. The project's settings (§13.13)
<repo>/.hivemind/worker-rules.md Committed. The project's own worker rules, added to the standing rules (§12.4)
<repo>/.hivemind/specs/          Committed. Specs from outside the repository, copied in by spec runs
<repo>/.hivemind/work.db         Ignored. The work graph's SQLite store (§11.2)
<repo>/.hivemind/run/            Ignored. Per-run and per-ticket files, the event stream, check logs
<repo>/.hivemind/reports/        Ignored. Run reports
<repo>/.hivemind/hivemind.log    Ignored. The human log
```

Worktrees: `<repo>-worktrees/<ticket>` beside a registered checkout (orchestra's layout), `<data>/projects/<id>/worktrees/<ticket>` for a cloned one; branch `hivemind/<ticket>`. A worktree is kept for review when its ticket is set aside, removed with its branch when it merges \[DECISION; closes `hivemind.md` §17's worktree item\].

### 10.3 Base, fetching and pushing

- \[DECISION\] Before a run, when Base has an upstream, the engine fetches it and fast-forwards Base if the upstream moved and Base has no commits of its own; if they diverged, the run starts on Base as it is and says so (`BASE_DIVERGED`).
- `push` (project setting): `never` (default for a registered checkout, as orchestra), `after_run` (default for a cloned one: push Base once the run's full check passes), `after_merge`. A push that isn't a fast-forward is not made and is reported (`PUSH_REFUSED`); the engine never forces.
- The tickets ref syncs on its own schedule, independent of `push` (§11.4).
- \[DECISION\] (H47) Pull requests come after v1: a later `push: pull_request` pushes a run's Base as a branch once its full check passes and opens one pull request for the run, the run report as its body. v1 merges into Base as orchestra does.

## 11. The work graph

### 11.1 The tracker interface

\[DECISION\] Every ticket operation goes through the `WorkGraph` service (`hivemind.md` §9), the only code that knows the backend. The local tracker (§11.2–11.5) is the only backend built. Adapters for JIRA, Linear or GitHub Issues MAY follow; each defines its own sync, and declares which capabilities it has (questions, typed results, area labels), the engine degrading where one lacks them.

**Interface:** `ready` (orchestra's rule, §13.3), `show`, `list`, `create`, `update`, `claim`, `release`, `close`, `reopen`, `defer`, `undefer`, `addDependency`, `removeDependency`, `label`, `unlabel`, `note`, `fileDiscovered`, `ask` (a question), `answer`, `history`, `changes` (a stream).

### 11.2 The local store (SQLite)

\[DECISION\] SQLite through `@effect/sql-sqlite-node` (`node:sqlite`), one database per project at `<repo>/.hivemind/work.db`, git-ignored, the engine its single writer, WAL, writes that must be atomic in `withTransaction`, forward-only migrations bundled with `fromRecord` or `fromGlob`, a backup before each migration, and a database newer than the engine refused (`hivemind.md` §9). `sql.reactive` streams the backlog and the ready set to the run view.

**Schema** (extends `hivemind.md`'s for orchestra's ticket model):

```sql
CREATE TABLE tickets (
  id TEXT PRIMARY KEY,                -- <prefix>-<short hash>, e.g. hm-a1b2; children <parent>.<n>
  type TEXT NOT NULL CHECK (type IN ('task','bug','feature','chore','epic','question')),
  title TEXT NOT NULL,
  description TEXT, design TEXT, acceptance TEXT,
  status TEXT NOT NULL CHECK (status IN ('open','in_progress','deferred','closed')),
  resolution TEXT CHECK (resolution IN ('done','no_change','cancelled','superseded','answered')),
  priority INTEGER NOT NULL DEFAULT 2 CHECK (priority BETWEEN 0 AND 4),
  defer_until INTEGER,
  assignee TEXT,                      -- the worker's session id while in progress
  model TEXT,                         -- per-ticket provider/model override (H3)
  metadata TEXT NOT NULL DEFAULT '{}',-- JSON: files, spec, predicted_files, result_kind, …
  result TEXT,                        -- JSON hand-off from finish (§13.5)
  options TEXT, recommended INTEGER,  -- questions: 2–4 options, the recommended index
  answer TEXT,                        -- questions: the maintainer's answer
  merged_commit TEXT,                 -- set by the merge queue
  close_reason TEXT,
  created_at INTEGER NOT NULL, updated_at INTEGER NOT NULL,
  clocks TEXT NOT NULL                -- JSON: each field's hybrid logical clock (§11.3)
);
CREATE TABLE labels (ticket_id TEXT NOT NULL REFERENCES tickets(id), label TEXT NOT NULL,
  PRIMARY KEY (ticket_id, label));
CREATE TABLE edges (
  from_id TEXT NOT NULL REFERENCES tickets(id), to_id TEXT NOT NULL REFERENCES tickets(id),
  kind TEXT NOT NULL CHECK (kind IN ('blocks','parent','discovered_from','related')),
  PRIMARY KEY (from_id, to_id, kind));
CREATE TABLE notes (id TEXT PRIMARY KEY, ticket_id TEXT NOT NULL REFERENCES tickets(id),
  author TEXT NOT NULL, body TEXT NOT NULL, at INTEGER NOT NULL);
CREATE TABLE events (seq INTEGER PRIMARY KEY, ticket_id TEXT, type TEXT NOT NULL,
  actor TEXT, payload TEXT, at INTEGER NOT NULL);
CREATE TABLE leases (               -- engine-local; never synced
  ticket_id TEXT PRIMARY KEY REFERENCES tickets(id), engine_id TEXT NOT NULL,
  expires_at INTEGER NOT NULL);
```

**Rules**

- Reject cycles on insert of `blocks` and `parent` edges. Index `edges(to_id)`, `tickets(status, priority, created_at)`, `labels(label)`, `events(ticket_id)`.
- Every mutation appends an `events` row in the same transaction, with its actor (the maintainer through a client, a worker's session, an organ, the engine).
- Rows decode into Schema classes from `contracts`; the same types reach the UI unchanged.
- A lease renews from its worker's activity; an expired lease (the engine crashed) returns the ticket to the carry-over on the next start (§13.10), not straight to `ready`.

### 11.3 The tickets ref

\[DECISION\] The backlog's synced record is a git ref of its own, `refs/hivemind/tickets`, with no history shared with the code. Its tree holds one JSON file per ticket, `tickets/<id>.json` (fields as the schema above, labels and edges inside the file, notes as an array of `{id, author, body, at}`), plus `meta.json` (the ID prefix, the schema version) and `lease.json` (the run lease, §11.5).

- **Clocks** (H32): each field carries the hybrid logical clock of its last change, in a `clocks` object beside the values (`"clocks": {"title": "<hlc>", "status": "<hlc>", …}`). A clock is the wall time in milliseconds, a counter and the engine's ID; an engine's clock never runs behind any clock it has merged, so a host whose wall clock is behind still orders its changes after what it has seen. Two engines changing different fields of a ticket both keep their change; the same field goes to the later clock, ties broken by engine ID, so every engine merges to the same result.
- **Commits** (H33): the engine is the ref's only writer on its host. It batches changes for at most 2 s into one commit on the ref, and commits at once on a run's transitions (a claim, a close, a merge, a question asked or answered, the run lease), with the engine's ID as author.
- SQLite can always be rebuilt from the ref (`hivemind tickets rebuild`); the ref is what travels with the repository.
- Live-only state stays out of it: ticket leases (§11.2), attempt counts, transcripts. The run lease (§11.5) is the one lease on the ref.

### 11.4 Sync and merging

- **Cadence** (H33): when the repository has a remote, the engine pushes the ref after each commit it made (a failed push is retried with backoff; a non-fast-forward is fetched and merged first), and fetches it at a run's start, every 60 s during a run, before each claim, on demand (a client's refresh), and every 15 min for a project with no run.
- **Merging** \[DECISION\]: the engine merges the ref itself (not git's merge machinery), three-way per ticket file against the merge base:
  - a ticket changed on one side only: that side;
  - notes: the union, by note ID;
  - labels and edges: as sets, adds and removes reconciled against the base;
  - plain fields: the later hybrid logical clock wins, ties broken by engine ID;
  - `status`: a fixed precedence where both changed (`closed` over `in_progress` over `deferred` over `open`), a `reopen` after a close winning by its clock;
  - `assignee`: only the engine holding the run lease claims tickets (§11.5); a claim made under a lease that has since expired loses to the current holder's.
  
  The merged state is committed on the ref and applied to SQLite in one transaction. Merging is property-tested (§24).
- Ticket IDs carry a random part, so two engines never mint the same ID \[VERIFY collision rate for the chosen length\].

### 11.5 One engine runs a project at a time

\[DECISION\] (H34) Several engines MAY hold the same project (a laptop's checkout, a server's clone) and edit its backlog, but only one runs it at a time. The **run lease** is `lease.json` on the tickets ref: the engine's ID and name, its host, the run's ID, when it was last renewed, and its expiry.

- A run's start fetches the ref first. If another engine's lease hasn't expired, the run refuses to start (`PROJECT_LEASED`, exit 2), naming that engine, its host and when it last renewed. Otherwise the engine writes its own lease in a commit and pushes it at once; if the push loses to another engine's lease (the merge keeps the earlier lease), the start is refused the same way.
- The holder renews the lease every minute, with its regular sync, and removes it when the run ends. A lease not renewed for 5 minutes has expired (its engine crashed or lost the network); the next run may take it over and says so (`LEASE_TAKEN_OVER`); a holder that finds its lease taken over stops dispatching and drains.
- A claim is a field on the ticket (`assignee`: the engine's ID and the worker's session); only the lease's holder writes it.
- Edits from an engine not holding the lease (the maintainer filing tickets on the laptop while the server runs) sync as any change does; the holder sees them within a minute.
- The app shows, on every engine holding the project, which engine is running it.
- With no remote, the lease lives on the local ref and only guards engines sharing that checkout.
- **Without the remote** \[DECISION\] (H41): an engine that can't reach the remote at a run's start can't see a lease held elsewhere, so the run refuses to start (`REMOTE_UNREACHABLE`, exit 2, with the remote's error) unless `--offline` is given. An offline run's start line and the app say its lease is unchecked; the engine pushes the lease as soon as the remote answers.
- **A clash on reconnecting** (H41): if another engine's lease covers the same time, the lease with the earlier start wins. The later run stops dispatching, finishes and merges what it has running, and ends `DRAINED`, naming the other engine (`LEASE_CLASH`). A ticket both runs worked on is reconciled by the ref's merge rules (§11.4; a close wins), and the losing branch is kept for review.

### 11.6 Interop

- \[DECISION\] (H40) `hivemind tickets import --beads <file>` brings a Beads backlog (a `bd export` JSONL file, such as orchestra's and tutti's) into a project's work graph, mapping types, statuses, priorities, labels, dependencies (blocks, parent-child, discovered-from, related), notes, design, acceptance and metadata; a Beads ticket labelled `human` with options in its metadata (orchestra-tnei.1) becomes a `question` \[VERIFY Beads' JSONL fields at the version in use\]. IDs keep their prefix. An import into a project with tickets merges as §11.4 does. Export to Beads comes later, when someone needs it.
- Keep all SQL inside the local tracker's implementation and within SQLite's dialect, so a later engine (Turso Database, libSQL) is a Layer swap (`hivemind.md` §16).

## 12. Workers

### 12.1 Process model

\[DECISION\] Each worker is a Pi process in RPC mode over stdio, owned by the engine as an Effect scoped resource behind a `PiAgent` service; Pi's Promise and RPC surface never leaks past it (`hivemind.md` §7).

- Spawn: the engine distribution's Node running Pi's CLI as `pi --mode rpc --extension <the first-party extension> --session-dir <data>/projects/<id>/sessions/<ticket>/`, cwd the ticket's worktree, in a process group of its own. RPC mode takes JSONL commands on stdin and writes JSONL records on stdout, correlated by `id`; records are split on LF only. Every external command (git, checks, setup) is started in its own group the same way, so a terminal's or a service manager's signals reach the engine alone.
- Each agent's Pi gets a configuration folder of its own, holding only the model proxy's provider entries (§12.9); the user's own Pi configuration and credentials are never read \[VERIFY how Pi's configuration folder is set per process\].
- The engine drives an agent through RPC commands: `prompt` (the first prompt, a hand-back), `steer` (a maintainer's message), `abort`, `set_model`, `switch_session` (resume), `fork`, `get_session_stats`.
- Acquire = spawn + handshake. Release = `abort`, then terminate, then reap. Effect interruption maps to `abort`.
- A reaper per worker (as tutti's spec §3.2): if the engine dies without its finalizers (SIGKILL), each worker's group is terminated within seconds.
- A ticket's sessions live in its session folder; a worker's session is resumed with `switch_session` for its hand-backs, after a question is answered, and in the carry-over.
- Only the first-party extension loads. It takes part in Pi's `project_trust` step, which runs before a project's own extensions load, and refuses a project's own `.pi/` settings and packages until the user trusts that project (`hivemind.md` §12).

### 12.2 The first-party extension

| Integration | Pi API | Purpose |
|---|---|---|
| Tools | `pi.registerTool()` (TypeBox schemas) | The worker's control channel (§12.3) |
| Gate | `tool_call` event, which can block a call (`{block: true, reason}`) or change it; a handler that fails blocks the call | The permission policy (§12.6): path confinement, a command policy, blocking `git push`, approvals |
| Standing rules | `before_agent_start` event, editing `systemPromptOptions` | §12.4 |
| Settling | `agent_before_settle` (can ask for one more model call), then `agent_settled` | The nudge for a turn with no outcome (§12.3); tells the engine the agent has settled |
| Trust | `project_trust` | Refuses a project's own extensions until it is trusted (§12.1) |
| Metadata | `pi.appendEntry()` | Orchestration data in the session without spending context |
| Usage | message events | Tokens and model per response, a cross-check of the model proxy's count (§12.9) \[VERIFY the usage fields\] |

Model calls need no hook: they pass through the model proxy (§12.9), which grants permits and counts usage, so the extension doesn't use `before_provider_request` (which providers registered by extensions bypass, Pi issue #7368).

The extension's tools forward to the engine over a per-agent Unix domain socket whose path and token arrive in the agent's environment, speaking the Schema RPC protocol; the token scopes the session to its ticket \[DECISION; resolves `hivemind.md` §17's channel item\]. Tool schemas have one source in `contracts`: each tool's Effect Schema is turned into JSON Schema and handed to Pi wrapped as a TypeBox schema (TypeBox schemas are JSON Schema, `Type.Unsafe`); arguments are decoded again with the Effect Schema in the handler \[VERIFY Pi validates the wrapped schema as given\].

### 12.3 The control channel

\[DECISION\] Everything a worker tells the engine goes through these tools; nothing is read from its final message (it is never read).

| Tool | Does | Enforced by its schema and handler |
|---|---|---|
| `get_ticket` | The worker's ticket: description, design, acceptance, notes, labels, the results of the tickets it depends on (§13.5) | Only its own ticket and those linked to it |
| `file_ticket` | A follow-up or a discovered issue, linked (`blocks`, `discovered_from`, `related`) | A short title (≤ 60 characters); a follow-up that builds on this ticket blocks on it |
| `ask_question` | A question for the maintainer, blocking this ticket or another one | 2 to 4 options, one recommended, each a label and a one-line trade-off (orchestra-tnei.1). A decision about another ticket blocks that ticket |
| `add_note` | A note on its ticket | |
| `finish` | Ends the work: commits are on the branch, checks passed | The worktree is clean (no uncommitted or untracked files; §13.7 names them otherwise), `check_own` passed at HEAD (its commit and output file given), a result payload valid for the ticket's kind (§13.5) |
| `defer` | Ends the turn without finishing: a reason, the work committed so far | |
| `wait_for` | Ends the turn: another ticket must merge first, with what this one needs from it | The ticket is linked as blocked; nothing uncommitted |
| `ask_and_wait` | `ask_question`, then holds the agent on a `Deferred` up to `question_wait` (default 15 min) for the answer; past it, ends the turn as asked | As `ask_question` |

- An invalid payload fails the tool call inside Pi's loop, so the agent retries with the decode error in its context (`hivemind.md` §8).
- A turn about to end with no outcome tool gets one nudge, inside Pi: the extension's `agent_before_settle` handler asks for one more model call with the nudge ("call finish, defer, wait_for or ask_question"). A second settle with no outcome ends the session as unfinished: the ticket is set aside with a note (§14.5).
- The extension's own tools are always allowed by the gate.

### 12.4 Standing rules

\[DECISION\] The extension injects the standing rules into Pi's system prompt, so they survive compaction: orchestra's worker prompt, adapted (stay in the worktree, never push, run checks once in the foreground with their output kept, give a check its time, tests in new files named after the feature, changelog entries as new lines, throwaway files in a scratchpad, stop only through an outcome tool). The project's `.hivemind/worker-rules.md` is appended. Rules about an action live in that tool's description, not in the prompt. The template's version is recorded per project (`settings.templates`), and an older project's rules are flagged at a run's start (orchestra-nr1v.3).

### 12.5 Models

- \[DECISION\] `model` (project setting): a provider and model (`anthropic/claude-…`, `openai/…`, `openrouter/…`, `ollama/…`) and, where the provider takes one, a thinking or effort level; a ticket's `model` field overrides it.
- `model_fallbacks`: models tried in order when the chosen one fails to start (missing or rejected credentials, the provider unreachable). A usage limit is not a start failure (§14.2). A ticket's hand-backs and resumed sessions always use the model that wrote it.
- No credential reaches an agent: every model call goes through the engine's model proxy (§12.9), which holds the credentials from the secret store (§20.3).

### 12.6 Permissions

\[DECISION\] `permissions` (project setting) is enforced by the extension's `tool_call` gate:

| Level | Allowed | Asks the maintainer | Always refused |
|---|---|---|---|
| `workspace` (default) | Reads anywhere; writes and commands inside the worktree; network for package managers and the model | Writes outside the worktree, network-reaching commands not on the list, anything the command policy marks risky | `git push`, history rewrites of other branches, removals outside the worktree, `sudo` |
| `ask` | Reads | Every write and command | As above |
| `full` | Everything but the refusals | Nothing | `git push` |

A request the gate can't decide becomes an attention request (§15.1); every decision (allowed, asked and answered, refused) is logged and counted for the reviewer. `full` SHOULD be used only with containers (§12.7).

### 12.7 Isolation

\[DECISION\] A worktree per worker by default. A project MAY set `container` (an image, or a devcontainer file whose image and setup commands are read \[VERIFY which devcontainer properties to honour\]): each worker then runs in a container with its worktree mounted, Pi and the extension inside on the Linux engine distribution's Node (mounted read-only; a macOS engine fetches the Linux distribution for its architecture once), the network as configured. The engine, its model proxy and the extension's socket stay outside, the socket mounted \[VERIFY reaching the model proxy from inside a container on each runtime\].

- **Runtime** (H36): Docker or Podman, whichever is on the host, driven through the Docker-compatible CLI (`docker` or `podman`: run, exec, mounts, networks, the same flags for what the engine needs); `container.runtime` names one where both are present. Rootless is preferred: rootless Podman or rootless Docker on Linux, Docker Desktop or Podman Machine on macOS; a notice says when only a rootful daemon is found.
- **Limits:** every container gets memory and CPU limits (`container.memory`, `container.cpus`; default the worker's share of the machine, §7.3), so containers can't take the host as an unbounded VM can.
- Images are pulled before a run's first dispatch; a container is removed with its worker.

### 12.8 Supervision

\[DECISION\] A `Supervisor` fiber per worker applies `hivemind.md` §8's policies, adapted:

| Failure (tagged error) | Policy |
|---|---|
| `RateLimited` (a provider's rate limit, no reset known) | Retry with exponential backoff (`Schedule`) |
| `UsageLimited` (a quota with a reset) | The usage-limit hold (§14.2) |
| `AgentStuck` (no events for `stuck_after`, default 10 min, not in a tool known to be long) | Interrupt; restart the session once with a summary; then defer |
| `ToolDefect` | Defer the ticket; file a follow-up about the tool |
| `StartFailed` | The model fallbacks (§12.5); then the environment hold's count (§14.1) |
| `BudgetExceeded` | Interrupt the ticket's scope; set it aside (§14.4) |
| `HumanRejected` | Return the ticket to the queue with the reviewer's note |

### 12.9 The model proxy

\[DECISION\] (H29) Agents never hold a model credential; OAuth tokens expire within hours, and a worker can run longer. The engine runs a **model proxy**, and each agent's Pi is configured at spawn with every provider's base URL pointing at it and a per-agent token in place of a key \[VERIFY Pi takes a base URL override for each built-in provider\].

- **Where:** a loopback HTTP listener on the engine's host (127.0.0.1, an ephemeral port). The per-agent token is the only way in; it dies with the agent's session.
- **Each call:** the proxy checks the token, takes a permit (the usage-limit hold, §14.2; per-provider concurrency caps; budgets, §14.4), swaps the token for the current credential from the secret store (§20.3), and streams the provider's response back unchanged.
- **OAuth:** the proxy refreshes a subscription's token itself before it expires, through `pi-ai`'s OAuth refresh, one refresh per account at a time, writing the new token to the secret store. Where a provider's subscription requests differ from its API-key requests (Anthropic's, for one), the agent's Pi is given a placeholder token of the subscription kind, so it builds the subscription's request and the proxy swaps only the token \[VERIFY per OAuth provider\].
- **APIs:** the proxy passes through each provider API Pi uses (Anthropic Messages, OpenAI Responses and Chat Completions, Google), changing only authentication headers. Local models (Ollama, LM Studio) go through it too, with no credential, for the accounting.
- **What it reads:** each response's usage (tokens in, out and cached, from the API's final event), attributed by token to the agent's ticket, run and model (§14.4, §21); and refusals: a 429 with a reset time starts the usage-limit hold (§14.2), one without starts the `RateLimited` backoff (§12.8), a rejected credential is a `StartFailed` before the first token and a notice after.
- Organs' calls from the engine itself (§16) take the same permits and accounting.

## 13. The run

Behaviour is orchestra 0.9.1's unless this section says otherwise; the references are `docs/running.md` and `docs/workers.md`.

### 13.1 Starting

- **Start checks**, all problems listed together, exit code 2 (`SETUP`) when any: the project's checkout clean and on a branch; no other run of the project, on this engine or another (the run lease, §11.5; the remote reachable, or `--offline`); settings valid; the requirements list passing (§17); models' credentials present; the templates' versions (a notice).
- **Modes:** the current tickets (`--tickets`, the default), one ticket and its descendants (`--ticket <id>`), a spec (`--spec <file> [--refine]`), a feature (`--feature "<text>"`), an overnight run (`--overnight`, §15.6).
- **No ticket limit by default**; `--limit N` caps a run's dispatches (orchestra-5ls7). `--concurrent N` caps its workers (also bounded by the machine budget).
- **Nothing ready but questions:** the run opens the answer talk instead of ending (§15.3); nothing ready at all: `READY_EMPTY after 0 tickets`.

### 13.2 Dispatch

- Free slots are filled at once and on every change: a worker settling, a merge, an answer, the ready set moving.
- A ticket is claimed (§11.5), given a worktree from Base (created, or reused and rebased), set up (§13.8), and handed to a fresh worker with its first prompt (the ticket's ID and that its rules are in the system prompt).
- Its first prompt adds what the run knows: an earlier attempt's notes, a spec's sections it cites, the hand-off results of the tickets it depends on.

### 13.3 Ready

orchestra's rule \[DECISION\]: open, not deferred (or past `defer_until`), not an epic or a question, every `blocks` blocker **closed and merged** (`merged_commit` set, or its branch on Base, as git says; a blocker closed but not merged holds it: `CLOSED_UNMERGED`, orchestra-lx4d.1), the ticket it was filed from merged, and not set aside in this run. An epic closes itself once its children are closed and merged.

### 13.4 Footprints, holds and solo tickets

orchestra 0.9.1's rules \[DECISION\]:

- A footprint is the files, functions and area labels a ticket names in its title, description, design and acceptance (never its notes), its `files` metadata, the predictor's guess for a ticket naming nothing (§16), and, while it runs, the files its worktree changed.
- Two tickets overlap only on a file one of them is to change (its `files` metadata lists it, its text asks to change it, or its worker changed it); files git merges by union and the check commands' files never count (orchestra-jocn, -kmcz).
- A ready ticket overlapping a running one, a ticket merging, or one waiting to merge is held back; so are ready tickets while a `solo` ticket runs or waits first in line.
- Tickets past their work (waiting to merge, merging, resolving, fixing) keep their footprint and their holds until they merge, but not their slot: at most `concurrent` such tickets at once (orchestra-z8lr).

### 13.5 Outcomes and hand-offs

- `finish` → the verifier (§13.6), then the merge queue.
- `defer` → set aside, its worktree kept, its notes saying why; triage reads it (§16).
- `wait_for` → back in the queue until that ticket merges (`WAITS_FOR_MERGE`), the slot freed and the dispatch number reused.
- `ask_question` (own ticket) → the ticket waits on the question; its session is resumed with the answer.
- **Typed hand-offs** \[DECISION\]: `finish` carries a result: a summary, plus fields per `result_kind` (metadata) defined in `contracts` (e.g. an API contract changed, a migration added, a decision recorded). Tickets depending on it get these results in their first prompt and through `get_ticket`.

### 13.6 The verifier

\[DECISION\] When a worker calls `finish`, the engine starts a verifier: a separate Pi session in the worktree, read-only (Read, Grep, Glob, Bash for reading and running tests), given the ticket and the diff but none of the worker's reasoning, using `verifier.model` (default: a different provider's model from the worker's where one is configured, else the same). It reuses the worker's passing `check_own` at the same commit rather than running it again, and checks the suites on what the change touches. It answers through its `verdict` tool (`PASS`, or `FAIL` with findings). A FAIL goes back to the worker, the earlier rounds' findings carried, for at most `verifier.rounds` (default 3); a later round that disagrees with an earlier one says so (`DISAGREES WITH ROUND <n>`), and the worker asks the maintainer rather than going round again (orchestra-nr1v.12). `verifier.enabled` per project (default on).

### 13.7 The merge queue

orchestra 0.9.1's path \[DECISION\]:

1. Take the queue (an Effect `Queue` with one consumer).
2. If the branch's commits change nothing: close the ticket as `no_change`.
3. Rebase onto Base when Base moved; on a conflict, hand it back (`RESOLVING`, up to `hand_backs.resolve`, default 2, each within `resolve_timeout`, default 20 min).
4. Run the setup if the rebase changed a `setup_files` file (§13.8).
5. Run `check_fast` on **every** merge, a branch already on top of Base included (orchestra-hm5e). A failure in the ticket's own code hands it back (`FIXING`, up to `hand_backs.fix`, default 2); a worker that commits nothing gets one more check (orchestra-2jwy); a failure outside the ticket's code sets it aside to be rechecked once Base moves (`CHECKS_FAILED`, `RECHECK`).
6. Fast-forward Base, record `merged_commit`, close the ticket, remove the worktree and branch, push when `push: after_merge`.

- **The merge train** (orchestra-8qk2): while one ticket's check runs, the next queued ticket is rebased onto the first's rebased head in its own worktree and checked beside it (`merge.train`, default 1; none while the machine budget holds); if the first merges as checked, the second merges with no further check.
- **Hand-backs** (orchestra-nr1v.1, .2, -cgko): one check per hand-back, after rebasing onto the current Base; the prompt asks for what failed and the tests of the change, not the whole check; the hand-back's clock stops during the usage-limit hold and while the machine is held for load; at the limit or the deadline a finished rebase or a fix commit is taken from git, and otherwise the worker is interrupted and told the hand-back is over before its branch is put back (its attempt kept).
- Only the step from the fast-forward to the end of its notes is uninterruptible, with each git command's time limit restored inside it (tutti spec §3.4). An interrupted rebase runs `git rebase --abort`; an interrupted check leaves the ticket waiting to merge; a stop never rebases a ticket that hasn't started its merge (orchestra-thcc).
- `CLOSED_UNCOMMITTED`: a ticket whose worktree has uncommitted or untracked files can't `finish`; were it to reach the queue so (a carry-over), it is set aside with the files named (orchestra-gsaa).

### 13.8 Setup

orchestra 0.8.2–0.9.0's rules \[DECISION\]: the project's `setup` runs in each new worktree before its worker starts, and again in a reused one whose rebase changed a setup file; within the check's time limit; its output kept. A failing setup puts the ticket back in the queue until a merge or a probe (`SETUP_FAILED`), counts toward the environment hold when it fails ticket after ticket, and names its tickets at the run's end (`SETUP_WAITS`). A setup cut short is run again. A conflict hand-back after a rebase that changed a setup file runs the setup first.

### 13.9 The full check and the run's end

- A run that ended by itself and merged something runs `check_full` once on Base, in a fresh worktree with setup; a failure files a ticket.
- The end line: `READY_EMPTY`, `LIMIT_REACHED`, `DRAINED`, or the stop's kind, then what is left (`SETUP_WAITS`, `SCOPE_OPEN`, the questions waiting).
- The reviewer writes the run report (§16) to `.hivemind/reports/`, shown in the app.
- `push: after_run` pushes Base once the full check passed.

### 13.10 Stops, drains and carry-over

- **Drain** (`s` in the app, `hivemind run drain`): no new tickets; the running ones finish and merge; the run ends `DRAINED`.
- **Stop now** (`hivemind run stop`, the app's Stop): workers interrupted, their sessions saved; the run ends `INTERRUPTED`; a merge in its uninterruptible step finishes first.
- **Carry-over**: the workers and tickets a stopped or crashed run left, saved in `.hivemind/run/state.json` and the leases; the next run resumes their sessions (Pi's session resume) rather than starting over; a carried ticket already on Base is dropped with its worktree and branch removed (orchestra-nr1v.8).
- A ticket merged by hand during a run is noticed as Base moves: its label dropped, what it blocks freed, counted closed (orchestra-nr1v.9).

### 13.11 Exit codes and kinds

The CLI's exit codes and the event stream's stop kinds are orchestra's, minus the terminal's: 0 (done, drained, limit, nothing to run), 2 (setup problem, failed pre-flight, requirements missing, the project leased by another engine, the remote unreachable without `--offline`), 3 (not used: orchestra's stop for a stuck worker; Hivemind sets such a ticket aside, §14.5; the number stays reserved so the others match orchestra's), 4 (a git or tracker failure, an engine panic), 5 (the checkout dirty or off its branch), 6 (a branch that doesn't fast-forward), 7 (the environment hold gave up), 8 (the usage limit outlasted `usage_limit_wait`), 130 (stopped). `hivemind run start --wait` waits for the end and exits with them; without `--wait` the CLI returns once the run starts.

### 13.12 The event stream

`.hivemind/run/events.jsonl` and the `run events` RPC stream: orchestra's record kinds (`start`, `dispatch`, `queue`, `settled`, `closed`, `deferred`, `asked`, `answered`, `hold`, `warn`, `done`, `end`, …) with orchestra's fields where they apply (`docs/events.md`), versioned, plus `model`, `tokens`, `cost` and `engine` fields. Defined in `contracts`.

### 13.13 Settings

`.hivemind/settings.json` (committed), with the engine's settings for machine-wide values:

| Setting | Default | |
|---|---|---|
| `concurrent` | 2 | Workers for this project's runs |
| `model`, `model_fallbacks` | none | §12.5 |
| `permissions` | `workspace` | §12.6 |
| `container` | none | §12.7 |
| `check_own`, `check_fast`, `check_full` | from `init` | §13.7 |
| `check_timeout` | 30m | |
| `setup`, `setup_files` | from `init`; orchestra's default list of manifests and lockfiles | §13.8 |
| `requires` | `[]` | §17 |
| `push` | `never` / `after_run` | §10.3 |
| `verifier` | `{enabled: true, rounds: 3}` | §13.6 |
| `hand_backs`, `resolve_timeout` | 2, 2, 20m | §13.7 |
| `merge.train` | 1 | |
| `ticket_limit` | none | A worker past it is interrupted and its ticket set aside (§14.5) |
| `usage_limit_wait` | 6h | §14.2 |
| `question_wait` | 15m | §12.3 |
| `budgets` | none | §14.4 |
| `organs` | per organ: `{enabled, model}` | §16 |
| `container.runtime`, `container.memory`, `container.cpus` | whichever is present; the worker's share | §12.7 |
| `questions` | `auto` | §15.3 |
| `templates` | the versions `init` wrote | §12.4 |

Flags, then environment variables (`HIVEMIND_…`), then the project's settings, then the engine's, resolved once per run into one value; every problem listed (tutti spec §3.3 `AppConfig`).

## 14. Holds

### 14.1 The environment hold

orchestra's: when several tickets in a row settle within minutes without claiming or changing anything, or their setups fail one after another, the run holds (`HOLD: ENVIRONMENT`), probes the machine with a minimal agent after `env_probe` (10 min), and goes on (`PROBE_OK`) or ends (exit 7). A ticket stopped by a closed-but-unmerged blocker doesn't count.

### 14.2 The usage-limit hold

\[DECISION\] A model call refused for a quota with a reset time (a subscription's session limit, a provider's quota), from a worker or an organ, holds the **whole run**: no new tickets; running workers wait (their hand-back clocks stopped) and are told to carry on at the reset, naming an open hand-back; the hold lifts at its reset time whether or not a worker sees it lift (orchestra-8jw3). Past `usage_limit_wait` the run ends (exit 8), its workers kept for the carry-over. The model proxy (§12.9) sees the refusal first-hand and grants no permit for that provider while the hold is on, so no other agent spends a call finding the limit.

### 14.3 The machine budget

`HOLD: MACHINE` while the engine's live workers reach `machine_workers` (§7.3); lifted when one ends. The console's system stats say when the machine itself is overloaded (§18.3).

### 14.4 Cost and budgets

\[DECISION\] Tokens and cost per model call (counted by the model proxy, §12.9, priced by `pi-ai`'s tables \[VERIFY\]; a subscription's calls are counted in tokens, with their API-price equivalent shown as such) are recorded per ticket, run, model and project in telemetry (§21), and shown in the app, the report and the event stream. `budgets` (per project, run or ticket, in tokens or money) are optional and off by default: a ticket over its budget is interrupted and set aside (`BudgetExceeded`); a run over its budget stops dispatching (`HOLD: BUDGET`).

### 14.5 One worker's trouble

\[DECISION\] (H37) Conditions that stop the whole run in orchestra (a worker gone twice, blocked on a dialog, its state unknown) can't happen to Pi workers owned by the engine. What remains is one ticket's trouble, in every run, overnight or not: a ticket past `ticket_limit`, or a worker that settles twice with no outcome (§12.3), is set aside (its worktree and session kept for review, a note saying why), triage reads it (§16), the app notifies, and the run goes on. Nothing stops the whole run for one worker; run-wide conditions (the environment hold, §14.1; the usage limit, §14.2) still can.

## 15. Questions, permissions and talks

### 15.1 Attention requests

- **Questions:** tickets of type `question` with 2–4 options, one recommended, blocking the tickets that wait on them. Filed by workers (`ask_question`), organs, talks or the maintainer.
- **Permission requests:** from the gate (§12.6): the tool call, its arguments, why it asks; answered allow once, allow for this run (a run-scoped rule), or refuse with a note.
- Both are listed by `attention.list`, shown in every client, and answered in the app, in the CLI (`hivemind answer`), or by an agent through the CLI. The first answer wins (§8.5).

### 15.2 Answering a question

The app shows the question with its options, the recommended one first and marked, and a free-text line; "Talk it through" opens the answer talk for that question. The answer is recorded on the question (closing it as `answered`) and written into the blocked tickets' design, so their workers see it; the blocked tickets become ready, and a ticket whose own session asked is resumed with the answer.

### 15.3 The answer talk

\[DECISION\] A talk (§15.4) given the open questions in scope, each with its options (drafted by the talk, and marked so, for a question filed without them), the tickets it blocks, the ticket it was asked from and their notes; read-only tools plus the engine's question tools. It shows each question's options first, talks it through when asked, records the answer, and goes on to the next. With `questions: auto` (default), the app opens it by itself, without taking focus, as soon as a question in a watched run's scope is open; questions arriving meanwhile queue into the same talk; `questions: key` opens it only on the maintainer's action (orchestra-tnei.6). A run with nothing ready but questions opens it instead of ending, and takes up the freed tickets as they are answered; deferred tickets are mentioned, never released by it (orchestra-tnei.5).

### 15.4 Talks

Talks are Pi sessions the engine runs, in the project's checkout, with the maintainer, streamed to a client's chat pane (or the CLI's terminal), with the engine's ticket tools available to file what they decide. Modes \[DECISION\] (orchestra's one talk with modes, orchestra-lx4d.16):

- **refine** a spec before its breakdown (§15.5);
- **feature**: talk a feature through, filing an epic and its tickets, then run it;
- **answer**: §15.3;
- **run** (H44): the console's conversation with the orchestrator (§18.3), one per project, its Pi session kept across the app's restarts. Its read tools: the run's view and events, tickets (`show`, `why`, `ready`), a worker's recent transcript, the log. Its engine commands, as tools: message (steer) a worker; start, drain or undrain a run; hold a ticket until another merges (a `blocks` link); file, update, defer or label tickets; answer a question in the maintainer's words; open the answer talk. It acts on the maintainer's instructions, confirming in one line before it stops a run, closes or cancels a ticket, or allows a permission request. Every action it takes is an event (actor `run_talk`), shown in History. It edits no code and runs no git or shell command.

A talk's model is the project's, overridable per talk.

### 15.5 Spec runs

orchestra's (`docs/running.md`, A whole app from a spec) with its later lessons \[DECISION\]:

- The spec is committed under `.hivemind/specs/` when it comes from outside the repository; a run carries on from its last version by its SHA-256, in update mode for a changed spec.
- The headless breakdown (an organ session with read tools and the ticket tools) files the root epic and, when the spec needs a tool the machine lacks or the repository has no code, the **environment ticket** first (labelled `solo`, priority 0, every other ticket blocked on it); the run starts on the environment ticket while the breakdown goes on (orchestra-ldp7). The rest of the tree comes back in the breakdown's structured answer, filed by the engine in order (orchestra-aehg).
- The environment ticket writes the project's `requires` list (§17) and its pins, scaffolds, installs dependencies with `setup`; it doesn't install machine tools as ticket work (orchestra-coyp.2).
- `--refine` runs the refine talk first; the talk files the tree itself.

### 15.6 Overnight runs

orchestra's `--overnight` (orchestra-aq0o) \[DECISION\]: the current tickets, after a pre-flight checklist and the open questions answered.

- **Refuses to start** until fixed or overridden: a ticket limit set; `concurrent` under 2; Base failing `check_fast`; `permissions: ask` (workers would wait on the maintainer); a model's credentials missing or rejected (a cheap probe call); free disk on the engine's host under `overnight.min_free_disk` (default 20 GB).
- **Warns**, offering the fix: `usage_limit_wait` under 5h; no `ticket_limit`; another heavy run on the engine or heavy load on its host; older worker rules; the host on battery; how many tickets are ready and how many wait only on questions.
- **Then the questions**, one by one in the answer talk, until none is open (or the maintainer leaves some, which the checklist names with what they hold).
- **During the night:** the engine keeps its host awake (`caffeinate -i -w <pid>` on macOS, `systemd-inhibit` on Linux). A small `overnight` tag in the information colour (`c5`) marks the run in the console (§18.3).

## 16. Organs

\[DECISION\] Organs call a model through `pi-ai`, which has tool calling but no JSON-schema output mode: each organ answers through one forced `answer` tool whose arguments are decoded with the organ's Effect Schema, and an answer that fails to decode is retried once with the error. An organ that needs tools (the scout, the breakdown) is a short Pi session ending in the same `answer` tool. Each has `organs.<name>.model` (default the project's model) and can be turned off.

| Organ | When | Given | Answers | Its answer changes |
|---|---|---|---|---|
| Triage | A ticket set aside (deferred, unfinished) | The ticket, its notes, the session's last turns, git state | A cause (ticket, environment, instructions, …), a confidence, a note | A note on the ticket; two high-confidence `environment` verdicts start the environment hold |
| Predictor | A ready ticket naming no files, with more than one slot | The ticket and the repository's file list (tests left out unless the ticket mentions tests) | The files it will likely change | Its footprint (`predicted_files`) |
| Reviewer | A run's end | The run's events, filtered; commits; set-aside tickets; checks' failures; cost | The run report | Nothing; shown and saved |
| Scout | `init` | Read-only tools on the repository | The project's check suites, setup, stack, requirements | `init`'s proposals |
| Breakdown | A spec run | The spec, read tools, the ticket tools | The tree (§15.5) | The backlog |

Organ calls take the model proxy's permits, so the usage-limit hold and per-provider caps apply, and their cost is recorded (§12.9, §14.4).

## 17. Requirements

orchestra-coyp's list \[DECISION\]: `requires` in the project's settings, each entry a name, a `check` command (exit 0 when met), and either an `install` command (never `sudo`; refused at load otherwise) or a `manual` instruction, optionally per platform.

- Checked at every run's start, on `init`, and after a merge that changed the list; a failing check with an install runs it and checks again; anything still missing stops the run before its first ticket (`REQUIREMENTS_MISSING`, exit 2) with one list, each with its instruction.
- On the engine's host: for a remote engine, the requirements are the server's.
- Workers that need a machine tool add an entry (check, install or manual) and install it through that entry's command, never by hand outside the list.
- `hivemind requires --markdown` prints the list for the README; `init` keeps that section between markers.

## 18. The app

### 18.1 Stack

\[DECISION\] The renderer is a Foldkit program: one Schema-defined Model, Messages, a pure `update`, side effects as Commands (`hivemind.md` §11). Behaviour from `@foldkit/ui`, appearance from Tailwind over the design handoff's tokens (§18.2). Mounted: Monaco for diffs and code, xterm.js for agents' command output (read-only), Cytoscape.js for the ticket graph with a layered layout through its dagre or ELK extension, styled from the tokens (H40). Split panels and trees come from `@foldkit/ui` where it has them, otherwise they are built as Foldkit submodels in `packages/ui`; no React component library \[VERIFY `@foldkit/ui`'s components\].

| Concern | Foldkit construct |
|---|---|
| A connection to an engine (through the connection process) | ManagedResource |
| The run view, transcripts, attention, the backlog, the host's load | Subscriptions that emit Messages |
| Requests (start, drain, answer, message a worker) | Commands |
| Per-worker panes, the talk panel, dialogs | Submodels with OutMessages to the parent |

### 18.2 The design handoff and themes

\[DECISION\] (H42) The **Hivemind Console handoff** (`design_handoff_hivemind_console`, 2026-10-21: a README, `themes.css`, and two HTML references, the three themes side by side and Lichen Bright alone) is the app's visual source, kept in the repository under `design/console/`: its README, `themes.css`, the two HTML references, and both references rendered at their size (`reference.png`, `kits.png`). The design tool's runtime the HTML needs to open (`support.js`, `_ds/`) has no licence of ours and stays out of the public repository; the PNGs are what tests compare against. It is high fidelity: colours, type, spacing and radii are final, at a 1280 × 800 reference window. Its HTML is a reference recreated in Foldkit (its README's suggestion of React doesn't apply); only `themes.css` is used as it is.

- **Tokens:** `themes.css` is copied into `packages/ui` unchanged, as the only source of colours, fonts, radii and glows. Tailwind's theme maps onto its variables (`--bg`, `--side`, `--panel`, `--line`, `--fg`, `--muted`, `--ink`, `--c1` to `--c5`, `--r`, `--rb`, `--rw`, `--glow1`, `--glow2`, `--font`, `--mono`); components use only those, and a lint rule refuses colour literals anywhere else. Spacing keeps to the handoff's steps (4, 6, 8, 10, 12, 16, 18, 20 and 24 px).
- **Themes:** **Lichen Bright** (the default), **Classic** and **Neon Grid**: one layout, chosen by `data-theme` on the root. The theme is a setting of each app install, not the engine's, switched in Settings and the command palette, and applied at once. All three are dark; there is no light theme in v1.
- **Roles:** `c1` working, `c2` needs you, `c3` failed or set aside, `c4` merged or done, `c5` information (the orchestrator, organs, an overnight run's badge). Every status mark, the system stats' warnings (`c2`) and alarms (`c3`), and History's dots use them; nothing else carries status colour.
- **Fonts:** Archivo (Lichen Bright), Inter (Classic) and JetBrains Mono (Neon Grid, for everything) ship inside the app as WOFF2 files (SIL Open Font License), since the Content Security Policy loads nothing remote (§20.1).
- **Icons:** the handoff has none beyond Unicode glyphs (›, ⏎, ⌘); Lucide where icons are added.
- **Hover and focus** as the handoff says: rows and cards lighten slightly, buttons `brightness(1.08)`, focus a 2px outline in `--c1` with a 2px offset.
- **Screens the handoff doesn't draw** (§18.4) are derived from its tokens and its panel, card and list language until a second handoff covers them (§28).
- **Checked:** the console's scene with fixture data at 1280 × 800 is captured in each theme and compared with the handoff's references (§24).

### 18.3 The console

\[DECISION\] The handoff's screen, adapted to Hivemind's model (H43 to H46). Layout as the handoff: a 38px title bar; under it a 260px sidebar beside the main area (padding 20 × 24 px, a column with 16px gaps), holding the stats row, then the run talk (1fr) beside History (320px), 156px high, then the two ticket columns side by side, which take the remaining height and scroll each on its own.

- **Title bar** (H46): native window controls in the design's bar. On macOS the traffic lights sit inset on its left (`titleBarStyle: 'hiddenInset'`, centred on its 38px); on Windows and Linux the system's minimise, maximise and close sit on its right (`titleBarOverlay`, coloured from `--side` and `--fg`) \[VERIFY `titleBarOverlay` on Linux at the pinned Electron\]. The design's three squares are dropped; the rest stays: "Hivemind", the system stats, and the search field (⌘K, the command palette). The bar is a drag region; its controls aren't.
- **System stats** describe the selected project's engine host (§7.4), sent over the connection, never the app's own machine (the handoff's README reads Electron's process; that would be wrong for a remote engine). `MEM`: used and total, its bar `c2` while memory pressure warns and `c3` when critical. `CPU`: the 1-minute load as a share of the cores, its bar `c2` past 1.5× the cores and `c3` past 3× for a minute (orchestra-recj). `CORES`: one block per core, busy ones in `c1`. Refreshed about every second; hovering shows swapping, the merge check's memory and each worker's (orchestra-556h, -u6pv).
- **Sidebar: projects and their runs** (H43). Header "Projects" and the count; an "All projects" row first, then one row per project on every known engine: a chip in its colour (the five role colours in turn, by the project's index) holding its two-digit index; the project's name; its Base and, where more than one engine is known, the engine's name (`main · server-1`); and its run's state on the right: Running (`c1`), Waiting (`c2`: something needs you), Idle (no run). Selecting a project scopes the main area to it; with All, the columns and History show every project, each card marked with its project's chip. The pinned button is "+ Add project" (§10.1), adding an engine on the way when the project's host isn't known yet (§9.2).
- **Stats row**, for the scope: live workers against the slots (`c1`, "4 of 4"), done today (`c4`), set aside (`c3`), needs you (`c2`), in queue (`c5`, what holds them on hover), tokens (`c5`). At its right end, the selected project's run controls: Start run (its modes: the current tickets, one ticket, a spec, a feature, overnight) while it is idle; Drain and Stop while it runs. An overnight run carries a small `c5` "overnight" tag here and on its project's row. Holds and banners (§14) take a strip above the row: `c2` for a hold, `c3` for a stop, with the reason and, where known, when it lifts.
- **The run talk** (H44) fills the conversation panel: the scope's run talk (§15.4), its header giving the talk's model and its context use ("claude-opus-5-5 · 14k ctx"), its composer sending to it. When questions arrive, the answer talk takes the panel (`questions: auto`, §15.3), and the run talk comes back once they are answered. With All projects, it acts on the project a message names, or asks which.
- **History:** the scope's events from the event stream (§13.12), newest first, each a time, a dot and one line: dispatched (`c1`), merged (`c4`), asked or waiting on you (`c2`), failed, set aside or held (`c3`), organs and the run talk's actions (`c5`).
- **In progress:** the run's Current (working, verifying, waiting to merge, merging, a hand-back), as the handoff's card: the ID and the title; the **phase bar** (H45), segments for working, verifying, waiting to merge and merging, those passed filled, the current one in the card's colour (its project's), a hand-back marked on the merging segment; and the step, the worker's latest action from its Pi events (the tool and its main argument: "Running scripts/check-own.sh", "Editing src/auth/refresh.ts") or its place in the queue ("2nd to merge", "in the train"), with the elapsed time on the right. No percentage: Hivemind has no honest one. Clicking a card opens its worker view.
- **Needs your attention:** attention requests and tickets set aside, as the handoff's card with its kind tag. **Approval** (`c2`): a permission request; Approve allows it once, its menu allows it for this run or refuses it with a note. **Question** (`c5`): Answer opens its options inline, the recommended one first, a free-text line and "Talk it through". **Review** (`c4`): a ticket set aside (`CHECKS_FAILED`, `MERGE_CONFLICT`, a verifier that disagrees with itself, a ticket past its limit); Review opens its diff and notes. The waiting time on the right in `c2`; "Open session" opens the worker view. A resolved card leaves the column, its ticket back in progress where it goes on.

### 18.4 Other screens

\[DECISION\] (H24) Reached from the console (a card, a project, the command palette), in the handoff's tokens and language:

| Screen | Holds |
|---|---|
| Worker view | The worker's transcript as it happens (messages, thinking collapsed, tool calls and their output in xterm.js, diffs in Monaco), its ticket, its notes and cost; a composer to message or interrupt it (the worker is then steered: the engine sends it nothing until released) |
| Talks | The refine and feature talks, full height; the run and answer talks open here too from the console |
| Backlog | Epics and tickets, their dependencies (the graph), what is ready and why the rest isn't (`tickets.why`), questions, deferred tickets, the tickets a run finished; read-only (editing in the app comes later; the CLI and the run talk cover it) |
| Review | A ticket's branch diff, a run's merged diff, run reports, the log |
| Insights | From the engine's telemetry (§21): tokens and cost per model, ticket, run and project; time per phase (working, checks, the merge queue, hand-backs, holds); check durations and failures; hand-back and set-aside rates; models compared on similar tickets |
| Engines and projects | Known engines (local, SSH, WSL2, TLS), their versions and status; Add engine (§9.2); projects per engine; credentials and sign-ins (§20.3) |
| Settings | The project's, the engine's and the app's settings (the theme among them), with their sources |

### 18.5 Desktop rules

From `hivemind.md` §11: density over whitespace, monospace for agent output; keyboard first (a command palette, keys for approve, refuse, interrupt, drain, the answer talk); one status mark per ticket, in the role colours (working, waiting on you, blocked, failed, done); notifications for questions, permission requests, a run's end and a hold; never push each token through `update` (buffer, once per animation frame); virtualise long lists; lazy views for inactive workers. The handoff first; for what it doesn't cover, Linear, Codex desktop, Claude desktop and VS Code's workbench.

## 19. The CLI

\[DECISION\] The engine distribution's `hivemind` command. It talks to the local engine, or to `--engine <name>` (a known engine, reached as the app reaches it).

| Command | Does |
|---|---|
| `hivemind serve` | Runs the engine in the foreground (what the service runs) |
| `hivemind bridge` | Connects stdin and stdout to the local engine's socket, starting the engine when it isn't running; what SSH and WSL2 clients run (§8.2) |
| `hivemind service install\|uninstall\|status` | The login service |
| `hivemind engine status\|stop\|update\|version` | The engine |
| `hivemind token create\|list\|revoke` | Pairing tokens |
| `hivemind credentials set\|list\|remove\|login` | Model credentials in the engine's secret store |
| `hivemind project add <path\|url>\|list\|remove` | Projects |
| `hivemind init` | Sets a project up (scout, checks, setup, requirements, rules) |
| `hivemind run [--tickets\|--ticket <id>\|--spec <file> [--refine]\|--feature "<text>"\|--overnight] [--limit N] [--concurrent N] [--model …] [--offline] [--wait]` | Starts a run |
| `hivemind run status\|drain\|undrain\|stop` | A run |
| `hivemind follow` | The run's events as plain lines, orchestra's plain mode |
| `hivemind answer [<id>]` | Questions and permission requests, asked on the terminal with their options; the answer talk on the terminal |
| `hivemind talk refine\|feature\|answer\|run` | A talk on the terminal |
| `hivemind tickets list\|show\|create\|update\|close\|reopen\|defer\|dep\|label\|note\|ready\|why\|rebuild` | The backlog |
| `hivemind tickets import --beads <file>` | A Beads backlog into the project (§11.6) |
| `hivemind requires [--markdown]` | The requirements' state |
| `hivemind report [<run>]` | A run's report |
| `hivemind insights [<query>]` | Telemetry summaries and the named queries (§21) |

Every command has `--json` for scripts and agents. Flags take two dashes; one-letter shorthands one (orchestra-6rdu).

## 20. Security

\[DECISION\] The renderer is treated as an untrusted web page; agents as code running with the engine user's permissions, confined by the gate (and a container where configured); a remote engine as a server holding credentials (`hivemind.md` §12, extended).

### 20.1 App

- `contextIsolation: true`, `sandbox: true`, `nodeIntegration: false`, `webSecurity: true`; no `<webview>`; a strict Content Security Policy (bundled assets only, no remote scripts, no `eval`); navigation and `window.open` to external URLs blocked, links opened in the system browser.
- Electron fuses: cookie encryption, only-load-app-from-asar, embedded asar integrity validation, **RunAsNode disabled** (agents run on the engine's own Node, never through Electron) \[VERIFY per platform\].
- The preload exposes a minimal API (window controls, the folder picker, the app's version, update status, the MessagePort), each method validated with Schema.

### 20.2 Engine

- The local socket is mode 0600 in a folder only the user can enter. TLS listeners only when configured, with tokens hashed at rest and revocable.
- Each agent gets a minimal environment: its worktree, the model proxy's address and its per-agent token (§12.9), the extension's socket and token, nothing else; never a model credential.
- The extension's per-agent token scopes its tools to its own ticket and those linked to it.
- A project's own Pi settings and packages load only after the user trusts the project.

### 20.3 Credentials

\[DECISION\] Model credentials live in the engine host's secret store: the macOS Keychain; on Linux libsecret or KWallet; on a Linux host with no keyring, an encrypted file under `<data>/secrets/` with a passphrase given at the engine's start (or from a file only the user can read, for a service), which the engine warns about. They are entered through the app (sent once over the authenticated connection) or the CLI, never stored by the app for a remote engine, and never written to the work graph, the tickets ref, telemetry, logs or a worktree. Only the model proxy reads them (§12.9).

- **API keys** are set by name; a key MAY instead be a command whose output is the key (a password manager's CLI), run by the engine when the key is needed, as Pi's own `auth.json` allows.
- **Subscriptions:** `credentials.login` runs `pi-ai`'s OAuth sign-in in the engine (`models.login(provider, 'oauth', …)`) with the engine's secret store as its `CredentialStore`, and relays the sign-in URL or device code to the client that asked, so a remote engine signs in from the laptop's browser. Providers with OAuth in `pi-ai` today: Anthropic (Claude Pro and Max), OpenAI (sign in with ChatGPT), GitHub Copilot, OpenRouter \[VERIFY at the pinned version\]. Whether a provider's terms allow its subscription in a third-party agent is the user's to check; the app says so at sign-in.

### 20.4 Supply chain

Exact pins and a committed lockfile; CI verifies the checksums of every downloaded artifact; the engine distribution and its checksums are signed, and the app and `hivemind engine update` verify the signature before installing (orchestra's `checksums.txt.sig` scheme); third-party Pi packages disabled by default.

### 20.5 Licence and distribution

\[DECISION\] (H49) Hivemind is open source under the MIT licence, in a public repository, `github.com/noesis-sol/hivemind`. Releases are public GitHub Releases: the engine distributions, the app's installers, their signed checksums, and the install script's downloads. The app's updates and `hivemind engine update` read them. Signing uses the maintainer's Apple Developer ID and a Windows code-signing certificate, kept in CI's secrets. `THIRD_PARTY_NOTICES.md` lists every bundled component's licence (Node, Pi, Effect, Foldkit, DuckDB, Monaco, xterm.js, Cytoscape.js, the three fonts under the SIL Open Font License).

## 21. Telemetry

\[DECISION\] (H38) Telemetry is the engine's own, kept on its host and always on; nothing leaves the host unless the user configures an export.

- **Store:** DuckDB, embedded in the engine through `@duckdb/node-api`, at `<data>/telemetry.duckdb`, one per engine across its projects. Only the engine opens it (DuckDB allows one writing process per file); clients query through the RPC's Insights group, by named queries. SQLite stays the work graph (§11.2), and telemetry never writes to it.
- **What is recorded:** model calls (from the model proxy: provider, model, tokens in, out and cached, cost, latency, time to first token, refusals and errors), tool calls (name, duration, the gate's decision), checks (kind, duration, result), merges, hand-backs, holds, worker sessions (ticket, model, duration, outcome), organ calls, and the host's load samples (§7.4), each tied to its project, run and ticket.
- **Writes** go through DuckDB's Appender from one fiber, in batches; a crash loses at most the open batch.
- **Reads in place:** the event stream (`events.jsonl`) and Pi's session files are queried where they lie, and the work graph is attached read-only for joins with tickets \[VERIFY that the `json` and `sqlite` extensions can be bundled with the engine rather than downloaded at run time, and that DuckDB reads the work graph safely while the engine writes it\].
- **Retention:** raw rows for 90 days, daily aggregates kept (engine settings).
- **Uses:** the cost figures (§14.4), the run report (§16), the app's Insights screen (§18.4), `hivemind insights`.
- **Export**, optional and off by default: Parquet files on demand, or OTLP to an endpoint the user configures (an OpenTelemetry Collector, ClickHouse or another store behind it), carrying Effect's spans and metrics (`@effect/opentelemetry`) and the recorded rows. Exports carry metadata only (counts, durations, model IDs, token totals, cost, error tags), never prompts, code, file contents, secrets or absolute paths (`hivemind.md` §10's rules). No Hivemind-hosted telemetry service in v1.

## 22. Packaging, signing and distribution

### 22.1 The engine distribution

\[DECISION\] Per engine target (§4): a tar.gz holding a pinned Node runtime, the engine bundle (`apps/engine` and `packages/engine` bundled), Pi and the first-party extension at exact versions, and migrations bundled (no runtime reads of source folders). `hivemind` is a small launcher running the bundled Node on the bundle. Signed checksums per release; macOS binaries signed and notarised.

### 22.2 The app

\[DECISION\] electron-builder, in CI, for the app's five targets (§4): dmg and zip per architecture (macOS, Developer ID, hardened runtime, notarised, every Mach-O signed), AppImage, deb and rpm (Linux), NSIS (Windows, Authenticode). Updates through electron-updater for macOS (from the zip), Windows (NSIS) and the AppImage; deb and rpm are downloads updated by hand until a package repository is worth running (H39). The macOS and Linux apps embed the engine distribution for their own architecture, for the local engine; the Windows app embeds the Linux x64 distribution and installs it in WSL2 (§7.6).

### 22.3 CI

- Engine: build and test on macOS x64, macOS arm64, Linux x64, Linux arm64.
- App: build and package on the five targets; smoke-test every artifact: launch; start or reach an engine; open a fixture repository; run one ticket with a stub model to its merge; close cleanly with no orphan processes.
- Remote: an end-to-end test with an engine in a container reached through the SSH bridge.
- Windows: the app installing and reaching an engine in WSL2, where a CI runner offers WSL2 \[VERIFY; otherwise on a dedicated Windows machine before each release\].

### 22.4 Native inventory

Anything with native code MUST appear here.

| Component | Native code | Status |
|---|---|---|
| Electron | Yes | In (the app) |
| Node runtime (engine distribution) | Yes, per engine target | In (new) |
| SQLite | None; built into Node (`node:sqlite`) | In |
| Pi coding agent | Unknown | \[VERIFY\] all four engine targets |
| DuckDB (`@duckdb/node-api`) | Yes, prebuilt per platform | In, engine only (H38): prebuilt for macOS and Linux, x64 and arm64 \[VERIFY each and glibc's floor\]; its `json` and `sqlite` extensions bundled, never downloaded at run time; signed with the engine distribution on macOS |
| Monaco, xterm.js, Cytoscape.js (with dagre or ELK) | None | In |
| OpenSSH client | System's, not bundled | Required on clients for remote engines; the bridge runs over it (§8.2) |
| WSL2 | System's, not bundled | Optional on Windows, for a local engine (§7.6) |
| Docker or Podman | System's, not bundled | Optional, for containers (§12.7) |
| `node-pty` (interactive shells) | Yes | Out of v1 |
| Beads `bd` | Yes | Out (not used; its JSONL is imported, §11.6) |
| Turso, libSQL, chDB | Yes | Out (`hivemind.md` §16) |

## 23. Developing with coding agents

\[DECISION\] (H48) Hivemind is built by orchestra 0.9 from this spec: a spec run in a new repository, whose breakdown files the environment ticket first and then each milestone's tickets, M0's spikes first (one per `[VERIFY]` that a later decision rests on; a spike that disproves a fact files a question on the decision it affects). orchestra's Beads backlog in that repository is the development tracker only; the product doesn't use Beads (H11).

Otherwise unchanged from `hivemind.md` §14 \[DECISION\]: Foldkit, Effect v4 and Pi vendored as git subtrees under `repos/`, pinned and refreshed on every upgrade; Foldkit's agent skills and `FOLDKIT.md`; source-first Effect skills; after every file write, a type check and oxlint on the changed files, errors returned to the agent in-band (a Claude Code `PostToolUse` hook; a development-only Pi package); Foldkit DevTools MCP in development; the order of work (contracts, then interfaces, then tests against in-memory layers, then implementations; for UI, Model and Messages first, then story tests, `update`, `view`).

## 24. Testing and quality gates

\[DECISION\] Every layer is tested against Effect Layers swapped for test implementations, so no unit test needs a real model, a real network or a browser (`hivemind.md` §15, extended).

| Layer | Approach | Tools |
|---|---|---|
| Work graph | Real migrations against in-memory SQLite; leases and timeouts with `TestClock` | `@effect/vitest` |
| Tickets ref and merging | Property tests of the per-field merge (commutativity where it should hold, no lost notes, claims), against real git in temp folders | `@effect/vitest`, `fast-check` |
| Run loop | Whole-run scenarios with a `FakePiAgent` replaying scripted event streams and tool calls, an in-memory git, `TestClock`; orchestra's scenarios as the checklist (dispatch, footprints, holds, merge queue, train, hand-backs, carry-over, usage limit, environment hold) | `@effect/vitest` |
| Git-subject behaviour | Real git in temp folders | `@effect/vitest` |
| Protocol | Client and engine over an in-memory transport, a real socket and a stdio pipe (the bridge); reconnect and resume; version mismatch | `@effect/vitest` |
| Model proxy | Against recorded provider streams per API: usage counted, a 429 with and without a reset, a rejected key, an OAuth token refreshed mid-session, two agents sharing one refresh | `@effect/vitest` |
| Telemetry | An in-memory DuckDB: batched writes, retention, the named queries over fixture events and sessions | `@effect/vitest` |
| Run lease | Two engines on one remote in temp folders: a refused start, renewal, expiry and takeover | `@effect/vitest` |
| Pi contract | Real pinned Pi in RPC mode with the extension, against a stub model provider, on every Pi upgrade \[VERIFY stub-provider support\] | Vitest |
| UI state | Foldkit story tests on `update` | `foldkit/story` |
| UI behaviour | Foldkit scene tests with accessible locators | `foldkit/scene` |
| UI look | The console with fixture data at 1280 × 800, captured in each theme and compared with the handoff's rendered references (`design/console/*.png`); colour-literal lint | The packaged-app test tool's screenshots |
| Packaged app | Smoke tests per target (§22.3), driving the packaged app | Playwright's Electron support \[VERIFY it attaches to a packaged, fused build on each target\] |
| Remote | An engine in a container over SSH, end to end | CI |

**Lessons from orchestra's tests** \[DECISION\]: a test's environment is airtight (a PATH holding only its fakes and the real tools it names; no inherited terminal variables; a lint against reading the process environment in tests), and tests assert on typed events, not log wording (orchestra-nqbn, -163s, -lx4d.12, -lx4d.21).

**Gates on every pull request:** type check, lint, format check; unit, story and scene tests; migrations up from empty and from the previous release's fixtures; the ticket-file schema's previous version readable; no new native-inventory entry without review.

## 25. Milestones

| Milestone | Delivers |
|---|---|
| M0 Spikes | Pi in RPC mode on the bundled Node with the extension (tools with wrapped schemas, the gate, `agent_before_settle`, `project_trust`, the system prompt, a per-process configuration folder) \[the VERIFYs of §12\]; the model proxy in front of Pi with a base URL override, for an API key and an Anthropic subscription; Effect RPC over a socket, a stdio pipe, a WebSocket and a MessagePort relay; the SSH bridge from macOS, Linux and Windows clients, and through `wsl.exe`; `node:sqlite` and DuckDB in the engine's Node on all four engine targets; Foldkit in Electron with the connection process |
| M1 Engine core | The daemon, projects, the work graph in SQLite, the CLI's ticket commands (Beads import included), the RPC server, the model proxy for API keys; a run with dispatch to Pi, `finish`, and the merge queue's basic path (rebase, `check_fast`, fast-forward); `follow` |
| M2 The run loop | Footprints and holds, solo tickets, the verifier, hand-backs, setup, `check_own` and `check_full`, rechecks, carry-over, the environment and usage-limit holds, the event stream, reports, the train |
| M3 The app | The design tokens and the three themes; the console (the projects sidebar, the system stats, the stats row, History, In progress with the phase bar, Needs your attention with questions and permissions inline); engines and projects; the worker view; the local engine's lifecycle; notifications |
| M4 Remote | Installing and updating engines over SSH and in WSL2, the bridge, TLS and tokens, credentials on the host and subscription sign-ins, pushing and fetching, the tickets ref, its sync and the run lease |
| M5 Organs and talks | Triage, predictor, reviewer, scout; `init`; requirements; the talks (refine, feature, answer, run); spec runs |
| M6 Overnight and 1.0 | Overnight runs, the machine budget and the system stats, budgets, telemetry and the Insights screen, containers, packaging and signing for every target, updates; a parity audit against orchestra 0.9.1 (§26) |

## 26. Parity with orchestra 0.9.1

The parity audit (M6) checks each orchestra feature against Hivemind. Its starting matrix:

| orchestra | Hivemind |
|---|---|
| Beads backlog (`bd`) | The work graph (§11); `hivemind tickets` for people; the extension's tools for workers |
| Workers in Herdr, tmux or cmux tabs; hooks; screen reading | Pi processes owned by the engine; state from Pi's events and the control channel (§12) |
| `orchestra hook`, the Stop hook, settle heuristics, nudges | `finish`/`defer`/`wait_for`/`ask_question`; one nudge for a turn without an outcome (§12.3) |
| Claude Code's permission modes, the auto classifier | The neutral policy enforced by the gate (§12.6) |
| The verifier subagent | The engine's verifier session (§13.6) |
| `claude -p` organs | `pi-ai` calls or short Pi sessions (§16) |
| The worker prompt appended to Claude Code's system prompt | Standing rules injected by the extension (§12.4) |
| The dashboard (Bubble Tea) and plain mode | The app (§18) and `hivemind follow` (§19) |
| `orchestra update` | App and CLI updates; engines updated over SSH (§9.2) |
| Runs registry and `--machine-workers` across runs | One engine per machine and its budget (§7.3) |
| Everything in §13–§17 | Same behaviour, as specified |
| Herdr/tmux/cmux, lost agent names, tabs left open, adoption of live workers, `●`/`⏺` bullets, hook input types, Claude's usage-limit screens, `--chrome`, MCP server choice for Claude | Not applicable |

## 27. Changes to `hivemind.md`'s decisions

| `hivemind.md` | Was | Now | Reason |
|---|---|---|---|
| §2 Product context | An app supervising Pi agents with typed hand-offs and a shared work graph | A port of orchestra (dispatch, footprints, merge queue, checks, questions, organs) with `hivemind.md`'s typed hand-offs and supervision | The maintainer wants orchestra's behaviour on this stack (H13, H27) |
| §4 Process architecture | The orchestrator is an Electron `utilityProcess`; closing the app aborts sessions | The engine is a separate long-lived Node daemon, local or remote; runs survive the app; a connection `utilityProcess` owns transports | The engine must run locally or on a remote server, headless (H4, H5) |
| §5 Node | Electron's bundled Node for every process | The engine ships its own pinned Node; Electron's Node only for the app | The same engine runs on servers without Electron (H5) |
| §6 Channels and versioning | Renderer to orchestrator over a MessagePort; no backward compatibility needed (one build) | Renderer to engine through the connection process (socket, SSH, TLS); a versioned protocol supporting the previous version | Engines and apps update separately, on different machines (H6, H22) |
| §7 Agent child runtime \[OPEN\] | `ELECTRON_RUN_AS_NODE` or a bundled Node | The engine distribution's Node | Resolved by H5; lets the RunAsNode fuse stay disabled |
| §7 Extension channel \[OPEN\] | A per-agent Unix socket, or an alternative | A per-agent Unix socket with a ticket-scoped token; tools include outcomes and questions with options | Resolved (H14) |
| §7 Tools | `next_task`, `report_result`, `file_issue`, `ask_human`, `read_board`, `post_note` | `get_ticket`, `file_ticket`, `ask_question`, `ask_and_wait`, `add_note`, `finish`, `defer`, `wait_for` | The engine dispatches tickets (orchestra's model), so agents don't pull work; outcomes are explicit (H13, H14) |
| §8 Coordination | Agents claim work with leases | The engine dispatches and claims; leases kept engine-local for crash recovery | orchestra's model (H13) |
| §8 Budgets | Token and cost budgets per task and run | Optional, off by default | Decided with the maintainer (H21) |
| §9 Work graph | SQLite, git-ignored; sync deferred (Turso later) | SQLite as the live store plus continuous sync through `refs/hivemind/tickets`, one JSON file per ticket, merged per field; schema extended for orchestra's tickets (types, statuses, design, acceptance, notes, labels, questions with options, merged commit) | The backlog must travel with the repository and between engines (H11, H12) |
| §3, §16 Windows | Out of v1 | The app on Windows x64 (engine remote or in WSL2); the engine stays macOS and Linux | Decided with the maintainer (H25) |
| §12 Secrets | Electron `safeStorage` for API keys | Model credentials in the engine host's secret store; the app's `safeStorage` only for its own secrets (pairing tokens) | A remote engine must work with the app closed (H26) |
| §12 RunAsNode fuse \[OPEN\] | SHOULD be disabled; conflicts with spawning Pi through Electron | Disabled | Pi runs on the engine's Node (H5) |
| §12 Containers \[OPEN\] | A later option | A per-project option in v1 | Safe broad permissions on servers (H16) |
| §13 Native inventory | Electron only | Plus the engine's Node runtime | H5 |
| §17 On-disk names \[OPEN\] | Undecided | `hivemind`, `.hivemind/`, `refs/hivemind/tickets`, branches `hivemind/<id>` | Decided (§10.2) |
| §17 Worktree lifecycle \[OPEN\] | Undecided | Kept when set aside, removed when merged; orchestra's locations | Decided (§10.2) |
| §3 macOS builds, Linux scope \[OPEN\] | Undecided | Per-arch macOS builds; Windows arm64 and Linux armv7l not built; deb and rpm updated by hand | Smaller downloads; wait for demand (H39) |
| §5 Type checker \[OPEN\] | `tsc` or `tsgo` | `tsgo`, with `tsc` as a fallback | Speed when agents type-check after every write (H40) |
| §6 Event batching \[OPEN\] | About 50 ms | 50 ms, an engine setting; the run view at about 20 Hz | Decided (H40) |
| §7 Models and credentials | API keys injected into each agent's environment at spawn | A model proxy in the engine holds every credential; agents get a per-agent token; subscriptions are signed in by the engine | OAuth tokens expire mid-session, and agents on a remote engine shouldn't hold keys (H29) |
| §9 Beads interop | MAY ship after v1 | Beads JSONL import in v1, export later | Existing orchestra and tutti backlogs move over (H40) |
| §10 Telemetry | Effect telemetry buffered in a SQLite outbox and shipped to ClickHouse, opt-in | DuckDB in the engine, always on and local, feeding the app's figures and Insights; export (Parquet, OTLP) optional, metadata only | The maintainer wants an embedded store that gathers data on the engine's host (H38) |
| §11 Look | Tailwind tokens: a neutral palette, one accent, light and dark themes; shadcn/ui's visual language as the reference | The Hivemind Console handoff's tokens and layout: three dark themes, Lichen Bright the default; the handoff first among the references | The maintainer's design handoff (H42) |
| §11 Graph library \[OPEN\] | Cytoscape.js, undecided | Cytoscape.js with a dagre or ELK layout | Decided (H40) |
| §13 Native inventory, §16 DuckDB | Out; deferred to an Insights panel | In, engine only, for telemetry | H38 |

## 28. Open questions

Settled with the maintainer on 2026-10-21 (H29–H49), except where marked; the `[VERIFY]`s they leave are M0's work (§25).

- [x] **Pi's APIs** (§12, §16, §20.3): `pi --mode rpc` (JSONL, correlated by `id`); `before_agent_start` for the standing rules; `agent_before_settle` for the nudge; `tool_call` blocking; `project_trust`; `--session-dir` and `switch_session`; organs through a forced `answer` tool (`pi-ai` has no JSON-schema mode); subscription sign-ins through `pi-ai`'s `models.login`. `before_provider_request` is no longer needed (the model proxy, H29). Still to verify in M0: base URL overrides, a per-process configuration folder, usage fields, a stub provider for tests, native code.
- [x] **Agents' credentials** (§12.9): a model proxy in the engine (H29).
- [x] **Effect RPC transports** (§8): `effect/unstable/rpc`, NDJSON over Effect's `Socket`, `RpcServer.makeProtocolWebsocket`. M0 verifies the stdio pipe and the MessagePort relay; the fallback stays a thin tagged protocol with Schema-encoded frames.
- [x] **SSH** (§8.2, §9.1): a stdio bridge, no forwarding (H30).
- [x] **WSL2** (§7.6, §22.2): automated, through the same bridge (H31).
- [x] **Ticket-file clocks** (§11.3): a hybrid logical clock per field (H32).
- [x] **Sync intervals** (§11.3, §11.4): commit within 2 s, push at once, fetch every 60 s in a run, 15 min idle (H33).
- [x] **Two engines on one project** (§11.5): forbidden, through the run lease (H34).
- [x] **A run's start without the remote** (§11.5): refused unless `--offline`; on a clash, the later run drains (H41).
- [x] **`machine_workers` default** (§7.3): half the cores, at least 2 (H35).
- [x] **Containers** (§12.7): Docker or Podman, rootless preferred, with limits (H36).
- [x] **Stops that remain** (§14.5): always set aside (H37).
- [x] **Stream intervals** (§8.3): about 20 Hz and 50 ms, as settings (H40; set without a separate question, as `hivemind.md` leaned).
- [x] **Telemetry** (§21): DuckDB, local and always on; export optional; no hosted service in v1 (H38).
- [x] **Builds** (§4, §22.2): per-arch macOS; no Windows arm64 or armv7l; deb and rpm by hand (H39).
- [x] **UI gaps** (§18): Cytoscape.js with dagre or ELK (H40); split panels and trees from `@foldkit/ui` or as Foldkit submodels; Playwright's Electron support for the packaged app (set without a separate question; both carry a `[VERIFY]`).
- [x] **Type checker** (§6): `tsgo`, `tsc` as fallback (H40).
- [x] **Beads JSONL interop** (§11.6): import in v1, export later (H40).
- [x] **The console's design** (§18.2, §18.3): the handoff, its sidebar, chat, progress and window controls adapted (H42 to H46).
- [ ] **A second design handoff** (§18.4): the screens the console handoff doesn't draw (the worker view, the backlog and its graph, review, Insights, the talks, engines and projects, settings) and its flows (adding a project, starting a run, the command palette). Derived from the console's language until then.

## 29. References

- orchestra 0.9.1: `github.com/noesis-sol/orchestra` (docs: running, workers, setup, organs, dashboard, events; `skills/orchestra/SKILL.md`).
- tutti: `github.com/noesis-sol/tutti` (spec `.orchestra/specs/tutti-spec.md`, `docs/parity.md`, the carry-over epic tutti-kkb).
- `hivemind.md` (2026-10-21) and its references: Pi (extensions, packages, `earendil-works/pi`, `sjunepark/pi-effect`), Effect v4 (SqlClient, Migrator, SqliteClient for Node, sql-clickhouse, RPC), Foldkit, the work-graph references (Beads, taskgraph, clu, OpenTasks), the deferred engines (Turso, libSQL, LiveStore), the Electron landscape.
- The Hivemind Console design handoff (`design_handoff_hivemind_console`, 2026-10-21), kept under `design/console/`.
- Added in the refinement: Pi's RPC mode, extension events and `pi-ai`'s OAuth sign-ins (Pi's docs and source at the pinned version; issue #7368 on `before_provider_request`); DuckDB's Node package (`@duckdb/node-api`) and its Appender; Effect RPC's socket and WebSocket protocols; Cytoscape.js's dagre and ELK layouts; `tsgo`; Playwright's Electron support.
