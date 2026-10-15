# PipelineCraft (working name): §1 Purpose, goals, non-goals · §2 Glossary

_2026-10-15 · spec draft v0 · status: draft (agent A1)_

- **Scope:** why the hub exists, who uses it, what it will not do, how we know v1 works, what it is called, and one shared vocabulary for every later section (§3–§16).
- **Project goal it serves:** a suite of local-first creative tools that hand work to each other (AssetCraft → StoryCraft → FilmCraft, later PrepressCraft) without the cloud, driven the same way by apps, by `craftctl` and by MCP clients.
- **Immediate area of improvement:** every app can already be driven locally, but one at a time, each with its own handshake and no way to find the others. This section names that gap, sets measurable targets for closing it, and fixes the terms so the next nine agents don't drift.

**Evidence basis.** Every sibling-app claim below is from a **static read** of the shallow clones in `/home/claude/src` at these commits. Nothing was built or run. Where a claim comes from the earlier study in `assetcraft` `docs/sibling-apis/` (read at release tags) and was not re-checked at HEAD, it says "(study)".

| Repo         | Commit    | Repo          | Commit    |
|--------------|-----------|---------------|-----------|
| `assetcraft` | `3a41b2b` | `designcraft` | `7ed5697` |
| `photocraft` | `fd28da5` | `effectcraft` | `2c08f67` |
| `lightcraft` | `38adc17` | `vectorcraft` | `0946bc5` |
| `filmcraft`  | `9ed9650` | `soundcraft`  | `3a28a65` |
| `pdfcraft`   | `ac89a11` | `artcraft`    | `d4e4ac6` |
| `wordcraft`  | `3292ba0` | `craft-libs`  | `8f5a283` |

---

## 1. Purpose, goals, non-goals, success criteria, naming

### 1.1 Problem statement

There are two APIs in Paul's world. Both are real and both work. Neither lets one Craft app hand work to another.

**API 1: local app control (what PipelineCraft builds on).** Each app's command registry, reached three ways: the headless `<app>-cli`, an MCP server on stdio, and an opt-in loopback control channel (`--control`) whose methods are `engine.execute {command, params}` and `engine.commands`.

**API 2: the Storyteller web API** (https://storyteller-docs.netlify.app/). ArtCraft's website and account backend: accounts, media files, credits, generation. In the suite it appears only as the "ArtCraft account" AI provider inside AssetCraft. It does not control local apps.

#### What the local surfaces look like today (static read)

The shape is shared, which is the good news: every control channel speaks JSON lines on loopback with the same two core methods.

| Evidence                                                            | What it shows                                                                                   |
|---------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| `designcraft` `crates/ui-egui/src/control.rs:146,152` @ `7ed5697`   | `"engine.execute" \| "ui.menu.invoke" \| "command"` and `"engine.commands"` are dispatched here |
| `effectcraft` `crates/ui-egui/src/control.rs:176,195` @ `2c08f67`   | Same two methods; `engine.commands` maps to the engine's own `command.list`                     |
| `designcraft` `crates/mcp/src/tools.rs:404,529` @ `7ed5697`         | The MCP server calls the *same* `engine.execute` / `engine.commands` on its backend             |
| `photocraft` `apps/photocraft/src/control_server.rs:20` @ `fd28da5` | `TcpListener::bind(("127.0.0.1", port))`: loopback only                                         |
| `filmcraft` `apps/filmcraft/src/control_server.rs:27` @ `9ed9650`   | Same loopback bind                                                                              |
| `lightcraft` `apps/lightcraft/src/control_server.rs:28` @ `38adc17` | Same loopback bind                                                                              |

The gaps are in everything around that shared core:

| Gap                                | Evidence                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            | Effect on app-to-app work                                                 |
|------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------|
| One app at a time                  | Each client (`<app>-cli --connect`, the app's own MCP) talks to exactly one app, e.g. `designcraft` `apps/designcraft-cli/src/main.rs:12` @ `7ed5697` (`mcp [--connect PORT]`)                                                                                                                                                                                                                                                                                                                                                      | A caller that needs two apps holds two connections with two dialects      |
| `--control` means different things | Port: `photocraft` `apps/photocraft/src/main.rs:198` @ `fd28da5`, `lightcraft` `apps/lightcraft/src/main.rs:756` @ `38adc17`, `filmcraft` `apps/filmcraft/src/args.rs:81` @ `9ed9650`, `designcraft` `apps/designcraft/src/main.rs:252` @ `7ed5697`. **File path**: `pdfcraft` `apps/pdfcraft/src/main.rs:17,191` @ `ac89a11` (random port, writes `{"port","token","pid"}` to the file, `:459`)                                                                                                                                    | A launcher can't start every app the same way                             |
| Varied auth                        | PhotoCraft: first line must be `auth {token}` (`photocraft` `apps/photocraft/src/control_server.rs:75` @ `fd28da5`; token from `--control-token` / `--control-token-file`, `main.rs:205,210`). PdfCraft: token in the control file (above). WordCraft: key in `<settings>/control-key.<instance>`, mode 0600 (`wordcraft` `apps/wordcraft/src/main.rs:7` @ `3292ba0`). LightCraft, FilmCraft, DesignCraft: no `auth` handling found in their `control_server.rs` (grep, static). EffectCraft, VectorCraft, SoundCraft: none (study) | Three handshakes plus "none"; no single client can attach to all          |
| No discovery                       | Only PdfCraft publishes where it listens (the control file). Everyone else needs the caller to pick the port up front. VectorCraft and SoundCraft both default to 7979 (study: `vectorcraft` `crates/mcp/src/lib.rs:33`, `soundcraft` `apps/soundcraft-cli/src/main.rs:362`)                                                                                                                                                                                                                                                        | Nothing can answer "which Craft apps are running, and where?"             |
| No events                          | No control channel pushes anything; MCP servers declare `"subscribe": false` (`lightcraft` `crates/mcp/src/server.rs:168` @ `38adc17`, `effectcraft` `crates/automation/src/server.rs:173` @ `2c08f67`). PhotoCraft has pull-only `jobs.list` / `jobs.cancel` (`photocraft` `crates/ui-egui/src/control.rs:440-441` @ `fd28da5`)                                                                                                                                                                                                    | A chain can't wait for "render done" without polling each app its own way |
| No effect metadata                 | No `CommandSpec` declares side effects. Nearest: `journal: bool` "false for queries" (`photocraft` `crates/engine/src/commands.rs:27` @ `fd28da5`; `effectcraft` `crates/engine/src/commands/mod.rs:115` @ `2c08f67`). MCP `readOnlyHint` is set per tool by hand-kept name lists (`lightcraft` `crates/mcp/src/tools.rs:387` @ `38adc17`; `designcraft` `crates/mcp/src/tools.rs:76` @ `7ed5697`)                                                                                                                                  | A hub (or an MCP client) can't tell safely which calls write user files   |

**Why the Storyteller API doesn't fill the gap.** It is a cloud account service. It knows the user, their media files and credits, and runs generation. It has no idea which apps are on Paul's machine, can't reach a loopback socket, and isn't meant to: routing local app traffic through it would add the cloud to a local-first suite and mix account auth with app control. It stays one AI provider inside AssetCraft (`assetcraft` `AGENTS.md` non-negotiable 7 @ `3a41b2b`).

**Why the local surfaces don't fill it either.** They are excellent single-app remotes. What's missing is the layer above them: find the apps, attach with the right handshake, call a command in any app by one address, know what a call will touch, and pass the result (an asset, a timeline, a job) to the next app. AssetCraft already does a hand-rolled slice of this for Open In and video posters (`assetcraft` `STATUS.md` step 8a, 7a @ `3a41b2b`; `crates/engine/src/open_in.rs`, `crates/thumbs/src/decoders/video.rs`), and its plan reserves an L4 `bridge` crate for "Sessions with the sibling apps" (`assetcraft` `AGENTS.md:36` @ `3a41b2b`). PipelineCraft is that layer, made once and shared.

### 1.2 Goals

Five verbs, three kinds of caller.

**Actors**

| Actor             | Who it is                                                                                                 | How it reaches the hub (v1)                 |
|-------------------|-----------------------------------------------------------------------------------------------------------|---------------------------------------------|
| Craft apps        | AssetCraft first (its planned L4 `bridge`), then StoryCraft, later any sibling that wants to call another | Hub client library (Rust), loopback         |
| `craftctl`        | The hub's command-line client, for Paul in PowerShell and for scripts (§12)                               | Same client library                         |
| MCP clients       | Claude (Claude Code / Desktop) and Paul's local llama.cpp stack                                           | The hub's MCP front door, stdio first (§10) |
| Apps being driven | Every Craft app with a command registry; reached through an adapter, never changed by the hub             | Their existing control channel, CLI or MCP  |

**Goals**

| ID  | Goal     | Means                                                                                                                       | Serves              |
|-----|----------|-----------------------------------------------------------------------------------------------------------------------------|---------------------|
| G1  | Discover | List installed and running Craft apps, their version, transport, health and command list, with no port typed by hand        | All three workflows |
| G2  | Call     | Run any command in any app by one address (`<app>/<command.id>` + JSON params), with the per-app handshake hidden           | Open In, Thumbnails |
| G3  | Chain    | Hand one step's output (an asset ref, a file, an OTIO timeline) to the next step, across apps                               | Plan → FilmCraft    |
| G4  | Observe  | One event stream for app up/down, job progress/done/failed, and asset changes, even where the app itself only polls         | Thumbnails, recipes |
| G5  | Automate | Saved recipes run headless with a dry run first, and the same surface exposed to MCP clients with effect-based confirmation | Recipes, MCP        |

Goals sit on the suite's house rules, not beside them: the hub is itself "everything is a command" (`assetcraft` `AGENTS.md` non-negotiable 3), never crashes on hostile input (non-negotiable 4), and never writes user files except by calling an app command whose effects say so (non-negotiable 5).

### 1.3 Non-goals (v1)

| ID   | Non-goal                                            | Reason                                                                                                                                                                |
|------|-----------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| NG1  | Replacing or re-defining any app's command registry | The app's registry stays the source of truth; the hub mirrors `engine.commands` and adds metadata in adapter config. Anything else forks the suite                    |
| NG2  | Routing the Storyteller web API                     | Different system (cloud accounts and credits), different auth; it stays one AI provider in AssetCraft (§1.1)                                                          |
| NG3  | Cloud sync, remote or LAN access                    | v1 binds loopback only, like every app's control channel (§1.1 bind citations). LAN opens a threat model §11 doesn't cover yet                                        |
| NG4  | Elo integration                                     | Project rule: Elo stays out of these apps (`assetcraft` `AGENTS.md` non-negotiable 7)                                                                                 |
| NG5  | 3D workflows                                        | No 3D app is in the suite's control surface; AssetCraft's `mesh` crate is thumbnails and preview only (`STATUS.md` 7b)                                                |
| NG6  | Driving app windows by pixels or synthesized input  | Only registry commands. `ui.*` commands exist in some apps but the hub treats them like any other command, with no UI scripting of its own                            |
| NG7  | A new scripting language                            | A recipe is a JSON list of command calls, close to what apps already accept (PhotoCraft actions/droplets, DesignCraft `script` JSONL, FilmCraft `run <script.jsonl>`) |
| NG8  | Changing an app's auth or behaviour on its behalf   | The hub adapts to each app; standardization is asked for upstream as small PRs (§14), never patched in                                                                |
| NG9  | Installing, updating or bundling apps               | Discovery reads what's installed (AssetCraft's `open_in.rs` approach); installers stay each app's MSI                                                                 |
| NG10 | A WASM / web hub                                    | A browser page can't open loopback TCP to the apps and shouldn't (§11 browser-origin rejection). Desktop only in v1                                                   |

### 1.4 Success criteria

v1 is done when the three acceptance workflows pass end to end on Paul's Windows machine, from both `craftctl` and an MCP client, and the robustness targets hold. Sequence diagrams and failure paths are §9 (agent A6); this section fixes what "pass" means.

**Acceptance workflows**

| ID  | Workflow           | Passes when                                                                                                                                                                                                                                                                                                                                                     | Commands it leans on today (static)                                                                                                                                                                                                                                                                    |
|-----|--------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| AC1 | Open In            | (a) An AssetCraft selection of 1 and of 10 files opens in PhotoCraft, LightCraft and VectorCraft. (b) If the app is already running with control on, the files go to **that** window (no second instance). (c) If it isn't running, the hub launches it and attaches. (d) A missing app or a refused path returns a typed error to AssetCraft; nothing crashes. | PhotoCraft `app.open` (`photocraft` `crates/ui-egui/src/control.rs:1217` @ `fd28da5`; paths must sit under `--automation-read-root`, study O4); LightCraft `library.import`; VectorCraft `app.open` (study)                                                                                            |
| AC2 | Thumbnails         | (a) For each `test_data/library` file AssetCraft can't decode itself, the hub returns a PNG from the owning app as a job. (b) Cancelling the job stops the app-side work. (c) Any failure falls back to AssetCraft's type icon, never to an error dialog. (d) Every file ends with a thumbnail or a fallback.                                                   | VectorCraft `file.place.info {path, thumbnail}` (`vectorcraft` `crates/ui-egui/src/place.rs:192` @ `0946bc5`); `effectcraft-cli render-frame` (`effectcraft` `apps/effectcraft-cli/src/main.rs:570` @ `2c08f67`); `wordcraft-cli render` (`wordcraft` `apps/wordcraft-cli/src/main.rs:57` @ `3292ba0`) |
| AC3 | Plan → FilmCraft   | (a) Storyline Builder blocks export to OTIO. (b) The hub imports it into FilmCraft. (c) The FilmCraft timeline has the same clip count, order and durations (± 1 frame). (d) The dry run lists every step and writes nothing.                                                                                                                                   | FilmCraft `file.import` (`filmcraft` `crates/engine/src/commands.rs:1331` @ `9ed9650`) with interchange detection that accepts `.otio` (`crates/engine/src/interchange.rs:11-16`); AssetCraft OTIO writer is planned (`AGENTS.md` L3 `interchange`)                                                    |
| AC4 | Same three via MCP | AC1–AC3 run from Claude and from the local llama.cpp stack through the MCP front door; every write tool asks for confirmation per §10.3; read-only tools don't                                                                                                                                                                                                  | Hub MCP front door (new)                                                                                                                                                                                                                                                                               |

**Draft quantitative targets (flagged for review: Paul decides; A9 finalizes in §13.4)**

These are proposals, not measurements. Each needs a baseline run on Paul's RTX 5090 Windows box before it becomes a gate.

| ID  | Measure                                                        | Draft target                                                                    | How measured                                              |
|-----|----------------------------------------------------------------|---------------------------------------------------------------------------------|-----------------------------------------------------------|
| P1  | Hub-added latency on a read-only call, warm connection         | p50 ≤ 2 ms, p95 ≤ 10 ms                                                         | `craftctl call` against the fake app (§13.2), 1,000 calls |
| P2  | `craftctl apps` with 10 known apps                             | ≤ 1 s cold, ≤ 100 ms warm                                                       | Wall clock, hub started on demand                         |
| P3  | Attach to an app already running with `--control`              | ≤ 500 ms incl. handshake                                                        | Hub log timestamps                                        |
| P4  | Event delivery, app state change → subscriber                  | p95 ≤ 100 ms (≤ poll interval + 50 ms where the hub has to poll)                | Fake app emitting timestamped events                      |
| P5  | AC1 Open In, app already running                               | ≤ 1 s from command to app reply                                                 | AssetCraft → hub → app round trip                         |
| R1  | App killed mid-call                                            | Caller gets a typed error within 2 s (or the call timeout); hub stays up        | Kill the app process during a long call                   |
| R2  | Hostile frames: malformed JSON, > 4 MiB lines, wrong handshake | Rejected and connection closed; no panic; hub stays up                          | Fuzz 10,000 random frames per adapter                     |
| R3  | Stale manifest (process gone)                                  | Marked down and cleaned within one discovery pass                               | Write a manifest with a dead pid                          |
| R4  | No orphans                                                     | Zero child processes left after hub exit or crash                               | Process list before/after                                 |
| R5  | Soak                                                           | 24 h of 1 call/s plus event traffic: no growth in RSS beyond 10 % after warm-up | Long run on Windows                                       |
| R6  | Never crash rule                                               | `cargo xtask ci` clean: no `unsafe`, no `unwrap`/`expect`/`panic` outside tests | Workspace lints (inherited)                               |

The 4 MiB line figure in R2 mirrors the limit the study found in EffectCraft and VectorCraft control servers (study: `effectcraft` `apps/effectcraft/src/control_server.rs:21-23`, `vectorcraft` `apps/vectorcraft/src/control_server.rs:26-28`); SoundCraft allows about 86 MiB for plugin state (study), so the hub's own limit must be per adapter.

### 1.5 Naming

"PipelineCraft" is a working name. It reads as a render-farm pipeline and is long on a command line. Paul wants something better. Candidates in the `<Word>Craft` pattern:

**Collision check method.** Case-insensitive whole-word grep of `*.rs`, `*.md`, `*.toml` across all 13 clones for the full name and the bare word. Existing suite names found: PhotoCraft, LightCraft, FilmCraft, PdfCraft (formerly PrintCraft, `pdfcraft` `ROADMAP.md:346` @ `ac89a11`), WordCraft, DesignCraft, EffectCraft, VectorCraft (formerly DrawCraft, `vectorcraft` `ROADMAP.md:162` @ `0946bc5`), SoundCraft, ArtCraft, AssetCraft, plus GridCraft, DeckCraft, CADCraft (`craft-libs` `docs/theme-infrastructure.md:156` @ `8f5a283`). Planned in Paul's notes: StoryCraft, PrepressCraft. Outside-world check is from general knowledge only, not a trademark search.

| #   | Name           | Rationale (one line)                                                               | Full name in repos | Bare word in repos                                                                                                     | Outside the suite                               | Verdict                     |
|-----|----------------|------------------------------------------------------------------------------------|--------------------|------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------|-----------------------------|
| 1   | **RelayCraft** | It passes work from app to app, like a relay baton; says "hand-off", not "control" | 0 files            | 2 files, both comments (`effectcraft` `apps/effectcraft-web/src/frames.rs`, `wordcraft` `crates/engine/src/sample.rs`) | Generic word, no well-known Craft product       | **Recommended**             |
| 2   | HubCraft       | Literal: the suite's hub; matches the glossary term                                | 0 files            | 10 files, all unrelated ("hub" of a wheel transition in FilmCraft, a download host in ArtCraft)                        | Generic                                         | Strong runner-up; plain     |
| 3   | CueCraft       | A stage manager calling cues to each department; fits a media-production crew      | 0 files            | 43 files: **cue points** on markers (e.g. `effectcraft` `crates/engine/src/tests_markers.rs:17`)                       | Generic                                         | Avoid: clashes with markers |
| 4   | LoomCraft      | Weaves separate apps into one fabric                                               | 0 files            | 1 file (sample text)                                                                                                   | "Loom" is a well-known video app                | Possible; brand echo        |
| 5   | ConductCraft   | Conducts the apps like an orchestra                                                | 0 files            | 0 files                                                                                                                | None known                                      | Clean but awkward to say    |
| 6   | SwitchCraft    | A switchboard that patches calls between apps; pun on "witchcraft"                 | 0 files            | "switchboard" 0 files                                                                                                  | Pun is memorable; several small products use it | Possible; jokey             |
| 7   | BridgeCraft    | Bridges the apps                                                                   | 0 files            | 136 files; AssetCraft's planned L4 `bridge` crate (`AGENTS.md:36`) and AssetCraft is "Bridge-style"                    | Echoes Adobe Bridge (clean-room concern)        | Reject                      |
| 8   | MeshCraft      | A mesh of connected apps                                                           | 0 files            | 404 files; AssetCraft `crates/mesh` is 3D meshes                                                                       | 3D connotation                                  | Reject                      |

Also considered and dropped: **StageCraft** (ILM's virtual-production brand, an obvious clash in Paul's field) and **FlowCraft** (227 hits for "flow", and close to Microsoft's former "Flow" product, a clean-room concern).

**Recommendation A1-N1.** Adopt **RelayCraft**: binary `relaycraft` (hub daemon), client `craftctl` unchanged (it names the suite, not the hub, so it survives any rename), crate prefix `relaycraft-*`. It is short, collision-free in all 13 repos, describes the hand-off job, and doesn't suggest the hub *controls* apps (it relays to their own registries, NG1). Runner-up: HubCraft. **Owner: Paul** (decision D1 in §15). Until Paul decides, every section keeps writing "PipelineCraft" so the rename is one search-and-replace.

---

## 2. Glossary

One meaning per term, used the same way in every section. Where the suite already has the term, the hub uses the suite's meaning and cites it. Where the suite uses a word for something else, the collision is called out.

### 2.1 Quick reference

| Term                    | One-line definition                                                                   | Suite precedent                                                      |
|-------------------------|---------------------------------------------------------------------------------------|----------------------------------------------------------------------|
| App                     | One installed Craft program with a command registry                                   | Each repo's `apps/<app>` binary                                      |
| Command                 | One registered, named action an app can run                                           | `CommandSpec` in every engine                                        |
| Command id              | A command's stable dotted name, e.g. `nav.back`                                       | `CommandSpec.id`                                                     |
| Params                  | The JSON object passed to a command                                                   | `Session::execute(id, params)`                                       |
| Effect (command effect) | The declared side-effect class of a command                                           | None; nearest is `journal: bool`                                     |
| Job                     | A long-running call with an id, progress, cancel and a result                         | PhotoCraft `jobs.list` / `jobs.cancel`                               |
| Event                   | A message the hub pushes to subscribers when something changes                        | None on control channels; EffectCraft render-queue events in-process |
| Asset ref               | A hub-level reference two apps agree means the same asset                             | None                                                                 |
| Manifest                | A small file an app (or the hub for it) publishes saying where and how to reach it    | PdfCraft's control file `{"port","token","pid"}`                     |
| Adapter                 | Hub code that speaks one transport to one app and normalizes it                       | AssetCraft planned L4 `bridge`                                       |
| Hub                     | The PipelineCraft daemon that discovers, routes and observes                          | None                                                                 |
| Session                 | **The app's engine state object.** Not a hub concept                                  | `Session` in every engine                                            |
| Connection              | One authenticated client link to the hub (the word the hub uses instead of "session") | None                                                                 |
| Recipe                  | A saved, ordered list of hub calls across apps, run with a dry run first              | PhotoCraft actions/droplets, DesignCraft scripts (single-app)        |
| Control channel         | An app's opt-in loopback JSON-lines server started by `--control`                     | `apps/<app>/src/control_server.rs`                                   |
| Headless CLI            | `<app>-cli`, runs commands with no window                                             | `apps/<app>-cli`                                                     |
| MCP front door          | The hub's single MCP server that exposes the whole suite                              | Per-app MCP servers today                                            |
| Local app control       | API 1: the registries reached by CLI, MCP and control channel                         | All of the above                                                     |
| Storyteller web API     | API 2: ArtCraft's cloud account backend                                               | `artcraft` Storyteller services                                      |

### 2.2 Definitions

**App.** One installed Craft program that exposes a command registry: PhotoCraft, LightCraft, FilmCraft, PdfCraft, WordCraft, DesignCraft, EffectCraft, VectorCraft, SoundCraft, AssetCraft, and future StoryCraft/PrepressCraft. Identified by a lowercase **app id** equal to its binary stem (`photocraft`, `filmcraft`), the same stems AssetCraft's Open In catalog uses (`assetcraft` `STATUS.md` step 8a table @ `3a41b2b`). An app may have several running **instances**; an instance is one process (pid) of an app. *Not an app for hub purposes:* the storytold ArtCraft desktop client, which is the Storyteller front end and has no `--control` channel (grep, `artcraft` @ `d4e4ac6`, static).

**Command.** One registered action in an app's registry, with a handler, label, menu placement, optional shortcut and a params description. The suite type is `CommandSpec` (`assetcraft` `crates/engine/src/commands.rs:13` @ `3a41b2b`; `photocraft` `crates/engine/src/commands.rs:15` @ `fd28da5`; `effectcraft` `crates/engine/src/commands/mod.rs:101` @ `2c08f67`; same name in filmcraft, lightcraft, pdfcraft, designcraft, soundcraft, vectorcraft, wordcraft engines). The hub never defines commands for an app (NG1); it lists them via `engine.commands` or `<app>-cli commands --json`. The hub has a small set of its **own** commands (`hub.apps`, `hub.call`, …, defined in §6) that follow the same rules.

**Command id.** The stable dotted string that names a command inside one app, e.g. `nav.back` (`assetcraft` `crates/engine/src/commands.rs:14-15`), `file.import` (`filmcraft` `crates/engine/src/commands.rs:1331`). Ids are unique **per app**, not across the suite (`file.open` exists in several apps). The hub's address for a command is therefore **`<app>/<command.id>`**, e.g. `filmcraft/file.import`; §6.2 defines the full addressing grammar.

**Params.** The JSON object passed with a command, exactly as the app's `Session::execute(id, params)` receives it (`assetcraft` `crates/engine/src/lib.rs:211` @ `3a41b2b`, `params: &Value`). The hub passes params through unchanged. Today each registry describes params as a free-form string (`CommandSpec.params`, "a compact human-readable form", `assetcraft` `crates/engine/src/commands.rs:22-23`); JSON Schema per command is an opportunity for §6.3.

**Effect (always written "command effect" in prose; JSON field `effects`).** The declared side-effect classes of a command: `read`, `write-user-file`, `write-app-state`, `external-network`, `destructive` (outline §6.4; A4 owns the final set). Used for MCP annotations, confirmation policy and per-client permissions. **Collision:** in the suite, "effect" already means visual effects: PhotoCraft layer effects (`photocraft` `crates/ui-egui/src/layer_style.rs:17` @ `fd28da5`, `photocraft_doc::effects`), FilmCraft clip effects (`filmcraft` `crates/engine/src/commands.rs:3010`, `"effect":"motion"|index`), and the EffectCraft app itself. No `CommandSpec` has a side-effect field today; the nearest is `journal: bool`, "false for queries" (`photocraft` `crates/engine/src/commands.rs:27`; `effectcraft` `crates/engine/src/commands/mod.rs:115`). AssetCraft's Stage 1.5 roadmap adds "command effects" (BRIEF), so AssetCraft can be the first app that declares them natively.

**Job.** A call whose work outlives the request: it gets a **job id** at once, reports progress, can be cancelled, and ends `done` (with a result location) or `failed` (with an error). Suite precedent: PhotoCraft's `jobs.list` and `jobs.cancel {job}` over the control channel (`photocraft` `crates/ui-egui/src/control.rs:440-441` @ `fd28da5`), and EffectCraft's `renderQueue.render {"wait": false}` (`effectcraft` `crates/ui-egui/tests/ui_web_offload.rs:216` @ `2c08f67`). Where an app has no job model, the adapter wraps a blocking call or a CLI process as a hub job (§8.3).

**Event.** A message the hub pushes to subscribed clients: app up/down, job progress/done/failed, selection changed, asset changed (§8.2). Each has a topic, a source (`<app>` or `hub`), a timestamp and a payload. **No control channel pushes events today** (§1.1); the nearest are in-process enums such as EffectCraft's render-queue events (`effectcraft` `crates/engine/src/render_queue.rs:99`, `#[serde(tag = "event")]`). The hub synthesizes events from polling until apps can push them.

**Asset ref.** The hub's value for "this asset", which two apps must resolve to the same thing: at least an absolute path, with optional content hash, XMP document id and AssetCraft id (outline §7.1, D4; A5 owns the schema). No suite type exists. Today apps pass plain path strings (`{"paths":[str]}` in `filmcraft` `file.import`, `commands.rs:1331`). An asset ref is never a Storyteller media-file token; those belong to API 2.

**Manifest.** A small JSON file that says where and how to reach one running app instance: app id, version, pid, transport, port, auth reference, capabilities (outline §5.2; A3 owns the schema and location). Suite precedent: PdfCraft's `--control <file>`, which writes `{"port","token","pid"}` atomically with owner-only permissions (`pdfcraft` `apps/pdfcraft/src/main.rs:17,459` @ `ac89a11`). For apps that don't publish one, the hub writes it for them when it launches them.

**Adapter.** Hub code that speaks one transport (control channel, headless CLI, or MCP stdio) and turns it into the hub's one contract. One adapter per transport, with per-app quirks (port vs file `--control`, handshake kind, line limits, path roots) in config, not code (outline §4.4). Closest suite precedent: AssetCraft's planned L4 `bridge` crate, "Sessions with the sibling apps" (`assetcraft` `AGENTS.md:36` @ `3a41b2b`), and its existing hand-rolled `filmcraft-cli` driver (`crates/thumbs/src/decoders/video.rs`).

**Hub.** The PipelineCraft daemon: discovery, adapters, router, event bus, job manager, MCP front door (§4.1). On-demand, loopback-only, one per user login (D5, NG3). It holds each app's credentials and the clients authenticate to it once (§11.2).

**Session.** **Reserved for the suite meaning:** an app's engine state object through which every command runs. AssetCraft: "One window's worth of navigator state" (`assetcraft` `crates/engine/src/lib.rs:121-122` @ `3a41b2b`), with `execute`, "the only way state changes" (`:210-211`). Every sibling engine has the same `Session` (e.g. `effectcraft` `crates/ui-egui/src/control.rs:195`, `app.session.execute`). The hub must not use "session" for its own client links.

**Connection.** One authenticated link from a client (an app, `craftctl` or the MCP front door) to the hub, carrying its identity and permissions (§11.4). Used instead of "session" (A1-O3).

**Recipe.** A saved, named, ordered list of hub calls (`<app>/<command.id>` + params), where a step may take an earlier step's result as input; always run with a dry run before commit (§8.4–8.5, §9.4). Single-app precedents the format should stay close to: PhotoCraft actions and droplets (`photocraft` `apps/photocraft-cli/src/lib.rs:33-39` @ `fd28da5`, steps as `{"command": id, "params": {…}}`), DesignCraft `script FILE.jsonl` (`designcraft` `apps/designcraft-cli/src/main.rs:8`), FilmCraft `run <script.jsonl>` (study). **Collision:** `artcraft` uses `recipes` as a module name for HTTP client helpers (e.g. `crates/desktop/artcraft/src/services/grok/util/upload_image_media_file_to_grok.rs:16` @ `d4e4ac6`); unrelated, no action needed.

**Control channel.** An app's opt-in loopback server started at launch with `--control` (a port in most apps, a file path in PdfCraft), JSON lines, envelope `{"id","method","params"}` → `{"id","ok","result"|"error"}`, methods including `engine.execute {command, params}` and `engine.commands` (`photocraft` `apps/photocraft/src/main.rs:14-16` @ `fd28da5`; `designcraft` `crates/ui-egui/src/control.rs:4-5` @ `7ed5697`). PdfCraft's is JSON-RPC 2.0 (study). It is the only way to drive an app that is **already open**.

**Headless CLI.** `<app>-cli`: runs commands with no window, lists them (`commands`), and usually also hosts the app's MCP server (`mcp`) and a client for a running app (`--connect` / `app --port`). E.g. `assetcraft-cli commands` / `exec` (`assetcraft` `apps/assetcraft-cli/src/main.rs:200-201` @ `3a41b2b`), `designcraft-cli` (`apps/designcraft-cli/src/main.rs:41`). `<app>-cli commands --json` is the authoritative live command list (study README).

**MCP front door.** The hub's one MCP server that exposes the merged registry of every reachable app as tools and resources, with effect-based annotations and confirmations (§10). Distinct from each app's own MCP server, which serves one app (e.g. `designcraft-cli mcp [--connect PORT]`, `apps/designcraft-cli/src/main.rs:12`) and stays usable on its own.

**The two APIs.**
- **Local app control (API 1)**: the app command registries, reached through the headless CLI, per-app MCP over stdio, and the control channel. PipelineCraft builds on this and only this.
- **Storyteller web API (API 2)**: https://storyteller-docs.netlify.app/, ArtCraft's website and account backend (accounts, media files, credits, generation). Used only as the ArtCraft-account AI provider inside AssetCraft. Never routed through the hub (NG2). In `artcraft` it lives under `crates/desktop/artcraft/src/services/storyteller/` (@ `d4e4ac6`).

---

## Findings and opportunities from this section

| ID    | Scope                   | Opportunity                                                                        | Suggested fix                                                                                                                                                                                        | Owner                                             | Status |
|-------|-------------------------|------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------|--------|
| A1-O1 | Discovery, launch       | `--control` takes a port in most apps and a file in PdfCraft (§1.1)                | Adapter config field `control_arg: port \| file`. Upstream ask: a suite-wide `--control-file <path>` that publishes a PdfCraft-style manifest, next to `--control <port>`                            | Claude (adapter); upstream apps (PR, §14)         | open   |
| A1-O2 | Vocabulary              | "Effect" already means visual effects in the suite                                 | Always write "command effect" in prose; JSON field `effects`; never bare "effect" in hub docs                                                                                                        | Paul (confirm); Claude (apply)                    | open   |
| A1-O3 | Vocabulary              | "Session" is the app engine object everywhere                                      | Hub uses "connection" for client links; integration pass rejects "hub session"                                                                                                                       | Claude (integration agent)                        | open   |
| A1-O4 | Protocol, MCP, security | No registry declares command effects; MCP read-only hints are hand-kept name lists | Hub infers conservatively (unknown = write + destructive) from adapter config; AssetCraft Stage 1.5 declares effects natively as the reference; upstream PR adds an `effects` field to `CommandSpec` | A4, A7 (spec); Claude (AssetCraft); upstream apps | open   |
| A1-O5 | Events                  | No control channel pushes events                                                   | Adapters poll (`jobs.list` where it exists) and the hub emits synthetic events; propose an upstream `events.subscribe` method later                                                                  | A6 (spec); upstream apps                          | open   |
| A1-O6 | Recipes                 | Three single-app script formats already exist                                      | Recipe step = `{"app","command","params"}`, a superset of PhotoCraft's `{"command","params"}`, so a one-app action imports with `app` filled in                                                      | A6, A7                                            | open   |
| A1-N1 | Naming                  | "PipelineCraft" is a placeholder                                                   | Adopt RelayCraft (runner-up HubCraft); keep `craftctl`                                                                                                                                               | Paul (D1)                                         | open   |

## Not verified

Everything above is a static read; nothing was built or run. These need a live run on Paul's Windows machine.

**Next action (Paul):**

1. Confirm each app's live command list and that `engine.commands` matches it (repeat per installed app).
   ```powershell
   Set-Location F:\Script-Projects\ArtCrafting\photocraft
   cargo run -q -p photocraft-cli -- commands --json > $env:TEMP\photocraft-commands.json
   (Get-Content $env:TEMP\photocraft-commands.json -Raw | ConvertFrom-Json).Count
   ```
   Expected: a number near the study's 863 engine commands (exact count differs by version); no error.
2. Confirm the `--control` argument split (port vs file).
   ```powershell
   Set-Location F:\Script-Projects\ArtCrafting\pdfcraft
   cargo run -q -p pdfcraft -- --control $env:TEMP\pdfcraft-control.json
   # in a second PowerShell window:
   Get-Content $env:TEMP\pdfcraft-control.json
   ```
   Expected: JSON with `port`, `token`, `pid`; the port is random. Then `Set-Location F:\Script-Projects\ArtCrafting\lightcraft; cargo run -q -p lightcraft -- --control 7980` starts LightCraft listening on `127.0.0.1:7980` with no token.
3. Confirm "no auth" on LightCraft, FilmCraft, DesignCraft, and the VectorCraft/SoundCraft 7979 clash.
   ```powershell
   Set-Location F:\Script-Projects\ArtCrafting\lightcraft
   $c = New-Object System.Net.Sockets.TcpClient('127.0.0.1', 7980)
   $s = $c.GetStream(); $w = New-Object System.IO.StreamWriter($s); $r = New-Object System.IO.StreamReader($s)
   $w.WriteLine('{"id":1,"method":"engine.commands","params":{}}'); $w.Flush(); $r.ReadLine().Substring(0,200); $c.Close()
   ```
   Expected: a `{"id":1,"ok":true,...}` reply with no handshake. Repeat for FilmCraft and DesignCraft on their ports. Start VectorCraft and SoundCraft with their defaults: expect the second one to fail to bind 7979.
4. Confirm FilmCraft imports an `.otio` through `file.import` (AC3 precondition).
   ```powershell
   Set-Location F:\Script-Projects\ArtCrafting\filmcraft
   cargo run -q -p filmcraft-cli -- describe file.import
   ```
   Expected: the `file.import` entry with params `{"paths":[str],...}` (`apps/filmcraft-cli/src/main.rs:309`). Then export any sequence with `file.exportOtio` and re-import it with `file.import {"paths":["<that>.otio"]}`; expect the same clip count.
5. Baseline the draft targets P1–P5 once the fake app exists (§13.2); until then, time one `engine.commands` round trip on LightCraft with step 3's script wrapped in `Measure-Command`. Expected: well under 50 ms for the app alone, which sets the floor for P1.

## Open questions

1. **Name (D1):** RelayCraft, HubCraft, or something else? (Paul)
2. **Local llama.cpp stack as an MCP client:** does Paul's FastAPI/Instructor stack already speak MCP over stdio, or does it need streamable HTTP on loopback (or a thin MCP client shim) for AC4? This decides whether §10.5's "stdio first" covers both MCP actors in v1. (Paul)
3. **Effect class names:** keep the outline's five (`read`, `write-user-file`, `write-app-state`, `external-network`, `destructive`), and should `destructive` be a flag on a write class rather than its own class? (A4 proposes, Paul decides)
4. Are the draft P/R numbers the right order of magnitude for how Paul works (e.g. is 1 s acceptable for Open In into a running app)? (Paul)
5. Which apps count as v1 targets for AC1: the six in AssetCraft's Open In catalog, or those plus VectorCraft (needed for AC1) and EffectCraft (study O12)? (Paul)

## Inputs for downstream agents

- **All agents:** use §2 terms exactly. "Session" = app engine only; hub client links are "connections". Write "command effect", never bare "effect". Address commands as `<app>/<command.id>`; app id = lowercase binary stem. Keep writing "PipelineCraft" until D1 is decided.
- **A2 (§3):** re-verify at HEAD the "(study)" claims used here: EffectCraft/VectorCraft/SoundCraft no-auth, the 7979 clash, line limits, PdfCraft JSON-RPC 2.0, FilmCraft `run <script.jsonl>`. Record the `--control` port-vs-file split per app.
- **A3 (§4–5):** PdfCraft's control file is the manifest precedent; the adapter needs a `control_arg: port | file` config and per-app line limits (4 MiB vs ~86 MiB). AssetCraft's L4 `bridge` is the natural home if D2 lands as "crate inside AssetCraft".
- **A4 (§6):** no registry has effect metadata or JSON Schema params; plan for adapter-side inference (A1-O4) and free-form `params` strings. Command ids are unique per app only.
- **A5 (§7):** apps take plain path arrays today (`{"paths":[str]}`); an asset ref must degrade to a path for every existing command.
- **A6 (§8–9):** no pushed events anywhere; PhotoCraft `jobs.list/cancel` and EffectCraft `renderQueue.render {wait:false}` are the job precedents. AC1–AC4 and the "passes when" clauses above are the acceptance definitions; FilmCraft OTIO import goes through `file.import` detection (`interchange.rs:11-16`).
- **A7 (§10, §12):** AC4 needs both Claude and the local llama.cpp stack (open question 2). Recipe steps should be a superset of PhotoCraft's `{"command","params"}` (A1-O6). Keep `craftctl` as the client name regardless of D1.
- **A8 (§11):** every app binds 127.0.0.1; auth is token-first-line (PhotoCraft), token file (PdfCraft), per-request key file (WordCraft) or none. NG3 and NG10 (no LAN, no browser hub) are the v1 trust boundary.
- **A9 (§13):** P1–P5 and R1–R6 are drafts for §13.4; R2's 4 MiB must be per-adapter.
- **A10 (§15):** D1 recommendation is RelayCraft (runner-up HubCraft), with the collision table in §1.5 as the argument.
