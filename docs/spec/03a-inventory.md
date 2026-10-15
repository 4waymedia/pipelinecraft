# §3a Current landscape: PhotoCraft, LightCraft, FilmCraft, PdfCraft, WordCraft

_2026-10-15 · spec draft v0 · status: draft (agent A2a)_

- **Scope:** the local-app-control surfaces (`<app>-cli`, MCP over stdio, `--control`) of five storytold Crafting Apps, re-verified at current `main` HEAD, plus what they share through craft-libs. The other apps are in A2b's half (§3b). The Storyteller web API is out of scope.
- **Project goal it serves:** PipelineCraft needs one adapter contract over every app. This section records what each app exposes today, so §4 (adapters), §6 (protocol), §7 (asset refs) and §11 (auth) are built on facts and not on assumptions.
- **Immediate area of improvement:** each app is close to the shared `engine.execute` / `engine.commands` shape, but there are five auth models, three framings, two id styles and four ways of saying "read-only". The O# list at the end names the small fixes that would close most of the gap.

**Evidence rules.** Every claim is a static read of the shallow clones in `/home/claude/src/<repo>`. Nothing was built or run. Citations are `repo` `path:line` @ commit. The HEAD commits are below. "Study" means the earlier static study `assetcraft/docs/sibling-apis/*.md`, which was read at each app's release tag. A difference between that study and HEAD is called **drift**.

| Repo         | HEAD commit | HEAD date  | Workspace version | Study tag / commit |
|--------------|-------------|------------|-------------------|--------------------|
| `photocraft` | `fd28da5`   | 2026-10-15 | 0.9.0             | v0.9.0 / `c699e3c` |
| `lightcraft` | `38adc17`   | 2026-10-14 | 0.8.0             | v0.8.0 / `16dbb90` |
| `filmcraft`  | `9ed9650`   | 2026-10-14 | 0.8.0             | v0.8.0 / `f03e215` |
| `pdfcraft`   | `ac89a11`   | 2026-10-14 | 0.8.0             | v0.8.0 / `9931329` |
| `wordcraft`  | `3292ba0`   | 2026-10-15 | 0.7.0             | v0.7.0 / `fc41918` |
| `craft-libs` | `8f5a283`   | 2026-10-14 | 0.1.0             | n/a                |

**How commands were counted.** A Python regex pass over each engine's non-test source (everything before the first `#[cfg(test)]` in each file). It collects the first string literal of every spec constructor (`cmd!`, `query!`, `spec!`, `CommandSpec::new`, PdfCraft's `c(`, and `CommandSpec { id: "…" }`), keeps ids that contain a dot, and de-duplicates. Ids built with `format!`/`concat!` (PhotoCraft's adjustment and stack-mode families, FilmCraft's `camera_keys!`) are **not** expanded, so the HEAD counts below are lower bounds. The study expanded those by hand, so a lower HEAD number is not necessarily drift. The live, authoritative count is `<app>-cli commands --json` (see Not verified).

---

## 3a.1 Inventory at a glance

| App        | Engine cmds (HEAD, literal) | Study count | UI-only cmds (HEAD) | MCP tools               | `--control` auth                           | Framing             | Push events | Long-job progress                                         |
|------------|-----------------------------|-------------|---------------------|-------------------------|--------------------------------------------|---------------------|-------------|-----------------------------------------------------------|
| PhotoCraft | ≥ 712                       | 863         | 79 `UI_COMMANDS`    | 25                      | 256-bit token, `auth` first frame          | JSON lines          | none        | `jobs.list` poll; MCP `notifications/progress` (one cmd)  |
| LightCraft | ≥ 304                       | 312         | ~179                | 28 helpers + 1 per cmd  | **none**                                   | JSON lines          | none        | `activity.list` poll; MCP progress on headless export     |
| FilmCraft  | ≥ 644                       | 702         | 138 `uic!`          | 17                      | **none**                                   | JSON lines          | none        | `jobs.list` / `export.queue.*` poll; MCP progress (export) |
| PdfCraft   | 192 (registry) + 163 tools  | 194 + 163   | (in registry)       | 163 (15 + 2 `--compact`) | 128-bit token in control file, `auth` first | **JSON-RPC 2.0** lines | none     | none                                                      |
| WordCraft  | 629                         | 617         | ~30 `ui.*`          | 16                      | 256-bit key in key file, every request or `auth` once | JSON lines (`key` field) | none | none                                                |

Sources for each cell are in the per-app sections.

---

## 3a.2 PhotoCraft (`photocraft` @ `fd28da5`)

### Transports

| Surface            | Detail                                                                                                                                                                                                                                                                                       | Source                                                                                                           |
|--------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------|
| CLI binary         | `photocraft-cli`. Subcommands: `convert`, `info`, `run (<file>\|--new <json>) --cmd <id> [--params <json>]… [--out]`, `batch`, `droplet`, `commands [--json] [--filter]`, `mcp`, `serve`                                                                                                   | `photocraft` `apps/photocraft-cli/src/lib.rs:14-55` (usage), `:72-100` (table) @ `fd28da5`                       |
| `commands --json`  | Bare JSON array, registry order: `{id, label, menu:[…], shortcut, params:"<prose>", enabled, disabledReason, needsDocument, keywords?}`. `params` is a human-readable string, not a schema                                                                                                    | `apps/photocraft-cli/src/lib.rs:535-554`; `crates/engine/src/commands.rs:143-154`; `docs/control-protocol.md:180-208` @ `fd28da5` |
| `serve`            | Headless JSON-lines RPC on stdio or `127.0.0.1:<port>`, same `auth` handshake. Methods: `engine.execute`, `jobs.list/cancel`, `engine.commands`, `session.list`, `doc.open/place/new/save/exportLayers/exportDataSets/importDataSets/inspect/render/select/close`, `batch`, `methods`        | `crates/automation/src/rpc.rs:38-57` @ `fd28da5`                                                                 |
| MCP                | `photocraft-cli mcp` (headless, in-process session) or `mcp --bridge 127.0.0.1:<port> --control-token-file <f>` (forwards to the running app). rmcp, stdio only. Resources `photocraft://document`, `photocraft://commands`                                                                    | `apps/photocraft-cli/src/lib.rs:583`; `crates/automation/src/server/conventions.rs:7-8` @ `fd28da5`              |
| MCP tool mapping   | 25 hand-written tools, not one per command. The registry is reached through `command_list` / `command_run {id, params, wait?}` / `command_batch {steps, stop_on_error?}` (≤ 256 steps)                                                                                                         | `crates/automation/src/server.rs:581-956` (25 `#[tool(` attributes) @ `fd28da5`                                  |
| `--control`        | `photocraft --control <port>` or `PHOTOCRAFT_CONTROL_PORT`. Binds `127.0.0.1:<port>`. A bad port exits with code 2. Port `0` is accepted but the chosen port is never published (see O7)                                                                                                     | `apps/photocraft/src/main.rs:131-166`, `:198-204`, `:275`; `apps/photocraft/src/control_server.rs:18-27` @ `fd28da5` |
| Framing            | JSON lines. Request `{"id","method","params"}` → reply `{"id","ok":true,"result"}` or `{"id","ok":false,"error":"<string>"}`. 1 MiB request line, 16 connections, 30 s I/O timeout, 60 s reply deadline                                                                                     | `apps/photocraft/src/control_server.rs:49-117`; `crates/automation/src/security.rs:15-23`; `crates/ui-egui/src/control.rs:48` @ `fd28da5` |
| Control methods    | `engine.execute` / `ui.menu.invoke` `{command\|id, params, wait?}`, `engine.commands`, `jobs.list`, `jobs.cancel`, `ui.menu.list`, `ui.inspect`, `ui.set`, `ui.dialog.*`, `ui.window.*`, `ui.pointer`, `ui.click`/`move`, `ui.key`, `ui.type`, `ui.resize`, `ui.gpu.simulateLoss`, `ui.focus`, `ui.screenshot`, `ui.context.choose`, `app.open`, `app.save`, `app.quit` | `crates/ui-egui/src/control.rs:393-1263` @ `fd28da5`                                                            |

### Auth

- Token: 256 bits from the OS CSPRNG, as 64 lowercase hex characters (`crates/automation/src/security.rs:74-85`, `:88-93`).
- Sources, in order: `--control-token <hex>`, `--control-token-file <path>`, env `PHOTOCRAFT_CONTROL_TOKEN` / `PHOTOCRAFT_CONTROL_TOKEN_FILE` (`security.rs:246-250`). A missing token file is created with `create_new` and a fresh token. **Mode 0600 is applied on Unix only**; on Windows the file inherits the folder's ACL (`security.rs:140-153`; `docs/control-protocol.md:19`). With no token and no file, the token is printed to stderr as `photocraft: control token: <hex>` (`apps/photocraft/src/main.rs:284-289`).
- Wire: the first frame must be `{"id":…,"method":"auth","params":{"token":"<hex>"}}`. Success → `{"id","ok":true,"result":{"authenticated":true}}`. Anything else → `{"ok":false,"error":"authentication required"}` and the connection closes. Constant-time compare. A second `auth` gets `"already authenticated"` and the connection stays open (`security.rs:96-131`; `control_server.rs:74-87`).
- Extra capability gate: `--automation-read-root` / `--automation-write-root`. Over control and MCP, `app.open {path}` takes a **relative** path under the read root, and every `file.*` command that takes an ambient path is refused (`docs/control-protocol.md:163`; `crates/ui-egui/src/control.rs:1217-1255`; `crates/automation/src/workspace.rs`). The CLI `run`/`batch` subcommands are trusted-local, so paths work there.

### Command registry

- Spec: `CommandSpec {id, label, menu, shortcut, params: &str, enabled, run, journal}` (`crates/engine/src/commands.rs:14-28`). `journal: false` marks a query, and that is the only effects-like field. `cmd!` sets `journal: true` (`:129-133`). Lookup `find()` (`:174`), registry `build()` (`:445`).
- Id style: dotted camelCase that follows Photoshop's menus: `layer.new.layer`, `file.export.quickExport`, `image.adjustments.curves` (`commands.rs:1`).
- Count: ≥ 712 literal engine ids at HEAD (study: 863 including generated families). UI-only ids: 79 in `UI_COMMANDS` (`crates/ui-egui/src/menus.rs:23-107`); study counted 76. **Drift: +3 UI ids.**
- Params schema: prose string only. Strict checking applies to the ids listed in `crates/engine/src/param_check.rs`. MCP tools have real JSON Schemas (rmcp `Parameters<T>`), but only for the 25 tools.
- Effects metadata on the wire: **none**. `journal` is not emitted by `list_entry` (`commands.rs:143-154`). MCP tool annotations are hand-coded per tool name: `readOnlyHint` for 7 tools, `destructiveHint` for the rest except `doc_new`/`doc_open`/`doc_place` (`crates/automation/src/server/conventions.rs:11-24`).

### Events and long jobs

- No push events on control, `serve` or MCP (no subscribe/notify code in `crates/ui-egui/src/control.rs` or `crates/automation/src/rpc.rs`).
- Background jobs: long commands (every `filter.*`, content-aware fill/scale, photomerge, ABR import, file opens and saves) run as jobs. `engine.execute … "wait": false` returns `{job, pending: true}`. Poll `jobs.list` (`state`, `progress` 0–1, result or error) and stop with `jobs.cancel {job?}` (`docs/control-protocol.md:210-212`; `crates/engine/src/jobs.rs:844-867`).
- MCP progress: `notifications/progress` is sent only for `command_run` of `file.export.renderVideo` in headless mode, and cancellation is honoured (`crates/automation/src/server/progress.rs:9-40`; `conventions.rs:52-57`).

### Asset handling for handoff

| Job                     | Surface                                                                                                       | Source                                                                         |
|-------------------------|---------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------|
| Open in running app     | control `app.open {"path":"<relative to read root>"}` → `{path, name, warnings}`                              | `crates/ui-egui/src/control.rs:1217-1255` @ `fd28da5`                          |
| Open headless           | MCP `doc_open {path}`; `serve` `doc.open`; CLI `run <file> --cmd …`                                           | `crates/automation/src/server.rs:592-593`; `rpc.rs:38-57`                      |
| Place as layer          | MCP `doc_place` / `serve` `doc.place` (scoped); `file.placeEmbedded` is refused over control                  | `server.rs:604-607`; `docs/control-protocol.md:371`                            |
| Export                  | `app.save {path}` (scoped); MCP `doc_save`/`doc_export`; `file.export.*` and `file.saveACopy` via CLI `run`   | `control.rs:1256`; `server.rs:729-741`                                         |
| Thumbnail / render      | `serve` `doc.render {index?, maxSide? (1024; 0 = full), path?}` → PNG; MCP `render_preview`; no `thumbnail` command | `crates/automation/src/rpc.rs:141-148`; `server.rs:782-783`                 |

### Drift since the study

| Item                          | Study (v0.9.0)                     | HEAD `fd28da5`                                         |
|-------------------------------|------------------------------------|--------------------------------------------------------|
| UI-only command count         | 76                                 | 79 (`menus.rs:23-107`)                                 |
| Control dispatch line range   | `control.rs:385-1250`              | `control.rs:387-1263`                                  |
| Auth, framing, MCP tool count | token, JSON lines, 25              | unchanged                                              |

---

## 3a.3 LightCraft (`lightcraft` @ `38adc17`)

### Transports

| Surface            | Detail                                                                                                                                                                                                                                                                         | Source                                                                                      |
|--------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------|
| CLI binary         | `lightcraft-cli`. Subcommands: `run [--demo\|--library DIR\|--import PATH…\|--connect [ADDR]] [--script FILE\|-] [--keep-going] CMD [k=v\|'{json}']…`, `mcp`, `render`, `snapshot`, `merge`, `synth-merge`, `denoise-serve`, `commands [--json]`, `controls [--json]`, `calibrate` | `lightcraft` `apps/lightcraft-cli/src/main.rs:28-111` @ `38adc17`                           |
| `run` output       | One JSON line per command: `{"command","ok","result"\|"error","ms"}`. `--script` reads JSON lines `{"command","params"}` or `{"method",…}`                                                                                                                                       | `apps/lightcraft-cli/src/main.rs:32-46`                                                     |
| `commands --json`  | Pretty JSON array from `engine.commands` on a demo headless session: `{id, label, menu, shortcut, params:"<prose>", enabled, disabled_reason?}` plus a synthetic `app.export` entry                                                                                            | `apps/lightcraft-cli/src/main.rs:886-906`; `crates/engine/src/cmd/mod.rs:66-91`; `crates/mcp/src/headless.rs:135-140` |
| MCP                | Hand-written JSON-RPC 2.0 over stdio, protocol `2025-06-18` (also `2025-03-26`, `2024-11-05`). `lightcraft-cli mcp` (headless) or `mcp --connect [ADDR]` (default `127.0.0.1:7980`) bridges to the control channel                                                         | `crates/mcp/src/server.rs:16-18`; `crates/mcp/src/lib.rs:31` @ `38adc17`                    |
| MCP tool mapping   | 17 helpers + 11 connect-only + **one generated tool per command**, named `cmd_<id with . → _>` (e.g. `develop.set` → `cmd_develop_set`), unless `--compact`. Generated tools take free-form params (`additionalProperties: true`)                                              | `crates/mcp/src/tools.rs:14`, `:78-336`, `:337-368`                                         |
| MCP resources      | `lightcraft://document`, `://commands`, `://library`, `://photos`, `://photo/active`, `://develop/active`, `://controls`                                                                                                                                                       | `crates/mcp/src/server.rs:35-41`                                                            |
| `--control`        | `lightcraft --control PORT` or `LIGHTCRAFT_CONTROL_PORT`. A bad port is logged and ignored (no control server, no non-zero exit)                                                                                                                                              | `apps/lightcraft/src/main.rs:672-673`, `:723-732`, `:748-756`, `:887-889`                   |
| Framing            | JSON lines, `{"id","method","params"}` → `{"id","ok","result"\|"error"}`. Request-only: a non-request line gets one error and the connection closes (anti-HTTP smuggling). 4 MiB lines, 16 connections, 5 s no-frame give-up, 60 s timeout                                     | `apps/lightcraft/src/control_server.rs:22-28`; `docs/control-protocol.md:1-29`              |
| Control methods    | `engine.execute` (aliases `ui.menu.invoke`, `command`), `engine.commands`, `ui.menu.list`, `ui.menu.tree`, `ui.inspect`, `ui.widgets`, `ui.clickWidget`/`dragWidget`/`hoverWidget`, `ui.move`, `ui.click`/`drag`, `ui.pointer`, `ui.key`, `ui.text`, `ui.clipboard`, `ui.scroll`, `ui.zoom`, `ui.set`, `ui.dialog.confirm`/`cancel`, `ui.resize`, `ui.zoomFactor`, `ui.screenshot`, `ui.render`, `app.quit` | `crates/ui-egui/src/control.rs:201-458` @ `38adc17` |

### Auth

- **None** on `--control`: "The port has no authentication, so only enable it when you need it" (`docs/control-protocol.md:27`). `control_server::start(port, ctx)` takes no secret (`apps/lightcraft/src/control_server.rs:26`).
- MCP stdio: none (the caller owns the process). `--connect` sends nothing extra.
- Mitigations that exist: loopback bind, request-only framing, one process per library via an exclusive OS lock on `<library>/catalog.lock` (`crates/catalog/src/lock.rs:1-19`).
- Not control auth, but nearby: `denoise-serve` has its own shared token (`--token-env`, ≥ 16 characters) for a LAN tile server (`apps/lightcraft-cli/src/main.rs:84-96`). PipelineCraft should not confuse the two.

### Command registry

- Spec: `CommandSpec {id, label, menu, shortcut, params, enabled, run, journal, explicit_targets}` (`crates/engine/src/cmd/mod.rs:47-63`). `cmd!(query …)` sets `journal: false` (`:123-130`). Registry built from 32 modules (`:133-170`). Serialized form `CommandInfo` has **no** journal field (`:66-91`).
- Id style: dotted camelCase: `library.import`, `photo.rate`, `develop.set`, `activity.list`.
- Count: ≥ 304 literal engine ids (study: 312). UI ids in `LANGUAGE_COMMANDS` + `UI_COMMANDS`: about 179 by regex (study: 170) (`crates/ui-egui/src/menus.rs:16`, `:53`). **Drift: about +9 UI ids** (the exact number needs a live run).
- `app.export` is still not an engine command; the headless MCP backend fakes it into `engine.commands` (`crates/mcp/src/headless.rs:127-153`; `crates/engine/src/cmd/export.rs:2`).
- Params schema: prose string. Effects on the wire: MCP annotations only, and **every generated `cmd_*` tool gets `destructiveHint: true`**, including queries, because `journal` isn't exported (`crates/mcp/src/tools.rs:370-388`).

### Events and long jobs

- No push events on control. MCP resources declare `subscribe: false` (study, `docs/mcp.md:206-223`; not re-read).
- Long tasks: one activity registry, `activity.list` / `activity.cancel {id|all}`, and `ui.inspect` reports `activity` (`docs/background-tasks.md:1-10`; `crates/engine/src/cmd/activity.rs:10-26`).
- MCP progress: `notifications/progress {progressToken, progress, total, message}` and `notifications/cancelled` for headless batch export (`crates/mcp/src/progress.rs:1`, `:111`, `:138-139`; `crates/mcp/src/backend.rs:18-23`).

### Asset handling for handoff

| Job                     | Surface                                                                                                                       | Source                                                         |
|-------------------------|-------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------|
| Open in running app     | `engine.execute library.import {paths:[abs…], mode?: add\|copy\|move, album?}`, then `library.select {ids, active}`; or `library.browse {path}` (in place, not added) | `crates/engine/src/cmd/library.rs:549`, `:1392`; `crates/engine/src/cmd/browse.rs:21` |
| Dry run of import       | `library.importPreview` (query)                                                                                               | `crates/engine/src/cmd/library.rs:1375`                        |
| Open in Develop         | `ui.set {"view":"detail","right":"edit"}`                                                                                     | `crates/ui-egui/src/control.rs:342`                            |
| Export                  | `app.export {path\|dir, ids?, preset?, format, longEdge…}` (desktop, MCP `export`, CLI `run`); `export.presets`               | `crates/mcp/src/headless.rs:130-153`; `crates/engine/src/cmd/export.rs:163` |
| Thumbnail / render      | control `ui.render {id?, size? (1600), path?}`; MCP `render_photo`; CLI `render <IN> -o <OUT> --size N`                       | `crates/ui-egui/src/control.rs:437-443`; `apps/lightcraft-cli/src/main.rs:54-67` |

### Drift since the study

| Item                    | Study (v0.8.0)       | HEAD `38adc17`                               |
|-------------------------|----------------------|----------------------------------------------|
| UI command count        | 170                  | ~179                                          |
| MCP progress            | not mentioned        | `crates/mcp/src/progress.rs` (export only)    |
| Auth, framing, tools    | none, JSON lines, 28 | unchanged                                     |

---

## 3a.4 FilmCraft (`filmcraft` @ `9ed9650`)

### Transports

| Surface            | Detail                                                                                                                                                                                                                                                                          | Source                                                                             |
|--------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------|
| CLI binary         | `filmcraft-cli`. Subcommands: `exec <id> [k=v…\|'<json>']`, `run <script.jsonl\|->` (`{"id","params"}` per line), `commands [filter] [--json]`, `describe <id>`, `inspect`, `import`, `export`, `render`, `probe`, `bench-decode`, `mcp`. Global `--bridge 127.0.0.1:PORT` runs any subcommand against the live app | `filmcraft` `apps/filmcraft-cli/src/main.rs:63-130` @ `9ed9650`                    |
| `commands --json`  | Output of `command.list`: `{id, label, menu, shortcut, shortcuts:[…], defaultShortcut, params:"<prose>", enabled}`                                                                                                                                                              | `crates/engine/src/commands.rs:3373-3383`                                          |
| MCP                | rmcp, stdio. `filmcraft-cli mcp [--project p.fcproj\|--demo]` headless, or `mcp --bridge 127.0.0.1:<port>`. Resources `filmcraft://document`, `filmcraft://commands`                                                                                                            | `apps/filmcraft-cli/src/main.rs:98`, `:107`; `crates/automation/src/server.rs:590-591` |
| MCP tool mapping   | 17 hand-written tools (`command_list`, `command_run`, `command_batch`, `doc_inspect`, `render_preview`, `project_inspect`, `sequence_inspect`, `media_import`, `render_frame`, `ui_*`). Registry reached through `command_run`                                                    | `crates/automation/src/server.rs:311-526` (17 `#[tool(`)                           |
| `--control`        | `filmcraft --control <port>` or `FILMCRAFT_CONTROL_PORT`; a bad port is a usage error. `--control-input-only` accepts widget input only from the control server                                                                                                                | `apps/filmcraft/src/args.rs:11-14`, `:81-87`, `:110-117`                            |
| Framing            | JSON lines, same request-only, close-on-junk policy as LightCraft. 4 MiB lines, 16 connections                                                                                                                                                                                  | `apps/filmcraft/src/control_server.rs:21-30`; `docs/control-protocol.md:1-21`       |
| Control methods    | `engine.execute`/`ui.menu.invoke`, `engine.commands`, `ui.menu.list`, `ui.inspect`, `perf.stats`, `ui.elements`, `ui.set`, `ui.panel.show`/`close`, `ui.click`/`move`, `ui.drag`, `ui.scroll`, `ui.key`, `ui.type`, `ui.timeline.hit`/`locate`, `ui.playback`, `ui.screenshot`, `ui.resize`, `ui.focus`, `app.quit` | `crates/ui-egui/src/control.rs:103-463` @ `9ed9650` |

### Auth

- **None**: "The port has no authentication, so only enable it while you use it" (`docs/control-protocol.md:16`). `control_server::start(port, ctx)` takes no secret (`apps/filmcraft/src/control_server.rs:25`). MCP bridge has none either.

### Command registry

- Spec: `CommandSpec {id, label, menu, shortcut, params, enabled, run, journal}`; `query!` sets `journal: false` (`crates/engine/src/commands.rs:22-47`). `build()` at `:1136`; `command.list` at `:3373`.
- Id style: dotted camelCase: `file.import`, `sequence.addEdit`, `file.exportMedia`, `export.queue.add`.
- Count: ≥ 644 literal engine ids (study: 702 with `camera_keys!` and local macros expanded). UI ids: 138 `uic!` literals in `crates/ui-egui/src/menus.rs:18-21` (study: 211, which included generated `window.workspace.*` ids; needs a live check).
- Params: prose string. Effects on the wire: none (`journal` not in `command.list`). MCP tools carry hand-written annotations (`read_only_hint`, `destructive_hint`, `idempotent_hint`, `open_world_hint`) (`crates/automation/src/server.rs:309`, `:334`, `:343`, `:370`).

### Events and long jobs

- No push events on control.
- Jobs: `jobs.list`, `jobs.cancel {job}` (`crates/engine/src/commands.rs:1606-1608`), and an export queue with `export.queue.add/list/start/stop/cancel/clear/move/remove/retry` (ids found in `crates/engine/src`; `export.queue.add` at `crates/engine/src/export_tools.rs:717`).
- MCP progress: `file.exportMedia` with `wait: true` runs as a job and reports `notifications/progress`, polled every 100 ms, and stops on cancel (`crates/automation/src/long_job.rs:1-66`).

### Asset handling for handoff

| Job                   | Surface                                                                                                                                        | Source                                                                    |
|-----------------------|------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------|
| Open project          | `engine.execute file.open {"path": str}` (absolute, no sandbox)                                                                                | `crates/engine/src/commands.rs:1514-1515`                                 |
| Import media          | `file.import {"paths":[str], "bin"?, "imageSequence"?, "replace"?}`; MCP `media_import`; CLI `--bridge … exec file.import paths='[…]'`         | `crates/engine/src/commands.rs:1331-1338`; `crates/automation/src/server.rs:408` |
| Export                | `file.exportMedia` (long job), `export.queue.add`, `file.exportInterchange` (OTIO / FCPXML / EDL …), CLI `export <out> --preset …`            | `crates/engine/src/commands.rs:1445`, `:1597-1600`                        |
| Frame / poster        | `file.exportFrame {target: source\|program, item\|sequence, …}`; MCP `render_frame`; CLI `render --seconds S --out f.png`                     | `crates/engine/src/keyboard.rs:556-575`; `server.rs:417`                  |
| Media info            | CLI `probe <media>` (JSON)                                                                                                                     | `apps/filmcraft-cli/src/main.rs:94-96`                                    |

### Drift since the study

None found in auth, framing, method list or MCP tool count. Line numbers match the study within a few lines.

---

## 3a.5 PdfCraft (`pdfcraft` @ `ac89a11`)

### Transports

| Surface            | Detail                                                                                                                                                                                                                                                                                                       | Source                                                                                   |
|--------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------|
| CLI binary         | `pdfcraft-cli`. Subcommands: `info`, `render`, `text`, `edit`, `combine`, `extract`, `split`, `check`, `audit`, `tools`, `run <tool> [k=v…]` / `run --script steps.json` (JSON **array**), `batch`, `ui --control FILE <method> …`, `mcp [--root DIR] [--compact]`                                           | `pdfcraft` `apps/pdfcraft-cli/src/main.rs:83-106` @ `ac89a11`                            |
| No `commands --json` | There is **no** `commands` subcommand. The nearest is `tools`: `[{name, description, read_only, command, input_schema}]` (`destructive` is left out). Registry ids are listed by the MCP/`run` tool `command_list`                                                                                         | `apps/pdfcraft-cli/src/main.rs:740-747`                                                  |
| `command_list`     | `{id, label, menu, shortcut, enabled, tool, params: <JSON Schema of the linked tool>\|null}`, filter and `enabled_only`                                                                                                                                                                                      | `crates/automation/src/lib.rs:1843-1870`                                                 |
| MCP                | Hand-written JSON-RPC 2.0 over stdio; protocols `2025-06-18`, `2025-03-26`, `2024-11-05`. Headless only (no bridge to the GUI). Resources `pdfcraft://doc/{doc}/info`, `/text`, `/page/{page}/text`, `/page/{page}/image?dpi=`                                                                              | `crates/automation/src/mcp.rs:1-26`                                                      |
| MCP tool mapping   | Exactly the 163 automation tools (`[a-z_]` names, e.g. `doc_open`, `page_render`); `--compact` lists 15 core tools plus `tool_search` / `tool_call`. `command_run {id, params}` runs a registry id only through its linked tool, else "has no headless tool"                                                 | `crates/automation/src/tools.rs:1-4`, `:117`; `crates/automation/src/conventions.rs:11-13` |
| `--control`        | `pdfcraft --control <file>` (a **file**, not a port). Binds `127.0.0.1:0` (OS-chosen port) and writes `{"port","token","pid"}` to `<file>` atomically (temp file + rename, owner-only on Unix). The app refuses to start if it can't publish                                                                 | `apps/pdfcraft/src/main.rs:16-19`, `:191`, `:373-375`, `:432-499`; `crates/ui-egui/src/control.rs:789-793` |
| Framing            | Newline-delimited **JSON-RPC 2.0**: `{"jsonrpc":"2.0","id","method","params"}` → `{"jsonrpc":"2.0","id","result"}` or `{"error":{"code","message"}}`. Codes `-32700` parse, `-32600` too long, `-32001` unauthenticated, `-32000` method error, `-32002` 30 s timeout. 4 KiB line cap before auth            | `crates/ui-egui/src/control.rs:786-857`, `:900`                                          |
| Control methods    | `ui.state`, `ui.commands`, `ui.command {id}` (no params forwarded), `ui.dock`, `ui.dock.reset`, `ui.set {key, value}`, `ui.open {path}`, `ui.inspect`, `ui.click`, `ui.focus`, `ui.resize`, `ui.quit`, `ui.move`, `ui.drag`, `ui.type`, `ui.key`, `ui.screenshot`. No `engine.execute` / `engine.commands` | `crates/ui-egui/src/control.rs:1-41`, `:279-350`                                         |
| Single instance    | Windows only: a files-only launch hands absolute paths to the running app over a Unix domain socket `%LOCALAPPDATA%\PdfCraft\instance-<SESSIONNAME>.sock` (`{"open":[…]}` → `ok`), then exits                                                                                                                | `apps/pdfcraft/src/single_instance.rs:1-21`, `:123-136`; `apps/pdfcraft/src/main.rs:216-224` |

### Auth

- Token: **128 bits** from `getrandom`, 32 hex characters (`crates/ui-egui/src/control.rs:908-915`). (PhotoCraft and WordCraft use 256 bits.)
- Storage: only in the control file that the caller names, never logged (`apps/pdfcraft/src/main.rs:439-442`). On Unix the temp file is created 0600; on Windows it inherits the folder ACL (`main.rs:459-499`).
- Wire: the first request must be `{"jsonrpc":"2.0","id":1,"method":"auth","params":{"token":"…"}}` → `{"result":{"ok":true}}`; anything else → `-32001` and close. Constant-time compare (`control.rs:831-838`, `:903-905`).
- Client: `pdfcraft-cli ui --control FILE <method>` refuses a control file that is a symlink or not owned by the user (study, `apps/pdfcraft-cli/src/main.rs:870-912`; not re-read).

### Command registry

- Two tables. **(1) Registry** `pdfcraft_engine::commands::COMMANDS`: `CommandSpec {id, label, menu: Option<&str>, shortcut, needs, in_text, icon}` with **no params and no run function** (`crates/engine/src/commands.rs:116-128`, `:163-372`). **(2) Tools** `pdfcraft_automation::tools()`: `ToolDef {name, title, description, input_schema: JSON Schema (draft 2020-12 subset), read_only, destructive, command: Option<registry id>}` (`crates/automation/src/tools.rs:10-23`, `:76-115`).
- Id style: dotted **snake_case** in the registry (`file.open_recent`, `file.save_as`, `view.layout.two_up`), and `[a-z_]` tool names (`doc_open`, `page_render`).
- Count: 192 registry ids at HEAD (study: 194; `edit.*` 22 → 20). 163 tools; 64 distinct registry ids are linked from a tool. **Drift: −2 registry ids.**
- Effects: the **best in this half of the suite**. Every tool has `read_only` and `destructive`, mapped to MCP `readOnlyHint` / `destructiveHint` / `idempotentHint` (`crates/automation/src/mcp.rs:482`). The registry itself has `needs` (enablement) only.

### Events and long jobs

- No push events. No job registry or progress in MCP (no progress code in `crates/automation/src/mcp.rs`). `ui.open` can reply `{"accepted","pending":true}`, then `ui.state.pending_opens` lists it (polling) (`crates/ui-egui/src/control.rs:39-40`).

### Asset handling for handoff

| Job                   | Surface                                                                                                         | Source                                                              |
|-----------------------|-----------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------|
| Open in running app   | control `ui.open {path}` (one path) after `auth`; or a plain files-only launch on Windows (single-instance hand-off) | `crates/ui-egui/src/control.rs:297`; `apps/pdfcraft/src/single_instance.rs:1-21` |
| Open headless         | tool `doc_open {path}` → doc id                                                                                 | `crates/automation/src/tools.rs:132`                                |
| Export                | tools `doc_save`, `doc_export_images`, `doc_export_text`, `doc_export_office`, `doc_export_all_images`; CLI `combine`, `extract`, `split` | `tools.rs:144`, `:863`                                 |
| Thumbnail / render    | tool `render_preview {max_side (1024), page?}`; `page_render` (96 dpi default); CLI `render f.pdf --page N --dpi D --out x.png` | `tools.rs:161`, `:1262`                                     |

---

## 3a.6 WordCraft (`wordcraft` @ `3292ba0`)

### Transports

| Surface            | Detail                                                                                                                                                                                                                                                   | Source                                                                       |
|--------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------|
| CLI binary         | `wordcraft-cli`. Subcommands: `convert`, `info`, `text`, `inspect`, `render <file> <out.png> [--page N] [--scale S]`, `run [--file F\|--template T] --cmd 'id={json}'… [--save OUT]`, `commands [--json]`, `parity`, `mcp`, `zotero`. No script-file mode     | `wordcraft` `apps/wordcraft-cli/src/main.rs:15-57` @ `3292ba0`               |
| `commands --json`  | `Registry::describe()`: `[{id, label, location, shortcut, params:"<prose>", mutates}]`. No `enabled`                                                                                                                                                     | `apps/wordcraft-cli/src/main.rs:411-416`; `crates/engine/src/lib.rs:120-128` |
| MCP                | Hand-written JSON-RPC 2.0 over stdio. `wordcraft-cli mcp` headless (with `--automation-read-root` / `--automation-write-root`) or `mcp --connect HOST:PORT` to the app. Resources `wordcraft://document`, `wordcraft://commands`                            | `apps/wordcraft-cli/src/main.rs:31-37`, `:240`; `crates/mcp/src/server.rs:28-29`, `:150-163` |
| MCP tool mapping   | 16 tools: `list_commands`, `execute {command, params}`, `batch`, `new_document`, `open_document`, `save_document`, `type_text`, `select_text`, `get_text`, `inspect_document`, `render_page`, `parity`, plus app-only `screenshot`, `click`, `key`, `ui_inspect`. Annotations carry `readOnlyHint` only | `crates/mcp/src/tools.rs:51-52`, `:57-147`              |
| `--control`        | `wordcraft --control <port>` or `WORDCRAFT_CONTROL_PORT`; a bad port exits 2. Binds `127.0.0.1:<port>`                                                                                                                                                 | `apps/wordcraft/src/main.rs:3-8`, `:270-274`; `apps/wordcraft/src/control_server.rs:39-46`; `docs/control-protocol.md:3-5` |
| Framing            | JSON lines with a top-level `key`: `{"id","key","method","params"}` → `{"id","ok":true,"result"}` / `{"id","ok":false,"error"}`. Bad JSON closes the connection. 1 MiB lines; 30 s to the first authorised request; 60 s reply timeout                  | `apps/wordcraft/src/control_server.rs:19-22`, `:195-245`; `docs/control-protocol.md:7-41` |
| Control methods    | `engine.execute` (alias `command`), `engine.commands {search?}`, `document.inspect`, `ui.inspect`, `ui.click`, `ui.clickText`, `ui.move`, `ui.drag`, `ui.press`, `ui.release`, `ui.wheel`, `ui.key`, `ui.text`, `ui.screenshot`, `ui.render`, `ui.parity`, `ui.resize`, `ui.focus`, `app.quit`. **Any other method name runs as a command id** | `crates/ui-egui/src/control.rs:119-300`; `docs/control-protocol.md:49-70` |

### Auth

- Key: 256 bits from `getrandom`, 64 hex characters (`apps/wordcraft/src/control_server.rs:78-82`).
- Storage: `<settings>/control-key.<instance>`, where `<instance>` is the Flatpak instance id or `local-<port>`. `<settings>` is `%APPDATA%\WordCraft` on Windows, overridable by `WORDCRAFT_PROFILE_DIR` (`crates/control-key/src/lib.rs:1-49`). A stale file is replaced; 0600 on Unix only; removed when the window closes (`control_server.rs:84-97`; `apps/wordcraft/src/main.rs:7-8`).
- Wire: top-level `"key"` on every request, **or** `{"method":"auth","key":"…"}` (or `params.key` for `auth` only) once per connection → `{"ok":true,"result":{}}`. A wrong or missing key → `{"ok":false,"error":"unauthorized"}` and close. Checked on every request (`control_server.rs:212-235`).
- MCP `--connect` reads `WORDCRAFT_CONTROL_KEY` or the key file again on every reconnect, and sends it only to loopback addresses (`docs/control-protocol.md:42-47`).

### Command registry

- Spec: `CommandSpec {id, label, location, shortcut, params: &str, mutates, enabled, run}`, built with `CommandSpec::new(…).key().params().pure().when()` (`crates/engine/src/lib.rs:58-100`). `Registry {get, all, describe, by_shortcut}` (`:103-133`).
- Id style: dotted camelCase: `file.open`, `file.exportPng`, `para.set`, `table.insert`.
- Count: 629 `CommandSpec::new` ids at HEAD, 234 of them `.pure()` (study: 617). Largest namespaces: `para` 50, `table` 48, `insert` 44, `format` 39. **Drift: +12 ids** (for example `insert` 40 → 44, `mailings` 17 → 19, new `link.*`).
- Effects: `mutates` (false = `.pure()`) is the only app here that **publishes** an effects flag in `commands --json`. Caveat: it means "changes the document", not "touches user files": `file.save`, `file.saveAs` and `file.open` are all `.pure()` (`crates/engine/src/cmd/file.rs:68-76`). The control `engine.commands` reply leaves `mutates` out and adds `enabled` instead (`crates/ui-egui/src/control.rs:135-149`), so the two lists differ.
- Params: prose string.

### Events and long jobs

- No push events, no job registry, and MCP ignores `notifications/progress` (`crates/mcp/src/server.rs:122-123`).

### Asset handling for handoff

| Job                   | Surface                                                                                                                         | Source                                                                        |
|-----------------------|---------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------|
| Open in running app   | `engine.execute file.open {"path", "data"?: base64, "password"?}`. **Still skips the unsaved-changes prompt** (see O5)          | `crates/engine/src/cmd/file.rs:68-71`; `crates/ui-egui/src/control.rs:124-133` |
| Open headless         | MCP `open_document {path}`; CLI `run --file F`                                                                                  | `crates/mcp/src/tools.rs:57-147`                                              |
| Export                | `file.exportPdf {path, markup?}`, `file.save`/`file.saveAs {path}` (format by extension), CLI `convert <in> <out>`              | `crates/engine/src/cmd/file.rs:72-91`                                         |
| Thumbnail / render    | `file.exportPng {path, page?, scale?}`; control `ui.render {path?, page?, scale?}` (base64 PNG when no path); MCP `render_page`; CLI `render` | `crates/engine/src/cmd/file.rs:91`; `docs/control-protocol.md:66`      |

### Drift since the study

| Item                              | Study (v0.7.0)                         | HEAD `3292ba0`                                                                                 |
|-----------------------------------|----------------------------------------|------------------------------------------------------------------------------------------------|
| Engine command count              | 617                                    | 629                                                                                            |
| Key file name                     | `control-key.local-<port>`             | `control-key.<instance>` (Flatpak id or `local-<port>`); `WORDCRAFT_PROFILE_DIR` override       |
| Control `file.close` on dirty doc | not guarded                            | refused unless `{"force": true}` (`crates/ui-egui/src/control.rs:110-114`)                      |
| Control `file.open` on dirty doc  | replaces silently                      | **unchanged**: control calls `WordApp::execute`, and only `WordApp::run` checks `discards_document` (`crates/ui-egui/src/lib.rs:744-760`, `:802`, `:2355-2363`) |
| Workspace deps                    | path only, no version                  | unchanged (`Cargo.toml:15-30`)                                                                 |

---

## 3a.7 Shared code (craft-libs @ `8f5a283`)

- craft-libs has seven crates: `atomic`, `cms`, `pdf`, `print`, `quit`, `settings`, `ui` (`craft-libs` `Cargo.toml:23-34` @ `8f5a283`). **None of them is a control, command-registry or MCP crate.** The only control-related text is a note that `craft-ui` docking `Action` serializes for control-channel requests (`docs/ui-docking.md:62`), and that it does not infer an app command registry (`docs/ui-docking.md:138`).
- What the five apps take from craft-libs: `craft-ui` (all five; pinned to rev `5c874d5` in photocraft, lightcraft, filmcraft and pdfcraft) and `craft-print` (photocraft, pdfcraft, wordcraft) (`photocraft` `Cargo.toml:13`, `:41`; `lightcraft` `Cargo.toml:13`; `filmcraft` `Cargo.toml:76`; `pdfcraft` `Cargo.toml:16-18`; `wordcraft` `crates/ui-egui/Cargo.toml:12`, `apps/wordcraft/Cargo.toml:71`).
- Everything PipelineCraft talks to is a **per-app copy**: each app has its own `control_server.rs`, its own `control.rs` dispatcher, its own MCP server (rmcp in PhotoCraft and FilmCraft; hand-written JSON-RPC in LightCraft, PdfCraft and WordCraft) and its own token code. LightCraft's and FilmCraft's `control_server.rs` look like siblings (same `MAX_LINE = 4 MiB`, `MAX_CONNECTIONS = 16`, request-only policy; `lightcraft` `apps/lightcraft/src/control_server.rs:22-24`, `filmcraft` `apps/filmcraft/src/control_server.rs:21-23`), but they are copies, not a shared crate.

---

## 3a.8 Cross-app comparison

| Aspect              | PhotoCraft                                   | LightCraft                                  | FilmCraft                                     | PdfCraft                                        | WordCraft                                       |
|---------------------|----------------------------------------------|---------------------------------------------|-----------------------------------------------|-------------------------------------------------|-------------------------------------------------|
| Control flag        | `--control <port>`                           | `--control <port>`                          | `--control <port>`                            | `--control <file>` (random port)                | `--control <port>`                              |
| Bad port            | exit 2                                       | logged, no server                           | usage error                                   | n/a                                             | exit 2                                          |
| Framing             | JSON lines `{id,method,params}`→`{id,ok,…}`  | same, request-only                          | same, request-only                            | JSON-RPC 2.0 lines                              | JSON lines + top-level `key`                    |
| Auth                | 256-bit token, `auth` first frame            | none                                        | none                                          | 128-bit token from control file, `auth` first   | 256-bit key file, every request or `auth` once  |
| Secret location     | `--control-token-file` path / env / stderr   | n/a                                         | n/a                                           | caller-named control file `{port,token,pid}`    | `%APPDATA%\WordCraft\control-key.local-<port>`  |
| Generic execute     | `engine.execute {command, params, wait?}`    | `engine.execute {command, params}`          | `engine.execute {command, params}`            | `ui.command {id}` (no params)                   | `engine.execute {command, params}`              |
| Command list        | `engine.commands`                            | `engine.commands`                           | `engine.commands`                             | `ui.commands`                                   | `engine.commands {search?}`                     |
| CLI list            | `commands --json`                            | `commands --json`                           | `commands --json`                             | `tools` (no `commands`)                         | `commands --json`                               |
| Id style            | dotted camelCase                             | dotted camelCase                            | dotted camelCase                              | dotted snake_case (+ `[a-z_]` tools)            | dotted camelCase                                |
| Params schema       | prose                                        | prose                                       | prose                                         | **JSON Schema** (tools)                         | prose                                           |
| Effects metadata    | `journal` (internal only)                    | `journal` (internal only)                   | `journal` (internal only)                     | `read_only` + `destructive` per tool            | `mutates` (in CLI list only; doc-state meaning) |
| MCP start           | `mcp` / `mcp --bridge ADDR` + token          | `mcp` / `mcp --connect [ADDR]`              | `mcp` / `mcp --bridge ADDR`                   | `mcp [--root] [--compact]` (headless only)      | `mcp` / `mcp --connect HOST:PORT`               |
| MCP tools           | 25 generic                                   | 28 + `cmd_*` per command                    | 17 generic                                    | 163 typed (+ compact mode)                      | 16 generic                                      |
| Push events         | none                                         | none                                        | none                                          | none                                            | none                                            |
| Progress            | `jobs.list` poll; MCP progress (renderVideo) | `activity.list` poll; MCP progress (export) | `jobs.list` / `export.queue.*`; MCP progress  | none                                            | none                                            |
| Open-file command   | `app.open {path}` (relative to read root)    | `library.import {paths}` (+ `library.select`) | `file.open {path}` / `file.import {paths}`  | `ui.open {path}`                                | `file.open {path}` (no save prompt)             |
| Export command      | `app.save {path}`; CLI `run --cmd file.*`    | `app.export {…}`                            | `file.exportMedia`, `file.exportInterchange`  | tools `doc_save`, `doc_export_*`                | `file.saveAs`, `file.exportPdf`                 |
| Render-to-file      | `serve` `doc.render {maxSide, path}`         | `ui.render {id, size, path}`                | `file.exportFrame`; CLI `render`              | tool `render_preview {max_side}`                | `file.exportPng`; `ui.render`                   |
| Single instance     | macOS Apple events only                      | one process per library (lock)              | none                                          | Windows UDS hand-off                            | none                                            |

---

## 3a.9 Convergence opportunities

Each row says what the hub can normalize on its own side (adapter config) and what is worth a small upstream PR. The hub-side fix always comes first, so PipelineCraft never waits on upstream.

| ID  | Opportunity                                                                                                                                                       | Scope / goal it serves              | Suggested fix                                                                                                                                                                                                                         | Owner                               | Status |
|-----|-------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------|--------|
| O1  | Three framings: JSON lines `{id,ok}` (PhotoCraft, LightCraft, FilmCraft), JSON lines with `key` (WordCraft), JSON-RPC 2.0 (PdfCraft)                               | §4 adapters, §6 protocol            | Hub: one `control` adapter with two codecs, `suite-jsonl` and `jsonrpc2`, chosen per app in config. The hub's own front door speaks `suite-jsonl` (D3: mirror `engine.execute`)                                                         | Claude (hub)                        | open   |
| O2  | Five auth models; LightCraft and FilmCraft have none                                                                                                              | §11 security                        | Hub: per-app auth strategy in config (`token-frame`, `token-file-jsonrpc`, `key-file`, `none`). Upstream: small PR to LightCraft and FilmCraft adding PhotoCraft's `--control-token-file` + `auth` frame (code is in `photocraft` `crates/automation/src/security.rs:74-250`) | Claude (hub); LightCraft, FilmCraft | open   |
| O3  | No app publishes its chosen port; only PdfCraft writes a discovery file (`{port,token,pid}`)                                                                      | §5 discovery                        | Hub: launch apps with explicit ports it assigns and remembers. Upstream (D6 candidate): adopt PdfCraft's control-file shape as the suite manifest: `--control-file <path>` writing `{app, version, pid, port, token, protocol}`         | Claude (hub); craft-libs / each app | open   |
| O4  | Effects metadata exists internally (`journal`, `mutates`, `read_only`/`destructive`) but only PdfCraft publishes a usable form; LightCraft's generated MCP tools all say `destructiveHint: true` | §6.4 effects, §10.2 MCP hints | Hub: seed an effects table per app from what is published (PdfCraft tools, WordCraft `mutates`) and hand-curate the rest. Upstream: one-line PR per app to emit `"readOnly": !journal` in `command.list` (PhotoCraft `commands.rs:146-149`, LightCraft `cmd/mod.rs:66-91`, FilmCraft `commands.rs:3379`) | Claude (hub); each app | open |
| O5  | WordCraft control `file.open` still replaces a dirty document with no prompt (study O5, still open at HEAD); `file.close` is now guarded                           | §9.1 Open In WordCraft              | Hub: mark WordCraft `file.open` over control as `destructive` and refuse it while `document.inspect` reports dirty, unless the caller confirms. Upstream: route control `file.open` through `discards_document` as `file.close` already is | Claude (hub); WordCraft             | open   |
| O6  | PdfCraft control can't run registry commands with params (`ui.command {id}` only) and has no `engine.execute`                                                     | §6 protocol, §9.5 PrepressCraft     | Hub: for PdfCraft, route writes to a headless `pdfcraft-cli mcp` / `run` adapter and use control only for `ui.open` and `ui.state`. Upstream: an `engine.execute {command, params}` alias that runs the linked tool                     | Claude (hub); PdfCraft              | open   |
| O7  | PhotoCraft accepts `--control 0` but prints `127.0.0.1:0`, so the OS-chosen port is lost                                                                          | §5 discovery                        | Hub: never pass 0 to PhotoCraft. Upstream: print `listener.local_addr()` (`apps/photocraft/src/control_server.rs:27`), or write it to the control file from O3                                                                         | Claude (hub); PhotoCraft            | open   |
| O8  | No push events anywhere; progress is polled (`jobs.list`, `activity.list`, `export.queue.list`) or comes as MCP progress for one command per app                  | §8 events and jobs                  | Hub: a poller per app that turns job lists into hub events (`job.progress`, `job.done`). Upstream later: an opt-in `events.subscribe` on control, after the hub proves the event shape                                                 | Claude (hub); later each app        | open   |
| O9  | Command lists differ in keys (`disabledReason` vs `disabled_reason`; `menu` array vs `location` string vs `Option<&str>`; WordCraft CLI has `mutates` but no `enabled`, control has the reverse) | §6.3 registry merge | Hub: normalize to one `CommandInfo {app, id, label, menuPath[], shortcut, paramsDoc, paramsSchema?, enabled, disabledReason?, effects}` in the adapter. Upstream: none needed                                     | Claude (hub)                        | open   |
| O10 | PdfCraft has no `commands --json`, and its `tools` output omits `destructive`                                                                                    | §13.2 contract tests                | Hub contract tests call `pdfcraft-cli tools` and MCP `command_list`. Upstream: add `destructive` to `tools` output (`apps/pdfcraft-cli/src/main.rs:743`)                                                                              | Claude (hub); PdfCraft              | open   |
| O11 | Bridge flag names differ: `--bridge` (PhotoCraft, FilmCraft) vs `--connect` (LightCraft, WordCraft); PdfCraft MCP can't bridge                                    | §4 adapters                         | Hub: per-app `mcp_bridge_flag` in config. Not worth an upstream PR on its own                                                                                                                                                       | Claude (hub)                        | open   |
| O12 | Token files are 0600 on Unix only; on Windows they inherit the folder ACL (PhotoCraft, PdfCraft, WordCraft)                                                       | §11 security on Paul's machine      | Hub: keep its own copies of secrets in the OS keyring and place any token files it asks apps to create under `%LOCALAPPDATA%\PipelineCraft\` with a user-only ACL. Check the default ACLs live (Not verified #6)                       | Claude (hub); Paul (verify)         | open   |
| O13 | Path rules differ: PhotoCraft needs relative paths under `--automation-read-root`; LightCraft, FilmCraft, PdfCraft, WordCraft take absolute paths                  | §7.4 path rules                     | Hub: the PhotoCraft adapter launches with the library root as read root and rewrites absolute asset paths to root-relative ones; refuses assets outside the root with a clear error                                                   | Claude (hub)                        | open   |
| O14 | No `thumbnail {path, maxSide}` command in any of the five (the study's O8 still holds)                                                                            | §9.2 Thumbnails                     | Hub: a per-app recipe (open → render → close) in adapter config, using the render rows in §3a.8. Upstream: propose one suite command id, `thumbnail.render {path, maxSide, out}`, starting with the app that needs it first           | Claude (hub); each app              | open   |

---

## Not verified

Everything above is a static read. These need a live run on Paul's Windows machine. Build each CLI once (`cargo build --release -p <app>-cli`) before step 1.

**Next action (Paul)**

1. Live command counts (confirms the counts in §3a.1).
   ```powershell
   Set-Location F:\Script-Projects\ArtCrafting
   foreach ($a in 'photocraft','lightcraft','filmcraft','wordcraft') {
     Push-Location ".\$a"
     $n = (cargo run -q --release -p "$a-cli" -- commands --json | ConvertFrom-Json).Count
     "$a commands: $n"
     Pop-Location
   }
   ```
   Expected: PhotoCraft about 863 (engine only), LightCraft about 313 (engine + synthetic `app.export`), FilmCraft about 702, WordCraft about 629.
2. PdfCraft tools.
   ```powershell
   Set-Location F:\Script-Projects\ArtCrafting\pdfcraft
   (cargo run -q --release -p pdfcraft-cli -- tools | ConvertFrom-Json).Count
   ```
   Expected: 163.
3. PhotoCraft auth handshake.
   ```powershell
   Set-Location F:\Script-Projects\ArtCrafting\photocraft
   Start-Process .\target\release\photocraft.exe -ArgumentList '--control','7878','--control-token-file',"$env:LOCALAPPDATA\PipelineCraft\pc.token"
   Start-Sleep 5
   $t = (Get-Content "$env:LOCALAPPDATA\PipelineCraft\pc.token").Trim()
   $c = [Net.Sockets.TcpClient]::new('127.0.0.1',7878); $s = $c.GetStream()
   $w = [IO.StreamWriter]::new($s); $r = [IO.StreamReader]::new($s); $w.AutoFlush = $true
   $w.WriteLine("{`"id`":`"auth`",`"method`":`"auth`",`"params`":{`"token`":`"$t`"}}"); $r.ReadLine()
   $w.WriteLine('{"id":2,"method":"engine.commands"}'); $r.ReadLine().Length
   ```
   Expected: `{"id":"auth","ok":true,"result":{"authenticated":true}}`, then a long reply.
4. LightCraft and FilmCraft have no auth. Same script against `lightcraft.exe --control 7980` and `filmcraft.exe --control 9876` without the `auth` line. Expected: `engine.commands` answers directly.
5. WordCraft key file and the O5 behaviour.
   ```powershell
   Set-Location F:\Script-Projects\ArtCrafting\wordcraft
   Start-Process .\target\release\wordcraft.exe -ArgumentList '--control','7981'
   Start-Sleep 5
   Get-Content "$env:APPDATA\WordCraft\control-key.local-7981"
   ```
   Expected: 64 hex characters. Then type into the document, and send `{"id":1,"key":"<k>","method":"engine.execute","params":{"command":"file.open","params":{"path":"F:\\Script-Projects\\ArtCrafting\\assetcraft\\test_data\\library\\<any>.docx"}}}`. Expected (O5): the typed text is lost with no prompt.
6. Token-file ACLs on Windows (O12).
   ```powershell
   Set-Location F:\Script-Projects\ArtCrafting
   icacls "$env:LOCALAPPDATA\PipelineCraft\pc.token"
   icacls "$env:APPDATA\WordCraft\control-key.local-7981"
   ```
   Expected: only your user, SYSTEM and Administrators. Any other principal is an O12 finding.
7. PhotoCraft port 0 (O7): `photocraft.exe --control 0` should print `listening on 127.0.0.1:0`.
8. PdfCraft control file: `pdfcraft.exe --control "$env:LOCALAPPDATA\PipelineCraft\pdf.json"`, then `Get-Content` it. Expected: `{"port":<n>,"token":"<32 hex>","pid":<n>}`.

## Open questions

1. **Effects source of truth.** Should the hub's effects table come from upstream (O4 PRs, so it's always accurate) or stay hand-curated hub config (so it ships now)? Recommendation: both. Curate now, and replace rows as upstream PRs land. Paul decides how many upstream PRs to send (ties to D6).
2. **PdfCraft write path.** Is it acceptable for the hub to drive PdfCraft writes through a headless `pdfcraft-cli mcp` process (a separate session from the open window) rather than the GUI (O6)? Recommendation: yes for batch and recipes. The GUI is used only for Open In.
3. **Unauthenticated apps.** Should the hub refuse to attach to LightCraft and FilmCraft control ports until O2 lands, or attach with a warning? Recommendation: attach only to ports the hub itself launched (so it knows the process is the app), and log it. A8 (§11) should decide.
4. The UI-command counts for LightCraft (~179) and FilmCraft (138 vs study 211) disagree enough that the study and this pass may be counting different things. The live `engine.commands` count over control (with UI ids) settles it.

## Inputs for downstream agents

- **A3 (architecture, discovery):** no app writes a manifest except PdfCraft's control file (`{port, token, pid}`). Discovery v1 must be launch-and-remember. PdfCraft needs a file path, not a port. LightCraft refuses a second process per library (`catalog.lock`). PdfCraft on Windows hands a plain files-only launch to the running instance (avoid that when the hub wants its own instance: pass `--control` or `--new-window`).
- **A4 (protocol):** adapters need two codecs (O1), five auth strategies (O2), and a normalized `CommandInfo` (O9). `wait: false` → `{job, pending: true}` exists only in PhotoCraft. Error replies are a plain string (`error`) in four apps and `{code, message}` in PdfCraft. Timeout semantics: PhotoCraft distinguishes `timeout: not run` (safe to retry) from `timeout: no reply in 60s` (may have applied). LightCraft's 5 s no-frame give-up is also "never ran".
- **A5 (asset refs):** open-file commands differ in id and params (§3a.8 row "Open-file command"). PhotoCraft needs root-relative paths (O13). WordCraft `file.open` takes optional base64 `data`, the only byte-level handoff among the five. FilmCraft `file.import` takes `paths[]` and a `bin`.
- **A6 (events, jobs):** no push events in any of the five. Pollable job lists are `jobs.list` (PhotoCraft, FilmCraft), `activity.list` (LightCraft) and `export.queue.list` (FilmCraft). MCP `notifications/progress` exists in PhotoCraft (renderVideo), LightCraft (headless export) and FilmCraft (`file.exportMedia`).
- **A7 (MCP front door):** three MCP styles: generic (PhotoCraft, FilmCraft, WordCraft), per-command (LightCraft `cmd_*`), and typed tools with schemas (PdfCraft). Only PdfCraft's hints are trustworthy. LightCraft marks every generated tool destructive.
- **A8 (security):** LightCraft and FilmCraft are unauthenticated. Token sizes are 256 bits (PhotoCraft, WordCraft) and 128 bits (PdfCraft). Unix-only 0600 (O12). PhotoCraft prints the token to stderr when no file is given, so the hub should always pass `--control-token-file`.
- **A9 (upstream):** the smallest useful PRs, in order: O4 (`readOnly` in `command.list`, three one-line PRs), O2 (token handshake for LightCraft/FilmCraft), O5 (WordCraft open guard), O3 (control-file manifest), O7, O10.
- **Merge note for the integrator:** this file uses O1–O14 locally; renumber when merging with §3b.
