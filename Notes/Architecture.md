# ProcPM Architecture
## 1. What ProcPM Actually Does

ProcPM is a single tool that replaces the "open five terminals" workflow for running a multi-service project. Concretely, it:

- Reads a project's `procpm.toml` to learn what services exist, how to start them, in what order, and how to restart them.
- Starts, stops, and restarts those services as supervised child processes.
- Tracks their runtime state (PID, CPU, RAM, uptime, crash/restart counts) in a local SQLite database.
- Captures each service's stdout/stderr into per-service log files, with tailing/follow support.
- Applies restart policies (e.g. `on-failure`) with a bounded retry count, so a broken service fails permanently instead of crash-looping forever.
- Resolves and validates a dependency graph between services (start order, circular-dependency detection).
- Exposes all of the above through a CLI, a TUI dashboard, and (optionally) an HTTP API.
- Lets third parties extend behavior via plugins.
- Self-diagnoses its own setup via `procpm doctor`.

In short: it's a **local process supervisor + project manager**, scoped to one project directory, persisted across invocations.

---

## 2. Major Systems

|System|Responsibility|
|---|---|
|**Config Loader**|Parses `procpm.toml`, validates schema, resolves defaults|
|**Dependency Resolver**|Builds a DAG from `depends_on`, computes start order, detects cycles|
|**Process Supervisor**|Spawns, stops, restarts OS processes; applies restart policy|
|**Monitor**|Polls/observes running processes for CPU, RAM, uptime, exit codes|
|**Log Manager**|Captures stdout/stderr per service, writes to `.procpm/logs/`, serves tail/follow|
|**State Store (SQLite)**|Durable record of projects, services, runs, crashes, restarts, events|
|**Plugin Host**|Loads plugins, exposes hook points, mediates plugin access to core systems|
|**CLI**|Command parsing, rich terminal output, one-shot commands|
|**TUI**|Long-lived interactive dashboard view (Textual)|
|**API Server**|Optional HTTP interface exposing the same operations as CLI/TUI|
|**Doctor**|Runs read-only checks against the other systems and reports pass/warn/fail|

The **Core** (Supervisor + Monitor + Log Manager + State Store + Dependency Resolver) is the actual engine. CLI, TUI, and API are three interchangeable "views" onto that same Core — none of them contain business logic themselves (see 5).

---

## 3. How the Systems Communicate

**[design decision — this is the central architectural choice a tool like this needs to make]**

Two things need to talk to each other in a fundamentally different way, so they should use two different channels:

### a) Core internals (Supervisor ↔ Monitor ↔ Log Manager ↔ State Store)

These live **in-process**, inside a single long-running background daemon (`procpmd`), and communicate via direct function calls / an in-memory event bus. There's no need for IPC between them — they're not separate programs, they're modules of the same engine.

- Supervisor emits events (`service.started`, `service.crashed`, `service.exited`) onto an internal event bus.
- Monitor and State Store subscribe to that bus: State Store persists the event, Monitor updates live metrics.
- Log Manager attaches directly to each child process's stdout/stderr pipes at spawn time — it doesn't go through the event bus, since log data volume is much higher than event volume.

### b) Daemon ↔ Clients (CLI, TUI, API)

The CLI, TUI, and (network) API are all separate OS processes from the daemon, so they talk to it over a **local IPC socket** (Unix domain socket / named pipe), using a small request/response + subscribe protocol:

- **Commands** (`start`, `stop`, `restart`, `status`) → request/response, like an RPC call.
- **Live state** (TUI dashboard, `logs --follow`) → the client subscribes and the daemon streams events/log lines over the same socket.
- The HTTP API server is really just another client of this socket, translating HTTP requests into the same IPC calls — it does not talk to the Supervisor directly.

This keeps exactly one process ever touching the actual OS processes and the SQLite file, which avoids the classic multi-writer SQLite locking problem and avoids two CLI invocations racing to supervise the same service.

**Why not have the CLI supervise processes directly (no daemon)?** Because `procpm status` needs to reflect services started by a _previous_ `procpm start` invocation, potentially in a different terminal. That requires a persistent background process — the daemon — rather than each CLI call being self-contained.

---

## 4. What Data Each System Owns

Ownership means: **only this system writes this data; everyone else reads it through that system's interface.**

|System|Owns|
|---|---|
|**Config Loader**|The parsed, validated in-memory representation of `procpm.toml`|
|**Dependency Resolver**|The computed start-order graph (derived, not persisted — recomputed from config)|
|**Process Supervisor**|The live OS handle for each running process (PID, process object, restart counter _for the current daemon lifetime_)|
|**Monitor**|Point-in-time resource metrics (CPU%, RAM) — ephemeral, not persisted, sampled on demand|
|**Log Manager**|Log files on disk (`.procpm/logs/*.log`) and any in-memory tail buffers|
|**State Store (SQLite)**|The durable historical record: projects, services, past runs, start/stop timestamps, crash events, restart counts _across daemon restarts_, plugin registry, config version history|
|**Plugin Host**|Loaded plugin instances and their registered hooks|

Notice there are **two places "restart count" can live**: the Supervisor's in-memory counter (used to enforce the `restart = "on-failure"` limit during the current daemon session) and the State Store's historical count (used for `procpm status` reporting and `doctor`). The Supervisor's counter is authoritative for _decisions_; the State Store's is authoritative for _reporting_. The Supervisor writes through to the State Store on every restart so the two never drift.

---

## 5. Where Business Logic Lives

**All business logic lives in the daemon's Core — never in the CLI, TUI, or API.**

Concretely:

- "Should this service restart?" → Supervisor (reads policy from Config).
- "What order do services start in?" → Dependency Resolver.
- "Is this a circular dependency?" → Dependency Resolver.
- "Has this service crashed too many times?" → Supervisor, checked against Config's restart limit.
- "Is port 3000 already in use?" → Doctor, calling into Monitor/OS.

The CLI's `start` command, the TUI's "start" keybinding, and the API's `POST /services/{name}/start` endpoint should all resolve to **the exact same daemon call**. None of them independently decide _whether_ to restart a service or _what order_ to start things in — they just collect arguments from the user and forward them.

**Why this matters:** if restart logic lived in the CLI instead, the TUI and API would need to reimplement it identically, and a plugin hooking into "before restart" would only fire for CLI-triggered restarts, not TUI ones. Centralizing in the daemon avoids all of that.

---

## 6. How Errors Are Handled

Three different kinds of failure need three different handling strategies:

1. **A supervised service crashes** (e.g. `python worker.py` exits non-zero) → Not a ProcPM error. Handled by policy: Supervisor checks `restart` config, retries up to the configured limit with the counter shown to the user (`Restarting... 1/5`), then marks the service `permanently failed` in the State Store and stops retrying. This is normal, expected operation, not an exception.
    
2. **A ProcPM operational error** (bad `procpm.toml`, port conflict, missing command, DB corruption) → Surfaced as a structured error with a category (config / runtime / io / plugin), not a raw stack trace. The CLI renders it as a readable message with a suggested fix; `procpm doctor` proactively checks for the most common of these _before_ they cause a failed start. **[design decision]**: define a small `ProcPMError` type hierarchy (`ConfigError`, `DependencyError`, `PortInUseError`, `PluginError`, etc.) so CLI/TUI/API can each format the same error appropriately for their surface (colored terminal text vs. HTTP status code vs. TUI modal).
    
3. **A daemon-level fault** (the daemon itself crashes or is killed) → Supervised services are OS child processes, so killing the daemon doesn't necessarily kill them (depends on process-group configuration — worth deciding deliberately: do you want services to die with the daemon, or survive it?). On daemon restart, it should reconcile: read the State Store's "services that were running," check if their PIDs are still alive, and either re-adopt them or mark them as `unknown/stopped`.
    

General rule: **errors from user code (the services) are data, not exceptions — errors from ProcPM itself are exceptions.**

---

## 7. How Processes Are Represented

A process moves through a small state machine, tracked by the Supervisor and mirrored into the State Store:

```
pending → starting → running → (stopping → stopped)
                         ↓
                      crashed → restarting → running
                         ↓
                  permanently_failed
```

Each service instance is represented as a record containing:

- **Identity**: service name, the project it belongs to
- **Definition** (from Config): command, port, `depends_on`, restart policy
- **Runtime** (from Supervisor, ephemeral): PID, current state, process handle
- **Metrics** (from Monitor, ephemeral, sampled): CPU%, RAM, uptime
- **History** (from State Store, durable): start/stop timestamps, restart count, exit codes, crash timestamps

The key design point: **the "process" as ProcPM models it is not just the OS PID.** It's a logical entity that persists across restarts — PID 18242 crashing and being replaced by PID 18391 is still "the same service," and the restart count/history has to survive that PID change. The OS PID is just the current runtime's transient handle on it.

---

## 8. How Configuration Works

- `procpm.toml` is the single source of truth for what services exist and how they behave — it's declarative, not imperative.
- The Config Loader parses it into a validated in-memory model at daemon startup (and on explicit reload, e.g. `procpm reload` or file-watch — worth deciding whether config changes are picked up live or require a restart **[open question]**).
- Validation happens _before_ anything is started: unknown keys, missing required fields, and dependency references to services that don't exist should all be caught here, and surfaced by `procpm doctor`.
- The State Store keeps a **version history of configuration** (per the spec's "configuration versions" data), so changes to `procpm.toml` over time are auditable — useful for answering "why did this service's restart policy change" after the fact.
- Config is read-only to every other system. Nothing downstream (Supervisor, Plugin Host, API) mutates it; if a plugin wants to influence behavior, it does so through hooks (§9), not by editing the parsed config in place.

---

## 9. How Plugins Interact With ProcPM

Plugins extend the Core without being compiled into it, via a **hook-based extension model** running inside the Plugin Host:

- The Plugin Host loads plugins at daemon startup (registered in `procpm.toml` or a plugin manifest) and keeps them isolated from directly touching the Supervisor's internals.
- Plugins register for **lifecycle hooks** emitted on the same internal event bus the Core uses (§3): `before_start`, `after_start`, `on_crash`, `before_restart`, `on_permanently_failed`, `on_log_line`, etc.
- Plugins can also **register new CLI subcommands** and **new API routes**, which the CLI/API systems discover from the Plugin Host at startup rather than hardcoding.
- Plugins interact with Core data (service list, logs, metrics) through a defined **plugin API surface** — not by reaching into the State Store's SQLite file directly. This keeps the schema free to change without breaking plugins, and lets the Plugin Host enforce permissions (e.g. a "read-only" plugin can't call `stop`).
- Failures inside a plugin should be caught at the Plugin Host boundary and reported as `PluginError` (§6) — a broken plugin must not be able to crash the daemon or block a service from starting.

This mirrors how the CLI/TUI/API are "views" on the Core (§5): plugins are best thought of as a fourth kind of client, just one that lives inside the daemon process and reacts to events instead of issuing commands from outside.

---

## 10. How the CLI / TUI / API Interact With Core

All three are **thin clients over the same IPC protocol** described in §3 — this is the load-bearing design constraint that keeps the whole system consistent:

|Interface|Nature|Talks to daemon via|
|---|---|---|
|**CLI**|One-shot command, prints, exits|IPC request/response|
|**TUI**|Long-lived interactive session|IPC request/response for actions + subscribes to the event stream for live updates (status table, logs)|
|**API**|Long-lived HTTP server, potentially remote/multi-client|Same IPC socket internally; translates HTTP verbs to daemon calls, and can expose Server-Sent Events/WebSocket for the same event stream the TUI consumes|

None of the three:

- decide restart policy,
- resolve dependency order,
- touch SQLite directly, or
- spawn/kill OS processes directly.

They only format input into a command and format the daemon's response for their medium (rich terminal tables, a Textual widget tree, or JSON). This is what guarantees that `procpm start`, the TUI's start action, and an API-triggered start all behave identically — because they're literally the same daemon call underneath.

---

## Summary Diagram

```
                     ┌─────────────┐
                     │ procpm.toml │
                     └──────┬──────┘
                            │ parsed by
                            ▼
                    ┌───────────────┐
                    │ Config Loader │
                    └───────┬───────┘
                            │
        ┌───────────────────────────────────────┐
        │              procpmd (daemon)          │
        │                                         │
        │  Dependency Resolver → Supervisor       │
        │        │                  │             │
        │        ▼                  ▼             │
        │     Monitor          Log Manager        │
        │        │                  │             │
        │        └──────┬───────────┘             │
        │               ▼                         │
        │         internal event bus ── Plugin Host│
        │               │                         │
        │               ▼                         │
        │          State Store (SQLite)           │
        └───────────────┬─────────────────────────┘
                         │ local IPC socket
          ┌──────────────┼──────────────┐
          ▼              ▼              ▼
        CLI            TUI          API Server
```