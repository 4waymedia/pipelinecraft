# §3 Current landscape, part b: DesignCraft, EffectCraft, VectorCraft, SoundCraft, ArtCraft, craft-libs, AssetCraft

_2026-10-15 · spec draft v0 · status: draft (agent A2b)_

- **Scope:** the local-control surfaces (`<app>-cli`, MCP over stdio, `--control`) of DesignCraft, EffectCraft, VectorCraft, SoundCraft and ArtCraft; the shared `craft-libs` crates; and AssetCraft's own command surface today and as planned (§3.4). PhotoCraft, LightCraft, FilmCraft, PdfCraft and WordCraft are in part a (sibling agent).
- **Project goal it serves:** PipelineCraft's hub contract (§6) has to sit on what the apps already expose, so that AssetCraft → StoryCraft → FilmCraft handoffs work without the cloud. This section is the factual base for the adapters (§4.4) and the protocol (§6).
- **Immediate area of improvement:** four of the five apps share one control envelope, but they differ on auth, port defaults, catalog shape, id style and effects metadata. There's no shared control crate. ArtCraft has no local control at all. The `--json` discovery call that §3.5 relies on silently returns nothing in two apps.

> Everything here comes from a **static read** of shallow clones. Nothing was built or run. Counts are regex counts over the source and can be off by a few ids. The authoritative numbers come from the live runs in **Not verified**.

## 3.0 Sources and drift

| Repo          | HEAD read  | Date       | Previous study (`assetcraft` `docs/sibling-apis/`) | Drift since that study                                                                                  |
|---------------|------------|------------|-----------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| `designcraft` | `7ed5697`  | 2026-10-14 | v0.8.0 `11a51d5`                                    | Engine commands 459 → 463; UI commands 106 → 134; MCP tools 24 → 25; MCP gained `--automation-read-root/-write-root`; CLI line numbers shifted |
| `effectcraft` | `2c08f67`  | 2026-10-14 | v0.10.0 `05955c0`                                   | MCP tools 33 → 36; key command lines (`file.open`, `renderQueue.*`) unchanged; regex count 728 (study said 731)        |
| `vectorcraft` | `0946bc5`  | 2026-10-15 | v0.11.0 `0a33cff`                                   | No surface drift found: 727 `cmd!` + 16 `style_cmds!` = 743, 285 UI commands, 32 tools                         |
| `soundcraft`  | `3a28a65`  | 2026-10-14 | v0.7.0 `3a28a65`                                    | None (same commit)                                                                                       |
| `artcraft`    | `d4e4ac6`  | 2026-10-11 | not studied                                         | First read                                                                                               |
| `craft-libs`  | `8f5a283`  | 2026-10-15 | not studied                                         | First read                                                                                               |
| `assetcraft`  | `3a41b2b`  | 2026-10-14 | n/a                                                 | Stage 1 steps 1–8 done (`STATUS.md:5`)                                                                   |

The clones are shallow, so no tags are visible. Cites below are `repo` `path:line` @ the commit above.

---

## 3.1b Per-app inventory

### DesignCraft (InDesign-style layout), `7ed5697`, workspace v0.8.0

**1. Transports**

| Transport | Detail | Cite |
|---|---|---|
| CLI binary | `designcraft-cli`. Subcommands: `run`, `commands [FILTER]`, `describe ID`, `script [FILE\|-] [--connect PORT]`, `app [--port P] CMD [JSON]` / `app --method M [JSON]`, `mcp`, `perf`, `bench`, `validate-pdf`, `links`, `--version` | `apps/designcraft-cli/src/main.rs:1-19`, dispatch `:92-138` |
| `commands` shape | Always pretty JSON (no `--json` flag): engine catalog only, `[{id, label, menu: [str], shortcut, params, enabled, disabled_reason?}]`. **`commands --json` treats `--json` as FILTER and prints `[]`** | `main.rs:101-111`; `crates/engine/src/cmd/mod.rs:85-108` |
| MCP stdio | `designcraft-cli mcp [--connect PORT\|HOST:PORT] [--sample] [--automation-read-root D] [--automation-write-root D]`. Hand-written JSON-RPC 2.0, protocol `2025-06-18` (+`2025-03-26`, `2024-11-05`). Backends: `Headless` (in-process `Session`) or `Remote` (the control channel). The roots apply only headless | `main.rs:166`; `crates/mcp/src/lib.rs:1-14`; `crates/mcp/src/server.rs:11-12` |
| MCP tools | 25: `list_commands`, `execute`, `batch` (the generic registry mirror), plus curated `inspect_document`, `get_story`, `set_story_text`, `new_document`, `open_document`, `save_document`, `place_image`, `render_page`, `export_png`, `select_tool`, `pointer`, `key`, `type_text`, and app-only `screenshot`, `click`, `drag`, `menu_list`, `ui_inspect`, `ui_set`, `dialog_set/confirm/cancel`. Snake_case. Does **not** implement the suite `command_*` convention (see O-B4) | `crates/mcp/src/tools.rs:83-391`, `call_tool :528-610` |
| `--control` | `designcraft --control <port>` or `DESIGNCRAFT_CONTROL_PORT`. A bad value is logged and ignored. Binds `127.0.0.1:<port>`. NDJSON. Request `{"id","method","params"}` → `{"id","ok":true,"result"}` / `{"id","ok":false,"error"}`. 4 MiB lines, 16 connections, 60 s reply timeout. Request-only guard: a non-request line gets one error and the connection closes | `apps/designcraft/src/main.rs:231-252, 347-349`; `apps/designcraft/src/control_server.rs:20-24, 26-37, 80-107, 132-157` |
| Control methods | `engine.execute`/`ui.menu.invoke`/`command {command, params}`, `engine.commands`, `document.inspect`, `ui.inspect`, `ui.menu.list`, `ui.tool.select/list`, `ui.pointer`, `ui.key`, `ui.move`, `ui.click/drag`, `ui.text`, `ui.set`, `ui.dialog.*`, `ui.resize`, `ui.focus`, `ui.screenshot`, `ui.render`, `app.open`, `app.save`, `app.export`, `app.quit` | `crates/ui-egui/src/control.rs:146-401` |
| Default port | 7979 (`DEFAULT_PORT` in the MCP crate; `app` subcommand default) | `crates/mcp/src/lib.rs:29`; `apps/designcraft-cli/src/main.rs:472` |

**2. Auth:** none. "The port has no authentication, so only enable it while you use it" (`docs/control-protocol.md:26-27`). The only protection is the request-only line guard (`control_server.rs:80-107`) against cross-origin `fetch` smuggling. MCP is stdio, so it has no auth either, and its file confinement applies only headless (`main.rs` USAGE, `:39`).

**3. Command registry:** `CommandSpec {id, label, menu: &[&str], shortcut, params: &'static str, enabled, run, journal, undoable}` (`crates/engine/src/cmd/mod.rs:71-82`). The `cmd!` forms are plain (journal + undo), `query` (neither) and `noundo` (journal only) (`:137-148`). `command_specs()` is at `:150`.
- **Count:** 463 engine ids, from a regex over `cmd!(… "id"` in `crates/engine/src`. The largest namespaces are `object` 56, `table` 35, `file` 29, `type` 28, `style` 21, `layout` 21.
- **UI commands:** 134 more in `UI_COMMANDS`, a tuple table (`crates/ui-egui/src/menus.rs:8-224`). `engine.commands` returns them with `"ui": true` (`control.rs:63-69`).
- **Id style:** `namespace.camelCase` (`file.exportPdf`, `file.saveACopy`).
- **Params:** free-text doc strings, with no JSON Schema.
- **Effects:** only `journal`/`undoable`. Nothing says "writes user file" or "network". MCP annotations carry `readOnlyHint` per tool (`tools.rs:70-78`).

**4. Events/progress:** none. The control channel is request/reply only. MCP ignores `notifications/progress`/`cancelled` (`crates/mcp/src/server.rs:124`).

**5. Handoff commands**

| Need | Command | Cite |
|---|---|---|
| Open | `file.open {path}` (.designcraft / .idml) → `{index, documentFonts, warnings}`; control `app.open {path}` wraps it | `crates/engine/src/cmd/file.rs:20-22`; `control.rs:398` |
| Place asset | `file.place {path\|base64, name?, frame?, pdfPage?, …}`; MCP `place_image` | `cmd/file.rs:102` |
| Export | `file.exportPdf {path?, pages?, standard?: x4\|a2b, …}` → `{path, bytes, pages, warnings}`; `file.exportIdml`, `file.exportEpub`, `file.exportHtml`, `file.package`; `file.serialize` → base64 | `cmd/export.rs:11-13`; `cmd/interchange.rs:12`; `cmd/file.rs:49` |
| Render to file | **No engine command.** Control `ui.render {path?, page?, scale?, bleed?}`; `app.export` → UI-only `app.exportPng`; MCP `render_page` / `export_png`; CLI `run --in F --export out.png` | `control.rs:372, 400`; `menus.rs:909`; `tools.rs:478, 563-564`; `main.rs:247` |

### EffectCraft (After Effects-style motion), `2c08f67`, workspace v0.10.0

**1. Transports**

| Transport | Detail | Cite |
|---|---|---|
| CLI binary | `effectcraft-cli`. Subcommands: `info`, `commands [--filter] [--enabled]`, `exec <id> [--params JSON]` / `exec --list [--schemas]`, `run <id> <json>…`, `props`/`get`/`set`, `render-frame`, `render`, `bench`, `script FILE.jsx`, `mcp [--autosave\|--bridge PORT]`. Global `--json`, `--project`, `--bridge PORT` (one-shot commands against a running GUI) | `apps/effectcraft-cli/src/main.rs:1-38`, dispatch `:443-604` |
| `commands --json` shape | Compact JSON `[{id, label, menu, shortcut, params, enabled, why, schema?}]`. `--schemas` adds a derived JSON Schema. The disabled-reason key is **`why`**, not `disabled_reason` | `main.rs:485, 790-806`; `crates/engine/src/commands/query.rs:9-30` |
| MCP stdio | `effectcraft-cli mcp` (headless, empty project) or `mcp --bridge PORT` (GUI). Read and write roots confine file access headless | `main.rs:604`; `docs/mcp.md:1-17` |
| MCP tools | 36. Legacy: `list_commands`, `describe_command`, `execute_command`, `run_script`, `get_*`, `set_property`, `render_frame`, `open_project`, `save_project`, `undo/redo/history`, `batch`, bridge-only `ui_*`/`screenshot`/`control`. **Suite convention:** `command_list`, `command_run`, `command_batch`, `doc_inspect`, `render_preview` ("the same core conventions as FilmCraft #28"). Resources: `effectcraft://document`, `effectcraft://commands` | `crates/automation/src/tools.rs:544-973`; `docs/mcp.md:19-31, 44-47` |
| `--control` | `effectcraft --control <port>` or `EFFECTCRAFT_CONTROL_PORT`. Port must be 1–65535; a bad port **exits 1** before the window opens. Loopback NDJSON with the same envelope and guard as DesignCraft (4 MiB, 16 connections, 60 s). The window doesn't take focus, and crash recovery is skipped | `apps/effectcraft/src/main.rs:76-117, 168-169, 217`; `apps/effectcraft/src/control_server.rs:19-37, 58-64, 79`; `docs/control-protocol.md:5-26` |
| Control methods | `engine.execute`/`ui.menu.invoke`, `engine.commands {filter?, enabledOnly?}`, `ui.menu.list`, `ui.inspect`, `ui.elements`, `ui.set`, `ui.panel.show/close`, `ui.click/move/drag/scroll/key/type`, `ui.viewer.locate/hit`, `ui.timeline.locate`, `ui.playback`, `ui.screenshot`, **`render.frame`**, `ui.resize`, `ui.focus`, `app.quit` | `crates/ui-egui/src/control.rs:176-458` |
| Default port | None built in (the docs use 9877) | `docs/control-protocol.md:6` |

**2. Auth:** none on the control port (`docs/control-protocol.md:26`). The GUI has no automation roots: `--automation-read-root` is rejected as an unknown option (`apps/effectcraft/src/main.rs:571`, test). MCP headless can confine file access with roots (`docs/mcp.md:6-17`).

**3. Command registry:** `CommandSpec {id, label, menu, shortcut, alt_shortcuts, params: &'static str, enabled, run, journal}` (`crates/engine/src/commands/mod.rs:101-117`). Macros: `cmd!` (`:119`), `query!` (`:139`), and `fe!`/`res!`/`fast!`/`chan!`/`view_cmd!`/`style_cmd!`/`stub!` wrappers.
- **Count:** 728 literal ids from a regex over every wrapper macro. The code comment says "~2,000 ids" (`:259`), and `effectcraft-cli info` prints `command_specs().len()` (`main.rs:450`), so the live count may be higher because some macros generate ids.
- **UI commands:** 111 `uic!` entries (`crates/ui-egui/src/menus.rs:29+`).
- **Id style:** `namespace.camelCase` (`renderQueue.add`, `file.exportTimeline`).
- **Params:** a free-text doc string, **plus** a derived JSON Schema (`params_schema`, `mod.rs:341-360`). The schema has properties with `description` only and no types. It's served by `command.describe` and `command.list {schemas:true}`.
- **Effects:** `journal` only (`query!` means read). MCP annotations give readOnly/destructive/idempotent from the name lists `READ_ONLY` and `WRITES_FILES` (`tools.rs:35-51, 66, 83`).

**4. Events/progress:** this is the only app here with push. MCP sends `notifications/progress` (with `_meta.progressToken`) and honours `notifications/cancelled` for a headless blocking `renderQueue.render` (`crates/automation/src/long_job.rs:1-6`; `docs/mcp.md:54-85`). Otherwise jobs are polled: `jobs.list`, `jobs.wait` (`crates/engine/src/commands/panels_cmds.rs:230-232`). The control channel has no push.

**5. Handoff commands**

| Need | Command | Cite |
|---|---|---|
| Open | `file.open {path}` (.ecproj); `file.import {paths, …}` for media | `crates/engine/src/commands/file.rs:732, 738` |
| Export | `renderQueue.add {comp?, format?, output?…}` + `renderQueue.render {wait?}`; `render.preRender`; `file.exportTimeline {path, format: otio\|fcpxml\|xml\|edl\|aaf\|omf}` (the FilmCraft path); `file.exportLottie` | `commands/render_queue.rs:1044, 1048`; `commands/timeline.rs:219` (study line, not re-checked); `commands/lottie.rs:57` |
| Render to file | Control `render.frame {comp?, time?, max_side?, path?, base64?}` → `{comp, time, width, height, path}`; CLI `render-frame … --out p.png --json`; MCP `render_frame` / `render_preview` | `control.rs:446`; `main.rs:570`; `tools.rs:728, 944` |
| Capabilities | `app.capabilities` → version, counts, export formats | `commands/query.rs:212` |

### VectorCraft (Illustrator-style vector), `0946bc5`, workspace v0.11.0

**1. Transports**

| Transport | Detail | Cite |
|---|---|---|
| CLI binary | `vectorcraft-cli`. Subcommands: `mcp [--connect ADDR\|--headless] [roots]`, `run [--in F] [--cmd ID [--params JSON]]… [--export F]…`, `commands`, `convert IN OUT`, `info FILE`, `bench`, `perf` | `apps/vectorcraft-cli/src/main.rs:67-113`, dispatch `:130-132` |
| `commands` shape | Pretty JSON from `Headless::call("engine.commands")`. Extra arguments are ignored, so `commands --json` works by accident. Shape is `CommandInfo {id, label, menu: [str], shortcut, params, enabled, disabled_reason?}` | `main.rs:214-219`; `crates/engine/src/cmd/mod.rs:119-144` |
| MCP stdio | `vectorcraft-cli mcp`. **By default it connects to `127.0.0.1:7979`** and falls back to headless. Roots imply `--headless`. Protocol `2026-07-28` (`MODERN_VERSION`), `2025-06-18`, `2025-03-26`, `2024-11-05` | `main.rs:71-76`; `crates/mcp/src/lib.rs:33`; `crates/mcp/src/server.rs:13-15` |
| MCP tools | 32, including the suite convention `command_list`, `command_run`, `command_batch`, `doc_inspect`, `render_preview`, plus `ui_inspect`, `ui_screenshot` and legacy `list_commands`/`run_command`/`open_file`/`export`/…. 3 resources, 4 templates, 5 prompts | `crates/mcp/src/tools.rs:79-461` (`:83-114` convention); `docs/mcp.md:260-264` |
| `--control` | `vectorcraft --control <port> [--automation-read-root D] [--automation-write-root D]` or `VECTORCRAFT_CONTROL_PORT`. Loopback NDJSON, same envelope. 4 MiB lines, 16 connections, plus per-connection deadlines (5 min idle, 30 s per line, 60 s write). A request the app hasn't started when it times out is **withdrawn** ("will not run") | `apps/vectorcraft/src/control_server.rs:24-33, 37-45, 47-59, 60-72, 162`; `docs/control-protocol.md:55-58` |
| Control methods | `engine.execute`/`ui.menu.invoke`/`command`, `engine.commands`, `document.inspect`, `ui.inspect`, `ui.menu.list`, `ui.contextMenu.list`, `ui.tool.*`, `ui.pointer/key/move/click/drag/wheel/text/set`, `ui.dialog.*`, `ui.resize`, `ui.focus`, `ui.screenshot`, **`ui.render`**, **`app.open`**, `app.save`, **`app.export`**, `app.quit` | `crates/ui-egui/src/control.rs:198-451` |

**2. Auth:** none (`docs/control-protocol.md:22`). Optional **automation roots** on the GUI itself confine file reads and writes (`:55-58`), the strongest confinement of the four.

**3. Command registry:** `CommandSpec {id, label, menu, shortcut, params, enabled, run, journal}` (`crates/engine/src/cmd/mod.rs:103-116`). `cmd!` has a `query` form (`:186-193`) and `style_cmds!` generates 16 ids (`cmd/textstyles.rs:53`).
- **Count:** 727 `cmd!` + 16 = 743 engine ids, plus 285 `UI_COMMANDS` tuples (`crates/ui-egui/src/menus.rs:54`). Both go into `engine.commands` with `"ui": true` (`control.rs:129-135`).
- **Id style:** `namespace.camelCase`. Note that **`file.open` is UI-only**; the engine opens with `document.open`.
- **Effects:** `journal` only. MCP annotations mark every non-read-only tool as destructive (`tools.rs:74`).

**4. Events/progress:** none. Request/reply only. MCP ignores progress notifications (`server.rs:246`).

**5. Handoff commands**

| Need | Command | Cite |
|---|---|---|
| Open | `document.open {path, pages?, …}` (23 readable formats); control `app.open {path}` → UI `file.open` | `crates/engine/src/cmd/fileio/mod.rs:86`; `control.rs:442` |
| Export | `document.export {path?, format?, artboard?, scale?, …}`; `document.exportPdf`, `document.exportForScreens`; CLI `convert IN OUT` | `fileio/mod.rs:113`; `main.rs:221` |
| Thumbnail without opening | `file.place.info {path, thumbnail: px}` → `thumbnailBase64` (≤512 px); `document.pdfInfo {thumbnail}` | `cmd/place/mod.rs:65`; `fileio/mod.rs:158` (study line) |
| Render to file | Control `ui.render {path?, scale?, artboard?}`, `app.export`; MCP `render_preview` | `control.rs:415, 444` |

### SoundCraft (Pro Tools-style DAW), `3a28a65`, v0.7.0

**1. Transports**

| Transport | Detail | Cite |
|---|---|---|
| CLI binary | `soundcraft-cli`. Subcommands: `info FILE`, `convert`, `run [--cmd ID[=JSON]]… [--bounce OUT]`, `script FILE`, `commands [FILTER]`, `describe ID`, `app [--port P] METHOD [JSON]`, `mcp [--connect PORT] [--demo]`, `parity`, `plugins` | `apps/soundcraft-cli/src/main.rs:3-19` |
| `commands` shape | **A text table only** (`id label menu`). `commands --json` treats `--json` as FILTER and prints **nothing**, exit 0. JSON is available only per id through `describe ID` | `main.rs:304-315, 317-326` |
| MCP stdio | `soundcraft-cli mcp [--demo]` (headless) or `--connect PORT` (GUI). Protocol `2025-06-18`. 16 tools: `list_commands`, `execute`, `batch`, `inspect_session`, `new_session`, `open_session`, `save_session`, `import_audio`, `bounce_mix`, `parity`, app-only `screenshot`/`ui_inspect`/`ui_set`/`menu_invoke`/`click`/`drag`/`key`. **No** `command_*` convention, and no tool annotations | `main.rs:389-420`; `crates/automation/src/mcp.rs:7, 20-97, 21` |
| `--control` | `soundcraft --control <port>`. **`0` picks a free port and prints `SOUNDCRAFT_CONTROL_PORT=<n>` on stdout**, the only port-discovery mechanism among these apps. Loopback NDJSON with the same envelope. Lines up to about 86 MiB (sized for plugin state), 16 connections, **90 s** timeout | `apps/soundcraft/src/control_server.rs:20-24, 26-43, 97-137`; `docs/control-protocol.md:9, 28-38` |
| Control methods | `engine.execute`/`command`, `engine.commands`, `engine.parity`, `session.inspect`/`document.inspect`, `ui.inspect`, `ui.menu.list/invoke`, `ui.set`, `ui.move/click/drag/scroll/zoom/key/text`, `ui.screenshot`, `app.quit` | `crates/ui-egui/src/control.rs:98-235` |
| Default port | 7979 (`app` subcommand) | `apps/soundcraft-cli/src/main.rs:362` |

**2. Auth:** none, and **there is no request-only guard either.** A line that isn't JSON gets `{"ok":false,"error":"bad JSON…"}` and the connection **stays open**, so the next JSON line runs (`control_server.rs:116-132`). From the static read, a browser's cross-origin `fetch` POST to `127.0.0.1:<port>` with a JSON body could run a command. Its header lines would be rejected one by one, and the body line would execute. DesignCraft, EffectCraft and VectorCraft close the connection on the first non-request line. Recorded as **O-B2**; needs a live check (Not verified, step 6).

**3. Command registry:** `CommandSpec {id, label, menu, shortcut, params, enabled, run, journal, undoable, view}` (`crates/engine/src/cmd/mod.rs:49-64`). `cmd!` has `query`, `noundo` and `view` forms (`:93-103`). `CommandInfo` always serializes `disabled_reason` (it's null when enabled) (`:66-75`).
- **Count:** 477–478 engine ids (regex 477; the study counted 478) plus about 66 UI ids.
- **Id style:** **`namespace.snake_case`** (`session.save_as`, `file.import_audio`, `file.bounce_mix`). This is the only snake_case app in the suite.

**4. Events/progress:** none. MCP has no notifications. `transport.status` and `session.inspect` are polled.

**5. Handoff commands:** open `session.open {path}` → `{name, tracks, missing}` (`crates/engine/src/cmd/file.rs:12`). Media import is `file.import_audio {path|paths, track?, at?}` (`:58`). Export is `file.bounce_mix {path, format?, …}` and `file.bounce_stems {dir}` (`:87-88`). **No render or thumbnail command:** only `ui.screenshot` of the window. Note that control `engine.execute session.open` skips the save prompt (study, finding at `effectcraft-vectorcraft-soundcraft.md:563`).

### ArtCraft (storytold/artcraft, credit-based AI app), `d4e4ac6`

ArtCraft is structured differently from the rest of the suite. **It has no local control surface at all.**

| Question | Finding | Cite |
|---|---|---|
| What ships | Two desktop apps in one workspace. (1) The **Tauri app** (default member, TypeScript/React frontend under `frontend/apps/artcraft`). (2) **`artcraft-native`**, a new pure-Rust egui app ("no Tauri, no webview" except a Midjourney sign-in window) with Create Image/Video/Audio built | `Cargo.toml:5-46` (`default-members = ["crates/desktop/artcraft"]`); `AGENTS.md:7-9`; `artcraft-native/README.md:1-12` |
| `<app>-cli` | None. No CLI binary in the workspace | `Cargo.toml:5-43` |
| MCP stdio | None locally | grep of `artcraft-native/src`, `crates/desktop/artcraft/src` |
| `--control` | None. `artcraft-native` reads no arguments (`main.rs` goes straight to `eframe::run_native`). The Tauri app has no deep-link or single-instance plugin | `artcraft-native/src/main.rs:17-27`; `crates/desktop/artcraft/Cargo.toml:72-84` |
| Command registry | No `CommandSpec`/`Session::execute`. The Tauri app exposes **63 Tauri IPC commands** (in-process JS ↔ Rust only, not reachable from outside) such as `generate_image_command`, `estimate_video_cost_command`, `storyteller_get_credits_command`. They are `snake_case_command` names, not dotted ids | `crates/desktop/artcraft/src/lib.rs:227-291` |
| Events | Internal only: `artcraft-native`'s backend returns results to the UI as `Event`s over channels | `artcraft-native/README.md` (Architecture table) |
| Handoff | None locally. Generated media goes to the user's **cloud library** (Storyteller media files). Files are downloaded with `download_media_file_command` | `lib.rs:230-232` |

**How the Storyteller API client is separated.** The separation is clean, and AssetCraft's ArtCraft-account provider can reuse it.
- `crates/api_clients/artcraft/artcraft_api_defs` holds the wire types (serde + utoipa, no HTTP).
- `artcraft_client` holds the typed endpoints (`credits`, `generate`, `jobs`, `media_files`, `omni_gen`, `users`, …). It's reqwest-based, has **no Tauri dependency**, and uses cookie credentials `StorytellerCredentialSet {avt, session}`.
- `artcraft_router` handles provider routing (fal, Midjourney, Grok, WorldLabs …).
- Cites: `crates/api_clients/artcraft/*/Cargo.toml` `[dependencies]`; `artcraft_client/src/credentials/storyteller_credential_set.rs:6-9`.
- `artcraft-native` consumes them as "Internal (Tauri-free) API crates shared with the Tauri app" (`artcraft-native/Cargo.toml:14-21`).
- Credentials are a **JSON file**, `~/Artcraft/credentials/native_session.json`, seeded from the Tauri cookie jar. They are not kept in the OS keyring (`artcraft-native/src/backend/session.rs:1, 14-17, 87-88`).

> **Watch-out (two APIs):** the Storyteller web API now defines **server-side MCP sessions**: `POST /v1/mcp/session/create|refresh|revoke`, `GET /v1/mcp/session/list` (`crates/api_clients/artcraft/artcraft_api_defs/src/mcp_sessions/create_mcp_session.rs:7`, `list_mcp_sessions.rs:6`). That is a **remote** MCP endpoint of the web backend. It is **not** local app control and stays out of PipelineCraft's scope (outline header). §2's glossary should name it so nobody confuses it with the suite's stdio MCP.

---

## 3.2b Auth variance (this half)

| App         | Control auth | Line guard (closes on a non-request line) | File confinement | Cite |
|-------------|--------------|-------------------------------------------|------------------|------|
| DesignCraft | none         | yes                                       | MCP headless roots only | `designcraft` `docs/control-protocol.md:26-27`; `control_server.rs:80-107` |
| EffectCraft | none         | yes                                       | MCP headless roots only; the GUI rejects the flags | `effectcraft` `docs/control-protocol.md:26`; `docs/mcp.md:6-17` |
| VectorCraft | none         | yes, plus deadlines and request withdrawal | GUI **and** MCP roots | `vectorcraft` `docs/control-protocol.md:22, 55-58` |
| SoundCraft  | none         | **no**                                     | MCP headless roots (not with `--connect`) | `soundcraft` `control_server.rs:97-137`; `docs/mcp.md:20-69` (study) |
| ArtCraft    | n/a (no local control) | n/a                              | n/a              | — |
| AssetCraft  | n/a (not built yet)    | n/a                              | n/a              | `assetcraft` `crates/engine/src/lib.rs:3` ("(later) MCP") |

The part-a apps (PhotoCraft token, PdfCraft token file, WordCraft key file) are the only ones with auth (`assetcraft` `docs/sibling-apis/README.md:36-46`).

## 3.3b Command naming and catalog shape

| App         | Id style                     | Open-file id     | Catalog entry keys                                         | Disabled key      | UI ids in `engine.commands` |
|-------------|------------------------------|------------------|-------------------------------------------------------------|-------------------|-----------------------------|
| DesignCraft | `ns.camelCase`               | `file.open`      | `id, label, menu[], shortcut, params, enabled`              | `disabled_reason` | yes, `"ui": true`           |
| EffectCraft | `ns.camelCase`               | `file.open`      | `id, label, menu[], shortcut, params, enabled, schema?`     | `why`             | no (menu dispatcher only)   |
| VectorCraft | `ns.camelCase`               | `document.open` (engine); `file.open` is UI | same as DesignCraft                  | `disabled_reason` | yes, `"ui": true`           |
| SoundCraft  | `ns.snake_case`              | `session.open`   | same keys                                                    | `disabled_reason` (always present) | see `control.rs:106` |
| AssetCraft  | `ns.camelCase`               | `file.open` (OS default app); `file.openIn {app}` | `id, label, menu: str, shortcut, params` | none (no `enabled`) | n/a |

What they agree on: dotted `namespace.verb` ids; a free-text `params` doc string; `engine.execute {command, params}` and `engine.commands` as control methods (four apps); and the envelope `{id, method, params}` → `{id, ok, result|error}`. What diverges: snake_case versus camelCase, the open verb (`file.open` / `document.open` / `session.open`), `why` versus `disabled_reason`, `menu` as an array versus a string, and whether UI ids are listed.

---

## 3.4 AssetCraft's own surface (`assetcraft` @ `3a41b2b`)

**What exists today**

| Piece | State | Cite |
|---|---|---|
| `Session::execute(id, &Value) -> CmdResult` | The single state-changing entry point. It bumps a `generation` counter | `crates/engine/src/lib.rs:210-215` |
| `CommandSpec` | `{id, label, menu: &str, shortcut, params: &str, run}`. **No `enabled`, no `journal`/`undoable`, no effects** | `crates/engine/src/commands.rs:13-26`; `cmd!` `:34-38` |
| Registry | **32 commands**, regex-counted: `nav.*` (10), `view.*` (8), `formats.heic`, `select.*` (5), `file.open`, `file.run`, `file.openIn`, `file.reveal`, `item.probe`, `cache.clear`, `app.state`, `app.commands`, `app.openInTargets` | `commands.rs:39-152` |
| Catalog | `app.commands` → `[{id, label, menu, shortcut, params}]` | `commands.rs:143, 592-601` |
| Errors | `CmdError::{Unknown, BadParams, Fs, Unavailable}`. The CLI maps these to exit codes 0/1/2 | `lib.rs:77-86`; `apps/assetcraft-cli/src/main.rs:8-9` |
| CLI | `assetcraft-cli list \| probe \| thumb \| commands [--json] \| exec <id> [<json>] \| help`. `commands --json` runs `app.commands` | `apps/assetcraft-cli/src/main.rs:25-40, 185-203, 481-486` |
| Handoff today | `file.openIn {app: photocraft\|lightcraft\|filmcraft\|pdfcraft\|designcraft\|wordcraft, paths?}` launches the program with path argv (no shell, detached). It does **not** use a control channel and has no EffectCraft/VectorCraft/SoundCraft entries | `commands.rs:122-129`; `STATUS.md:23` (8a) |
| MCP, `--control`, serve | **Not present.** The lib doc says "(later) MCP" | `crates/engine/src/lib.rs:3`; grep finds no `TcpListener`/`mcp` in `apps`, `crates` |
| Thumbnails as a command | Not a command. `thumb` calls `assetcraft_thumbs::render` directly | `apps/assetcraft-cli/src/main.rs:4-6` |

**What is planned, and where it is written**

- `docs/assetcraft-plan.md:26` (principle 3): "everything is a command, with an opt-in loopback MCP server and control channel".
- `docs/assetcraft-plan.md:39` (layout): `assetcraft-cli` is "headless: list, probe, thumbs, exec, mcp".
- `docs/assetcraft-plan.md:58-67` (stage table): Stages 0, 1, 2, 3, **3.5 (Develop)**, 4–7. **There is no Stage 1.5 row.** Stage 6 (`:64`, `:221`) is "Recipes … any-CLI step, hot folders", which is where a cross-app client is needed.
- `docs/sibling-apis/README.md:89-91`: "Stage 1.5" appears only as an *optional* slot for new thumbnail formats (SVG/AI/EPS, EffectCraft, audio). That README also recommends `crates/craft-control` "in AssetCraft's engine … until craft-libs has one" (O3 there).
- The brief's "Stage 1.5 adds serve, MCP, `--control` and command effects" is **not yet in the plan or STATUS at `3a41b2b`**. Recorded as O-B9.

---

## 3.6b craft-libs (`craft-libs` @ `8f5a283`)

| Crate            | Purpose (from its `lib.rs`/README)                                         | Used by (this half, pinned rev) |
|------------------|-----------------------------------------------------------------------------|---------------------------------|
| `craft-atomic`   | Crash-safe atomic file write and replace (same-dir temp, fsync, rename)      | (PhotoCraft per grep)           |
| `craft-cms`      | Pure-Rust ICC v2/v4 colour management                                        | (PhotoCraft)                    |
| `craft-pdf`      | Shared PDF read, render and optional write (hayro/krilla, vendored)          | DesignCraft `c348732`           |
| `craft-print`    | System print spooler (CUPS, Windows)                                         | DesignCraft, VectorCraft `d991bf6` |
| `craft-quit`     | Platform-correct quit (macOS `terminate:`)                                   | —                               |
| `craft-settings` | `config.toml` storage, atomic saves, backup, migration                       | —                               |
| `craft-ui`       | egui controls, icon catalogs, transactional docking                          | all four apps + AssetCraft `5c874d5` |
| `craft-pdf-read-only-contract` | Test-only contract crate (`publish = false`)                  | —                               |

Cites: `craft-libs` `Cargo.toml:15-33`; `README.md:22-44`; each app's root `Cargo.toml` (`designcraft` `:15-16, 73`).

**There is no shared control, command, registry or MCP crate.** craft-libs explicitly leaves commands to each app: "Applications own their assets, themes, layouts, commands and document semantics" (`crates/ui/src/lib.rs:3-4`), and "No generic group docking or app command registry is inferred" (`docs/ui-docking.md:138`).

Evidence that one is needed:
- **The same transport was copied three times.** DesignCraft and EffectCraft `control_server.rs` are near line-for-line identical (doc comment, `MAX_LINE`, `MAX_CONNECTIONS`, `Line` enum, `parse_line`, `serve_lines`). VectorCraft forked the same file and added deadlines and request withdrawal. SoundCraft has an older variant without the guard.
- **The same MCP shape was hand-written four times**, each with a `Backend` trait and `Headless`/`Remote` backends (`designcraft` `crates/mcp/src/lib.rs:5-14`, `vectorcraft` `crates/mcp/src/lib.rs:1-13`).
- **`CommandSpec` was hand-written five times**, with five different field sets.

The convergence opportunity is O-B1.

---

## 3.7b Cross-app comparison

| App         | Transports               | Framing                         | Auth | Id style       | Effects metadata               | Events/progress                       | Open-file command                 | Export / render command                                      |
|-------------|--------------------------|---------------------------------|------|----------------|--------------------------------|---------------------------------------|-----------------------------------|---------------------------------------------------------------|
| DesignCraft | cli, MCP (headless/remote), `--control` | NDJSON `{id,method,params}`→`{id,ok,…}`, 4 MiB, guard | none | `ns.camelCase` | `journal`, `undoable`; MCP `readOnlyHint` | none                                  | `file.open {path}` / `app.open`  | `file.exportPdf`; PNG only via `ui.render` / `app.export`     |
| EffectCraft | cli (`--bridge`), MCP (headless/bridge), `--control` | same, 4 MiB, guard     | none | `ns.camelCase` | `journal`; MCP read/destructive/idempotent | MCP progress + cancel (render only); `jobs.*` poll | `file.open {path}`               | `renderQueue.add`+`render`; `render.frame {path}`             |
| VectorCraft | cli, MCP (remote-first), `--control` | same, 4 MiB, guard + deadlines | none (roots) | `ns.camelCase` | `journal`; MCP read/destructive | none                                  | `document.open {path}` / `app.open` | `document.export {path}`; `ui.render`; `file.place.info` thumb |
| SoundCraft  | cli, MCP (headless/connect), `--control` (port 0) | same, ~86 MiB, **no guard**, 90 s | none | `ns.snake_case` | `journal`, `undoable`, `view`; no MCP hints | none                                  | `session.open {path}`            | `file.bounce_mix {path}`; no render                           |
| ArtCraft    | **none local** (63 Tauri IPC commands, in-process) | n/a                    | n/a  | `snake_case_command` | n/a                      | internal only                         | n/a                               | n/a (cloud library)                                           |
| AssetCraft  | cli only (`exec`, `commands --json`) | n/a yet                   | n/a  | `ns.camelCase` | **none**                       | none (watcher is internal)            | `file.open` (OS) / `file.openIn` (spawn) | `assetcraft-cli thumb` (not a command)                  |

---

## 3.8b Convergence opportunities

IDs carry a `B` so the merge with part a can renumber them.

| ID    | Finding (scope)                                                                 | Project goal it serves                         | Suggested fix                                                                                         | Owner |
|-------|---------------------------------------------------------------------------------|-----------------------------------------------|--------------------------------------------------------------------------------------------------------|-------|
| O-B1  | No shared control/registry/MCP crate. The transport is copied ×3 and diverging, `CommandSpec` ×5, the MCP server ×4 (§3.6b) | One hub contract; fewer adapter quirks        | A `craft-control` crate family. (a) `craft-control-proto`: envelope types, error codes, a `CommandInfo` with `effects`, and port-file discovery. (b) A server: the guarded NDJSON loop (VectorCraft's version with deadlines) plus an optional token handshake. (c) A client. (d) An MCP shim that maps the registry to `command_list/run/batch`. Incubate it in AssetCraft as `crates/craft-control` (layers L0/L1, no UI) per `docs/sibling-apis/README.md:91`, then offer it upstream | Claude (incubate); craft-libs (adopt) |
| O-B2  | SoundCraft's control server has no request-only guard, so a cross-origin HTTP body line may execute (static read) | Safety of any hub that tells users to enable `--control` | Port DesignCraft's `parse_line`/`serve_lines` guard (`designcraft` `control_server.rs:80-157`) into `soundcraft` `control_server.rs:97-137`. Verify first (Not verified, step 6) | SoundCraft (upstream); Paul files the issue |
| O-B3  | `commands --json` doesn't work everywhere: DesignCraft prints `[]`, SoundCraft prints nothing, VectorCraft ignores the flag | §3.5 live verification; hub discovery (§5)     | Suite rule: `<app>-cli commands --json` prints the full `engine.commands` array. Until then the hub adapter calls DesignCraft `commands`, VectorCraft `commands`, and SoundCraft per-id `describe` or control `engine.commands` | Each app; Claude (adapter quirks table) |
| O-B4  | The suite MCP convention `command_list`, `command_run`, `command_batch`, `doc_inspect`, `render_preview` and `<app>://document\|commands` ("FilmCraft #28") is in EffectCraft and VectorCraft (and PhotoCraft, FilmCraft, PdfCraft, LightCraft per grep) but **not** in DesignCraft, SoundCraft or WordCraft | PipelineCraft's MCP front door (§4.1) can be a thin aggregator | Adopt the #28 tool set as the hub's per-app MCP contract. Ask DesignCraft and SoundCraft to add it. The hub namespaces tools as `<app>.command_run` | Claude (spec §6); DesignCraft, SoundCraft |
| O-B5  | No effects metadata beyond `journal`/`undoable`. Nothing declares writes-user-file, network or destructive | §6.4 effects; dry-run and undo rules (§8.5)    | Add `effects: &[Effect]` to `CommandSpec` (read, writes-app-state, writes-user-file, network, destructive), defaulted from the existing `query`/`noundo` forms. AssetCraft does it first (O-B9); the hub falls back to `journal == false` meaning read | AssetCraft (Claude); each app later |
| O-B6  | Params are free text everywhere. Only EffectCraft derives a schema, and it has no types | §6.3 JSON Schema per command                   | Hub v1 accepts `params` docs plus EffectCraft-style derived schemas. Longer term `craft-control-proto` defines a typed `ParamSpec` | Claude; upstream |
| O-B7  | Port collisions and no discovery: 7979 is the default in DesignCraft MCP/`app`, VectorCraft MCP and SoundCraft `app`. Only SoundCraft supports `--control 0` with a printed port | §5.1–5.2 discovery manifest                    | The hub always launches with explicit ports (or `0` where supported) and records them. Suite rule: write `<config dir>/control.json {pid, port, protocol, token?}` on bind | Claude (hub); each app |
| O-B8  | Id style and open verbs diverge (`snake_case` in SoundCraft; `file.open` / `document.open` / `session.open`; `why` / `disabled_reason`) | §6.2 addressing; §9.1 Open In                  | The hub defines capability aliases (`asset.open`, `asset.export`, `asset.thumbnail`) mapped per app in adapter config, not code (§4.4). Never rename upstream ids | Claude (spec) |
| O-B9  | AssetCraft's Stage 1.5 (serve, MCP, `--control`, effects) is in the brief but not in `docs/assetcraft-plan.md` or `STATUS.md` | AssetCraft is the first hub client and server  | Add a Stage 1.5 row and section to the plan. Its steps: (1) `effects` + `enabled` on `CommandSpec`, (2) `assetcraft-cli commands --json` with the full shape, (3) `crates/craft-control` server and `--control`, (4) `assetcraft-cli mcp` with the #28 tools, (5) `thumb.render` as a command. Exit: `engine.commands` over the port matches `commands --json` | Paul (approve); Claude (draft) |
| O-B10 | AssetCraft `file.openIn` launches a new process and lacks EffectCraft, VectorCraft and SoundCraft | §9.1 Open In                                   | Add the three to `open_in.rs` (study O12). "Open In v2" sends `app.open` / `file.open` / `session.open` to a running instance through the O-B1 client | Claude |
| O-B11 | ArtCraft has no local control, CLI or MCP. Its natural handoff is files coming out of the cloud library | Generated assets entering the pipeline (Stage 5) | The hub treats ArtCraft as **not a hub app**. AssetCraft's ArtCraft provider (`artcraft_client`, Tauri-free) downloads results into a watched folder, and AssetCraft's watcher picks them up. If upstream wants local control later, `artcraft-native` is the place (egui, could adopt O-B1) | Claude (AssetCraft provider); upstream ArtCraft (optional) |
| O-B12 | ArtCraft credentials sit in a JSON file, not the OS keyring | AssetCraft rule "secrets in OS keyring"        | AssetCraft's provider stores its own session in the keyring and does **not** read `native_session.json` without asking the user | Claude |
| O-B13 | Only EffectCraft pushes progress (MCP, render only). Nothing pushes over control | §8 events and jobs                             | Hub v1 polls (`jobs.list`, `transport.status`) behind one `job.*` abstraction. Suite rule: an optional `events.subscribe` on the control channel emitting `{"event":…}` lines | Claude (hub); upstream |

---

## Not verified

Everything above is a static read. Run these on Paul's Windows machine, in a scratch profile, against throwaway files.

**Next action (Paul)**

1. Catalog counts and shape, DesignCraft (expected: one pretty JSON array of about 463 objects with `disabled_reason` on disabled entries. The second call shows the `--json` bug: expected `[]`):
   ```powershell
   Set-Location F:\Script-Projects\ArtCrafting\designcraft
   cargo run -q -p designcraft-cli -- commands | Out-File $env:TEMP\dc-commands.json -Encoding utf8
   (Get-Content $env:TEMP\dc-commands.json -Raw | ConvertFrom-Json).Count
   cargo run -q -p designcraft-cli -- commands --json
   ```
2. EffectCraft (expected: a count of about 728 or more, matching `info`'s `commands`, and entries carrying a `why` key):
   ```powershell
   Set-Location F:\Script-Projects\ArtCrafting\effectcraft
   cargo run -q -p effectcraft-cli -- commands --json | Out-File $env:TEMP\ec-commands.json -Encoding utf8
   (Get-Content $env:TEMP\ec-commands.json -Raw | ConvertFrom-Json).Count
   cargo run -q -p effectcraft-cli -- info --json
   ```
3. VectorCraft (expected: about 743 engine entries; any extra arguments are ignored):
   ```powershell
   Set-Location F:\Script-Projects\ArtCrafting\vectorcraft
   cargo run -q -p vectorcraft-cli -- commands | Out-File $env:TEMP\vc-commands.json -Encoding utf8
   (Get-Content $env:TEMP\vc-commands.json -Raw | ConvertFrom-Json).Count
   ```
4. SoundCraft (expected: about 478 text lines from the first call and **no output** from the second):
   ```powershell
   Set-Location F:\Script-Projects\ArtCrafting\soundcraft
   (cargo run -q -p soundcraft-cli -- commands | Measure-Object -Line).Lines
   cargo run -q -p soundcraft-cli -- commands --json
   ```
5. Control channel round trip, DesignCraft (expected: a reply line starting `{"id":1,"ok":true,"result":[` that also contains `"ui":true` entries). Start the app first in another window with `cargo run -q -p designcraft -- --control 7981`:
   ```powershell
   Set-Location F:\Script-Projects\ArtCrafting\designcraft
   $c = [System.Net.Sockets.TcpClient]::new('127.0.0.1', 7981)
   $s = $c.GetStream(); $w = [IO.StreamWriter]::new($s); $r = [IO.StreamReader]::new($s)
   $w.WriteLine('{"id":1,"method":"engine.commands"}'); $w.Flush()
   $r.ReadLine().Substring(0,80); $c.Close()
   ```
6. **SoundCraft guard check (O-B2).** Start `cargo run -q -p soundcraft -- --control 0 --no-audio` and note the printed `SOUNDCRAFT_CONTROL_PORT=<n>`. Then send an HTTP-shaped preamble followed by a harmless read. Expected if O-B2 is real: two or more `bad JSON` error lines, then a **successful** `{"id":7,"ok":true,…}` reply. Compare with DesignCraft on step 5's port: one error, then the connection closes.
   ```powershell
   Set-Location F:\Script-Projects\ArtCrafting\soundcraft
   $port = <n>
   $c = [System.Net.Sockets.TcpClient]::new('127.0.0.1', $port)
   $s = $c.GetStream(); $w = [IO.StreamWriter]::new($s); $r = [IO.StreamReader]::new($s)
   $w.WriteLine('POST / HTTP/1.1'); $w.WriteLine('Host: 127.0.0.1'); $w.WriteLine('')
   $w.WriteLine('{"id":7,"method":"engine.parity"}'); $w.Flush()
   1..3 | ForEach-Object { $r.ReadLine() }; $c.Close()
   ```
7. Check that the ArtCraft workspace has no CLI binary (expected: only `artcraft`, `artcraft-native` and test or library targets):
   ```powershell
   Set-Location F:\Script-Projects\ArtCrafting\artcraft
   cargo metadata --no-deps --format-version 1 | ConvertFrom-Json | ForEach-Object { $_.packages.targets | Where-Object { $_.kind -contains 'bin' } | Select-Object name }
   ```

If VectorCraft or SoundCraft aren't cloned under `F:\Script-Projects\ArtCrafting\`, use the installed `vectorcraft-cli.exe` / `soundcraft-cli.exe` from Program Files instead of `cargo run`.

## Open questions

1. **Stage 1.5 scope (O-B9).** Should AssetCraft's plan get a Stage 1.5 for serve, MCP, `--control` and effects before Stage 2, merged with the optional thumbnail-formats slot in `docs/sibling-apis/README.md:89`? Or does it stay a PipelineCraft prerequisite inside a later stage? (Paul)
2. **Where `craft-control` lives (O-B1).** Incubate inside AssetCraft and offer it to craft-libs later, or propose it to craft-libs now through Paul's forks (DesignCraft/EffectCraft PRs)? This decides whether the hub depends on upstream adoption. (Paul)
3. **ArtCraft's role (O-B11).** Confirm that ArtCraft is **not** a hub-routed app in v1, and that the server-side Storyteller MCP sessions (`/v1/mcp/session/*`) stay out of scope even though they're "MCP". (Paul)

## Inputs for downstream agents

- **§2 Glossary:** define "Storyteller MCP session" (remote, web API) separately from "app MCP" (stdio, local). Define `journal` (the replayable command log) as distinct from undo.
- **§4.4 Adapters:** one NDJSON control adapter covers DesignCraft, EffectCraft, VectorCraft and SoundCraft, since the envelope is identical. Per-app quirks to put in config:
  - catalog source (`commands` versus `commands --json` versus `describe`; see O-B3)
  - disabled key (`why` or `disabled_reason`)
  - whether UI ids are included
  - timeout (60 s; 90 s for SoundCraft)
  - max line size (SoundCraft about 86 MiB)
  - port strategy (SoundCraft `0` plus a stdout line; the others need an explicit port)
  - GUI roots (VectorCraft only)
  - bad-port behaviour (EffectCraft exits 1; DesignCraft ignores it)
  - ArtCraft has **no adapter**.
- **§5 Discovery:** no app writes a port file. The hub must own the port choice (O-B7). The default 7979 collides across three CLIs.
- **§6.1/6.2:** reuse the envelope `{id, method, params}` → `{id, ok, result|error}`, plus `engine.execute {command, params}` and `engine.commands`, unchanged. For MCP, reuse the #28 tool names (O-B4).
- **§6.4 Effects:** the source data available today is `journal` (all four), `undoable` (DesignCraft, SoundCraft), `view` (SoundCraft) and MCP hint lists (EffectCraft, VectorCraft, DesignCraft). Nothing marks network or user-file writes.
- **§8 Events:** only EffectCraft MCP `notifications/progress` for `renderQueue.render`. Everything else is polled (`jobs.list`/`jobs.wait`, `transport.status`).
- **§9.1 Open In:** the verbs are `file.open` (DesignCraft, EffectCraft), `document.open` or control `app.open` (VectorCraft), and `session.open` (SoundCraft). SoundCraft's control open skips the save prompt. AssetCraft today spawns processes and lists only six apps.
- **§9.2 Thumbnails:** VectorCraft `file.place.info {thumbnail}` (no document opened); EffectCraft `render.frame` / `render-frame`; DesignCraft `ui.render` or CLI `run --export` (no engine command); SoundCraft none.
- **§15 Decisions:** Stage 1.5 placement and `craft-control` home (Open questions 1–2).
