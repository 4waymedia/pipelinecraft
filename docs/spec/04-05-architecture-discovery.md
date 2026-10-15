# PipelineCraft (working name): §4 Architecture · §5 Discovery and app lifecycle

_2026-10-15 · spec draft v0 · status: draft (agent A3)_

- **Scope:** the hub's components and what each one owns (§4.1), where the code lives (§4.2), the crate list with `cargo xtask layers` rows (§4.3), the adapter model with every app's quirks as config (§4.4), a diagram (§4.5), and how the hub finds, attaches to, launches, health-checks and cleans up app instances (§5).
- **Project goal it serves:** a suite of local-first creative tools that hand work to each other (AssetCraft → StoryCraft → FilmCraft, later PrepressCraft) with no cloud. This section is the skeleton the protocol (§6), the asset refs (§7), events and jobs (§8) and the MCP front door (§10) hang on.
- **Immediate area of improvement:** today nothing can answer "which Craft apps are running, and how do I reach each one?" (01 §1.1 "No discovery"). This section gives a v1 answer that works with the apps **exactly as they ship today**, and a small manifest file to propose upstream so the answer gets simpler later.

**Evidence basis.** Static read only; nothing was built or run. Facts about sibling apps come from the wave-1 inventory and are cited by its IDs: `03a` = `03a-inventory.md` (O1–O14), `03b` = `03b-inventory.md` (O-B1–O-B13), `01` = `01-goals-glossary.md`. Where this section needed a detail those docs lack, it re-read the source and cites `repo` `path:line` @ commit. Commits: `assetcraft` `3a41b2b`, `photocraft` `fd28da5`, `lightcraft` `38adc17`, `filmcraft` `9ed9650`, `pdfcraft` `ac89a11`, `wordcraft` `3292ba0`, `designcraft` `7ed5697`, `effectcraft` `2c08f67`, `vectorcraft` `0946bc5`, `soundcraft` `3a28a65`. Terms follow the 01 §2 glossary exactly ("connection", not "session", for hub clients; "command effect").

**New facts found in this pass (not in wave 1):**

| ID     | Fact                                                                                                                                                                                                 | Source                                                                                                                                                                                                                                                                                                         |
|--------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| A3-F1  | O7 is not PhotoCraft-only. LightCraft, FilmCraft, DesignCraft and VectorCraft also log the **requested** port, not `local_addr()`, so `--control 0` loses the port in five apps. WordCraft refuses 0 (`NonZeroU16`), EffectCraft requires 1–65535. Only SoundCraft (stdout line) and PdfCraft (file) report an OS-chosen port | `lightcraft` `apps/lightcraft/src/control_server.rs:35`; `filmcraft` `apps/filmcraft/src/control_server.rs:34`; `designcraft` `apps/designcraft/src/control_server.rs:35`; `vectorcraft` `apps/vectorcraft/src/control_server.rs:69`; `photocraft` `apps/photocraft/src/control_server.rs:27`; `wordcraft` `apps/wordcraft/src/main.rs:334`; `soundcraft` `apps/soundcraft/src/control_server.rs:35-37` |
| A3-F2  | A failed bind does **not** stop most apps: the control server logs the error and returns, and the window opens with no control channel. A port the hub picked that someone else already holds therefore leads the hub to a **different** listener | `soundcraft` `apps/soundcraft/src/control_server.rs:28-33`; `photocraft` `control_server.rs:20-24`; `filmcraft` `control_server.rs:30`; `designcraft` `control_server.rs:31`; `vectorcraft` `control_server.rs:65` (PdfCraft is the exception: it refuses to start, `pdfcraft` `apps/pdfcraft/src/main.rs:432-449`) |
| A3-F3  | Every GUI app is built with `windows_subsystem = "windows"` in release, so stdout/stderr lines (SoundCraft's port line, PhotoCraft's printed token, every "failed to bind" line) are only visible if the launcher passes pipes | e.g. `soundcraft` `apps/soundcraft/src/main.rs:11`; `photocraft` `apps/photocraft/src/main.rs:20` (same attribute in all ten GUI `main.rs`, grep)                                                                                                                                                              |
| A3-F4  | PdfCraft never removes its control file on exit, so a stale `{port,token,pid}` survives a crash or a normal quit                                                                                     | `pdfcraft` `apps/pdfcraft/src/main.rs:373-375`, `:432-499` (only the staged temp file is removed, `:494`)                                                                                                                                                                                                       |
| A3-F5  | PdfCraft skips its Windows single-instance hand-off whenever `--control` (or `--new-window`) is given, so a hub launch always gets its own window                                                   | `pdfcraft` `apps/pdfcraft/src/main.rs:219`                                                                                                                                                                                                                                                                     |
| A3-F6  | Every app and every CLI answers `--version`; only EffectCraft reports its version over the control channel (`app.capabilities`)                                                                     | e.g. `lightcraft` `apps/lightcraft-cli/src/main.rs:168`, `effectcraft` `crates/engine/src/commands/query.rs:212` (grep of all nine)                                                                                                                                                                             |
| A3-F7  | VectorCraft's `mcp` connects to `127.0.0.1:7979` by default and only falls back to headless; a hub that wants headless must pass `--headless`                                                       | `vectorcraft` `apps/vectorcraft-cli/src/main.rs:4`, `:72-73`, `:169`                                                                                                                                                                                                                                          |

---

## 4. Architecture

### 4.1 Components and responsibilities

The hub is one on-demand daemon per user login (D5) plus thin clients. The daemon owns all app connections and all secrets; clients never talk to apps directly.

| Component           | Owns                                                                                                                                                                                                                                                                                                                                                              | Does not own                                                     | Crate (§4.3)            |
|---------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------|-------------------------|
| **Hub daemon**      | Process lifetime, single-instance lock, its own listener for connections (loopback, suite-jsonl framing per D3), its own manifest `hub.json` so clients can find it (§5.2), config load (built-in defaults + user overrides), the thread pool                                                                                                                       | Any app's commands; the wire contract (A4, §6)                   | `pipelinecraft` (app)   |
| **Discovery**       | Producing **candidates**: "app X, maybe instance pid P, maybe at port N, with auth hint H, from source S". Sources: config list of installed apps, suite run-dir manifests, app-native artefacts (PdfCraft control file, WordCraft key file), process probe (§5.1). Pure and testable with fakes; it never opens a socket                                          | Connecting, launching, deciding identity                         | `craft-hub-discovery`   |
| **Lifecycle**       | Turning candidates into **instances**: attach (handshake + identity check), launch (GUI with `--control`, or headless CLI/MCP), health, crash and stale cleanup, multiple instances, version gating (§5.3–5.6)                                                                                                                                                      | Transport bytes                                                  | `craft-hub-core`        |
| **Adapters**        | One per transport kind (§4.4): speak it, apply the per-app quirks from config, and return the hub's normalized types (`CommandInfo` per 03a O9, call outcome, job snapshot). Bounded reads, timeouts, no panics on hostile replies                                                                                                                                 | Routing, policy                                                  | `craft-hub-adapters`    |
| **Registry merge**  | The merged catalog `<app>/<command.id>` → `CommandInfo` per instance, refreshed on attach and on version change; command effects seeded from what apps publish and from the curated table (03a O4, 03b O-B5)                                                                                                                                                   | Defining commands (NG1)                                          | `craft-hub-core`        |
| **Router**          | Resolving an address (`<app>[@instance]/<command.id>` or a capability alias such as `asset.open`, 03b O-B8) to one instance and one adapter; choosing the transport (control vs headless, e.g. PdfCraft writes go headless, 03a O6); per-instance concurrency (at most 2 hub connections per app instance, apps allow 16, 03a §3a.2); calling the policy hook (A8) | Grammar of addresses (A4 §6.2), permission rules (A8 §11.4)      | `craft-hub-core`        |
| **Event bus**       | Topics, subscriptions, fan-out with bounded per-subscriber queues. **No app pushes events today** (01 §1.1; 03a O8; 03b O-B13), so most events come from **polling bridges**: per-app pollers that call a list method, diff it against the last snapshot and emit `job.*` events; discovery emits `app.up` / `app.down`. The one real push source is MCP `notifications/progress` from headless children (PhotoCraft renderVideo, LightCraft export, FilmCraft exportMedia, EffectCraft renderQueue) | Event payload schemas (A6 §8)                                    | `craft-hub-core`        |
| **Job manager**     | Hub job ids; wrapping each kind of long work as one hub job: an app job (`wait:false` → `{job, pending}` in PhotoCraft), a blocking call on a worker thread, a CLI child process, or an MCP call with progress; cancel mapping (`jobs.cancel`, `activity.cancel`, MCP `notifications/cancelled`, kill a child the hub owns); timeouts                         | Job semantics and states (A6 §8.3)                               | `craft-hub-core`        |
| **MCP front door**  | A thin **stdio** process (`pipelinecraft mcp`) that an MCP client spawns. It is a hub *client*: it connects to the daemon (starting it if needed), turns the merged registry into tools, and forwards calls. Streamable HTTP on loopback later, inside the daemon                                                                                                 | Tool naming and confirmation rules (A7 §10)                      | `craft-hub-mcp`         |
| **`craftctl`**      | The human/script client: `apps`, `commands`, `call`, `watch`, `run`. Starts the daemon on demand                                                                                                                                                                                                                                                                  | Subcommand design (A7 §12)                                       | `craftctl` (app)        |
| **Status panel** (optional, L6) | An egui view of instances, health, jobs and the event tail. Built as a library panel so it can run standalone (`pipelinecraft panel`) or dock inside AssetCraft. It is a client like `craftctl`; it only draws (`AGENTS.md` non-negotiable 3)                                                                                                    | Anything stateful                                                | `craft-hub-panel`       |

**Polling bridges in v1** (sources from 03a §3a.8 and 03b §3.7b; the intervals are draft numbers for P4 in 01 §1.4):

| App         | Poll call                                                      | Kind              | Emits                                   | Interval (draft)                            |
|-------------|----------------------------------------------------------------|-------------------|-----------------------------------------|---------------------------------------------|
| PhotoCraft  | control method `jobs.list`                                     | method            | `job.progress`, `job.done`, `job.failed` | 250 ms while a hub job is open there, else off |
| LightCraft  | `engine.execute activity.list`                                 | command           | same                                    | same                                        |
| FilmCraft   | `engine.execute jobs.list`; `export.queue.list`                | command           | same, plus `export.queue.changed`       | same                                        |
| EffectCraft | `engine.execute jobs.list`                                     | command           | same                                    | same                                        |
| SoundCraft  | `engine.execute transport.status`                              | command           | `transport.changed`                     | 1 s, only with a subscriber                 |
| PdfCraft    | control `ui.state` → `pending_opens`                           | method            | `open.done`                             | 250 ms after a `ui.open` until it clears    |
| All         | discovery pass (pid alive, §5.4)                                | hub-internal      | `app.up`, `app.down`                    | 1 s pid check, 5 s full pass                |
| WordCraft, DesignCraft, VectorCraft | none (no job list)                       | n/a               | the hub job ends when the call returns  | n/a                                         |

Pollers run only while someone needs them (an open hub job on that instance, or a subscriber to that topic), so an idle hub sends nothing to apps. Selection and asset-changed events have no cheap poll source in any app today; they are A6's call, and the opportunity is 03b O-B13 (`events.subscribe` upstream).

### 4.2 Deployment shape

Three options, judged against one fact from wave 1: the code the hub needs first (a guarded NDJSON control client, envelope types, a manifest writer) is the **same** code AssetCraft needs for its own Stage 1.5 `--control` and for "Open In v2" into a running app (03b O-B1, O-B9, O-B10; `assetcraft` `docs/sibling-apis/README.md:91` @ `3a41b2b`: "A shared control client (O3) belongs in AssetCraft's engine as `crates/craft-control` until craft-libs has one").

| Option                                        | For                                                                                                                                       | Against                                                                                                                                                                |
|-----------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| A. Separate `pipelinecraft` repo now          | Clean boundary; own release cadence; the name change (D1) is one repo rename                                                              | `craft-control` would be needed in two repos at once, so it is either copied (the very problem O-B1 names: the transport is already copied ×3) or published early before it is proven; two CIs for one small team |
| B. Crate family in AssetCraft, permanently     | One workspace, one `cargo xtask ci`, layers enforced today                                                                                 | The hub becomes "part of AssetCraft"; StoryCraft and others would depend on AssetCraft's repo; AssetCraft CI grows forever                                             |
| **C. Incubate in AssetCraft, promote later**  | Starts small (Paul's preference); `craft-control-*` is written once and used by both AssetCraft and the hub from day one; layers checked by the existing xtask | Needs a rule that hub crates never depend on `assetcraft-*` crates, so promotion is a move, not a rewrite (A3-O2)                                                     |

**Recommendation A3-R1 (feeds D2): option C, in two promotions.**

1. **Now (AssetCraft Stage 1.5 / hub Stage 0):** add the `craft-control-*` and `craft-hub-*` crates and the `pipelinecraft` and `craftctl` apps to the AssetCraft workspace (§4.3). AssetCraft's own `--control` server and its planned L4 `bridge` use `craft-control-*`; the hub uses the same crates.
2. **Promotion 1, `craft-control-*` → craft-libs:** when one sibling fork (DesignCraft or EffectCraft, whose `control_server.rs` files are near-identical, 03b §3.6b) builds against it in a PR. That is the D6 "one small PR" test.
3. **Promotion 2, `craft-hub-*` + binaries → their own repo** (named after D1): at hub Stage 2 (Hand off), or earlier if a second client (StoryCraft) needs the hub. The crate names have no `pipelinecraft-` prefix, so D1 never renames a crate.

Owner: Paul (decides D2); Claude (implements). The outline's D2 recommendation ("workspace crate + `craftctl`, promote at Stage 2") agrees; this adds the separate, earlier promotion of `craft-control-*`.

### 4.3 Crate list and layer-table rows

AssetCraft's layer rule lives in `assetcraft` `xtask/src/layers.rs:19-33` @ `3a41b2b` (`TABLE: &[(&str, Class)]`, names without the `assetcraft-` prefix, `short_name` at `:73` leaves other names unchanged). UI crates (`craft-ui`, `egui*`, `eframe`, `winit`, `rfd`) are allowed only from `UI_MIN_LAYER = 6` (`:36`, `:39`). Every workspace member must have a row or `cargo xtask layers` fails (`:95`). Current rows: `fs` L0; `probe`, `watch`, `mesh` L1; `thumbs` L2; L3/L4 planned (`boards`, `interchange`, `bridge`); `engine` L5; `ui-egui`, `mcp` L6; apps exempt.

| Crate                  | Layer  | Depends on (workspace)                                     | External deps (all permissive; Paul to confirm)                                    | Responsibility                                                                                                     |
|------------------------|--------|------------------------------------------------------------|-------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------|
| `craft-control-proto`  | L0     | none                                                       | `serde`, `serde_json`                                                               | Envelope types (suite-jsonl and JSON-RPC 2.0), `CommandInfo` with `effects`, error codes, **`Manifest`**, **`AppDescriptor`** (config schema), bounded line reader |
| `craft-control-client` | L1     | `craft-control-proto`                                      | none beyond std                                                                     | Loopback TCP client, the two codecs (03a O1), the auth strategies (03a O2), deadlines; reused by AssetCraft `bridge` |
| `craft-control-server` | L1     | `craft-control-proto`                                      | none beyond std                                                                     | Guarded NDJSON server (VectorCraft's variant: line guard, deadlines, withdrawal, 03b O-B1) with an optional token handshake and a manifest writer. Used by AssetCraft's `--control` and by the daemon's own listener |
| `craft-hub-discovery`  | L2     | `craft-control-proto`                                      | `sysinfo` (process list, MIT), `toml` (MIT/Apache)                                  | Candidate sources (§5.1), installed-app locator lifted from AssetCraft `open_in.rs` (`AppLocator`, `candidates`, `crates/engine/src/open_in.rs:166-174`), config load |
| `craft-hub-adapters`   | L2     | `craft-control-proto`, `craft-control-client`             | `semver` (MIT/Apache)                                                               | The four adapter kinds (§4.4), quirk application, version-ranged overrides                                        |
| `craft-hub-core`       | L3     | proto, client, server, discovery, adapters                 | none new                                                                            | Lifecycle, registry merge, router, event bus, pollers, job manager. Std threads and channels, like AssetCraft's pools |
| `craft-hub-mcp`        | L2     | `craft-control-proto`, `craft-control-client`             | none new (hand-written JSON-RPC 2.0, as LightCraft, PdfCraft and WordCraft do)     | The MCP front door shim: a hub client that serves MCP on stdio                                                     |
| `craft-hub-panel`      | L6     | `craft-control-proto`, `craft-control-client`             | `egui`, `craft-ui`                                                                  | Optional status panel                                                                                              |
| `pipelinecraft`        | Exempt | core, mcp, panel                                           |                                                                                     | Daemon binary: `pipelinecraft serve`, `pipelinecraft mcp`, `pipelinecraft panel`                                  |
| `craftctl`             | Exempt | proto, client                                              |                                                                                     | CLI client                                                                                                         |

**Exact rows to add to `TABLE`** (in layer order, next to the existing rows):

```rust
pub const TABLE: &[(&str, Class)] = &[
    ("fs", Class::Layer(0)),
    ("craft-control-proto", Class::Layer(0)),
    ("probe", Class::Layer(1)),
    ("watch", Class::Layer(1)),
    ("mesh", Class::Layer(1)),
    ("craft-control-client", Class::Layer(1)),
    ("craft-control-server", Class::Layer(1)),
    ("thumbs", Class::Layer(2)),
    ("craft-hub-discovery", Class::Layer(2)),
    ("craft-hub-adapters", Class::Layer(2)),
    ("craft-hub-mcp", Class::Layer(2)),
    // L3 (planned): boards, interchange. L4 (planned): bridge.
    ("craft-hub-core", Class::Layer(3)),
    ("engine", Class::Layer(5)),
    ("ui-egui", Class::Layer(6)),
    ("mcp", Class::Layer(6)),
    ("craft-hub-panel", Class::Layer(6)),
    ("assetcraft", Class::Exempt),
    ("cli", Class::Exempt),
    ("web", Class::Exempt),
    ("xtask", Class::Exempt),
    ("pipelinecraft", Class::Exempt),
    ("craftctl", Class::Exempt),
];
```

Notes on the rows:
- AssetCraft's planned `bridge` (L4) depends on `craft-control-client` (L1) and, once the hub exists, talks to the hub through it; with no hub running it falls back to talking to an app directly (today's Open In behaviour, `assetcraft` `STATUS.md` 8a). `engine` (L5) can use both.
- Same-layer crates never depend on each other (`craft-hub-discovery` and `craft-hub-adapters` are siblings; `craft-hub-core` above them joins them), which the existing rule enforces (`layers.rs:102-106`).
- **Island rule (A3-O2).** The layer table can't stop `craft-hub-core` (L3) from depending on `thumbs` (L2), which would block promotion. Proposed addition next to `TABLE`: `pub const ISLANDS: &[(&str, &[&str])] = &[("craft-", &["craft-"])];` meaning a workspace crate whose name starts with `craft-` may depend only on workspace crates starting with `craft-`. Small change to `check()`; owner Claude.
- `default-members = ["crates/*", "apps/*"]` (`assetcraft` `Cargo.toml:4`) would build the hub binaries on every AssetCraft build. Fine at first; list them explicitly if build time hurts (owner Claude, log only).

### 4.4 Adapter model

**Four adapter kinds, one per transport.** Every real control surface in 03a/03b maps onto one of them; the variance inside a kind is config.

| Adapter kind         | Transport                                                                   | Variants (config, not code)                                                                                                                                                                                                                                                         | Apps                                                                                     | Used for                                                                           |
|----------------------|-----------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------|
| `control-jsonl`      | Loopback TCP, suite-jsonl `{id,method,params}` → `{id,ok,result\|error}`     | **auth:** `none` (LightCraft, FilmCraft, DesignCraft, EffectCraft, VectorCraft, SoundCraft); `token-frame`, first frame `auth {token}` (PhotoCraft, 03a §3a.2); `key-field`, top-level `key` on every request or `auth` once (WordCraft, 03a §3a.6). **port:** explicit, or `0` reported on stdout (SoundCraft). **limits:** request line 1 MiB / 4 MiB / ~86 MiB; reply timeout 60 s / 90 s. **junk policy:** closes vs stays open (SoundCraft, 03b O-B2) | PhotoCraft, LightCraft, FilmCraft, WordCraft, DesignCraft, EffectCraft, VectorCraft, SoundCraft; AssetCraft (planned) | Driving a running GUI window; the only way into an app that is already open      |
| `control-jsonrpc`    | Loopback TCP, newline JSON-RPC 2.0, discovered from a **control file**       | Control file path chosen by the hub at launch; `auth` first with the file's token; method map `ui.commands` (list), `ui.command {id}` (**no params**), `ui.open {path}`, `ui.state`                                                                                                  | PdfCraft                                                                                 | Open In, state; anything with params is routed to `mcp-stdio` (03a O6)            |
| `cli-headless`       | Spawn `<app>-cli` per call or per script, read stdout, bounded, killed on timeout | argv template per app (`exec <id> <json>`, `run <script.jsonl>`, `run --cmd …`, `script FILE`); catalog argv (`commands --json` vs `commands` vs `tools`, 03b O-B3); output parser (JSON array, JSON lines, text table)                                                                | All nine + AssetCraft                                                                    | Catalog fetch with no app running, `--version`, one-shot batch work, thumbnails (AssetCraft's `filmcraft-cli --compact run` poster driver is the precedent, `STATUS.md` 7a) |
| `mcp-stdio`          | Spawn `<app>-cli mcp …` once, keep it, speak MCP JSON-RPC as a client        | Tool style: `suite-28` (`command_list`/`command_run`/`command_batch`; PhotoCraft, LightCraft, FilmCraft, EffectCraft, VectorCraft), `generic-execute` (`list_commands`/`execute`; WordCraft, DesignCraft, SoundCraft), `pdfcraft-tools` (typed tools, `command_run` only via a linked tool). Headless flags and roots per app | All nine (headless)                                                                      | Long headless jobs with **pushed** progress and cancel; PdfCraft writes            |

Not modelled as adapters: each app's `mcp --bridge/--connect` mode (the hub speaks the control channel itself; 03a O11), the `ui.*` UI-automation methods (callable as ordinary methods, NG6), LightCraft's `denoise-serve` LAN token (unrelated, 03a §3a.3), and ArtCraft, which has no local control surface (03b O-B11).

**Per-app adapter config.** Shipped as a built-in default file compiled into the hub (`include_str!`), with a user override file merged on top (§5.2 for the path). Quirks are rows here, never `if app == …` in code. Version-ranged `[[…override]]` tables let a quirk switch off when upstream fixes it. Values are from 03a/03b at the commits above; anything marked `# verify` is a static-read guess.

```toml
# craft-hub-adapters: apps.default.toml (schema 1). Paths use %VAR% on Windows.
schema = 1

[defaults]
launch            = "on-demand"     # never | on-demand
port_range        = [47600, 47699]  # hub-assigned control ports; avoids the 7979 default (03b O-B7)
attach_timeout_ms = 500             # P3
launch_timeout_ms = 10000
max_hub_connections = 2             # per instance; apps allow 16
max_reply_line    = 67108864        # hub-side read guard for replies (64 MiB)
pid_check_ms      = 1000
discovery_pass_ms = 5000

# ---------------------------------------------------------------- PhotoCraft
[apps.photocraft]
name           = "PhotoCraft"
exe            = "photocraft.exe"
cli            = "photocraft-cli.exe"
install_dirs   = ['%ProgramFiles%\PhotoCraft', '%LOCALAPPDATA%\Programs\PhotoCraft']
tested         = ">=0.9.0, <0.10.0"
identity_probe = ["image.adjustments.curves"]          # a command id only this app has
[apps.photocraft.control]
adapter          = "control-jsonl"
launch_args      = ["--control", "{port}", "--control-token-file", "{token_file}",
                    "--automation-read-root", "{read_root}"]
port_env         = "PHOTOCRAFT_CONTROL_PORT"
port_zero        = "lost"            # O7: prints 127.0.0.1:0; never pass 0
bad_port         = "exit"            # exit code 2
bind_failure     = "continues"       # A3-F2
max_request_line = 1048576
reply_timeout_ms = 60000
closes_on_junk   = true              # auth-first: anything else closes
auth             = { kind = "token-frame", token_hex_len = 64, source = "token_file" }
execute          = { method = "engine.execute", command_key = "command", wait = true }
commands         = { method = "engine.commands" }
health           = { method = "jobs.list" }          # verify: cheap
poll.jobs        = { method = "jobs.list", cancel = "jobs.cancel" }
[apps.photocraft.catalog]
source       = "cli"
args         = ["commands", "--json"]
disabled_key = "disabledReason"
[apps.photocraft.headless]
adapter = "mcp-stdio"
args    = ["mcp"]
tools   = "suite-28"
[apps.photocraft.paths]
mode = "relative-to-read-root"       # O13: app.open takes a path under --automation-read-root

# ---------------------------------------------------------------- LightCraft
[apps.lightcraft]
name           = "LightCraft"
exe            = "lightcraft.exe"
cli            = "lightcraft-cli.exe"
install_dirs   = ['%ProgramFiles%\LightCraft', '%LOCALAPPDATA%\Programs\LightCraft']
tested         = ">=0.8.0, <0.9.0"
identity_probe = ["develop.set"]
single_instance = "per-library-lock"  # <library>/catalog.lock: prefer attach over a second launch
[apps.lightcraft.control]
adapter          = "control-jsonl"
launch_args      = ["--control", "{port}"]
port_env         = "LIGHTCRAFT_CONTROL_PORT"
port_zero        = "lost"            # A3-F1
bad_port         = "ignored"         # logged, no server, exit 0
bind_failure     = "continues"
max_request_line = 4194304
reply_timeout_ms = 60000
first_frame_ms   = 5000              # gives up on a silent connection: connect lazily
closes_on_junk   = true
auth             = { kind = "none" }
execute          = { method = "engine.execute", command_key = "command" }
commands         = { method = "engine.commands" }
poll.jobs        = { command = "activity.list", cancel_command = "activity.cancel" }
[apps.lightcraft.catalog]
source       = "cli"
args         = ["commands", "--json"]   # demo session: `enabled` reflects the demo, refresh live on attach
disabled_key = "disabled_reason"
synthetic    = ["app.export"]           # injected by the headless backend, not an engine command
[apps.lightcraft.headless]
adapter = "mcp-stdio"
args    = ["mcp", "--compact"]          # verify: --compact keeps command_run
tools   = "suite-28"

# ---------------------------------------------------------------- FilmCraft
[apps.filmcraft]
name           = "FilmCraft"
exe            = "filmcraft.exe"
cli            = "filmcraft-cli.exe"
install_dirs   = ['%ProgramFiles%\FilmCraft', '%LOCALAPPDATA%\Programs\FilmCraft']
tested         = ">=0.8.0, <0.9.0"
identity_probe = ["sequence.addEdit"]
file_args      = ["--", "{paths...}"]   # the only app that understands `--` (STATUS 8a)
[apps.filmcraft.control]
adapter          = "control-jsonl"
launch_args      = ["--control", "{port}"]
port_env         = "FILMCRAFT_CONTROL_PORT"
port_zero        = "lost"
bad_port         = "exit"
bind_failure     = "continues"
max_request_line = 4194304
reply_timeout_ms = 60000                # verify
closes_on_junk   = true
auth             = { kind = "none" }
execute          = { method = "engine.execute", command_key = "command" }
commands         = { method = "engine.commands" }
poll.jobs        = { command = "jobs.list", cancel_command = "jobs.cancel" }
poll.export_queue = { command = "export.queue.list" }
[apps.filmcraft.catalog]
source = "cli"
args   = ["commands", "--json"]          # no disabled-reason key
[apps.filmcraft.headless]
adapter = "cli-headless"
args    = ["--compact", "run", "{script_jsonl}"]   # lines {"id","params"}
alt     = { adapter = "mcp-stdio", args = ["mcp"], tools = "suite-28" }

# ---------------------------------------------------------------- PdfCraft
[apps.pdfcraft]
name           = "PdfCraft"
exe            = "pdfcraft.exe"
cli            = "pdfcraft-cli.exe"
install_dirs   = ['%ProgramFiles%\PdfCraft', '%LOCALAPPDATA%\Programs\PdfCraft']
tested         = ">=0.8.0, <0.9.0"
identity_probe = []                      # the control file carries the pid: pid + exe check is identity
single_instance = "windows-handoff"      # skipped when --control is given (A3-F5)
[apps.pdfcraft.control]
adapter          = "control-jsonrpc"
launch_args      = ["--control", "{control_file}"]   # a FILE, random port (A1-O1)
port_zero        = "file"
bind_failure     = "refuses-to-start"
max_request_line = 4096                  # before auth
reply_timeout_ms = 30000                 # -32002
control_file_stale = "never-removed"     # A3-F4: check pid before trusting it
auth             = { kind = "token-frame", token_hex_len = 32, source = "control_file" }
commands         = { method = "ui.commands" }
execute          = { method = "ui.command", id_key = "id", params = "unsupported" }  # O6
health           = { method = "ui.state" }
poll.opens       = { method = "ui.state", field = "pending_opens" }
[apps.pdfcraft.catalog]
source = "cli"
args   = ["tools"]                       # no `commands`; `destructive` missing (O10)
also   = { adapter = "mcp-stdio", tool = "command_list" }
[apps.pdfcraft.headless]
adapter = "mcp-stdio"
args    = ["mcp", "--root", "{read_root}"]
tools   = "pdfcraft-tools"
route_writes_here = true                 # every call with params goes headless (O6)

# ---------------------------------------------------------------- WordCraft
[apps.wordcraft]
name           = "WordCraft"
exe            = "wordcraft.exe"
cli            = "wordcraft-cli.exe"
install_dirs   = ['%ProgramFiles%\WordCraft', '%LOCALAPPDATA%\Programs\WordCraft']
tested         = ">=0.7.0, <0.8.0"
identity_probe = []                      # the key file for this port is the identity (below)
[apps.wordcraft.control]
adapter          = "control-jsonl"
launch_args      = ["--control", "{port}"]
port_env         = "WORDCRAFT_CONTROL_PORT"
port_zero        = "refused"             # NonZeroU16
bad_port         = "exit"
bind_failure     = "continues"           # verify
max_request_line = 1048576
reply_timeout_ms = 60000
first_auth_ms    = 30000
closes_on_junk   = true
unknown_method   = "runs-as-command"     # NEVER send a hub-invented method name (e.g. "ping")
auth             = { kind = "key-field", mode = "per-request", token_hex_len = 64,
                     source = "key_file", key_file = '%APPDATA%\WordCraft\control-key.local-{port}',
                     key_file_env_dir = "WORDCRAFT_PROFILE_DIR" }
execute          = { method = "engine.execute", command_key = "command" }
commands         = { method = "engine.commands" }     # has `enabled`, not `mutates`
health           = { method = "document.inspect" }    # verify: cheap
[apps.wordcraft.catalog]
source    = "cli"
args      = ["commands", "--json"]       # has `mutates`, not `enabled`: merge both (O9)
menu_key  = "location"
[apps.wordcraft.headless]
adapter = "mcp-stdio"
args    = ["mcp", "--automation-read-root", "{read_root}", "--automation-write-root", "{write_root}"]
tools   = "generic-execute"

# ---------------------------------------------------------------- DesignCraft
[apps.designcraft]
name           = "DesignCraft"
exe            = "designcraft.exe"
cli            = "designcraft-cli.exe"
install_dirs   = ['%ProgramFiles%\DesignCraft', '%LOCALAPPDATA%\Programs\DesignCraft']
tested         = ">=0.8.0, <0.9.0"
identity_probe = ["file.exportIdml"]
[apps.designcraft.control]
adapter          = "control-jsonl"
launch_args      = ["--control", "{port}"]
port_env         = "DESIGNCRAFT_CONTROL_PORT"
port_zero        = "lost"
bad_port         = "ignored"
bind_failure     = "continues"
max_request_line = 4194304
reply_timeout_ms = 60000
closes_on_junk   = true
auth             = { kind = "none" }
execute          = { method = "engine.execute", command_key = "command" }
commands         = { method = "engine.commands", ui_flag = "ui" }   # UI ids marked "ui": true
health           = { method = "document.inspect" }                  # verify
[apps.designcraft.catalog]
source       = "cli"
args         = ["commands"]              # O-B3: `commands --json` reads --json as FILTER, prints []
disabled_key = "disabled_reason"
[[apps.designcraft.override]]            # example of a version-ranged fix (hypothetical version)
versions = ">=0.9.0"
catalog.args = ["commands", "--json"]
[apps.designcraft.headless]
adapter = "mcp-stdio"
args    = ["mcp"]
tools   = "generic-execute"

# ---------------------------------------------------------------- EffectCraft
[apps.effectcraft]
name           = "EffectCraft"
exe            = "effectcraft.exe"
cli            = "effectcraft-cli.exe"
install_dirs   = ['%ProgramFiles%\EffectCraft', '%LOCALAPPDATA%\Programs\EffectCraft']
tested         = ">=0.10.0, <0.11.0"
identity_probe = ["renderQueue.add"]
launch_side_effects = ["no-focus", "crash-recovery-skipped"]   # with --control (03b)
[apps.effectcraft.control]
adapter          = "control-jsonl"
launch_args      = ["--control", "{port}"]
port_env         = "EFFECTCRAFT_CONTROL_PORT"
port_zero        = "refused"             # 1-65535
bad_port         = "exit"                # exit 1
bind_failure     = "continues"           # verify
max_request_line = 4194304
reply_timeout_ms = 60000
closes_on_junk   = true
auth             = { kind = "none" }
execute          = { method = "engine.execute", command_key = "command" }
commands         = { method = "engine.commands" }
version          = { command = "app.capabilities", field = "version" }   # A3-F6
poll.jobs        = { command = "jobs.list" }
[apps.effectcraft.catalog]
source       = "cli"
args         = ["--json", "commands"]
schemas_args = ["exec", "--list", "--schemas"]
disabled_key = "why"
[apps.effectcraft.headless]
adapter = "mcp-stdio"
args    = ["mcp"]
tools   = "suite-28"

# ---------------------------------------------------------------- VectorCraft
[apps.vectorcraft]
name           = "VectorCraft"
exe            = "vectorcraft.exe"
cli            = "vectorcraft-cli.exe"
install_dirs   = ['%ProgramFiles%\VectorCraft', '%LOCALAPPDATA%\Programs\VectorCraft']
tested         = ">=0.11.0, <0.12.0"
identity_probe = ["document.open"]
[apps.vectorcraft.control]
adapter          = "control-jsonl"
launch_args      = ["--control", "{port}", "--automation-read-root", "{read_root}",
                    "--automation-write-root", "{write_root}"]
port_env         = "VECTORCRAFT_CONTROL_PORT"
port_zero        = "lost"
bind_failure     = "continues"
max_request_line = 4194304
reply_timeout_ms = 60000
idle_close_ms    = 300000                # 5 min idle
not_started_on_timeout = "withdrawn"     # a timed-out request that hadn't started will not run: retry-safe
closes_on_junk   = true
auth             = { kind = "none" }
execute          = { method = "engine.execute", command_key = "command" }
commands         = { method = "engine.commands", ui_flag = "ui" }
[apps.vectorcraft.catalog]
source       = "cli"
args         = ["commands"]              # extra args ignored
disabled_key = "disabled_reason"
[apps.vectorcraft.headless]
adapter = "mcp-stdio"
args    = ["mcp", "--headless"]          # A3-F7: default connects to 127.0.0.1:7979
tools   = "suite-28"

# ---------------------------------------------------------------- SoundCraft
[apps.soundcraft]
name           = "SoundCraft"
exe            = "soundcraft.exe"
cli            = "soundcraft-cli.exe"
install_dirs   = ['%ProgramFiles%\SoundCraft', '%LOCALAPPDATA%\Programs\SoundCraft']
tested         = ">=0.7.0, <0.8.0"
identity_probe = ["file.bounce_mix"]
id_style       = "snake_case"
[apps.soundcraft.control]
adapter          = "control-jsonl"
launch_args      = ["--control", "0"]
port_zero        = "reported-stdout"     # SOUNDCRAFT_CONTROL_PORT=<n>; needs a stdout pipe (A3-F3)
port_env         = "SOUNDCRAFT_CONTROL_PORT"
bind_failure     = "continues"
max_request_line = 90000000              # about 86 MiB for plugin state; verify exact
reply_timeout_ms = 90000
closes_on_junk   = false                 # O-B2: the hub must only ever send well-formed lines
auth             = { kind = "none" }
execute          = { method = "engine.execute", command_key = "command" }
commands         = { method = "engine.commands" }
poll.transport   = { command = "transport.status" }
[apps.soundcraft.catalog]
source = "mcp"                           # O-B3: `commands --json` prints nothing; text table otherwise
tool   = "list_commands"
disabled_key = "disabled_reason"         # always present, null when enabled
[apps.soundcraft.headless]
adapter = "mcp-stdio"
args    = ["mcp"]
tools   = "generic-execute"

# ---------------------------------------------------------------- AssetCraft (first native citizen)
[apps.assetcraft]
name           = "AssetCraft"
exe            = "assetcraft.exe"
cli            = "assetcraft-cli.exe"
install_dirs   = ['%ProgramFiles%\AssetCraft', '%LOCALAPPDATA%\Programs\AssetCraft']
identity_probe = ["file.openIn"]
[apps.assetcraft.control]                # planned, Stage 1.5 (03b O-B9): writes its own manifest
adapter  = "control-jsonl"
discover = "manifest"                    # no launch args needed once it publishes a manifest
auth     = { kind = "token-frame", token_hex_len = 64, source = "manifest" }
execute  = { method = "engine.execute", command_key = "command" }
commands = { method = "engine.commands" }
[apps.assetcraft.catalog]
source = "cli"
args   = ["commands", "--json"]          # today: app.commands shape {id,label,menu,shortcut,params}
[apps.assetcraft.headless]
adapter = "cli-headless"
args    = ["exec", "{id}", "{params_json}"]

# ---------------------------------------------------------------- not a hub app
[apps.artcraft]
enabled = false
reason  = "no local control surface; Storyteller web API is out of scope (NG2, 03b O-B11)"
```

**Placeholders** (`{port}`, `{token_file}`, `{control_file}`, `{read_root}`, `{write_root}`, `{script_jsonl}`, `{paths...}`) are filled by the lifecycle code, and each expands to exactly **one argv item** (no shell, like AssetCraft's `Launcher::spawn`, `crates/engine/src/launch.rs:15`). `{read_root}`/`{write_root}` default to the roots A5 defines for asset handoff (§7.4); until then, the folder of the asset being handed off.

### 4.5 Diagram

```
  CLIENTS (connections, one auth each, §11)
  ┌───────────────┐  ┌───────────────┐  ┌──────────────────────┐  ┌─────────────────┐
  │ AssetCraft    │  │ craftctl      │  │ MCP client           │  │ status panel    │
  │ (L4 bridge)   │  │ (PowerShell)  │  │ Claude / llama.cpp   │  │ (egui, optional)│
  └──────┬────────┘  └──────┬────────┘  └──────────┬───────────┘  └───────┬─────────┘
         │                  │              stdio   │                      │
         │                  │          ┌───────────┴──────────┐           │
         │                  │          │ pipelinecraft mcp    │           │
         │                  │          │ (craft-hub-mcp shim) │           │
         │                  │          └───────────┬──────────┘           │
         └────── loopback suite-jsonl (D3), found via hub.json ───────────┘
                                       │
  ┌────────────────────────────────────┴───────────────────────────────────────────┐
  │ HUB DAEMON  pipelinecraft serve   (one per login, on demand, 127.0.0.1 only)    │
  │                                                                                 │
  │  ┌──────────────┐  candidates  ┌─────────────┐   ┌────────────────────────────┐ │
  │  │ discovery    │─────────────▶│ lifecycle   │──▶│ instances: app@pid, state, │ │
  │  │ config list  │              │ attach /    │   │ version, adapter, catalog  │ │
  │  │ run-dir mf.  │              │ launch /    │   └─────────────┬──────────────┘ │
  │  │ app artefacts│              │ health /    │                 │                │
  │  │ process probe│              │ cleanup     │                 ▼                │
  │  └──────────────┘              └──────┬──────┘   ┌────────────────────────────┐ │
  │                                       │ app.up/  │ router                     │ │
  │  ┌──────────────┐   events            │ down     │ <app>[@inst]/<cmd>, alias, │ │
  │  │ event bus    │◀────────────────────┘          │ transport choice, policy   │ │
  │  │ topics, fan- │◀──── job.* ───┐                └─────────────┬──────────────┘ │
  │  │ out, bounded │               │                              │                │
  │  └──────────────┘        ┌──────┴───────┐                      │                │
  │         ▲                │ job manager  │◀─────────────────────┤                │
  │         │ diffs          └──────────────┘                      │                │
  │  ┌──────┴───────┐                                              │                │
  │  │ pollers      │ jobs.list / activity.list / export.queue.list / ui.state      │
  │  └──────┬───────┘                                              │                │
  │         │        ┌─────────────────────── adapters ────────────┴─────────────┐  │
  │         └───────▶│ control-jsonl │ control-jsonrpc │ cli-headless │ mcp-stdio │  │
  │                  └──────┬────────┴────────┬────────┴──────┬───────┴─────┬─────┘  │
  └─────────────────────────┼─────────────────┼───────────────┼─────────────┼────────┘
          TCP 127.0.0.1     │       TCP + control file        │ spawn       │ spawn, keep
     ┌──────────────────────┴───────┐  ┌──────┴─────┐  ┌──────┴─────┐  ┌────┴────────┐
     │ GUI apps with --control      │  │ PdfCraft   │  │ <app>-cli  │  │ <app>-cli   │
     │ PhotoCraft(token) WordCraft  │  │ GUI        │  │ exec / run │  │ mcp         │
     │ (key) LightCraft FilmCraft   │  │ JSON-RPC   │  │ commands   │  │ (progress   │
     │ DesignCraft EffectCraft      │  │            │  │ --version  │  │  pushed)    │
     │ VectorCraft SoundCraft (none)│  │            │  │            │  │             │
     │ AssetCraft (planned)         │  │            │  │            │  │             │
     └──────────────────────────────┘  └────────────┘  └────────────┘  └─────────────┘
       Storyteller web API: not connected to anything above (NG2)
```

---

## 5. Discovery and app lifecycle

### 5.1 How the hub finds apps

| Source                           | Works today, no upstream change?                                                                                                  | Finds hand-started apps?                                     | Gives port?                                       | Gives auth?                                     | Cost                                    | Risk                                                                                                         |
|----------------------------------|-----------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------|---------------------------------------------------|-------------------------------------------------|-----------------------------------------|--------------------------------------------------------------------------------------------------------------|
| **Config list** (installed apps) | Yes: env override → `PATH` → `%ProgramFiles%` / `%ProgramW6432%` / `%ProgramFiles(x86)%` → `%LOCALAPPDATA%\Programs` (AssetCraft's locator, `STATUS.md` 8a) | No (installed ≠ running)                                     | No                                                | No                                              | A few stats per app                    | None; also the source of `--version` and launch paths                                                        |
| **Manifest folder** (suite run dir) | Only for manifests the hub itself writes (apps it launched) and, later, for AssetCraft; no sibling writes one (03a O3, 03b O-B7) | Not until upstream adopts it                                 | Yes                                               | Yes (inline or by file reference)               | One directory listing                  | Stale files (handled §5.4); hostile content (bounded, validated)                                            |
| **App-native artefacts**         | Yes, two apps: PdfCraft control files the hub created (`{port,token,pid}`); WordCraft key files `%APPDATA%\WordCraft\control-key.local-<port>` (one per running port, removed on close, 03a §3a.6) | WordCraft: **yes** (the file name gives the port). PdfCraft: only at a path the hub knows | Yes                                     | Yes                                             | One listing each                       | PdfCraft file may be stale (A3-F4); WordCraft file may be stale after a crash (WordCraft replaces it itself) |
| **Process probe** (list processes, read argv) | Yes: `sysinfo` lists `photocraft.exe …` etc. with their command lines, so `--control 7980`, `--control-token-file <f>` and PdfCraft's `--control <file>` are visible | **Yes**, the only source that sees apps the user started by hand | When `--control <port>` is on argv; not when the port came from the `*_CONTROL_PORT` env var (reading another process's environment is not assumed, verify) | Token **file path** from argv (PhotoCraft), not the token | One process snapshot per pass          | Reads only same-user processes; argv can lie (identity is still checked at attach, §5.3)                     |
| **Port scan**                    | Technically                                                                                                                        | Partly                                                       | Yes                                               | No                                              | Up to 65,535 connects                  | **Rejected.** It sends bytes to unknown listeners, including unauthenticated apps and SoundCraft, whose server runs the next JSON line after junk (O-B2); slow; looks like malware to endpoint security; identifies nothing reliably |

**Recommendation A3-R2: a v1 hybrid of the first four, merged by `(app, pid)`, and no port scan.**

1. **Config list** gives the set of known, installed apps (exe, cli, version). It is static; recomputed on start and on `craftctl apps --refresh`.
2. **Run-dir manifests** (§5.2) give everything the hub launched before (so a restarted hub re-attaches) and, once adopted, everything apps publish themselves.
3. **App-native artefacts** add WordCraft instances (key files → ports) and PdfCraft instances (control files in the hub's own folder).
4. **Process probe** adds apps the user started by hand, with `--control <port>` parsed from argv. A running app without `--control` is listed as `running, not controllable`; the hub never tries to enable control on it.
5. Each candidate becomes an instance only after the attach check (§5.3). Two sources agreeing on the same pid merge into one candidate; on conflict, a manifest written by the app wins over one written by the hub, which wins over argv.

Then, **upstream later (A3-O1, feeds D6):** every app publishes the manifest below whenever `--control` is on, through `craft-control-server`'s writer (O-B1), so sources 3–4 become fallbacks for old versions.

### 5.2 Manifest schema and location

**Precedent.** PdfCraft's control file: `{"port","token","pid"}`, written by staging a new file next to the target (random name, `create_new`, owner-only on Unix) and renaming it over the target so a planted link is replaced, not followed, and a reader never sees half a file; the app refuses to start if it can't publish (`pdfcraft` `apps/pdfcraft/src/main.rs:432-499` @ `ac89a11`). The suite manifest keeps that write discipline and those three fields, and adds what the hub needs.

**Location.**

| Platform | Suite run dir (manifests, any app writes here)                  | Hub state dir (hub's own files)                         | Why                                                                                                                                                             |
|----------|-----------------------------------------------------------------|---------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Windows  | `%LOCALAPPDATA%\CraftApps\run\`                                 | `%LOCALAPPDATA%\PipelineCraft\` (`hub.json`, `hub.lock`, `apps.toml` overrides, `tokens\`, `logs\`) | `%LOCALAPPDATA%` is per user and **not roaming**: a port and pid are only true on this machine (`%APPDATA%` roams in domain profiles). Not `%TEMP%`, which cleanup tools empty while apps run. Default ACL is user + SYSTEM + Administrators (verify, 03a O12) |
| Linux    | `$XDG_RUNTIME_DIR/craft-apps/` (0700, cleared at logout), else `~/.local/state/craft-apps/run/` | `~/.local/state/pipelinecraft/`                         | Runtime dir is the XDG place for sockets and pid-like files                                                                                                     |
| macOS    | `~/Library/Application Support/CraftApps/run/`                  | `~/Library/Application Support/PipelineCraft/`          | Per user                                                                                                                                                        |
| Override | `CRAFT_RUN_DIR`                                                 | `PIPELINECRAFT_HOME`                                    | Tests and isolated runs, like `WORDCRAFT_PROFILE_DIR`                                                                                                           |

The run dir is named for the **suite**, not the hub, so an upstream app never has to know the hub's name (and D1 never moves it). File name: `<app>-<pid>.json`, one per instance, so instances never overwrite each other.

**Schema (manifest version 1), draft:**

```json
{
  "$comment": "Craft app manifest, version 1. Written atomically (stage + rename). Readers ignore unknown fields.",
  "type": "object",
  "required": ["manifest", "app", "pid", "control"],
  "properties": {
    "manifest":    { "const": 1 },
    "app":         { "type": "string", "pattern": "^[a-z][a-z0-9-]{1,31}$", "description": "app id = binary stem (01 §2.2)" },
    "app_version": { "type": "string", "maxLength": 64, "description": "semver of the running app" },
    "pid":         { "type": "integer", "minimum": 1 },
    "exe":         { "type": "string", "maxLength": 4096, "description": "full path of the running program, for pid-reuse checks" },
    "started_at":  { "type": "string", "format": "date-time" },
    "instance":    { "type": "string", "pattern": "^[A-Za-z0-9_-]{8,64}$", "description": "random per process; echoed by app.hello when that exists (A3-O3)" },
    "publisher":   { "enum": ["app", "hub"], "description": "who wrote the file; the hub only deletes files it wrote, plus provably stale ones (§5.4)" },
    "control": {
      "type": "object",
      "required": ["transport", "port", "codec"],
      "properties": {
        "transport": { "const": "tcp" },
        "host":      { "const": "127.0.0.1" },
        "port":      { "type": "integer", "minimum": 1, "maximum": 65535 },
        "codec":     { "enum": ["suite-jsonl", "jsonrpc2"] },
        "protocol":  { "type": "integer", "minimum": 1, "description": "control protocol major version (A4 §6.6)" }
      }
    },
    "auth": {
      "type": "object",
      "required": ["kind"],
      "properties": {
        "kind":       { "enum": ["none", "token-frame", "key-field"] },
        "token":      { "type": "string", "pattern": "^[0-9a-f]{32,128}$", "description": "inline secret; the file is then owner-only" },
        "token_file": { "type": "string", "maxLength": 4096, "description": "or: where the secret is; exactly one of token / token_file" }
      }
    },
    "capabilities": {
      "type": "object",
      "properties": {
        "methods": { "type": "array", "items": { "type": "string" }, "maxItems": 256 },
        "jobs":    { "type": "string", "description": "poll method or command id for jobs, e.g. jobs.list" },
        "events":  { "type": "boolean", "description": "true only once events.subscribe exists (O-B13)" },
        "effects": { "type": "boolean", "description": "true if engine.commands carries effects (O-B5)" }
      }
    }
  }
}
```

Example: the manifest the hub writes for a PhotoCraft it launched.

```json
{"manifest":1,"app":"photocraft","app_version":"0.9.0","pid":21844,
 "exe":"C:\\Program Files\\PhotoCraft\\photocraft.exe","started_at":"2026-10-15T17:02:11Z",
 "instance":"k3V9qL0wZp7c","publisher":"hub",
 "control":{"transport":"tcp","host":"127.0.0.1","port":47601,"codec":"suite-jsonl","protocol":1},
 "auth":{"kind":"token-frame","token_file":"C:\\Users\\paul\\AppData\\Local\\PipelineCraft\\tokens\\photocraft-47601.token"},
 "capabilities":{"methods":["engine.execute","engine.commands","jobs.list","jobs.cancel","app.open"],"jobs":"jobs.list","events":false,"effects":false}}
```

**Reading rules (hostile input, `AGENTS.md` non-negotiable 4):** at most 16 KiB per file and 256 files per pass; regular files only, no links followed; parse errors and schema violations make the file **ignored and reported**, never fatal; `token_file` must resolve inside the hub state dir or the app's own settings dir, else ignored. The hub's own `hub.json` uses the same schema with `"app":"pipelinecraft"` and is how clients find the daemon.

### 5.3 Attach vs launch

**Who starts apps with `--control`:** the hub, and only when a caller asks for something that needs an instance and none is attachable (`launch = "on-demand"`). Paul can still start apps by hand; the hub attaches to those when the process probe shows `--control`.

**Attach (an instance already running with control)**

1. Pid check: process alive, and its exe stem matches the app (pid reuse guard).
2. Connect to `127.0.0.1:<port>` within `attach_timeout_ms` (P3: 500 ms).
3. Handshake per auth kind: `token-frame` with the token from the manifest/token file/control file; `key-field` with the key read from WordCraft's key file **for that port**; `none` sends nothing.
4. **Identity check** (A3-F2 makes this mandatory): call the catalog method and require every `identity_probe` id to be present; PdfCraft and WordCraft use their file-held secret as identity instead (only that app could have answered with that token/key).
5. Read the version (§5.6), fetch and merge the catalog, emit `app.up`.

**Launch GUI (no attachable instance)**

1. Pick a port from `port_range` by binding `127.0.0.1:p` and releasing it (a race remains; step 4 catches it). SoundCraft gets `0` and reports its port; PdfCraft gets a control-file path in the hub state dir; PhotoCraft gets a fresh token written by the hub to `tokens\photocraft-<port>.token` before launch (PhotoCraft reads an existing token file and creates one only when it's missing, 03a §3a.2). Never pass a token on argv, and never rely on stderr (A3-F3).
2. Spawn the exe with `launch_args`, one argv item per placeholder, no shell, stdout/stderr piped to the hub log (this also captures "failed to bind" lines and SoundCraft's port line), detached so the window outlives the hub (AssetCraft's `DETACHED_PROCESS | CREATE_NEW_PROCESS_GROUP` precedent, `STATUS.md` 8a).
3. Wait for readiness: poll-connect every 100 ms up to `launch_timeout_ms` (10 s draft), or read the control file / stdout port line.
4. Attach steps 3–5. If identity fails (another listener held the port), and the child is still the hub's own just-launched process with no call made yet, the hub ends that child and retries once on another port; after that it reports `app.launch_failed` with the log tail.
5. Write the run-dir manifest with `publisher: "hub"`.

**Launch headless (batch, catalogs, thumbnails, recipes)**

- `cli-headless`: one child per call or per script, killed at the call timeout; bounded stdout.
- `mcp-stdio`: one long-lived child per app, started on first use, stopped after an idle period (draft: 5 min).
- Headless children are owned by the hub and must not outlive it (R4). Recommended mechanism on Windows: a job object with kill-on-close through a permissive crate (candidate `command-group`, license to confirm; the hub's own crates keep `unsafe` forbidden). A GUI window the hub launched is **not** an orphan: it belongs to the user, and the hub never closes it.

**When an app is running without `--control`:** listed as `running, not controllable`. For Open In, the hub falls back to today's launch-with-file-arguments behaviour (AssetCraft's catalog; PdfCraft hands files to the running window by itself), and says so in the reply. It never kills or restarts a user's app.

**Unauthenticated apps** (six of nine): attaching to a hand-started one is a security decision for A8 (03a open question 3). This section's default: attach to unauthenticated instances found by the process probe or launched by the hub, never to a port that only appears in user config; config key `attach_unauthenticated = "probed-or-launched"`.

### 5.4 Health, crash and stale cleanup

| Check                      | How                                                                                                                                                                                                                           | Interval (draft)                                              | On failure                                                                                                                                   |
|----------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------|
| Liveness                   | Pid alive and exe matches (`sysinfo`, only the tracked pids are refreshed)                                                                                                                                                    | 1 s                                                           | `app.down {reason:"exited"}`; in-flight calls fail with a hub error (A4) within 2 s (R1); hub jobs on it end `failed`                         |
| Reachability               | Connections are **lazy**: opened on first call, reused, reopened when the app closed them (LightCraft drops silent connections after 5 s; VectorCraft after 5 min idle). A failed reconnect while the pid lives marks the instance `unreachable` | on use                                                        | `app.unreachable`; next discovery pass retries                                                                                               |
| Readiness (optional)       | The configured `health` call, a cheap read; never a hub-invented method (WordCraft runs unknown methods as command ids)                                                                                                        | 30 s, only while a subscriber or open job needs that instance | 3 failures → `unreachable`                                                                                                                   |
| Discovery pass             | All sources (§5.1)                                                                                                                                                                                                            | 5 s while any client is connected; on demand otherwise         | New instances → attach; vanished → `app.down`                                                                                               |

No heartbeat file writes: apps don't write one today, and a timestamp rewritten every few seconds churns disks and cloud-sync folders for no gain over a pid check.

**Stale cleanup rules (R3: within one discovery pass):**
- A run-dir manifest whose pid is dead, or alive but a different exe, is stale. The hub deletes it if `publisher` is `hub`, or if `publisher` is `app` and it is stale by both checks; otherwise it ignores it.
- PdfCraft control files are only trusted when the pid inside is alive and is `pdfcraft.exe` (A3-F4). The hub deletes only control files under its own state dir.
- WordCraft key files are WordCraft's: never deleted by the hub (WordCraft replaces a stale one itself, 03a §3a.6). A key file with no live `wordcraft.exe --control <port>` is ignored.
- Token files under `tokens\` are deleted with the hub manifest that refers to them.

**Crashes.**
- *App crash:* GUI apps are never auto-restarted (they hold the user's documents, and EffectCraft skips crash recovery under `--control`, 03b). Headless children are restarted on the next call, at most 3 times in 60 s, then `app.headless_failed`.
- *Hub crash:* headless children die with it (job object). GUI apps keep running; on restart, the hub reads its own run-dir manifests and token files and re-attaches. Clients see their connections drop and reconnect via `hub.json`.
- *Single hub:* an exclusive OS lock on `hub.lock` (LightCraft's `catalog.lock` pattern, `lightcraft` `crates/catalog/src/lock.rs:1-19`, released by the OS when the process ends, so it is never stale). A second `pipelinecraft serve` exits and points at the running one.

### 5.5 Multiple instances of one app

- **Identity:** an instance is `<app>@<pid>`; `instance` (random) from the manifest is carried when present. Ports are not identities (they are reused).
- **Default instance for an address without `@`:** the only controllable instance; if there are several, the one the caller's connection used last, else the most recently attached. A4 owns the final grammar; this section's proposal for writes is stricter: with more than one candidate and no pin, a call whose command effects are not read-only fails with a typed "ambiguous instance" error listing the instances. Reads go to the default.
- **App limits that shape this:** LightCraft allows one process per library (`catalog.lock`), so the hub attaches rather than launching a second LightCraft on the same library. PdfCraft's hand-off is skipped under `--control` (A3-F5), so hub launches always get their own window. WordCraft's key file is per port, so multiple WordCraft instances are distinguishable. PhotoCraft, FilmCraft, DesignCraft, EffectCraft, VectorCraft and SoundCraft have no single-instance rule on Windows (03a §3a.8 row "Single instance"; 03b static read).
- **Headless instances** never count as the default for GUI-targeted calls (Open In); they are used only where the router picked the headless transport.

### 5.6 Version negotiation

| Layer                        | Source                                                                                                         | Rule                                                                                                                                                                       |
|------------------------------|----------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Manifest format              | `manifest` field                                                                                               | Known major (1) → read; unknown → ignore the file, report once                                                                                                             |
| Control protocol             | `control.protocol` in a manifest; today implied by the adapter kind                                            | The hub supports a range per codec; outside it → `unsupported`                                                                                                             |
| App version                  | Manifest `app_version`; else EffectCraft `app.capabilities`; else `<cli> --version` of the installed CLI next to the exe (A3-F6); else unknown | Matched against `tested` and `[[override]] versions` with semver                                                                                                           |
| Command presence             | The live catalog                                                                                               | The real capability test: aliases, recipes and Open In check that the command id exists in **this** instance, not that the version is new enough                         |

**When the app is older (or newer) than the hub expects:**

| Case                                        | State        | Behaviour                                                                                                         |
|---------------------------------------------|--------------|-------------------------------------------------------------------------------------------------------------------|
| Inside `tested`                             | `ok`         | Normal                                                                                                            |
| Outside `tested`, adapter handshake works   | `untested`   | Calls allowed; `craftctl apps` and `app.up` carry a warning; overrides for the nearest range apply                |
| Below a hard `min` (if set) or handshake fails | `unsupported` | Listed with the reason; calls refused with a typed error; headless CLI may still be used for `--version` only |
| Version unknown                             | `untested`   | As above                                                                                                          |

Caveat: `<cli> --version` reports the **CLI's** version. It is the GUI's version only when both come from the same install folder (the MSIs install both, `STATUS.md` 8a; verify). Hence the upstream ask A3-O3 (`app.hello`).

### 5.7 Rust type sketches

Sketches, not final code: no `unsafe`, no `unwrap`/`expect`/`panic`; everything from disk or the wire is parsed into these types and validated, errors returned.

```rust
// craft-control-proto (L0) ---------------------------------------------------

/// Lowercase binary stem, e.g. "photocraft" (01 §2.2). Validated on construction.
#[derive(Debug, Clone, PartialEq, Eq, Hash, serde::Serialize, serde::Deserialize)]
#[serde(try_from = "String", into = "String")]
pub struct AppId(String);

impl TryFrom<String> for AppId {
    type Error = ProtoError;
    fn try_from(s: String) -> Result<Self, Self::Error> {
        let ok = (2..=32).contains(&s.len())
            && s.starts_with(|c: char| c.is_ascii_lowercase())
            && s.chars().all(|c| c.is_ascii_lowercase() || c.is_ascii_digit() || c == '-');
        if ok { Ok(Self(s)) } else { Err(ProtoError::BadAppId) }
    }
}
impl From<AppId> for String { fn from(a: AppId) -> Self { a.0 } }

#[derive(Debug, Clone, Copy, PartialEq, Eq, serde::Serialize, serde::Deserialize)]
#[serde(rename_all = "kebab-case")]
pub enum Codec { SuiteJsonl, Jsonrpc2 }

#[derive(Debug, Clone, PartialEq, Eq, serde::Serialize, serde::Deserialize)]
#[serde(tag = "kind", rename_all = "kebab-case")]
pub enum ManifestAuth {
    None,
    TokenFrame { #[serde(default)] token: Option<Secret>, #[serde(default)] token_file: Option<String> },
    KeyField   { #[serde(default)] token: Option<Secret>, #[serde(default)] token_file: Option<String> },
}

/// Hex secret; Debug never prints it.
#[derive(Clone, PartialEq, Eq, serde::Serialize, serde::Deserialize)]
#[serde(try_from = "String", into = "String")]
pub struct Secret(String);
impl std::fmt::Debug for Secret {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result { f.write_str("Secret(..)") }
}

#[derive(Debug, Clone, serde::Serialize, serde::Deserialize)]
pub struct ControlEndpoint {
    pub transport: String,          // "tcp" only in v1
    #[serde(default = "loopback")]
    pub host: String,               // must be "127.0.0.1"
    pub port: u16,
    pub codec: Codec,
    #[serde(default)]
    pub protocol: Option<u32>,
}

#[derive(Debug, Clone, serde::Serialize, serde::Deserialize)]
pub struct Manifest {
    pub manifest: u32,
    pub app: AppId,
    #[serde(default)] pub app_version: Option<String>,
    pub pid: u32,
    #[serde(default)] pub exe: Option<String>,
    #[serde(default)] pub started_at: Option<String>,
    #[serde(default)] pub instance: Option<String>,
    #[serde(default)] pub publisher: Publisher,
    pub control: ControlEndpoint,
    #[serde(default)] pub auth: Option<ManifestAuth>,
    #[serde(default)] pub capabilities: Capabilities,
}

#[derive(Debug, Clone, Copy, Default, PartialEq, Eq, serde::Serialize, serde::Deserialize)]
#[serde(rename_all = "lowercase")]
pub enum Publisher { #[default] App, Hub }

#[derive(Debug, Clone, Default, serde::Serialize, serde::Deserialize)]
pub struct Capabilities {
    #[serde(default)] pub methods: Vec<String>,
    #[serde(default)] pub jobs: Option<String>,
    #[serde(default)] pub events: bool,
    #[serde(default)] pub effects: bool,
}

pub const MAX_MANIFEST_BYTES: usize = 16 * 1024;

impl Manifest {
    /// Parse hostile bytes: size cap, JSON, then semantic checks. Never panics.
    pub fn parse(bytes: &[u8]) -> Result<Self, ProtoError> {
        if bytes.len() > MAX_MANIFEST_BYTES { return Err(ProtoError::TooLarge); }
        let m: Manifest = serde_json::from_slice(bytes).map_err(|e| ProtoError::Json(e.to_string()))?;
        if m.manifest != 1 { return Err(ProtoError::UnknownManifestVersion(m.manifest)); }
        if m.control.host != "127.0.0.1" || m.control.transport != "tcp" { return Err(ProtoError::NotLoopback); }
        if m.control.port == 0 || m.pid == 0 { return Err(ProtoError::BadField("port/pid")); }
        if let Some(ManifestAuth::TokenFrame { token, token_file } | ManifestAuth::KeyField { token, token_file }) = &m.auth {
            if token.is_some() == token_file.is_some() { return Err(ProtoError::BadField("exactly one of token/token_file")); }
        }
        Ok(m)
    }
}

/// Static description of one app, deserialized from apps.default.toml + user overrides (§4.4).
#[derive(Debug, Clone, serde::Deserialize)]
pub struct AppDescriptor {
    #[serde(default = "yes")] pub enabled: bool,
    pub name: String,
    #[serde(default)] pub exe: Option<String>,
    #[serde(default)] pub cli: Option<String>,
    #[serde(default)] pub install_dirs: Vec<String>,
    #[serde(default)] pub tested: Option<String>,          // semver range, parsed in adapters
    #[serde(default)] pub identity_probe: Vec<String>,
    #[serde(default)] pub single_instance: Option<String>,
    #[serde(default)] pub control: Option<ControlConfig>,
    #[serde(default)] pub catalog: Option<CatalogConfig>,
    #[serde(default)] pub headless: Option<HeadlessConfig>,
    #[serde(default, rename = "override")] pub overrides: Vec<VersionOverride>,
}

#[derive(Debug, Clone, Copy, PartialEq, Eq, serde::Deserialize)]
#[serde(rename_all = "kebab-case")]
pub enum PortZero { Lost, Refused, ReportedStdout, File }

#[derive(Debug, Clone, serde::Deserialize)]
#[serde(tag = "kind", rename_all = "kebab-case")]
pub enum AuthConfig {
    None,
    TokenFrame { token_hex_len: usize, source: SecretSource },
    KeyField   { token_hex_len: usize, source: SecretSource, #[serde(default)] mode: KeyMode,
                 #[serde(default)] key_file: Option<String>, #[serde(default)] key_file_env_dir: Option<String> },
}
// ControlConfig, CatalogConfig, HeadlessConfig, VersionOverride, SecretSource, KeyMode: plain
// serde structs/enums mirroring the TOML keys in §4.4.

// craft-hub-adapters (L2) ------------------------------------------------------

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum AdapterKind { ControlJsonl, ControlJsonrpc, CliHeadless, McpStdio }

/// One live link to one app instance over one transport. Blocking with deadlines (std threads,
/// like AssetCraft's pools); the core runs each adapter on its own worker.
pub trait Adapter: Send {
    fn kind(&self) -> AdapterKind;
    /// Connect + handshake + identity check. Idempotent; reconnects if the app closed the link.
    fn attach(&mut self, deadline: std::time::Instant) -> Result<Hello, AdapterError>;
    /// The catalog, normalized (03a O9).
    fn commands(&mut self, deadline: std::time::Instant) -> Result<Vec<CommandInfo>, AdapterError>;
    /// Run one command. `Pending` when the app returned a job handle (PhotoCraft wait:false).
    fn execute(&mut self, call: &Call, deadline: std::time::Instant) -> Result<CallOutcome, AdapterError>;
    /// Snapshot of the app's jobs, for the polling bridge. Default: not supported.
    fn poll_jobs(&mut self, _deadline: std::time::Instant) -> Result<Vec<JobSnapshot>, AdapterError> {
        Err(AdapterError::Unsupported("job polling"))
    }
    fn cancel(&mut self, _job: &AppJobRef, _deadline: std::time::Instant) -> Result<(), AdapterError> {
        Err(AdapterError::Unsupported("cancel"))
    }
    /// Pushed progress (mcp-stdio only), drained without blocking.
    fn drain_progress(&mut self) -> Vec<ProgressNote> { Vec::new() }
    fn close(&mut self);
}

#[derive(Debug, Clone)]
pub struct Hello { pub app: AppId, pub version: Option<String>, pub identity: IdentityProof }

#[derive(Debug, Clone)]
pub enum IdentityProof { ProbeIds(Vec<String>), SecretFromAppFile, AppHello { instance: String } }

#[derive(Debug, Clone)]
pub enum CallOutcome { Done(serde_json::Value), Pending(AppJobRef) }

#[derive(Debug)]
pub enum AdapterError {
    Connect(std::io::Error),
    Auth,                              // rejected handshake
    Identity { expected: AppId },      // listener is not this app (A3-F2)
    Timeout { may_have_run: bool },    // PhotoCraft/VectorCraft distinguish "not run" (03a inputs)
    LineTooLong { limit: usize },
    Protocol(String),                  // malformed reply; connection dropped
    App { message: String },           // the app's own error string or {code,message}
    Unsupported(&'static str),         // e.g. PdfCraft ui.command with params (O6)
    ChildExited(Option<i32>),
}

// craft-hub-core (L3) ------------------------------------------------------------

#[derive(Debug, Clone, PartialEq, Eq, Hash)]
pub struct InstanceId { pub app: AppId, pub pid: u32 }

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum InstanceState { Candidate, Attaching, Ok, Untested, Unsupported, Unreachable, NotControllable, Down }

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum CandidateSource { RunDirManifest, AppArtefact, ProcessProbe, HubLaunched }
```

---

## Findings and opportunities from this section

| ID     | Scope                  | Opportunity                                                                                                         | Suggested fix                                                                                                                                                                                                                                                                                 | Owner                                     | Status |
|--------|------------------------|---------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------|--------|
| A3-R1  | §4.2 deployment, D2    | Hub and AssetCraft need the same control code first                                                                  | Incubate `craft-control-*` and `craft-hub-*` in the AssetCraft workspace; promote `craft-control-*` to craft-libs after one sibling PR builds on it; promote the hub to its own repo at hub Stage 2                                                                                          | Paul (D2); Claude                         | open   |
| A3-R2  | §5.1 discovery         | No app publishes where it listens                                                                                    | v1 hybrid: config list + run-dir manifests + app artefacts (WordCraft key files, PdfCraft control files) + process probe with argv parsing; no port scan                                                                                                                                       | Claude                                    | open   |
| A3-O1  | §5.2 manifest, D6      | PdfCraft's control file is the only manifest; nothing is suite-wide                                                 | Upstream PR via `craft-control-server`: write `<run dir>/<app>-<pid>.json` (schema §5.2) whenever `--control` is on; keep PdfCraft's `--control <file>` working. AssetCraft Stage 1.5 implements it first as the reference                                                                   | Claude (AssetCraft, crate); upstream apps | open   |
| A3-O2  | §4.3 layers            | `TABLE` can't keep hub crates free of `assetcraft-*` deps, which promotion needs                                    | Add `ISLANDS` (`craft-` crates depend only on `craft-` crates) to `xtask/src/layers.rs` `check()`, with a unit test like the existing ones                                                                                                                                                    | Claude                                    | open   |
| A3-O3  | §5.3 identity, §5.6    | No app can say "I am X, version V, instance I" over control (A3-F6), so identity uses probe ids and versions use the CLI | Upstream: one read-only control method `app.hello` → `{app, version, pid, instance, protocol, capabilities}`; EffectCraft's `app.capabilities` is the nearest precedent. Hub uses it when present                                                                                             | Claude (spec); upstream apps              | open   |
| A3-O4  | §5.3 launch            | `--control 0` loses the port in five apps (A3-F1), and a failed bind leaves the app running without control (A3-F2) | Hub: explicit ports from `port_range` + identity check + one retry. Upstream: log/print `listener.local_addr()` and exit non-zero when `--control` was asked for and the bind failed (PdfCraft's rule, `main.rs:432-437`); extends 03a O7                                                     | Claude (hub); upstream apps               | open   |
| A3-O5  | §5.4 stale cleanup     | PdfCraft's control file outlives the process (A3-F4)                                                                 | Hub: trust it only with a live `pdfcraft.exe` pid. Upstream: remove the file on clean exit                                                                                                                                                                                                    | Claude (hub); PdfCraft                    | open   |
| A3-O6  | §4.4 adapters          | WordCraft runs any unknown control method as a command id (03a §3a.6)                                                | Adapter config `unknown_method = "runs-as-command"`; the hub never sends invented methods (no `ping`); contract test asserts it. Upstream: return "unknown method" for names outside the method table                                                                                        | Claude (hub); WordCraft                   | open   |
| A3-O7  | §5.3 launch on Windows | Output from release GUI builds is invisible unless piped (A3-F3); SoundCraft's port line depends on it               | Hub always pipes stdout/stderr of GUI apps it launches into its log; verify on Windows that the pipe works with `windows_subsystem = "windows"`                                                                                                                                                 | Claude; Paul (verify)                     | open   |
| A3-O8  | §4.1 lifecycle         | `AppLocator` and the install catalog live in AssetCraft's engine (`open_in.rs`)                                      | Lift `candidates()` and the catalog into `craft-hub-discovery`; AssetCraft's engine depends on it (L2 under L5), so Open In and the hub share one discovery (also closes the STATUS step 8 "one discovery for both" opportunity)                                                              | Claude                                    | open   |
| A3-O9  | §4.4 / §5.6            | Quirks change as upstream fixes land                                                                                 | Version-ranged `[[override]]` tables in adapter config, checked by contract tests per installed version (§13.2)                                                                                                                                                                              | Claude                                    | open   |

## Not verified

Everything above is a static read. These need a live run on Paul's Windows machine. Use throwaway files.

**Next action (Paul):**

1. Port 0 and port loss (A3-F1). Expected: LightCraft logs `listening on 127.0.0.1:0` (no real port); SoundCraft prints a real `SOUNDCRAFT_CONTROL_PORT=<n>` when stdout is redirected.
   ```powershell
   Set-Location F:\Script-Projects\ArtCrafting\soundcraft
   $p = Start-Process .\target\release\soundcraft.exe -ArgumentList '--control','0','--no-audio' -RedirectStandardOutput "$env:TEMP\sc-out.txt" -PassThru
   Start-Sleep 5; Get-Content "$env:TEMP\sc-out.txt"; Stop-Process $p.Id
   ```
   This also checks A3-O7: if the file is empty, piped stdout does not work for release GUI builds.
2. Bind failure (A3-F2). Expected: the second app opens a window with no control server, and a connection to the port reaches the first app.
   ```powershell
   Set-Location F:\Script-Projects\ArtCrafting\designcraft
   Start-Process .\target\release\designcraft.exe -ArgumentList '--control','47650'
   Start-Sleep 5
   Set-Location F:\Script-Projects\ArtCrafting\vectorcraft
   Start-Process .\target\release\vectorcraft.exe -ArgumentList '--control','47650' -RedirectStandardError "$env:TEMP\vc-err.txt"
   Start-Sleep 5; Get-Content "$env:TEMP\vc-err.txt"
   ```
   Expected: `vectorcraft: control server failed to bind 127.0.0.1:47650: …` and VectorCraft still running.
3. Process probe sees argv (A3-R2). Expected: the command line includes `--control 47650`.
   ```powershell
   Set-Location F:\Script-Projects\ArtCrafting
   Get-CimInstance Win32_Process -Filter "Name like '%craft.exe'" | Select-Object ProcessId, Name, CommandLine
   ```
4. PdfCraft control file is stale after exit (A3-F4). Expected: the file still exists after the window closes, with the old pid.
   ```powershell
   Set-Location F:\Script-Projects\ArtCrafting\pdfcraft
   New-Item -ItemType Directory -Force "$env:LOCALAPPDATA\PipelineCraft" | Out-Null
   $p = Start-Process .\target\release\pdfcraft.exe -ArgumentList '--control',"$env:LOCALAPPDATA\PipelineCraft\pdf.json" -PassThru
   Start-Sleep 5; Get-Content "$env:LOCALAPPDATA\PipelineCraft\pdf.json"
   Stop-Process $p.Id; Start-Sleep 1; Test-Path "$env:LOCALAPPDATA\PipelineCraft\pdf.json"
   ```
   Expected: JSON `{"port":…,"token":"<32 hex>","pid":…}`, then `True`.
5. WordCraft key file appears per port and disappears on close (§5.1 artefact source). Expected: `control-key.local-47660` exists while the window is open and is gone after closing it normally (close the window by hand, not with `Stop-Process`).
   ```powershell
   Set-Location F:\Script-Projects\ArtCrafting\wordcraft
   Start-Process .\target\release\wordcraft.exe -ArgumentList '--control','47660'
   Start-Sleep 5; Get-ChildItem "$env:APPDATA\WordCraft\control-key.*"
   ```
6. WordCraft runs unknown methods as commands (A3-O6). With the key `$k` from step 5, send `{"id":1,"key":"<k>","method":"app.state","params":{}}`; expected: an "unknown command" style error from the registry, not "unknown method".
7. Run-dir ACL (§5.2). Expected: only your user, SYSTEM and Administrators.
   ```powershell
   Set-Location F:\Script-Projects\ArtCrafting
   New-Item -ItemType Directory -Force "$env:LOCALAPPDATA\CraftApps\run" | Out-Null
   icacls "$env:LOCALAPPDATA\CraftApps\run"
   ```
8. CLI version equals GUI version in one install (§5.6). Expected: the same version string twice.
   ```powershell
   Set-Location 'C:\Program Files\FilmCraft'
   .\filmcraft-cli.exe --version; (Get-Item .\filmcraft.exe).VersionInfo.ProductVersion
   ```
9. LightCraft `mcp --compact` still offers `command_run` (adapter config `# verify`). Expected: `command_run` in the `tools/list` reply.

## Open questions

1. **Run-dir name.** `%LOCALAPPDATA%\CraftApps\run\` is suite-neutral so upstream apps never name the hub. Is "CraftApps" the right suite word, or should it be `storytold` / `Crafting`? It has to be agreed with upstream before the first manifest PR (D6). (Paul)
2. **Attaching to hand-started unauthenticated apps.** Default here is "probed-or-launched". Is attaching to a LightCraft/FilmCraft/DesignCraft/EffectCraft/VectorCraft/SoundCraft window Paul started himself acceptable before O2 (token handshake upstream) lands? (A8 proposes, Paul decides)
3. **Who launches.** Should the hub ever launch a GUI app on its own (on-demand Open In into a closed app, AC1c), or only headless children, leaving GUI launches to AssetCraft's existing Open In? This draft says the hub may launch, per AC1c. (Paul)
4. **Crate prefix.** This draft uses `craft-hub-*` (rename-proof) where A1-N1 suggested `relaycraft-*`. The integrator should pick one. (Paul, D1)
5. **Async or threads.** The sketches are blocking with deadlines on std threads, matching AssetCraft. The PhotoCraft and FilmCraft MCP servers use rmcp (tokio). Is a thread-per-instance daemon fine for v1 (≤ 10 apps, ≤ 10 connections)? (A4/A9 confirm)

## Inputs for downstream agents

- **A4 (§6 protocol):** the daemon's listener reuses `craft-control-server` with the suite-jsonl codec (D3). Assumed address grammar `<app>[@<pid>]/<command.id>` plus capability aliases; this section proposes that writes with an ambiguous instance fail with a typed error (§5.5). Errors this layer raises and §6.5 should give codes: identity mismatch, auth rejected, timeout with `may_have_run`, line too long, unsupported over this transport, child exited, unsupported version, ambiguous instance, launch failed. The hub must never send invented method names to WordCraft (A3-O6), so any hub keep-alive is hub-side only.
- **A5 (§7 asset refs):** launch args need `{read_root}`/`{write_root}` for PhotoCraft (relative paths, O13), VectorCraft (GUI roots), WordCraft and PdfCraft headless. §7.4 should define how those roots are chosen; until then this section uses the asset's folder.
- **A6 (§8–9):** event sources are the polling bridges in §4.1 plus MCP progress from `mcp-stdio` children; `app.up`, `app.down`, `app.unreachable`, `app.launch_failed` come from lifecycle. Pollers run only while a job or subscriber needs them. AC1c "launch and attach" is §5.3; AC1b "same window" depends on attach finding the instance (§5.1).
- **A7 (§10, §12):** the MCP front door is a separate stdio shim process (`pipelinecraft mcp`) that is a hub client; `craftctl` starts the daemon on demand by reading `hub.json` (§5.2). `craftctl apps` should show state (`ok`/`untested`/`unsupported`/`unreachable`/`running, not controllable`) and the source.
- **A8 (§11):** secrets the hub holds: PhotoCraft token files it writes under `%LOCALAPPDATA%\PipelineCraft\tokens\`, PdfCraft control files it names, WordCraft key files it reads. Manifests may carry a token inline (owner-only file) or a `token_file`; decide which is allowed. Port scan is rejected. Attach policy for unauthenticated apps is open question 2.
- **A9 (§13, §14, §16):** layer rows in §4.3 and the island rule (A3-O2); the fake app must emulate each adapter kind and the quirks in §4.4 (port 0 kinds, bind failure, junk policy, WordCraft unknown method). Upstream asks from this section, smallest first: A3-O4 (print real port / exit on bind failure), A3-O5, A3-O1 (manifest), A3-O3 (`app.hello`). Stage 0 "See" = §5 discovery + health + `craftctl apps`.
- **A10 (§15):** D2 recommendation is A3-R1; D5 (on-demand daemon) is assumed throughout, with the `hub.lock` single-instance rule.
