# Agent Orchestrator Desktop — Technology Stack Spec

Oct 21, 2026 · @Michael Michailidis

## 1. Purpose, scope and conventions

This spec fixes the technology stack for a desktop app that runs and coordinates several Pi coding agents. Its reader is an AI agent that will refine it into an implementation plan.

In scope: technology choices, process boundaries, data stores, contracts, packaging, security and the development toolchain. Out of scope: product features, UX flows, pricing and branding.

**Markers**

- **MUST / SHOULD / MAY** follow RFC 2119.
- **\[DECISION\]** is settled. Change it only with a stated reason and a note in section 17.
- **\[OPEN\]** is undecided. Propose options with trade-offs and a recommendation.
- **\[VERIFY\]** is a fact read from docs on 2026-10-21. Check it against the current release or vendored source before building on it.

**Rules for the refining agent**

- Pin every dependency to an exact version. "latest" is never a valid value.
- Do not add a dependency that ships native binaries without an entry in the native inventory (section 13).
- Every dependency MUST run on all four targets in section 3. A gap on any target blocks adoption.
- Prefer the vendored source of Effect, Foldkit and Pi over web search when an API is in doubt.

## 2. Product context and stack at a glance

The app supervises several Pi agents working in parallel on one git repository. Each agent runs in its own process and git worktree, takes work from a shared dependency graph, and hands results to other agents through typed contracts.

The differentiator is coordination, not parallelism: typed hand-offs, a shared work graph, supervision, budgets and human approval points. Comparable products run agents side by side (Conductor, Superset, Nimbalyst, Maestro, Paseo, Codex desktop); Gas Town coordinates agents on top of Beads.

| Layer | Choice | Status |
| --- | --- | --- |
| Desktop shell | Electron | \[DECISION\] |
| Language | TypeScript, strict, ESM | \[DECISION\] |
| Core runtime | Effect v4, one exact version across every process | \[DECISION\] |
| Agent harness | Pi coding agent, one process per agent, wrapped in an Effect service | \[DECISION\] |
| UI framework | Foldkit (Elm Architecture built on Effect) | \[DECISION\] |
| UI components | `@foldkit/ui` headless components + Tailwind CSS | \[DECISION\] |
| Work graph (ticketing) | SQLite through Effect SQL (`@effect/sql-sqlite-node`, `node:sqlite`) | \[DECISION\] |
| Telemetry | Effect tracing and metrics, shipped to ClickHouse | \[DECISION\]; hosting \[OPEN\] |
| Platforms | macOS and Linux, x64 and arm64 | \[DECISION\] |
| Build | pnpm workspaces, electron-vite, Foldkit Vite plugin | SHOULD |

## 3. Platform targets

The app MUST run on every Mac and every mainstream Linux machine: four targets, each built, signed where applicable, and smoke-tested in CI. \[DECISION\]

| OS | Architectures | Package formats | Notes |
| --- | --- | --- | --- |
| macOS | x64 (Intel), arm64 (Apple Silicon) | dmg, zip | Developer ID signed and notarised. Minimum macOS follows the pinned Electron's floor \[VERIFY\]. Per-arch builds or a universal build \[OPEN\]. |
| Linux (glibc) | x64, arm64 | AppImage, deb, rpm | Minimum glibc follows Electron's floor \[VERIFY\]. Both X11 and Wayland. |

**Constraints**

- Every dependency with native code MUST ship builds for darwin-x64, darwin-arm64, linux-x64 and linux-arm64. Example of a blocked dependency today: `@tursodatabase/database` has no darwin-x64 build.
- v1 goal: zero native Node addons beyond Electron itself. SQLite comes from Node's built-in `node:sqlite`.
- Linux armv7l (32-bit ARM) \[OPEN\]. Electron supports it; decide on demand and CI cost.
- musl distributions (Alpine) are out of scope. Electron does not ship official musl builds \[VERIFY\].
- Windows is out of scope for v1. Code SHOULD stay portable: paths through Effect's `Path` and `NodePath`, shell assumptions confined to the agent layer.

## 4. Process architecture

The app runs as four kinds of process, and the main process stays thin: orchestration lives in a separate utility process, and each agent is its own OS process. \[DECISION\]

&#91;embedded content: process architecture · 4 process kinds, 2 data stores\]

Data flows renderer to orchestrator to agents; main only spawns the orchestrator and hands over the ports. Agents never open the database: the orchestrator is its sole writer.

| Process | Runs | Owns | MUST NOT |
| --- | --- | --- | --- |
| Renderer | Foldkit app | UI state (the Foldkit Model) | Access Node, Pi, the file system or any database |
| Main | Electron main | Windows, menus, app lifecycle, secret storage, updates, spawning the orchestrator, brokering MessagePorts | Run orchestration logic or long-running work |
| Orchestrator | Electron `utilityProcess` running the Effect runtime | Supervision, the work graph (sole SQLite writer), git worktrees, agent process lifecycle, telemetry pipeline | Render UI |
| Agent (one per running agent) | Pi coding agent in RPC mode plus the first-party Pi package | One agent session in one worktree | Open the work-graph database, talk to other agents directly |

**Rules**

- Everything with a lifetime is an Effect scoped resource. Closing the app's root scope aborts sessions, terminates agent processes and applies the worktree cleanup policy.
- The orchestrator is the only writer of the work graph. Agents reach it only through orchestrator-implemented tools.
- `node:sqlite` is synchronous, so its busy waits block the event loop. That is acceptable in the orchestrator process and never acceptable in main.
- Only the main process can create a `utilityProcess` \[VERIFY\]. The orchestrator spawns agents as ordinary child processes (section 7).

## 5. Language, runtime and repository

One TypeScript monorepo, one exact Effect version everywhere, and Foldkit's pinned Effect peer version sets that number. \[DECISION\]

**Versions**

- Effect v4: every `effect` and `@effect/*` package on the same exact version. Foldkit pins its peers exactly (`effect@4.0.0`, `@effect/platform-browser@4.0.0` when read) \[VERIFY current pin\]. The monorepo follows that pin and upgrades with Foldkit.
- Effect's SQL and RPC modules are marked unstable in the v4 API reference. Pin exactly and keep them behind the project's own services.
- Electron: latest stable major at project start, pinned. Stay within Electron's supported majors.
- Node: Electron's bundled Node. It MUST provide `node:sqlite`, `MessageChannel` and `child_process` \[VERIFY `node:sqlite` stability in the pinned Electron\].
- Pi (`@earendil-works/pi-coding-agent`): exact version, upgraded deliberately with its contract tests.
- Foldkit: exact version, pre-1.0 with breaking changes allowed in minors. Upgrade with the vendored subtree and `FOLDKIT.md` (section 14).

**Toolchain**

- pnpm workspaces. electron-vite builds main, preload and renderer; the renderer uses `@foldkit/vite-plugin`. The orchestrator is bundled as a separate entry \[VERIFY electron-vite utility-process entry support\].
- Type checking with `tsc` or `tsgo`, plus the Effect Language Service \[OPEN which checker\].
- oxlint with `@foldkit/oxlint-plugin`, and oxfmt, matching Foldkit's scaffold.

**Layout**

```text
apps/desktop/            Electron main, preload, packaging config
packages/contracts/      Effect Schema: domain types, IPC/RPC contracts, agent tool payloads (depends on effect only)
packages/orchestrator/   Effect services: Supervisor, AgentPool, PiAgent, WorkGraph, Worktrees, Telemetry
packages/pi-extension/   First-party Pi package loaded into every agent: tools and gates
packages/ui/             Foldkit renderer app
repos/                   Vendored git subtrees (Foldkit, Effect, Pi), read-only reference
```

`contracts` is the shared vocabulary. Every other package imports it; it imports nothing but `effect`.

## 6. IPC and contracts

Every message that crosses a process boundary is defined once in `packages/contracts` as Effect Schema and decoded on arrival. No untyped `ipcRenderer` channels exist. \[DECISION\]

**Channels**

| From | To | Transport | Carries |
| --- | --- | --- | --- |
| Renderer | Orchestrator | `MessageChannelMain` ports, brokered once by main | All app data: requests, streams, events |
| Renderer | Main | Preload `contextBridge`, minimal API | Window controls, folder picker, app version, update status |
| Orchestrator | Agent | Pi RPC over stdio | Prompts, aborts, Pi session events |
| Agent (Pi extension) | Orchestrator | Per-agent local channel \[OPEN, section 7\] | Tool calls: next task, report result, file issue, ask human |

**Protocol**

- Renderer and orchestrator talk through Effect's RPC module over the MessagePort \[VERIFY v4 RPC module path and a MessagePort transport\]. Fallback: a thin tagged request/response/stream protocol with Schema-encoded frames.
- After brokering the ports, main leaves the data path.
- Each preload method validates its input and output with Schema.

**Streaming**

- The orchestrator batches high-frequency events (tokens, tool output) before they cross, at about one flush per 50 ms \[OPEN exact interval\].
- The renderer applies stream updates at most once per animation frame.
- In Foldkit: the RPC connection is a ManagedResource, streams are Subscriptions that emit Messages, and requests are Commands.

**Versioning**

- All processes ship in one build, so cross-process contracts need no backward compatibility.
- Persisted data (SQLite) and telemetry rows are versioned through migrations (sections 9 and 10).

## 7. Agent layer (Pi + Effect)

Each agent is a Pi process in RPC mode, owned by the orchestrator as an Effect scoped resource behind a `PiAgent` service. Pi's Promise and RPC surface never leaks past that service. \[DECISION\]

**Process model**

- Spawn: Pi's CLI in RPC mode over stdio, one process per agent, cwd = the agent's worktree.
- Acquire = spawn + handshake. Release = Pi abort, then terminate, then reap. Effect interruption maps to Pi abort.
- Runtime for the child \[OPEN\]: Electron's binary with `ELECTRON_RUN_AS_NODE=1`, or a bundled Node. The first conflicts with disabling the RunAsNode fuse (section 12).
- Pi is bundled at an exact version and unpacked from the asar archive so child processes can load it \[VERIFY\]. Audit Pi for native dependencies on all four targets \[VERIFY\].
- Reference implementation for the boundary: `sjunepark/pi-effect` (interruption to `session.abort()`, loss-safe event streams). Write the project's own adapter; do not depend on it.

**First-party Pi package** (`packages/pi-extension`, loaded into every agent)

| Integration | Pi API | Purpose |
| --- | --- | --- |
| Tools | `pi.registerTool()` | `next_task`, `report_result`, `file_issue`, `ask_human`, `read_board`, `post_note` |
| Write and command gate | `tool_call` event | Confine file writes and bash cwd to the worktree; block `git push` and destructive commands per policy; route risky calls to human approval |
| Completion signal | `agent_settled` event | Tells the orchestrator the run has fully settled |
| Metadata | `pi.appendEntry()` | Stores orchestration data in the session without spending context |
| Provider permits | `before_provider_request` event | Ask the orchestrator for a rate-limit permit before each model call \[VERIFY the handler can await\] |

- Tool handlers never touch the database. They forward to the orchestrator over a per-agent channel: a Unix domain socket whose path arrives in the agent's environment, speaking the Schema RPC protocol \[OPEN versus alternatives\].
- Tool parameter schemas: Pi uses TypeBox, the project uses Effect Schema. Single source in `contracts`, converted through JSON Schema \[OPEN exact route\].
- Only the first-party package loads. Project-level and third-party Pi packages stay disabled unless the user opts in.

**Models and credentials**

- Pi's provider layer (`pi-ai`) makes all model calls.
- API keys live in OS secret storage (section 12) and are injected into each agent's environment at spawn. They are never written to the worktree \[VERIFY Pi's supported credential inputs\].

**Worktrees**

- One git worktree per agent task branch, created and removed as a scoped resource.
- Naming convention and cleanup policy (keep failed worktrees for inspection, or clean up) \[OPEN\].

## 8. Coordination model

Coordination has two tiers: live state in Effect primitives inside the orchestrator, and durable state in the SQLite work graph. Agents never talk to each other directly. \[DECISION\]

| Tier | Lives in | Holds | Effect primitives |
| --- | --- | --- | --- |
| Live | Orchestrator memory | Agent events, interrupts, broadcasts ("the API contract changed"), UI streams, pending human questions | `Queue`, `PubSub`, `SubscriptionRef`, `Deferred` |
| Durable | SQLite work graph | Plan, tasks, dependencies, claims and leases, results, provenance, audit events | `WorkGraph` service over `SqlClient` |

**Hand-offs**

- `report_result` takes a Schema-validated payload defined per task kind in `contracts`.
- An invalid payload fails the tool call inside Pi's loop, so the agent retries with the decode error in its context.
- Downstream tasks receive upstream results as structured hand-off context.

**Supervision** (a `Supervisor` service, one fiber per agent)

| Failure (tagged error) | Policy |
| --- | --- |
| `RateLimited` | Retry with exponential backoff (`Schedule`) |
| `AgentStuck` (no events for N minutes) | Timeout, then restart with a summary of the session |
| `ToolDefect` | Fail the task, file a follow-up issue |
| `BudgetExceeded` | Interrupt the task's scope |
| `HumanRejected` | Return the task to the queue with the reviewer's note |

**Limits and control**

- Global agent cap and per-provider cap, enforced with semaphores. Values configurable \[OPEN defaults\].
- Token and cost budgets per task and per run, tracked in `Ref` counters fed by Pi usage events.
- `ask_human` suspends the agent on a `Deferred` until the user answers in the UI. The agent's status shows "waiting on you".
- Cancelling a task interrupts its agent fiber, which aborts Pi and terminates the process. Cascading cancel along `blocks` edges SHOULD be offered.

## 9. Work graph (SQLite)

The ticketing system is the project's own dependency-aware work graph on SQLite, accessed only through a `WorkGraph` Effect service. Beads is a design reference, not a dependency. \[DECISION\]

**Engine**

- `@effect/sql-sqlite-node`, which uses Node's built-in `node:sqlite`: one serialised connection, WAL on by default, 5-second busy timeout. No native addon to rebuild or sign.
- The orchestrator is the single writer. Writes that must be atomic run in `withTransaction`.
- Location: one database per repository, at `<repo>/.<app>/work.db`, git-ignored by default \[OPEN folder name\]. App-level state lives in Electron's `userData`, not in the repo.

**Interface** (the only code that issues SQL against the work graph)

`ready`, `claimNext`, `renewLease`, `complete`, `fail`, `addDependency`, `fileDiscovered`, `cancelCascade`, `history`, `readyStream`.

**Initial schema**

```sql
CREATE TABLE tasks (
  id TEXT PRIMARY KEY,                 -- short hash id, e.g. t-a1b2
  kind TEXT NOT NULL,                  -- selects the result Schema in contracts
  title TEXT NOT NULL, body TEXT,
  status TEXT NOT NULL CHECK (status IN ('open','claimed','done','failed','cancelled')),
  priority INTEGER NOT NULL DEFAULT 2,
  assignee TEXT, lease_expires_at INTEGER,
  result TEXT,                         -- JSON hand-off for downstream tasks
  created_at INTEGER NOT NULL
);
CREATE TABLE edges (
  from_id TEXT NOT NULL REFERENCES tasks(id),
  to_id   TEXT NOT NULL REFERENCES tasks(id),
  kind TEXT NOT NULL CHECK (kind IN ('blocks','parent','discovered_from','related')),
  PRIMARY KEY (from_id, to_id, kind)
);
CREATE TABLE events (                  -- append-only audit log and UI activity feed
  seq INTEGER PRIMARY KEY, task_id TEXT, type TEXT NOT NULL, actor TEXT, payload TEXT, at INTEGER NOT NULL
);
CREATE VIEW ready AS
SELECT t.* FROM tasks t
WHERE (t.status = 'open' OR (t.status = 'claimed' AND t.lease_expires_at < unixepoch()))
  AND NOT EXISTS (
    SELECT 1 FROM edges e JOIN tasks b ON b.id = e.from_id
    WHERE e.to_id = t.id AND e.kind = 'blocks' AND b.status <> 'done');
```

Atomic claim:

```sql
UPDATE tasks SET status = 'claimed', assignee = ?, lease_expires_at = unixepoch() + ?
WHERE id = (SELECT id FROM ready ORDER BY priority, created_at LIMIT 1)
RETURNING *;
```

**Rules**

- Reject cycles on insert of `blocks` edges. Add indexes on `edges(to_id)`, `tasks(status, priority, created_at)` and `events(task_id)`.
- Every mutation appends an `events` row in the same transaction.
- Rows decode into Schema classes from `contracts`. The same types reach the UI unchanged.
- Leases renew from agent activity. An expired lease returns the task to `ready`.

**Migrations and reactivity**

- Effect `Migrator`, forward-only, loaded with `fromRecord` or `fromGlob` so migrations are bundled. Never `fromFileSystem`: paths inside a packaged asar break it.
- Back up the file with `SqliteClient.backup` before migrating. Refuse to open a database newer than the app's schema.
- `sql.reactive(["tasks"], …)` streams the ready queue and board to the UI. Writes invalidate the matching keys.

**Interop and future engines**

- Beads-compatible JSONL import and export MAY ship after v1 \[VERIFY Beads' JSONL format\].
- Keep all SQL inside the `WorkGraph` implementation and within SQLite's dialect. That keeps a later Layer swap to Turso Database or libSQL cheap (section 16).

## 10. Telemetry (ClickHouse)

The orchestrator records traces, metrics and domain events with Effect's built-in telemetry, buffers them locally, and ships them to a remote ClickHouse. The app MUST work fully with ClickHouse absent or unreachable. \[DECISION\]

**Instrumentation**

- Spans with `Effect.withSpan` around each agent turn, tool call, work-graph operation and IPC request. Metrics as counters and histograms. Structured logs.
- Export through `@effect/opentelemetry`.
- Pi-side facts come from Pi events collected by the orchestrator: token usage and model per response, tool durations (`tool_execution_end.durationMs`), errors \[VERIFY the usage fields on Pi's message events\].

**Pipeline**

1. Local outbox: a separate SQLite file (`telemetry.db`) in `userData`, never in the repo, so telemetry can't contend with the work graph.
2. Batched shipping with retry and backoff; bounded outbox size with oldest-first eviction.
3. Destination \[OPEN\]: (a) OTLP to an OpenTelemetry Collector with its ClickHouse exporter, standard schema; or (b) direct inserts with `@effect/sql-clickhouse` into app-defined tables. Leaning: (a) for spans and logs, (b) for domain analytics tables.

**Deployment**

- ClickHouse runs as a remote server, self-hosted or ClickHouse Cloud. The desktop app does not embed it: chDB or clickhouse-local would add native binaries for all four targets. \[DECISION\]
- Who hosts it, and whether end users can point the app at their own instance \[OPEN\].

**Privacy**

- Off by default; explicit opt-in.
- Default capture is metadata only: counts, durations, model ids, token totals, cost, error tags. Never prompts, code, file contents, secrets or absolute paths.
- Content capture, if ever offered, sits behind a separate explicit setting. A redaction step in the orchestrator runs before the outbox.

**Initial tables (sketch)**

| Table | Grain | Key columns |
| --- | --- | --- |
| `agent_runs` | One agent session | run id, task kind, model, provider, start, end, outcome, tokens in/out, cost |
| `llm_requests` | One model call | run id, model, latency, tokens in/out, cost, error tag |
| `tool_calls` | One tool execution | run id, tool name, duration, is error, gated (bool) |
| `task_events` | One work-graph event | task kind, event type, actor role, timestamp |

Engines, partitioning and TTL (for example MergeTree partitioned by month) \[OPEN\].

## 11. UI (Foldkit + Foldkit UI + Tailwind)

The renderer is a Foldkit program: one Schema-defined Model, Messages, a pure `update`, and side effects returned as Commands. Behaviour comes from `@foldkit/ui`, appearance from Tailwind. \[DECISION\]

**Foldkit mapping**

| Concern | Foldkit construct |
| --- | --- |
| RPC connection to the orchestrator | ManagedResource |
| Agent event streams, ready queue, board | Subscriptions that emit Messages |
| Requests (claim, approve, interrupt, open repo) | Commands (Effects returning Messages) |
| Per-agent panes, dialogs, menus | Submodels with OutMessages to the parent |
| Third-party DOM widgets | Mount |

**Components**

- `@foldkit/ui` headless primitives: dialog, menu, tabs, listbox, disclosure, virtual list \[VERIFY the current component set\].
- Tailwind CSS with design tokens as CSS variables: a neutral palette, one accent, tight radii, dense spacing, light and dark themes. shadcn/ui's visual language is the reference; its React components are not used.
- Mounted libraries (pure JavaScript, no native code): Monaco for diffs and code, xterm.js to render agent terminal output read-only, Cytoscape.js for the task and agent graph \[OPEN graph library\].
- Split panels and trees: from `@foldkit/ui` if available; otherwise Web Awesome components, used for presentation only \[OPEN\].
- Interactive shells typed into by the user need `node-pty`, a native addon. Out of v1 \[OPEN\].

**Performance**

- Never push each token through `update`. Buffer stream events and dispatch at most once per animation frame.
- Lazy-memoised views for inactive agents. Virtualise every long list and log.
- Enable Foldkit's slow-update warnings in development.

**Desktop UX rules**

- Density over whitespace: small type, tight rows, monospace for agent output.
- Keyboard first: command palette, shortcuts for approve, reject and interrupt, predictable focus between panes.
- One consistent agent status indicator: running, waiting on you, blocked, failed, done.
- Design references: Linear, Codex desktop, Claude desktop, and VS Code's workbench layout.

**Tooling**

- Foldkit DevTools and DevTools MCP in development builds only, removed from production builds \[VERIFY how to exclude them\].

## 12. Security

The renderer is treated like an untrusted web page, and agents are treated as code running with the user's full permissions, because Pi does not sandbox tool calls. \[DECISION\]

**Renderer and Electron hardening**

- `contextIsolation: true`, `sandbox: true`, `nodeIntegration: false`, `webSecurity: true`. No `<webview>`.
- Strict Content Security Policy: bundled assets only, no remote scripts, no `eval`.
- Block in-app navigation and `window.open` to external URLs; open links in the system browser.
- Electron fuses: enable cookie encryption, only-load-app-from-asar and embedded asar integrity validation \[VERIFY per platform\].
- The RunAsNode fuse SHOULD be disabled for hardening. That conflicts with spawning Pi through `ELECTRON_RUN_AS_NODE` (section 7) \[OPEN\].

**Agents**

- Every agent runs with the user's OS permissions. The `tool_call` gate is the main control: path confinement to the worktree, a command policy, and human approval for pushes, network-reaching commands and destructive operations.
- Each agent gets a minimal environment: only the credentials its provider needs.
- Only the first-party Pi package loads. A repository's own `.pi/` settings and packages load only after the user trusts that repository.
- Container isolation per agent is a later option \[OPEN\].

**Secrets**

- Electron `safeStorage`: macOS Keychain; on Linux, libsecret or KWallet.
- On Linux without a keyring, `safeStorage` can fall back to weak or plain-text storage \[VERIFY\]. The app MUST detect that and warn before storing keys.
- Secrets never enter the work graph, telemetry or worktrees.

**Supply chain**

- Exact pins and a committed lockfile. CI verifies checksums of every downloaded artifact.
- Third-party Pi packages are disabled by default.

## 13. Packaging, signing and distribution

electron-builder produces signed packages for the four targets in CI, and the v1 bundle contains no native code beyond Electron itself. \[DECISION on targets; electron-builder SHOULD, Electron Forge is the alternative\]

**Targets**

| Target | Artifacts | Signing | Updates |
| --- | --- | --- | --- |
| macOS x64, arm64 | dmg, zip | Developer ID, hardened runtime, notarised; every Mach-O in the bundle signed | electron-updater from the zip |
| Linux x64, arm64 | AppImage, deb, rpm | Package checksums; optional repository signing | AppImage self-update; deb and rpm through a package repository or manual \[OPEN\] |

**Rules**

- macOS entitlements: only what V8 and Electron need \[VERIFY the minimal set\].
- Linux: AppImage needs FUSE on the host; check Chromium's sandbox helper behaviour per distribution \[VERIFY\].
- Pi and the first-party Pi package are unpacked from the asar archive so agent child processes can load them \[VERIFY\].
- Bundled migrations only (section 9); no runtime reads of source directories.

**Native inventory** (anything with native code MUST appear here)

| Component | Native code | Status |
| --- | --- | --- |
| SQLite | None; built into Node (`node:sqlite`) | In |
| Pi coding agent | Unknown | \[VERIFY\] all four targets |
| Monaco, xterm.js, Cytoscape.js | None | In |
| `node-pty` (interactive shells) | Yes | Out of v1 |
| Beads `bd` binary | Yes (Go, embedded Dolt) | Out |
| Turso Database (`@tursodatabase/database`) | Yes; no darwin-x64 build | Out |
| DuckDB (`@duckdb/node-api`) | Yes | Out |
| chDB or clickhouse-local | Yes | Out |

**CI**

- Build matrix: macOS x64, macOS arm64, Linux x64, Linux arm64 runners.
- Smoke test every packaged artifact: launch, open a fixture repository, run one agent against a stub model, close cleanly with no orphan processes.

## 14. Developing with coding agents

The project will be built largely by coding agents on young, fast-moving libraries, so agents get pinned source to read, mechanical checks after every write, and a fixed order of work. \[DECISION\]

**Context**

- Vendor Foldkit, Effect v4 and Pi as git subtrees under `repos/`, each pinned to the exact installed version and refreshed on every upgrade. Read-only reference; application code imports from `node_modules`.
- Foldkit's tag pattern is `foldkit@<version>`. Replace `FOLDKIT.md` wholesale from that tag on every upgrade; project rules live in `AGENTS.md`.
- Install Foldkit's official agent skills. For Effect, use source-first skills (for example `joelhooks/pi-effect` or `pi-effect-harness` when developing inside Pi).
- Foldkit docs are available as Markdown (`.md` suffix, `llms-full.txt`) for agents without the subtree.

**Enforcement** (hooks, not prompts)

- After every file write: type-check and `oxlint` (with `@foldkit/oxlint-plugin`) on changed files, errors returned to the agent in-band.
- Claude Code: a `PostToolUse` hook. Pi: a `tool_result` handler in a development-only Pi package.

**Feedback**

- Foldkit DevTools MCP connected in development, so agents can inspect the Model, replay Messages and dispatch Messages against the running app.

**Order of work**

1. UI features: Model and Message changes first, reviewed by a human; then story tests; then `update`; then `view`. Point the agent at a named Foldkit example.
2. Orchestrator features: Schema contracts in `contracts` first; then the service interface; then tests against in-memory layers; then the implementation.

## 15. Testing and quality gates

Every layer is tested against Effect Layers swapped for test implementations, so no unit test needs a real model, a real repository or a browser. \[DECISION\]

| Layer | Approach | Tools |
| --- | --- | --- |
| Work graph | Real migrations against in-memory SQLite (`:memory:` Layer); leases and timeouts with `TestClock` | `@effect/vitest` |
| Orchestrator services | `FakePiAgent` Layer replaying scripted event streams; supervision policies driven by injected tagged errors | `@effect/vitest` |
| Pi contract | Real pinned Pi in RPC mode against a stub model provider, run in CI on every Pi upgrade \[VERIFY Pi stub-provider support\] | Vitest |
| UI state | Foldkit story tests on `update`, Commands resolved inline | `foldkit/story` |
| UI behaviour | Foldkit scene tests through the view with accessible locators | `foldkit/scene` |
| Packaged app | Smoke test per target (section 13) | Playwright for Electron \[OPEN\] |

**Gates on every pull request**

- Type-check, lint, format check.
- Unit, story and scene tests.
- Migrations: up from an empty database, and up from the previous release's database fixture.
- No new entry in the native inventory without review.

## 16. Non-goals and deferred options

Each option below was evaluated and set aside for v1. Do not reintroduce one without meeting its revisit condition.

| Option | Status | Reason | Revisit when |
| --- | --- | --- | --- |
| Windows | Out of v1 | Not a target | A product decision adds it |
| Beads (`bd` binary) | Not used | Go binary plus embedded Dolt to bundle and sign; a single-writer orchestrator makes an own graph cheap | Only JSONL interop is wanted |
| Turso Database | Preferred future engine | Concurrent writes, change data capture and local-first sync fit; pre-1.0, no darwin-x64 build, no Effect driver | 1.0, all four targets covered, sync story settled; then write an Effect driver and swap the Layer |
| libSQL | Deferred | Maintained, but new work goes to Turso Database; native addon | Turso stalls and sync is needed |
| LiveStore | Deferred | Beta; Effect SQL reactive queries cover live UI | Multi-device sync is needed |
| PGlite or Postgres | Deferred | Only useful for a server-backed team version | A Postgres-backed team or cloud mode is planned |
| DuckDB | Deferred | Analytics engine, not transactional | An on-demand Insights panel reading the SQLite file and Pi session JSONL |
| React and React Flow | Not used | Foldkit chosen | Never, unless Foldkit is replaced |
| Interactive shells (`node-pty`) | Deferred | Native addon | Users need to type into agent shells |
| Cloud sync and multi-device | Deferred | Not needed for a single-machine orchestrator | Team features are planned |

## 17. Open questions for the refining agent

Resolve these first, in this order: each one shapes the sections after it. For each, propose options, trade-offs and a recommendation, then update the section it names.

- [ ] **Agent child runtime** (sections 7, 12): `ELECTRON_RUN_AS_NODE` versus a bundled Node versus agents spawned by main. Decide together with the RunAsNode fuse.
- [ ] **Extension-to-orchestrator channel** (sections 6, 7): per-agent Unix domain socket versus an alternative. Must carry Schema RPC and survive agent restarts.
- [ ] **Tool schema single source** (section 7): Effect Schema to JSON Schema to TypeBox, or another route.
- [ ] **Provider permits** (sections 7, 8): confirm `before_provider_request` can await a permit; otherwise cap concurrency per agent.
- [ ] **Effect RPC over MessagePort** (section 6): confirm the v4 module and transport; otherwise specify the fallback protocol.
- [ ] **ClickHouse hosting and path** (section 10): who hosts, OTLP collector versus direct inserts, the consent flow.
- [ ] **On-disk names** (section 9): app name, per-repo folder name, git-ignore behaviour.
- [ ] **Worktree lifecycle** (section 7): naming, cleanup, keep-on-failure.
- [ ] **macOS builds** (sections 3, 13): universal versus per-arch, minimum macOS.
- [ ] **Linux scope** (sections 3, 13): armv7l, minimum glibc, update channels for deb and rpm.
- [ ] **Linux secrets without a keyring** (section 12): warn, refuse, or offer an encrypted file with a passphrase.
- [ ] **UI gaps** (section 11): graph library, split panels and trees, packaged-app test tool.
- [ ] **Beads JSONL interop** (section 9): in v1 or later.

## 18. References

Pages read while forming this spec, as of 2026-10-21.

**Pi**

- [Pi extensions](https://pi.dev/docs/latest/extensions) · [Pi packages](https://pi.dev/docs/latest/packages) · [earendil-works/pi](https://github.com/earendil-works/pi)
- [sjunepark/pi-effect](https://github.com/sjunepark/pi-effect) · [pi-effect-harness](https://pi.dev/packages/pi-effect-harness?page=) · [joelhooks/pi-effect](https://github.com/joelhooks/pi-effect)

**Effect v4**

- [API reference index](https://effect.website/docs/v4/api) · [SqlClient](https://effect.website/docs/v4/api/effect/sql/SqlClient) · [SqlModel](https://effect.website/docs/v4/api/effect/sql/SqlModel) · [Migrator](https://effect.website/docs/v4/api/effect/sql/Migrator)
- [SqliteClient for Node](https://effect.website/docs/v4/api/sql-sqlite-node/SqliteClient) · [SqliteMigrator for Node](https://effect.website/docs/v4/api/sql-sqlite-node/SqliteMigrator) · [sql-clickhouse](https://effect.website/docs/v4/api/sql-clickhouse)

**Foldkit**

- [Foldkit](https://foldkit.dev/) · [Get started](https://foldkit.dev/get-started) · [AI-assisted development](https://foldkit.dev/ai/overview) · [foldkit/foldkit](https://github.com/foldkit/foldkit/)

**Work graph design references**

- [Beads](https://github.com/gastownhall/beads/) · [Beads architecture](https://beads.gascity.com/architecture) · [taskgraph](https://github.com/BudEcosystem/taskgraph) · [clu](https://github.com/Arjia-Labs/clu) · [OpenTasks](https://github.com/alexngai/opentasks)

**Deferred engines**

- [Turso Database](https://github.com/tursodatabase/turso/) · [Turso TypeScript reference](https://docs.turso.tech/sdk/ts/reference) · [libSQL](https://docs.turso.tech/libsql) · [LiveStore](https://github.com/livestorejs/livestore)

**Electron landscape**

- [2026 audit of Electron apps](https://codenote.net/en/posts/famous-electron-apps-2026-research/) · [Agent orchestrator desktop apps](https://rywalker.com/research/t/electron)
