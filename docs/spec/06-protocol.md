# PipelineCraft (working name): §6 Hub protocol, the contract

_2026-10-15 · spec draft v0 · status: draft (agent A4)_

- **Scope:** the wire contract between hub clients (Craft apps, `craftctl`, the MCP front door) and the hub, plus the rules the hub uses when it calls an app through an adapter: framing, addressing, params and schemas, command effects, errors, versioning, and worked examples with real command ids.
- **Project goal it serves:** one way to call any command in any Craft app (G2 Call), hand results on (G3 Chain), watch what happens (G4 Observe) and run it safely from recipes and MCP (G5 Automate), so the AC1–AC4 workflows in §1.4 run without the cloud.
- **Immediate area of improvement:** the apps already agree on most of a wire shape (`{id,method,params}` → `{id,ok,result|error}`, `engine.execute`, `engine.commands`). What's missing is a typed error, a command effect on every command, a schema where one exists, and an address that spans apps. This section adds those on the hub side, without asking any app to change first.

**Evidence basis.** Static read of the shallow clones in `/home/claude/src` at the commits listed in `01-goals-glossary.md` (photocraft `fd28da5`, lightcraft `38adc17`, filmcraft `9ed9650`, pdfcraft `ac89a11`, wordcraft `3292ba0`, designcraft `7ed5697`, effectcraft `2c08f67`, vectorcraft `0946bc5`, soundcraft `3a28a65`, assetcraft `3a41b2b`). Nothing was built or run. Inventory facts come from `03a-inventory.md` (O1–O14) and `03b-inventory.md` (O-B1–O-B13) and are cited by ID. New citations below were re-checked at HEAD for details those docs lack (error strings, timeout wording, PdfCraft line caps). Terms follow §2 exactly: **connection** (never "session") for a hub client link, **command effect** in prose with JSON field `effects`, address `<app>/<command.id>`.

---

## 6.1 Transport and framing

### 6.1.1 Recommendation: mirror the suite's JSON-lines envelope (decision D3)

The hub's client-facing protocol is the suite's own control-channel envelope, one JSON object per line:

```text
request  → {"id": <id>, "method": "<method>", "params": {…}}
reply    ← {"id": <id>, "ok": true,  "result": <any>}
reply    ← {"id": <id>, "ok": false, "error": {"code": "…", "message": "…", …}}
event    ← {"event": "<topic>", "sub": "<subscription id>", "seq": <n>, "data": {…}}
```

The request and success shape is the one seven of the eight control channels already use:

| Evidence                                                                 | What it shows                                                                                       |
|--------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------|
| `photocraft` `apps/photocraft/src/control_server.rs:88-108` @ `fd28da5`  | Reads `id`, `method`, `params` (default `{}`); copies `id` into the reply                           |
| `photocraft` `crates/ui-egui/src/control.rs:183`, `:1268` @ `fd28da5`    | Replies `{"ok": false, "error": "<string>"}`; unknown method → ``unknown method `X` ``              |
| `lightcraft` `apps/lightcraft/src/control_server.rs:97-111` @ `38adc17`  | Same `ok`/`error` reply shape                                                                       |
| `filmcraft` `apps/filmcraft/src/control_server.rs:77` @ `9ed9650`        | Same                                                                                                |
| `designcraft` `apps/designcraft/src/control_server.rs:64` @ `7ed5697`    | Same                                                                                                |
| `effectcraft` `apps/effectcraft/src/control_server.rs:63` @ `2c08f67`    | Same                                                                                                |
| `vectorcraft` `apps/vectorcraft/src/control_server.rs:40-44` @ `0946bc5` | Same                                                                                                |
| `soundcraft` `apps/soundcraft/src/control_server.rs:126-132` @ `3a28a65` | Same                                                                                                |
| `wordcraft` `apps/wordcraft/src/control_server.rs:211-245` @ `3292ba0`   | Same shape plus a top-level `key` field for auth                                                    |
| `pdfcraft` `crates/ui-egui/src/control.rs:836-845`, `:900` @ `ac89a11`   | The outlier: JSON-RPC 2.0 lines, `{"jsonrpc":"2.0","id","result"}` / `{"error":{"code","message"}}` |

Two deliberate changes from the suite shape, both additive:

1. **`error` is an object, not a string.** It carries a stable `code`, `origin`, `retryable` and `ran` (§6.5). The app's original text is kept in `message` and `appError`. A client that only prints errors still has `error.message`.
2. **Event frames.** A line with an `event` key and no `ok` is a pushed event (§6.2.6; topics are A6's). No app pushes on its control channel today (A1-O5, O8, O-B13), so this frame exists only on the hub side.

**Compatibility methods.** The hub also answers `engine.execute {command, params}` and `engine.commands` with the suite's exact reply shape (`error` as a **string**, plus an extra `errorCode` field). `command` is a hub address (`filmcraft/file.import`). This lets an existing single-app client such as `designcraft-cli app --port` (`designcraft` `apps/designcraft-cli/src/main.rs:472` @ `7ed5697`) or a PowerShell `TcpClient` script point at the hub with no change except the command string.

### 6.1.2 Why not JSON-RPC 2.0

PdfCraft's control channel and every app's MCP server already speak JSON-RPC 2.0 (`pdfcraft` `crates/ui-egui/src/control.rs:786-857` @ `ac89a11`; MCP servers per 03a §3a.1 and 03b §3.1b). It was considered and set aside for the client→hub side:

| Criterion                        | Suite JSON lines `{id,ok}`                                                            | JSON-RPC 2.0                                                                                     | Verdict       |
|----------------------------------|---------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------|---------------|
| Matches what apps speak          | 7 of 8 control channels (table above)                                                 | PdfCraft control only                                                                            | JSON lines    |
| Matches existing tooling         | Every app's PowerShell/TcpClient recipe in 03a/03b "Not verified" works unchanged     | Needs a `jsonrpc` field and a different reply check                                              | JSON lines    |
| Success test in a script         | `$r.ok`                                                                               | `$r.error -eq $null`                                                                             | JSON lines    |
| Error codes                      | None today (string); the hub adds `error.code`                                        | Numeric codes, but only 5 reserved ones are standard; app codes are free-form (`-32000` for all) | Even          |
| Batching                         | `hub.batch` method with ordered steps and dry run (§6.2.4)                            | Array batches: unordered, no dry run, no stop-on-error                                           | JSON lines    |
| Notifications                    | `event` frames, explicit subscription                                                 | Requests without `id`; ambiguous with a client-sent notification                                 | Even          |
| Standard tooling for MCP clients | Not needed: the MCP front door (§10) is already JSON-RPC 2.0 for clients that want it | Native                                                                                           | MCP covers it |

Result: the suite shape for the hub's own socket, and JSON-RPC 2.0 only where it already exists: the PdfCraft control adapter, every adapter that talks to an app's MCP over stdio, and the hub's MCP front door. This is O1's two-codec plan (`suite-jsonl`, `jsonrpc2`) seen from the protocol side.

### 6.1.3 Socket: loopback TCP in v1, a local socket later

| Option                                                    | For                                                                                                                                                                                                                                                                                                                | Against                                                                                                                                |
|-----------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------|
| **Loopback TCP `127.0.0.1:<port>`** (v1)                  | What every app does (`photocraft` `apps/photocraft/src/control_server.rs:20`, `lightcraft` `:28`, `filmcraft` `:27`, per §1.1); `std::net` only, no extra crate; same code on Windows, macOS, Linux; PowerShell can test it                                                                                        | Any local process (and a browser, via DNS rebinding or `fetch`) can reach the port, so it needs a token and an HTTP guard (below, §11) |
| Windows named pipe `\\.\pipe\PipelineCraft-<user>`        | Per-user ACL by the OS; unreachable from a browser                                                                                                                                                                                                                                                                 | Not in `std`; needs `tokio::net::windows::named_pipe` or a crate; a second code path; PowerShell testing needs `NamedPipeClientStream` |
| AF_UNIX socket in `%LOCALAPPDATA%\PipelineCraft\hub.sock` | Already in the suite: PdfCraft's single-instance socket uses `uds_windows` (MIT) because `std` has no Unix sockets on Windows and Winsock would need `unsafe` (`pdfcraft` `apps/pdfcraft/src/single_instance.rs:1-21`; `apps/pdfcraft/Cargo.toml:54` @ `ac89a11`); per-user folder ACL; unreachable from a browser | Windows 10 1803+ only; a second code path                                                                                              |

**Recommendation A4-R1.** v1 listens on loopback TCP only, port chosen by the OS (`:0`) and published in the hub manifest file next to the token (A3 owns the location; assumed `%LOCALAPPDATA%\PipelineCraft\hub.json`, owner-only ACL per O12). The framing and every method are transport-independent, so an AF_UNIX listener (PdfCraft's `uds_windows` precedent) can be added in v1.x behind the same codec. **Owner: Claude (hub)**; transport choice confirmed by **A8** in §11.

### 6.1.4 Framing rules

| Rule                         | Value (v1 default)                                                                                                                                 | Precedent                                                                                                                      |
|------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------|
| Encoding                     | UTF-8, no BOM. Invalid UTF-8 is a `bad_frame` error and closes the connection                                                                      | All apps read `String` lines                                                                                                   |
| Delimiter                    | `\n`; a trailing `\r` is stripped; empty lines are ignored                                                                                         | `photocraft` `apps/photocraft/src/control_server.rs:71` skips blank lines                                                      |
| One frame per line           | Exactly one JSON **object**. Arrays, scalars and multi-object lines are `bad_frame`                                                                | Suite                                                                                                                          |
| Max line before `hub.hello`  | 4 KiB; longer → `frame_too_large`, close                                                                                                           | `pdfcraft` `crates/ui-egui/src/control.rs:808`, `:863` (`MAX_AUTH_LINE = 4 * 1024`)                                            |
| Max request line after hello | 4 MiB; longer → `frame_too_large`, close                                                                                                           | LightCraft, FilmCraft, DesignCraft, EffectCraft, VectorCraft 4 MiB (03a §3a.3–3a.4, 03b §3.1b); PhotoCraft and WordCraft 1 MiB |
| Max reply/event line         | 32 MiB. A larger result is refused as `result_too_large`, with advice to pass an output path (§7 handoff by file, A5)                              | SoundCraft accepts ~86 MiB for plugin state (03b §3.1b), so the hub→app side is per adapter, not this limit                    |
| Hub→app line limits          | Per adapter config, never above the app's own limit                                                                                                | R2 in §1.4                                                                                                                     |
| Request-only guard           | The first line that is not a JSON object (e.g. `POST / HTTP/1.1`) gets one `bad_frame` error and the connection closes                             | LightCraft/FilmCraft/DesignCraft guard (03a §3a.3; 03b §3.2b); SoundCraft's missing guard is O-B2                              |
| First frame                  | Must be `hub.hello` (§6.6) within 5 s, or the connection closes                                                                                    | PhotoCraft `auth` first frame (`apps/photocraft/src/control_server.rs:74-81`); LightCraft 5 s no-frame give-up (03a §3a.3)     |
| Idle timeout                 | 5 min with no frame and no open subscription                                                                                                       | VectorCraft 5 min idle (03b §3.1b)                                                                                             |
| Request `id`                 | Client-chosen, unique among the connection's in-flight requests. Integer (0 … 2^53−1) or string (1–128 chars). Reused in-flight id → `bad_request` | PhotoCraft examples use both `"auth"` and `1` (03a Not verified #3)                                                            |
| Unknown top-level keys       | Rejected (`bad_request`). Extensions go in `params` or an optional `meta` object (`meta.trace`, `meta.client`)                                     | Hostile-input rule (`assetcraft` `AGENTS.md` non-negotiable 4)                                                                 |
| In-flight per connection     | Up to 32; replies may arrive out of order and are matched by `id`                                                                                  | New (apps are one request at a time per connection; `photocraft` `control_server.rs:88-104` blocks on each reply)              |
| Connections                  | Up to 32 per hub; the 33rd gets `busy` and close                                                                                                   | Apps allow 16 (03a §3a.2)                                                                                                      |

**Hub→app ids.** The hub never forwards a client's `id` to an app. Each adapter connection numbers its own requests from 1. Two clients that both send `"id": 1` can't collide inside an app, and an app reply can't be delivered to the wrong client.

**Hub→app concurrency.** Every app serves one request at a time per connection, and runs commands on its UI thread anyway. An adapter keeps a small pool of connections per instance (default 2, max 4, well under the apps' 16) and queues the rest. A long `wait: true` call therefore can't block a quick `engine.commands` refresh.

---

## 6.2 Addressing and hub methods

### 6.2.1 Hub methods

All hub methods are commands in the hub's own registry under the `hub.` namespace, so they follow the suite rule "everything is a command" (`assetcraft` `AGENTS.md` non-negotiable 3) and get command effects like any other.

| Method            | Params (camelCase)                                                                        | Result                                                                        | Command effect               | Owner of semantics |
|-------------------|-------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------|------------------------------|--------------------|
| `hub.hello`       | `{protocol, client: {name, version, kind}, auth: {token}, features?}`                     | `{protocol, hub: {version}, connection, features, limits, deprecated}`        | read                         | A4 (§6.6), A8 auth |
| `hub.ping`        | `{}`                                                                                      | `{now}`                                                                       | read                         | A4                 |
| `hub.apps`        | `{running?: bool, installed?: bool}`                                                      | `[{app, version, instances: [{instance, mode, pid?, transport, health}]}]`    | read                         | A3 (§5)            |
| `hub.commands`    | `{app?, instance?, filter?, effects?: [class], kind?: cmd\|method\|tool, refresh?: bool}` | `{registryHash, commands: [CommandInfo]}`                                     | read                         | A4 (§6.3)          |
| `hub.describe`    | `{target}`                                                                                | `CommandInfo` with `paramsSchema` and `resultSchema` when known               | read                         | A4                 |
| `hub.route`       | `{capability, prefer?: [app], params?}`                                                   | `{candidates: [{target, params, effects}]}` in preference order; runs nothing | read                         | A4 (§6.2.5)        |
| `hub.call`        | `{target, params?, wait?: bool, timeoutMs?, dryRun?: bool, confirm?: string}`             | the command's result, `{job}` when `wait: false`, or a plan when `dryRun`     | of the target                | A4                 |
| `hub.batch`       | `{steps: [{target, params?}], stopOnError?: bool (true), dryRun?: bool, confirm?}`        | `{steps: [{target, ok, result\|error}]}`                                      | union of steps               | A4; refs A6        |
| `hub.subscribe`   | `{topics: [pattern], filter?}`                                                            | `{sub}` then `event` frames                                                   | read                         | A6 (§8)            |
| `hub.unsubscribe` | `{sub}`                                                                                   | `{}`                                                                          | read                         | A6                 |
| `hub.job.list`    | `{app?, state?}`                                                                          | `[{job, target, state, progress, result?, error?}]`                           | read                         | A6 (§8.3)          |
| `hub.job.get`     | `{job}`                                                                                   | one job                                                                       | read                         | A6                 |
| `hub.job.wait`    | `{job, timeoutMs?}`                                                                       | the job when done, or `timeout_not_run`-style `job_still_running`             | read                         | A6                 |
| `hub.job.cancel`  | `{job}`                                                                                   | `{cancelled: bool}`                                                           | write-app-state              | A6                 |
| `hub.app.launch`  | `{app, mode: gui\|headless}`                                                              | `{instance}`                                                                  | write-app-state              | A3 (§5.3)          |
| `hub.app.stop`    | `{instance}`                                                                              | `{}`                                                                          | write-app-state, destructive | A3                 |
| `engine.execute`  | `{command: <address>, params?}`                                                           | suite shape (§6.1.1 compatibility)                                            | of the target                | A4                 |
| `engine.commands` | `{app?}`                                                                                  | suite shape                                                                   | read                         | A4                 |

The job and subscription methods are named here so §8 can rely on them; A6 owns their payloads. `hub.app.*` is A3's; the names are a proposal.

**Key style.** Hub-defined JSON keys are camelCase (`timeoutMs`, `dryRun`, `disabledReason`), matching the suite majority (PhotoCraft `disabledReason`, `maxSide`; EffectCraft `enabledOnly`). Command effect class values are kebab-case per §2 (`write-user-file`). Error codes are snake_case (`unknown_command`). App params and results pass through untouched, whatever their style.

### 6.2.2 Target grammar

```abnf
target     = address / capability
address    = app [ "@" instance ] "/" [ kind ":" ] name
app        = %x61-7A *31( %x61-7A / DIGIT / "-" )       ; lowercase binary stem, §2 "App"
instance   = 1*16( ALPHA / DIGIT ) / "pid=" 1*10DIGIT / "gui" / "headless" / "new"
kind       = "cmd" / "method" / "tool"                    ; default "cmd"
name       = 1*128( ALPHA / DIGIT / "." / "_" / "-" )      ; upstream spelling, byte for byte
capability = "cap:" 1*64( %x61-7A / DIGIT / "." / "-" )
```

| Part       | Meaning                                                                                                                                                                                                                                                           |
|------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `app`      | App id from §2: `photocraft`, `filmcraft`, `assetcraft`, …                                                                                                                                                                                                        |
| `instance` | Omitted = resolve (below). `p1`, `p2` … = hub-assigned instance id from `hub.apps`. `pid=1234` = that process. `gui` = the attached GUI instance. `headless` = an adapter-owned CLI/MCP process. `new` = launch a fresh instance for this call (A3's policy)      |
| `kind`     | `cmd` = a registry command run through `engine.execute` (or the app's equivalent). `method` = a raw control-channel method that is not a registry command (`app.open`, `ui.render`, `render.frame`, `ui.open`). `tool` = an app MCP tool (`doc_open` in PdfCraft) |
| `name`     | The upstream id exactly as the app spells it                                                                                                                                                                                                                      |

Why `kind` exists: some of the most useful handoff calls are **not** registry commands. PhotoCraft `app.open` is a control method (`photocraft` `crates/ui-egui/src/control.rs:1217` @ `fd28da5`). EffectCraft `render.frame` is a control method (03b §3.1b). PdfCraft's typed surface is its 163 tools, and its control channel can't pass params to registry commands (O6). DesignCraft's control `app.open` wraps `file.open` (03b §3.1b), and AssetCraft already has a registry `app.*` namespace (`app.state`, `app.commands`, 03b §3.4). Without `kind`, `designcraft/app.open` and `assetcraft/app.commands` would need guessing. With it, the address says which surface runs.

Examples: `filmcraft/file.import`, `photocraft/method:app.open`, `pdfcraft@headless/tool:doc_open`, `lightcraft@p2/app.export`, `cap:timeline.export.otio`.

**Instance resolution when `@instance` is omitted** (deterministic; A3 may refine):

1. `kind = method` and the name starts `ui.` or `app.`: GUI instances only.
2. Exactly one GUI instance attached: use it.
3. No GUI instance and the adapter has a headless mode for this command: use (or start) the headless instance.
4. Two or more GUI instances: `ambiguous_target`, with `data.candidates` listing instance ids. The hub never picks one silently, because Open In must reach the window the user means (AC1b).
5. Nothing reachable: `app_unavailable`, with `data.launchable: bool`.

### 6.2.3 Id normalization: alias, never rename

Apps disagree on id style: dotted camelCase in most (`file.saveAs`, `file.exportPdf`), dotted snake_case in SoundCraft (`session.save_as`, `soundcraft` `crates/engine/src/cmd/file.rs:24` @ `3a28a65`) and PdfCraft's registry (`file.save_as`, 03a §3a.5), and `[a-z_]` names for PdfCraft tools.

Rules:

| ID | Rule                                                                                                                                                                                                                                                                                                                                                              |
|----|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| N1 | The **canonical id is the upstream id, byte for byte**. The hub never stores, emits or documents a rewritten id as if it were the app's                                                                                                                                                                                                                           |
| N2 | Matching is exact first. Ids are case-sensitive, as in every engine's `find()`                                                                                                                                                                                                                                                                                    |
| N3 | On an exact miss, the hub computes the **fold key**: lowercase, then remove every `_` and `-` inside segments, keep the dots. `session.save_as` → `session.saveas`; `file.saveAs` → `file.saveas`. If exactly one command **in that app** has the same fold key, the call runs it and the reply carries `"resolved": "<canonical id>"` plus a `folded_id` warning |
| N4 | At registry load, the hub checks fold keys for collisions per app. Any colliding ids lose folding (exact only) and the collision is logged. This is a contract test (§13.2)                                                                                                                                                                                       |
| N5 | Explicit **aliases** live in the app's profile (§6.3.2): `aliases = { "file.open" = "document.open" }` for VectorCraft, whose engine opens with `document.open` and whose `file.open` is UI-only (03b §3.1b). An alias is listed in `CommandInfo.aliases`, never as its own command                                                                               |
| N6 | Folding never crosses apps. Cross-app sameness ("open this asset") is a **capability** (§6.2.5), not an id                                                                                                                                                                                                                                                        |
| N7 | Folding applies to `cmd` and `tool` kinds, never to `method` (control methods are few and fixed per adapter)                                                                                                                                                                                                                                                      |
| N8 | A folded match still goes through the command-effect check of the **resolved** command, so a fold can never skip a confirmation                                                                                                                                                                                                                                   |

### 6.2.4 Batch

`hub.batch` runs steps in order on one connection. `stopOnError` defaults to true. Limit: 256 steps, the same as PhotoCraft's `MAX_BATCH_STEPS` (`photocraft` `crates/automation/src/security.rs:20-21` @ `fd28da5`). Steps may target different apps. The batch's command effects are the union of its steps', and one `confirm` covers the batch (the confirmation prompt lists every step). With `dryRun: true`, nothing is dispatched: each step is resolved, validated (§6.3) and given its effects, and the reply is the plan. Passing one step's result into a later step is A6's (§8.4); the step object reserves the keys `as` and `from` for it.

### 6.2.5 Capability routing

A **capability** is a hub-defined verb that several apps can satisfy with their own commands: "whoever can export OTIO". Capabilities are declared per app in its profile, as a mapping to a real address plus a params template. They never invent behaviour an app doesn't have (NG1).

| Capability             | Provided by (address and fixed params)                                                                                                                                                                  | Evidence                                                                                                                             |
|------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------|
| `asset.open`           | `photocraft/method:app.open {path}`; `filmcraft/file.open {path}`; `vectorcraft/document.open {path}`; `soundcraft/session.open {path}`; `pdfcraft/method:ui.open {path}`; `wordcraft/file.open {path}` | 03a §3a.8 row "Open-file command"; 03b §3.3b                                                                                         |
| `asset.import`         | `filmcraft/file.import {paths}`; `lightcraft/library.import {paths}`; `effectcraft/file.import {paths}`                                                                                                 | `filmcraft` `crates/engine/src/commands.rs:1331` @ `9ed9650`; 03a §3a.3; 03b §3.1b                                                   |
| `timeline.export.otio` | `filmcraft/file.exportOtio {path}`; `effectcraft/file.exportTimeline {path, format: "otio"}`                                                                                                            | `filmcraft` `crates/engine/src/commands.rs:1468` @ `9ed9650`; `effectcraft` `crates/engine/src/commands/timeline.rs:133` @ `2c08f67` |
| `asset.thumbnail`      | `vectorcraft/file.place.info {path, thumbnail}`; `effectcraft/method:render.frame {path, max_side}`; others by recipe (O14)                                                                             | 03b §3.1b; O14                                                                                                                       |

Resolution: `hub.route {capability}` lists candidates in order: the caller's `prefer` list, then a running GUI instance, then a headless one, then a launchable app. `hub.call {target: "cap:…"}` runs the first candidate. The reply always carries `"resolved": "<address>"`, so a caller (and the audit log, §11.5) knows which app ran. The capability names are a proposal that matches O-B8 (`asset.open`, `asset.export`, `asset.thumbnail`). **Broadcast** (send to every instance) is not offered for commands in v1, because a write to "all apps" has no sensible confirmation. Broadcast exists only as events (A6).

### 6.2.6 Event frame (placeholder for A6)

```json
{"event": "job.progress", "sub": "s1", "seq": 42, "source": "photocraft@p1", "at": "2026-10-15T09:12:03.120Z", "data": {"job": "j7", "progress": 0.4}}
```

`seq` is per subscription and strictly increasing, so a client can tell it dropped frames. Topic names, payloads, back-pressure and polling-to-event synthesis belong to §8 (A6).

---

## 6.3 Params and results

### 6.3.1 Where a schema comes from

The hub passes `params` to the app unchanged (§2 "Params"). A schema is for validation before dispatch, for `hub.describe`, and for MCP tool input schemas (§10). Sources, best first:

| Rank | Source                                   | Apps today                                                                                                                                                                                                           | Validation mode                                                |
|------|------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------|
| 1    | Hub **schema overlay** (curated, §6.3.2) | none yet; v1 writes them for the AC1–AC3 commands only                                                                                                                                                               | `strict`: reject before dispatch                               |
| 2    | App-published typed schema               | PdfCraft tools: `input_schema`, JSON Schema draft 2020-12 subset (`pdfcraft` `crates/automation/src/tools.rs:10-23` @ `ac89a11`); MCP tool schemas in PhotoCraft (25 tools) and others                               | `strict`                                                       |
| 3    | App-derived untyped schema               | EffectCraft `params_schema`: properties with `description` only, no types, `required`, and `additionalProperties: false` when the doc parses (`effectcraft` `crates/engine/src/commands/mod.rs:341-365` @ `2c08f67`) | `advisory`: check unknown and missing keys; warn, don't reject |
| 4    | Prose `params` doc string                | every other registry (`CommandSpec.params: &str`; 03a §3a.8 "Params schema")                                                                                                                                         | `opaque`: no validation; `paramsDoc` shown to callers          |
| 5    | Nothing                                  | PdfCraft registry ids without a linked tool (03a §3a.5)                                                                                                                                                              | `opaque`; `paramsSchema: {"type": "object"}`                   |

`CommandInfo.paramsSchemaSource` says which rank applied (`overlay`, `app`, `derived`, `doc`, `none`), and `validation` says the mode. In `advisory` mode the warnings come back in `warnings[]` and the call still runs. The app's own param checks (e.g. PhotoCraft `crates/engine/src/param_check.rs`, 03a §3a.2) stay the final word, and their errors map to `bad_params` (§6.5).

Results have no schema in any app today. The overlay may add `result` schemas for commands the hub chains (§8.4). Otherwise results are opaque JSON.

### 6.3.2 App profile: schema overlay and effects table

Each app gets one hub-side **profile** folder, `profiles/<app>/`, versioned with the hub and keyed by app version range. It is adapter config (outline §4.4), not code.

```text
profiles/
  filmcraft/
    profile.toml      # adapter quirks (A3: codec, auth, ports, limits), aliases, capabilities
    effects.toml      # curated command effects (§6.4.3)
    schemas.json      # schema overlay (this section)
```

`schemas.json` format:

```json
{
  "app": "filmcraft",
  "appVersion": ">=0.8.0, <0.9.0",
  "evidence": {"repo": "filmcraft", "commit": "9ed9650", "method": "static-read"},
  "commands": {
    "file.import": {
      "params": {
        "type": "object",
        "properties": {
          "paths": {"type": "array", "items": {"type": "string"}, "minItems": 1},
          "path": {"type": "string", "description": "single-path form the handler also accepts"},
          "bin": {"type": "integer", "minimum": 0},
          "imageSequence": {"type": "boolean"},
          "replace": {"type": "boolean"}
        },
        "additionalProperties": false
      },
      "evidence": "crates/engine/src/commands.rs:1331-1338"
    },
    "file.exportOtio": {
      "params": {"type": "object", "properties": {"path": {"type": "string"}}, "required": ["path"], "additionalProperties": false},
      "evidence": "crates/engine/src/commands.rs:1468"
    }
  }
}
```

Rules:

| ID | Rule                                                                                                                                                                                                                                        |
|----|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| S1 | An overlay row needs an `evidence` cite (path:line at the profile's commit). No evidence, no row                                                                                                                                            |
| S2 | Out-of-range app version: the overlay is **ignored** for that instance (fall back to ranks 2–5) and `hub.apps` reports `profile: stale`. A stale strict schema must never reject a valid call                                               |
| S3 | Overlay schemas use the same JSON Schema subset PdfCraft uses (draft 2020-12: `type`, `properties`, `required`, `items`, `enum`, `minimum`/`maximum`, `minItems`/`maxItems`, `additionalProperties`), so one validator serves ranks 1 and 2 |
| S4 | A contract test (§13.2) runs every overlay against the live `engine.commands` list: every overlay id must exist, and every `required` key must appear in the app's `params` doc string                                                      |
| S5 | Start small: v1 overlays cover only the commands AC1–AC3 call (about 15 ids). Everything else stays opaque until a workflow needs it                                                                                                        |

### 6.3.3 Normalized `CommandInfo`

This is O9's normalized entry, with the protocol fields added:

```json
{
  "address": "filmcraft/file.import",
  "app": "filmcraft",
  "kind": "cmd",
  "id": "file.import",
  "label": "Import…",
  "menuPath": ["File"],
  "shortcut": "Cmd+I",
  "enabled": true,
  "disabledReason": null,
  "paramsDoc": "{\"paths\":[str],\"bin\":binId?,\"imageSequence\":bool?,\"replace\":bool?}",
  "paramsSchema": {"type": "object", "…": "…"},
  "paramsSchemaSource": "overlay",
  "validation": "strict",
  "effects": {"classes": ["write-app-state"], "destructive": false, "undoable": true, "idempotent": false, "confirm": false, "source": "curated"},
  "aliases": [],
  "capabilities": ["asset.import"],
  "ui": false
}
```

Per-app key mapping into it: `disabledReason` ← `disabledReason` (PhotoCraft), `disabled_reason` (LightCraft, DesignCraft, VectorCraft, SoundCraft), `why` (EffectCraft); `menuPath` ← array `menu`, or WordCraft's `location` string split on ` › `, or PdfCraft's `Option<&str>`; `ui` ← `"ui": true` (DesignCraft, VectorCraft). Sources: 03a §3a.8, 03b §3.3b, O9.

---

## 6.4 The command effects model

### 6.4.1 Classes and flags: resolving A1's open question 3

A1 left open whether `destructive` is a fifth class or a flag on a write class (`01-goals-glossary.md` Open question 3). **Recommendation A4-R2: four classes plus a `destructive` flag.**

| Class              | Meaning                                                                                                      | Examples (real ids)                                                                                             |
|--------------------|--------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| `read`             | Changes no document, setting or file. May change view-only state (zoom, scroll)                              | `filmcraft/command.list`, `photocraft/jobs.list`, `lightcraft/library.importPreview`, `pdfcraft/tool:doc_info`  |
| `write-app-state`  | Changes in-app state: an open document, a project, a library catalog, a selection, preferences, open windows | `filmcraft/file.import`, `photocraft/method:app.open`, `lightcraft/library.import`, `designcraft/file.place`    |
| `write-user-file`  | Creates, overwrites, moves or deletes a file outside the app's own private state                             | `filmcraft/file.exportOtio`, `lightcraft/app.export`, `wordcraft/file.saveAs`, `pdfcraft/tool:doc_save`         |
| `external-network` | Talks to anything beyond the loopback interface                                                              | None found in a local app registry at HEAD (static). Reserved for AI providers (AssetCraft) and future commands |

| Flag          | Type          | Meaning                                                                                                                                                                                                     |
|---------------|---------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `destructive` | bool          | The call can lose work that isn't trivially recoverable: overwrite an existing file, delete content, discard unsaved changes, quit an app                                                                   |
| `undoable`    | bool or null  | The app records an undo step (`null` = unknown)                                                                                                                                                             |
| `idempotent`  | bool          | Repeating the call with the same params has no further effect. Gates automatic retry after `ran: maybe` (§6.5)                                                                                              |
| `confirm`     | bool, derived | The hub's default: `true` if `destructive`, or any class other than `read`/`write-app-state`, or the effects came from `default`. §10.3 and §11.4 may tighten it per connection, never loosen it below this |
| `source`      | enum          | `declared` (the app said so), `curated` (profile row), `inferred` (from app metadata, §6.4.4), `default` (unknown command, §6.4.2)                                                                          |

Why a flag and not a class:

1. **"Destructive" qualifies a write; it isn't a different surface.** `wordcraft/file.open` (replaces a dirty document, O5) and `filmcraft/file.exportOtio` over an existing file are both destructive, but one writes app state and the other writes a user file. A single `destructive` class would lose which.
2. **It matches every existing metadata source.** PdfCraft has two independent bools, `read_only` and `destructive` (`pdfcraft` `crates/automation/src/tools.rs:17-20` @ `ac89a11`). MCP has `readOnlyHint` and `destructiveHint` as separate annotations (PdfCraft `crates/automation/src/mcp.rs:482`). A flag maps 1:1 to both, which A7 needs for §10.2.
3. **Classes stay a set; flags stay scalars.** "Does this touch files?" and "can this lose work?" are different permission questions in §11.4.

The JSON field is `effects`, an object, so the glossary's field name holds and new flags can be added in a minor version. The class list is a **set**: `["write-app-state", "write-user-file"]` is normal for a command that edits a document and saves it.

**Gap noted (A4-F4).** AssetCraft `file.openIn` and `file.run` start other programs (03b §3.4, `assetcraft` `crates/engine/src/commands.rs:122-129` @ `3a41b2b`). That is none of the four classes. Interim: curate as `write-app-state` with `confirm: true` and a `note`. Paul decides whether v0.2 adds an `external-process` class.

### 6.4.2 Unknown commands default to the worst case

A command with no declared, curated or inferred effects gets:

```json
{"classes": ["write-app-state", "write-user-file"], "destructive": true, "undoable": null, "idempotent": false, "confirm": true, "source": "default"}
```

An app profile with `network_capable = true` also adds `external-network` to the default. No local app needs it at HEAD (static). This matches A1-O4 ("unknown = write + destructive"). Consequences: an MCP client sees every unknown command as destructive and must confirm, the hub never auto-retries it, and `hub.commands {effects: ["read"]}` never lists it. Being wrong in this direction costs a prompt; being wrong in the other direction can cost Paul's files.

### 6.4.3 Curated effects table: `profiles/<app>/effects.toml`

TOML, because it is hand-edited, row-heavy and needs comments. Exact rows win over pattern rules; the first matching pattern wins; anything unmatched falls to inference (§6.4.4), then to the default.

```toml
# profiles/filmcraft/effects.toml
[meta]
app         = "filmcraft"
app_version = ">=0.8.0, <0.9.0"
evidence    = { repo = "filmcraft", commit = "9ed9650", method = "static-read" }
network_capable = false

# Exact rows. `evidence` is required (rule S1 applies to effects too).
[command."command.list"]
classes  = ["read"]
evidence = "crates/engine/src/commands.rs:3373"

[command."file.import"]
classes  = ["write-app-state"]
undoable = true
evidence = "crates/engine/src/commands.rs:1331"

[command."file.exportOtio"]
classes     = ["write-user-file"]
destructive = true          # overwrites `path` if it exists (not confirmed live: Not verified #5)
idempotent  = true
evidence    = "crates/engine/src/commands.rs:1468"

[command."jobs.cancel"]
classes  = ["write-app-state"]
evidence = "crates/engine/src/commands.rs:1606-1608"

# Pattern rules, in order. `*` matches within one segment, `**` across segments.
[[rule]]
match   = "file.export*"
classes = ["write-user-file"]
destructive = true

[[rule]]
match   = "ui.**"
classes = ["write-app-state"]

# Tighten-only overrides of app-declared effects (§6.4.4 rule E2).
[override."file.open"]
destructive = true
reason      = "replaces the open project without a save prompt over control (needs live check)"
```

`craftctl` (§12) gets a `profile check <app>` command that loads the TOML, rejects unknown keys, unknown classes and missing `evidence`, and lists every live command that would fall to `default`. That list is the to-do list for curation.

### 6.4.4 Mapping existing metadata to command effects

Precedence, highest first:

| ID | Rule                                                                                                                                 |
|----|--------------------------------------------------------------------------------------------------------------------------------------|
| E1 | **Declared** effects from the app (AssetCraft Stage 1.5, O-B5/O-B9; any upstream `effects` field, O4) win, `source: declared`        |
| E2 | A curated `[override."id"]` can only **tighten** declared effects (add classes, set `destructive`/`confirm` true), never loosen them |
| E3 | With no declared effects, an exact curated row wins, then the first matching `[[rule]]`, `source: curated`                           |
| E4 | Otherwise inference from the metadata below, `source: inferred`                                                                      |
| E5 | Otherwise the default (§6.4.2), `source: default`                                                                                    |

Inference table (E4). Each row says what the source field really means, which is why some map less directly than their names suggest.

| App metadata                           | Where (cite)                                                                                                                                                                                                           | Visible to the hub at runtime?                                                              | Maps to                                                                                                                                                                                                                                                                                                                          |
|----------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| PdfCraft `read_only: true`             | `pdfcraft` `crates/automation/src/tools.rs:17-18` @ `ac89a11` ("Does not change any document or file")                                                                                                                 | Yes, MCP `tools/list` annotations and CLI `tools` (O10: `destructive` missing from `tools`) | `read`, `idempotent: true`                                                                                                                                                                                                                                                                                                       |
| PdfCraft `read_only: false`            | same                                                                                                                                                                                                                   | Yes                                                                                         | `write-app-state`; plus `write-user-file` for the tools PdfCraft itself lists as file writers in its idempotent set: `doc_save`, `doc_optimize`, `doc_export_data`, `measure_export`, `doc_export_images`, `doc_export_office`, `doc_export_all_images`, `doc_export_text`, `image_save` (`crates/automation/src/mcp.rs:482`)    |
| PdfCraft `destructive: true`           | `tools.rs:19-20` ("May remove content or overwrite files")                                                                                                                                                             | Yes (MCP `destructiveHint`)                                                                 | `destructive: true`                                                                                                                                                                                                                                                                                                              |
| `journal: false` (query forms)         | `photocraft` `crates/engine/src/commands.rs:27`; `lightcraft` `crates/engine/src/cmd/mod.rs:123-130`; `filmcraft` `crates/engine/src/commands.rs:22-47`; DesignCraft, EffectCraft, VectorCraft, SoundCraft (03b §3.1b) | **No.** Not serialized in any `command.list` (O4; `filmcraft` `commands.rs:3379`)           | `read`, **but only** for ids that match no write pattern (`file.*`, `*.export*`, `*.save*`, `jobs.cancel`, `activity.cancel`, `app.quit`). Counter-example: PhotoCraft `jobs.cancel` is `journal: false` and cancels work (`photocraft` `crates/engine/src/jobs.rs:853-860`)                                                     |
| `journal: true`                        | same                                                                                                                                                                                                                   | No                                                                                          | `write-app-state`; plus the write patterns above add `write-user-file`                                                                                                                                                                                                                                                           |
| WordCraft `mutates: false` (`.pure()`) | `wordcraft` `crates/engine/src/lib.rs:58-100`; published in `commands --json` only (03a §3a.6)                                                                                                                         | CLI only; the control `engine.commands` omits it (`crates/ui-egui/src/control.rs:135-149`)  | **Not** `read`. It means "doesn't change the document", and `file.open`, `file.save`, `file.saveAs` are all `.pure()` (`crates/engine/src/cmd/file.rs:68-76` @ `3292ba0`). Map to `read` only outside `file.*`/`*.export*`; `file.save*`/`file.export*` → `write-user-file`; `file.open` → `write-app-state`, `destructive` (O5) |
| WordCraft `mutates: true`              | same                                                                                                                                                                                                                   | CLI only                                                                                    | `write-app-state`, `undoable: true`                                                                                                                                                                                                                                                                                              |
| `undoable` (DesignCraft, SoundCraft)   | `designcraft` `crates/engine/src/cmd/mod.rs:81-82`; `soundcraft` `crates/engine/src/cmd/mod.rs:60-61`                                                                                                                  | No                                                                                          | sets `undoable`; `noundo` + `journal` → `write-app-state`, `undoable: false`                                                                                                                                                                                                                                                     |
| SoundCraft `view: true`                | `soundcraft` `crates/engine/src/cmd/mod.rs:62-63` ("Only touches `edit.zoom`")                                                                                                                                         | No                                                                                          | `read` (view-only state is allowed in `read`, §6.4.1)                                                                                                                                                                                                                                                                            |
| MCP hint lists (per tool)              | EffectCraft `READ_ONLY`, `WRITES_FILES` (`effectcraft` `crates/automation/src/tools.rs:37-38`, `:66`, `:83`); PhotoCraft `conventions.rs:11-24`; FilmCraft annotations (03a §3a.4)                                     | Yes, for `tool` kind only                                                                   | `readOnlyHint` → `read`; `WRITES_FILES` → `write-user-file`; `destructiveHint` → `destructive`. **Ignored** for LightCraft's generated `cmd_*` tools, which are all `destructiveHint: true` (03a §3a.3)                                                                                                                          |
| `"ui": true` entries                   | DesignCraft, VectorCraft `engine.commands` (03b §3.1b)                                                                                                                                                                 | Yes                                                                                         | no inference; falls to curated rules (`ui.**` → `write-app-state`)                                                                                                                                                                                                                                                               |

The practical consequence: the only runtime-visible inputs are PdfCraft's tool flags, WordCraft's `mutates` and MCP hints. `journal`, `undoable` and `view` are only in source. **Opportunity A4-F5:** `craftctl profile seed <app> --from-source <repo>` does a one-off static scan of the app's `cmd!`/`query!` forms at a pinned commit and writes candidate rows into `effects.toml` for Paul to review. The hub never reads app source at runtime. When O4's upstream PRs emit `readOnly` in `command.list`, the scan is no longer needed for that app.

---

## 6.5 Errors

### 6.5.1 Error object

```json
{
  "code": "disabled",
  "message": "command `file.exportOtio` is not available right now: no sequence is open",
  "origin": "app",
  "retryable": false,
  "ran": "no",
  "app": "filmcraft",
  "target": "filmcraft@p1/file.exportOtio",
  "appError": "command `file.exportOtio` is not available right now: no sequence is open",
  "data": {"reason": "no sequence is open"}
}
```

| Field       | Meaning                                                                                                                                                |
|-------------|--------------------------------------------------------------------------------------------------------------------------------------------------------|
| `code`      | Stable snake_case code from §6.5.2. Clients branch on this                                                                                             |
| `message`   | Human text. For app errors, the app's own text                                                                                                         |
| `origin`    | `hub` (the hub refused), `adapter` (the link between hub and app failed), `app` (the app answered with an error)                                       |
| `retryable` | The hub's advice: the same request may succeed later without changes                                                                                   |
| `ran`       | `no` (the app never started it: safe to re-send), `maybe` (it may have applied), `yes` (it ran and failed or partly applied). The basis for safe retry |
| `appError`  | The raw app error, unchanged: a string from suite-jsonl apps, `{code, message}` from PdfCraft, the MCP `content` text when `isError` is true           |
| `data`      | Code-specific detail (`candidates`, `suggestions`, `confirm`, `limit`, `reason`)                                                                       |

**Hub auto-retry rule.** The hub retries on its own only when `retryable && (ran == "no" || effects.idempotent || effects.classes == {read})`, at most twice, with 250 ms then 1 s backoff. Everything else goes back to the caller.

### 6.5.2 Code table

| Code                    | Origin  | Retryable | `ran` | Meaning                                                                                  |
|-------------------------|---------|-----------|-------|------------------------------------------------------------------------------------------|
| `bad_frame`             | hub     | no        | no    | Line isn't a JSON object or isn't UTF-8. Connection closes                               |
| `frame_too_large`       | hub     | no        | no    | Line over the limit (§6.1.4). Connection closes                                          |
| `bad_request`           | hub     | no        | no    | Envelope invalid: missing `method`, bad `id`, unknown top-level key                      |
| `hello_required`        | hub     | no        | no    | First frame wasn't `hub.hello`. Connection closes                                        |
| `unauthenticated`       | hub     | no        | no    | Bad or missing token in `hub.hello` (A8)                                                 |
| `protocol_version`      | hub     | no        | no    | No common protocol major; `data.supported` lists them                                    |
| `unknown_method`        | hub     | no        | no    | Not a hub method                                                                         |
| `bad_target`            | hub     | no        | no    | Target doesn't parse (§6.2.2)                                                            |
| `unknown_app`           | hub     | no        | no    | App id not installed or not known                                                        |
| `app_unavailable`       | hub     | yes       | no    | Known app, no reachable instance; `data.launchable`                                      |
| `ambiguous_target`      | hub     | no        | no    | Several instances match; `data.candidates`                                               |
| `no_capable_app`        | hub     | yes       | no    | No app provides the capability right now                                                 |
| `unknown_command`       | hub/app | no        | no    | Id not in the app's registry; `data.suggestions` from fold keys                          |
| `params_invalid`        | hub     | no        | no    | Failed a `strict` schema (§6.3); `data.errors`                                           |
| `confirmation_required` | hub     | no        | no    | Command effects need confirmation; `data.confirm` carries a one-use token (§10.3, §11.4) |
| `forbidden`             | hub     | no        | no    | The connection may not run this effect class (§11.4)                                     |
| `path_denied`           | hub/app | no        | no    | A path is outside an allowed root (PhotoCraft read root O13; VectorCraft roots)          |
| `busy`                  | hub/app | yes       | no    | Too many connections or in-flight requests                                               |
| `result_too_large`      | hub     | no        | yes   | Result over 32 MiB; re-run with an output path                                           |
| `timeout_not_run`       | adapter | yes       | no    | Deadline passed before the app started the request; it will not run                      |
| `timeout_unknown`       | adapter | no        | maybe | No reply in time; the command may still apply                                            |
| `app_disconnected`      | adapter | yes       | maybe | Connection to the app dropped mid-call (R1 in §1.4)                                      |
| `app_auth_failed`       | adapter | no        | no    | The app refused the hub's credentials. A hub config fault                                |
| `adapter_protocol`      | adapter | no        | maybe | The app's reply didn't parse or didn't match the request                                 |
| `disabled`              | app     | no        | no    | Command exists but isn't available now (`data.reason`)                                   |
| `bad_params`            | app     | no        | no    | The app rejected the params                                                              |
| `no_document`           | app     | no        | no    | Needs an open document, sequence or composition                                          |
| `not_found`             | app     | no        | yes   | A referenced object (layer, node, file) doesn't exist                                    |
| `app_busy`              | app     | yes       | no    | App can't take it now (library locked by another process, window not drawing)            |
| `not_persisted`         | app     | yes       | yes   | Applied in memory but not written to disk; the app retries                               |
| `cancelled`             | app/hub | no        | yes   | Cancelled by a job cancel; nothing (more) changed                                        |
| `app_internal`          | app     | no        | yes   | The app hit an internal bug and kept the document as it was                              |
| `app_error`             | app     | no        | yes   | Any other app error; text in `message`                                                   |
| `job_still_running`     | hub     | yes       | yes   | `hub.job.wait` deadline passed; the job continues                                        |
| `internal`              | hub     | no        | maybe | A hub bug. The hub stays up (never-crash rule)                                           |

### 6.5.3 Mapping app error shapes

Every suite-jsonl app returns `"error": "<string>"`. The engines share their wording closely, because every `EngineError` is a `thiserror` enum with the same three leading variants. The hub classifies by prefix (first match wins) and always keeps the raw string in `appError`.

| App text (pattern)                                                                      | Apps and cite                                                                                                                                                                                                                                                                                                                                                    | Hub code                                                                      |
|-----------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------|
| ``unknown command `X` ``                                                                | `photocraft` `crates/engine/src/lib.rs:146`; `lightcraft` `crates/engine/src/lib.rs:84`; `filmcraft` `crates/engine/src/lib.rs:82`; `designcraft` `crates/engine/src/lib.rs:38`; `effectcraft` `crates/engine/src/lib.rs:78`; `vectorcraft` `crates/engine/src/lib.rs:41`; `soundcraft` `crates/engine/src/lib.rs:38`; `wordcraft` `crates/engine/src/lib.rs:41` | `unknown_command`                                                             |
| ``command `X` is not available right now: R``                                           | PhotoCraft, LightCraft, FilmCraft, DesignCraft, EffectCraft, VectorCraft `EngineError::Disabled` (same files, next variant)                                                                                                                                                                                                                                      | `disabled`                                                                    |
| ``` `X` is not available: R``` / `not available: R`                                     | `soundcraft` `crates/engine/src/lib.rs:40`; `wordcraft` `crates/engine/src/lib.rs:45`                                                                                                                                                                                                                                                                            | `disabled`                                                                    |
| ``invalid parameters for `X`: M``                                                       | PhotoCraft, LightCraft, FilmCraft, DesignCraft, EffectCraft, VectorCraft `EngineError::BadParams`                                                                                                                                                                                                                                                                | `bad_params`                                                                  |
| ``bad parameters for `X`: M`` / `bad parameters: M`                                     | `soundcraft` `crates/engine/src/lib.rs:42`; `wordcraft` `crates/engine/src/lib.rs:43`                                                                                                                                                                                                                                                                            | `bad_params`                                                                  |
| `no active document` / `no active sequence` / `no active composition`                   | `photocraft` `lib.rs:152`; `designcraft` `lib.rs:44`; `vectorcraft` `lib.rs:47`; `filmcraft` `lib.rs:88`; `effectcraft` `lib.rs:84`                                                                                                                                                                                                                              | `no_document`                                                                 |
| `internal error in `X`` …                                                               | `designcraft` `lib.rs:48-50`; `vectorcraft` `lib.rs:54-55`; `soundcraft` `lib.rs:50`                                                                                                                                                                                                                                                                             | `app_internal`                                                                |
| `saved in memory but not written to disk: …`                                            | `lightcraft` `crates/engine/src/lib.rs:92-95` (`NotSaved`)                                                                                                                                                                                                                                                                                                       | `not_persisted`                                                               |
| `cancelled`                                                                             | `photocraft` `crates/engine/src/lib.rs:158-160`; `lightcraft` `crates/engine/src/lib.rs:106` (`Cancelled(String)`)                                                                                                                                                                                                                                               | `cancelled`                                                                   |
| `timeout: not run — …`                                                                  | `photocraft` `crates/ui-egui/src/control.rs:60-64` (dropped before dispatch, "safe to re-send")                                                                                                                                                                                                                                                                  | `timeout_not_run`                                                             |
| `timeout: the app did not start the request in time; it was withdrawn and will not run` | `vectorcraft` `apps/vectorcraft/src/control_server.rs:36-40`                                                                                                                                                                                                                                                                                                     | `timeout_not_run`                                                             |
| `the app drew no frame for 5 s, so the request was not run: …`                          | `lightcraft` `apps/lightcraft/src/control_server.rs:66-67`, `:105-106` (`HIDDEN_WINDOW_ERROR`)                                                                                                                                                                                                                                                                   | `app_busy` (`ran: no`)                                                        |
| `timeout: no reply in Ns — …`                                                           | `photocraft` `crates/ui-egui/src/control.rs:49-54`                                                                                                                                                                                                                                                                                                               | `timeout_unknown`                                                             |
| bare `timeout`                                                                          | `lightcraft` `control_server.rs:111`; `filmcraft` `control_server.rs:77`; `designcraft` `control_server.rs:64`; `effectcraft` `control_server.rs:63`; `soundcraft` `control_server.rs:126` (90 s); `wordcraft` `control_server.rs:245`                                                                                                                           | `timeout_unknown`                                                             |
| `too many control connections`                                                          | `lightcraft` `control_server.rs:45`; `filmcraft` `:44`; `designcraft` `:45`; `effectcraft` `:44`; `vectorcraft` `:81`; `soundcraft` `:60`                                                                                                                                                                                                                        | `busy`                                                                        |
| `request exceeds N bytes` / `… longer than …`                                           | `photocraft` `control_server.rs:61-66`; LightCraft, FilmCraft, DesignCraft, EffectCraft tests assert "longer than" (e.g. `filmcraft` `control_server.rs:237`)                                                                                                                                                                                                    | `adapter_protocol` (the hub sent too much; never shown as the client's fault) |
| `bad JSON: …`                                                                           | `photocraft` `apps/photocraft/src/control_server.rs:108`; `soundcraft` `control_server.rs:132`; `wordcraft` `control_server.rs:205-208` (and closes)                                                                                                                                                                                                             | `adapter_protocol`                                                            |
| `authentication required` / `unauthorized`                                              | PhotoCraft (03a §3a.2 Auth); `wordcraft` `control_server.rs:231`                                                                                                                                                                                                                                                                                                 | `app_auth_failed`                                                             |
| ``unknown method `X` ``                                                                 | `photocraft` `crates/ui-egui/src/control.rs:1268`                                                                                                                                                                                                                                                                                                                | `unknown_command` (for `method` kind)                                         |
| `ambient smart-object path denied` / `pass `path`` (scoped writes)                      | `photocraft` `crates/ui-egui/src/control.rs` tests `:2448`, `:2753`                                                                                                                                                                                                                                                                                              | `path_denied`                                                                 |
| anything else (`Other`, `Failed`, `Catalog`, `Media`, …)                                | every engine's catch-all `#[error("{0}")]`                                                                                                                                                                                                                                                                                                                       | `app_error`                                                                   |

PdfCraft and MCP adapters (JSON-RPC 2.0):

| App shape                                                               | Cite                                                       | Hub code                                           |
|-------------------------------------------------------------------------|------------------------------------------------------------|----------------------------------------------------|
| control `-32700` parse error                                            | `pdfcraft` `crates/ui-egui/src/control.rs:826` @ `ac89a11` | `adapter_protocol`                                 |
| control `-32600` line too long before auth                              | `control.rs:814`                                           | `adapter_protocol`                                 |
| control `-32001` "authenticate first"                                   | `control.rs:838`                                           | `app_auth_failed`                                  |
| control `-32000` method error (string message)                          | `control.rs:844`                                           | apply the string table above to `message`          |
| control `-32002` "the app did not answer within 30 s"                   | `control.rs:845`                                           | `timeout_unknown`                                  |
| MCP `-32601` method not found / `-32602` invalid params                 | `pdfcraft` `crates/automation/src/mcp.rs:25-27`            | `unknown_command` / `bad_params`                   |
| MCP `tools/call` result with `isError: true`, text in `content[0].text` | `pdfcraft` `crates/automation/src/mcp.rs:339`, `:346`      | string table applied to the text, else `app_error` |

Headless CLI adapters: a spawn failure is `app_unavailable`; a non-zero exit with JSON on stdout uses that JSON's error; otherwise `app_error` with the last 4 KiB of stderr in `data.stderr`. AssetCraft's own `CmdError::{Unknown, BadParams, Fs, Unavailable}` (`assetcraft` `crates/engine/src/lib.rs:77-86` @ `3a41b2b`) maps to `unknown_command`, `bad_params`, `app_error` (with `data.fs`), `disabled`.

**Timeout opportunity (A4-F3).** Only PhotoCraft and VectorCraft tell "never ran" apart from "may have applied". Six apps reply bare `"timeout"`, which the hub must treat as `ran: maybe`, so it can't retry a write after a slow frame. VectorCraft's withdrawal (`vectorcraft` `apps/vectorcraft/src/control_server.rs:36-40`) is the pattern to copy upstream.

---

## 6.6 Versioning

### 6.6.1 Protocol version and `hub.hello`

The protocol version is `"MAJOR.MINOR"`. v1 ships as `"1.0"`. Drafts before v1 are `"0.x"`, where any change may break.

```json
→ {"id": 1, "method": "hub.hello", "params": {
     "protocol": "1.0",
     "client": {"name": "assetcraft", "version": "0.4.0", "kind": "app"},
     "auth": {"token": "<64 hex from hub.json>"},
     "features": ["events", "jobs"]}}
← {"id": 1, "ok": true, "result": {
     "protocol": "1.0",
     "hub": {"name": "pipelinecraft", "version": "0.1.0"},
     "connection": "c-7f3a",
     "features": ["events", "jobs", "capabilities"],
     "limits": {"maxRequestLine": 4194304, "maxReplyLine": 33554432, "maxInFlight": 32, "maxBatchSteps": 256},
     "deprecated": []}}
```

| Rule | Text                                                                                                                                                                                          |
|------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| V1   | The client sends the highest version it speaks. The hub replies with `min(client, hub)` within the same major, or `protocol_version` with `data.supported: ["1"]` and closes                  |
| V2   | `client.kind` is `app`, `craftctl` or `mcp`. It feeds §11.4 permissions; it is not proof of identity (A8)                                                                                     |
| V3   | `hub.hello` doubles as authentication, like PhotoCraft's `auth` first frame (`photocraft` `apps/photocraft/src/control_server.rs:74-81`). Where the token lives and how it is checked is A8's |
| V4   | `features` is the negotiated intersection. A client must not call methods of a feature the hub didn't return                                                                                  |

### 6.6.2 Deprecation rules

| ID  | Rule                                                                                                                                                                                                                                                         |
|-----|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| D-1 | Within a major, only additive changes: new methods, new optional params, new result fields, new error codes, new event topics, new effect flags. Clients must ignore unknown result fields and handle unknown error codes by `origin` and `retryable`        |
| D-2 | A new **effect class** is a minor change, but every older client must treat an unknown class as needing confirmation                                                                                                                                         |
| D-3 | Removing or renaming a hub method or field: mark it deprecated for at least one minor release; every reply that uses it carries `warnings: [{"code": "deprecated", "use": "<replacement>"}]`; `hub.hello` lists it in `deprecated`; remove at the next major |
| D-4 | The hub supports the current and the previous major (N and N−1)                                                                                                                                                                                              |
| D-5 | **App ids are not hub versions.** When an upstream id disappears, the hub keeps no ghost command. If the app names a replacement, the profile adds an alias (N5) with `since` and `until`; otherwise calls get `unknown_command` with `data.suggestions`     |
| D-6 | `hub.commands` returns `registryHash` (a hash over every instance's command list and the profiles in use), so clients can cache and refresh cheaply. App version changes are reported by A3's discovery (§5.5)                                               |

---

## 6.7 Worked examples

Each example shows the client↔hub frames, then (indented under `adapter`) what the hub sends to the app. Command ids and app reply shapes are from the cited source. **Values (paths, ids, counts) are illustrative**; nothing was run. Every example assumes `hub.hello` already succeeded on the connection.

### Example 1: list FilmCraft's commands (read, suite-jsonl adapter)

```json
→ {"id": 10, "method": "hub.commands", "params": {"app": "filmcraft", "filter": "file.export"}}
   adapter → filmcraft@p1  {"id": 1, "method": "engine.commands", "params": {}}
   adapter ← filmcraft@p1  {"id": 1, "ok": true, "result": [{"id": "file.exportOtio", "label": "OpenTimelineIO…", "menu": ["File", "Export"], "shortcut": null, "shortcuts": [], "defaultShortcut": null, "params": "{\"path\":str}", "enabled": false}, "…"]}
← {"id": 10, "ok": true, "result": {"registryHash": "b3:9c1e…", "commands": [
     {"address": "filmcraft/file.exportOtio", "app": "filmcraft", "kind": "cmd", "id": "file.exportOtio",
      "label": "OpenTimelineIO…", "menuPath": ["File", "Export"], "shortcut": null,
      "enabled": false, "disabledReason": null, "paramsDoc": "{\"path\":str}",
      "paramsSchema": {"type": "object", "properties": {"path": {"type": "string"}}, "required": ["path"], "additionalProperties": false},
      "paramsSchemaSource": "overlay", "validation": "strict",
      "effects": {"classes": ["write-user-file"], "destructive": true, "undoable": null, "idempotent": true, "confirm": true, "source": "curated"},
      "aliases": [], "capabilities": ["timeline.export.otio"], "ui": false}]}}
```

FilmCraft's control `engine.commands` runs the engine's `command.list` (`filmcraft` `crates/ui-egui/src/control.rs:111-114` @ `9ed9650`), whose entry keys are at `crates/engine/src/commands.rs:3379`. It has no disabled-reason key, so `disabledReason` is null even when `enabled` is false. The hub could fill it with a `hub.describe` later; that's an O9 detail.

### Example 2: open a file in a running PhotoCraft (write-app-state, token auth, root-relative path)

```json
→ {"id": 11, "method": "hub.call", "params": {"target": "photocraft/method:app.open", "params": {"path": "F:\\Library\\shoots\\2026\\IMG_0412.psd"}}}
   adapter → photocraft@p1 (connection opened earlier with {"id": 1, "method": "auth", "params": {"token": "<hex>"}})
             {"id": 2, "method": "app.open", "params": {"path": "shoots/2026/IMG_0412.psd"}}
   adapter ← photocraft@p1 {"id": 2, "ok": true, "result": {"path": "shoots/2026/IMG_0412.psd", "name": "IMG_0412.psd", "warnings": []}}
← {"id": 11, "ok": true, "result": {"path": "shoots/2026/IMG_0412.psd", "name": "IMG_0412.psd", "warnings": []},
   "resolved": "photocraft@p1/method:app.open"}
```

`app.open` is a control method, not a registry command (`photocraft` `crates/ui-egui/src/control.rs:1217-1253` @ `fd28da5`). It reads through the automation read root, so the adapter rewrites the absolute path to one relative to the root it launched PhotoCraft with (O13). An absolute path outside the root never reaches the app: the hub returns `path_denied`. The path rule itself is A5's (§7.4). This is AC1's call.

### Example 3: export from LightCraft (write-user-file → confirmation)

```json
→ {"id": 12, "method": "hub.call", "params": {"target": "lightcraft/app.export", "params": {"dir": "F:\\Exports\\Shoot-0412", "format": "jpeg", "longEdge": 2048}}}
← {"id": 12, "ok": false, "error": {"code": "confirmation_required", "origin": "hub", "retryable": false, "ran": "no",
     "message": "lightcraft/app.export writes files (write-user-file) and needs confirmation",
     "target": "lightcraft@p1/app.export",
     "data": {"confirm": "cf-41d2", "effects": {"classes": ["write-user-file"], "destructive": false, "confirm": true, "source": "curated"}, "expiresInMs": 120000}}}
→ {"id": 13, "method": "hub.call", "params": {"target": "lightcraft/app.export", "params": {"dir": "F:\\Exports\\Shoot-0412", "format": "jpeg", "longEdge": 2048}, "confirm": "cf-41d2"}}
   adapter → lightcraft@p1 {"id": 5, "method": "engine.execute", "params": {"command": "app.export", "params": {"dir": "F:\\Exports\\Shoot-0412", "format": "jpeg", "longEdge": 2048}}}
   adapter ← lightcraft@p1 {"id": 5, "ok": true, "result": {"files": ["F:\\Exports\\Shoot-0412\\IMG_0412.jpg"]}}
← {"id": 13, "ok": true, "result": {"files": ["F:\\Exports\\Shoot-0412\\IMG_0412.jpg"]}, "resolved": "lightcraft@p1/app.export"}
```

In the GUI, `app.export` is a UI command that `engine.execute` reaches (`lightcraft` `crates/ui-egui/src/menus.rs:219`, `:1878` @ `38adc17`); it replies `{"files": […]}` (`crates/ui-egui/src/control.rs:529-530`). The params come from the synthetic catalog entry (`crates/mcp/src/headless.rs:137`). The confirm token is bound to this connection, target and params hash, and is single-use. Who may skip it is §10.3/§11.4.

### Example 4: a PdfCraft call through the JSON-RPC 2.0 adapter

```json
→ {"id": 14, "method": "hub.call", "params": {"target": "pdfcraft@headless/tool:doc_open", "params": {"path": "F:\\Library\\print\\proof-v3.pdf"}}}
   adapter → pdfcraft-cli mcp (stdio; initialize already done when the process started)
             {"jsonrpc": "2.0", "id": 3, "method": "tools/call", "params": {"name": "doc_open", "arguments": {"path": "F:\\Library\\print\\proof-v3.pdf"}}}
   adapter ← {"jsonrpc": "2.0", "id": 3, "result": {"content": [{"type": "text", "text": "…"}],
              "structuredContent": {"doc": 1, "name": "proof-v3.pdf", "path": "F:\\Library\\print\\proof-v3.pdf", "pages": 12, "dirty": false, "editable": true, "read_only_reason": null},
              "isError": false}}
← {"id": 14, "ok": true, "result": {"doc": 1, "name": "proof-v3.pdf", "path": "F:\\Library\\print\\proof-v3.pdf", "pages": 12, "dirty": false, "editable": true, "read_only_reason": null},
   "resolved": "pdfcraft@p2/tool:doc_open"}
```

`doc_open` is linked to registry id `file.open` and has a typed schema (`pdfcraft` `crates/automation/src/tools.rs:132-134` @ `ac89a11`). The adapter returns `structuredContent` when present (`crates/automation/src/mcp.rs:503-505`), else the text content as `{"text": …}`. The field names follow `summary()` (`crates/automation/src/lib.rs:2127-2135`); whether `doc_open` returns exactly that is **not verified**. Effects: `read_only: false` → `write-app-state`, `source: inferred`. The control channel is still used for Open In into the open window: `pdfcraft/method:ui.open` becomes `{"jsonrpc":"2.0","id":4,"method":"ui.open","params":{"path":"…"}}` after the `auth` frame (03a §3a.5). The headless-for-writes split is O6.

### Example 5: id folding on SoundCraft

```json
→ {"id": 15, "method": "hub.call", "params": {"target": "soundcraft/session.saveAs", "params": {"path": "F:\\Audio\\mix-v2.scsession"}, "confirm": "cf-77a0"}}
   adapter → soundcraft@p1 {"id": 9, "method": "engine.execute", "params": {"command": "session.save_as", "params": {"path": "F:\\Audio\\mix-v2.scsession"}}}
   adapter ← soundcraft@p1 {"id": 9, "ok": true, "result": {"…": "…"}}
← {"id": 15, "ok": true, "result": {"…": "…"}, "resolved": "soundcraft@p1/session.save_as",
   "warnings": [{"code": "folded_id", "message": "`session.saveAs` resolved to `session.save_as`; use the canonical id"}]}
```

`session.save_as` is SoundCraft's real id (`soundcraft` `crates/engine/src/cmd/file.rs:24` @ `3a28a65`). Rule N3 applies because exactly one SoundCraft id folds to `session.saveas`. The confirm token here came from an earlier `confirmation_required` reply, because a curated row marks the command `write-user-file`. The result shape and file extension are illustrative.

### Example 6: errors (disabled, then an ambiguous timeout)

```json
→ {"id": 16, "method": "hub.call", "params": {"target": "filmcraft/file.exportOtio", "params": {"path": "F:\\Plan\\cut.otio"}, "confirm": "cf-0b19"}}
   adapter → filmcraft@p1 {"id": 7, "method": "engine.execute", "params": {"command": "file.exportOtio", "params": {"path": "F:\\Plan\\cut.otio"}}}
   adapter ← filmcraft@p1 {"id": 7, "ok": false, "error": "command `file.exportOtio` is not available right now: no sequence is open"}
← {"id": 16, "ok": false, "error": {"code": "disabled", "origin": "app", "retryable": false, "ran": "no", "app": "filmcraft",
     "target": "filmcraft@p1/file.exportOtio",
     "message": "command `file.exportOtio` is not available right now: no sequence is open",
     "appError": "command `file.exportOtio` is not available right now: no sequence is open",
     "data": {"reason": "no sequence is open"}}}

→ {"id": 17, "method": "hub.call", "params": {"target": "wordcraft/file.exportPdf", "params": {"path": "F:\\Docs\\brief.pdf"}, "confirm": "cf-5e2c"}}
   adapter ← wordcraft@p1 {"id": 3, "ok": false, "error": "timeout"}
← {"id": 17, "ok": false, "error": {"code": "timeout_unknown", "origin": "adapter", "retryable": false, "ran": "maybe", "app": "wordcraft",
     "target": "wordcraft@p1/file.exportPdf", "message": "WordCraft did not reply within 60 s; the export may still complete",
     "appError": "timeout"}}
```

The disabled text comes from FilmCraft's `EngineError::Disabled` (`filmcraft` `crates/engine/src/lib.rs:84-85`) with the reason from `has_seq` (`crates/engine/src/commands.rs:75-76`) @ `9ed9650`. The WordCraft reply is its 60 s fallback (`wordcraft` `apps/wordcraft/src/control_server.rs:245` @ `3292ba0`). Because `ran` is `maybe`, the hub does not retry the export; the caller can check whether the file appeared.

### Example 7: a dry-run batch with a capability step

```json
→ {"id": 18, "method": "hub.batch", "params": {"dryRun": true, "steps": [
     {"target": "filmcraft/file.import", "params": {"paths": ["F:\\Library\\clips\\A001.mov", "F:\\Library\\clips\\A002.mov"]}},
     {"target": "cap:timeline.export.otio", "params": {"path": "F:\\Plan\\rough.otio"}}]}}
← {"id": 18, "ok": true, "result": {"dryRun": true, "steps": [
     {"target": "filmcraft/file.import", "resolved": "filmcraft@p1/file.import", "ok": true, "validation": "strict",
      "effects": {"classes": ["write-app-state"], "destructive": false, "undoable": true, "confirm": false, "source": "curated"}},
     {"target": "cap:timeline.export.otio", "resolved": "filmcraft@p1/file.exportOtio", "ok": true, "validation": "strict",
      "alternatives": ["effectcraft/file.exportTimeline"],
      "effects": {"classes": ["write-user-file"], "destructive": true, "idempotent": true, "confirm": true, "source": "curated"}}],
   "effects": {"classes": ["write-app-state", "write-user-file"], "destructive": true, "confirm": true},
   "confirm": "cf-9d61"}}
```

Nothing is dispatched. FilmCraft is preferred for the capability because a GUI instance is running; EffectCraft's `file.exportTimeline` with `format: "otio"` is the alternative (§6.2.5). The dry-run reply already carries the confirm token, so a caller that accepts the plan re-sends the same batch with `"dryRun": false, "confirm": "cf-9d61"`. A dry run can't know whether step 2 will be `disabled` (no sequence yet); that's why the batch defaults to `stopOnError: true`. This is the shape of AC3's last two steps.

### Example 8: subscribe to job and app events

```json
→ {"id": 19, "method": "hub.subscribe", "params": {"topics": ["job.*", "app.*"], "filter": {"app": "photocraft"}}}
← {"id": 19, "ok": true, "result": {"sub": "s1"}}
← {"event": "app.up", "sub": "s1", "seq": 1, "source": "photocraft@p1", "at": "2026-10-15T09:12:00.004Z", "data": {"instance": "p1", "version": "0.9.0", "mode": "gui"}}
→ {"id": 20, "method": "hub.call", "params": {"target": "photocraft/filter.blur.gaussianBlur", "params": {"radius": 4}, "wait": false}}
   adapter → photocraft@p1 {"id": 12, "method": "engine.execute", "params": {"command": "filter.blur.gaussianBlur", "params": {"radius": 4}, "wait": false}}
   adapter ← photocraft@p1 {"id": 12, "ok": true, "result": {"job": 31, "pending": true}}
← {"id": 20, "ok": true, "result": {"job": "j7", "pending": true}, "resolved": "photocraft@p1/filter.blur.gaussianBlur"}
   adapter → photocraft@p1 {"id": 13, "method": "jobs.list", "params": {}}       (poll, every 250 ms while j7 runs)
← {"event": "job.progress", "sub": "s1", "seq": 2, "source": "photocraft@p1", "at": "…", "data": {"job": "j7", "progress": 0.4}}
← {"event": "job.done", "sub": "s1", "seq": 3, "source": "photocraft@p1", "at": "…", "data": {"job": "j7", "result": {"…": "…"}}}
```

`wait: false` → `{job, pending: true}` is PhotoCraft's documented behaviour (`photocraft` `crates/ui-egui/src/control.rs:33`, `:311` @ `fd28da5`), and `jobs.list` returns `{"jobs": […]}` with `state` and `progress` (`crates/engine/src/jobs.rs:850`; 03a §3a.2). The hub maps the app's job number to its own job id (`j7`) because job numbers are per app. The command id `filter.blur.gaussianBlur` follows PhotoCraft's Photoshop-menu naming (`commands.rs:1`) but **its exact spelling is not verified** (filter ids are generated by `format!`, 03a counting note); `craftctl commands photocraft --filter blur` settles it. Event topics, poll rate and payloads are A6's (§8).

---

## 6.8 Rust type sketches

Sketches for the hub's `proto` crate (layer and crate name are A3's; §4.3). They follow the house rules: no `unwrap`/`expect`/`panic`, untrusted input deserialized into closed types, unknown fields rejected where the contract says so.

```rust
use serde::{Deserialize, Serialize};
use serde_json::Value;
use std::collections::BTreeSet;

/// Client-chosen request id: integer (0..=2^53-1) or string (1..=128 chars). Checked after parse.
#[derive(Debug, Clone, PartialEq, Eq, Hash, Serialize, Deserialize)]
#[serde(untagged)]
pub enum Id {
    Num(u64),
    Str(String),
}

#[derive(Debug, Deserialize)]
#[serde(deny_unknown_fields)]
pub struct Request {
    pub id: Id,
    pub method: String,
    #[serde(default)]
    pub params: Value, // must be an object; checked by the router, not here
    #[serde(default)]
    pub meta: Option<Meta>,
}

#[derive(Debug, Default, Deserialize)]
#[serde(deny_unknown_fields, rename_all = "camelCase")]
pub struct Meta {
    pub trace: Option<String>,
}

/// One outgoing line. `Reply` always carries `ok`; `Event` never does.
#[derive(Debug, Serialize)]
#[serde(untagged)]
pub enum Frame {
    Reply(Reply),
    Event(EventFrame),
}

#[derive(Debug, Serialize)]
pub struct Reply {
    pub id: Option<Id>, // None only for frame-level errors before an id could be read
    pub ok: bool,
    #[serde(skip_serializing_if = "Option::is_none")]
    pub result: Option<Value>,
    #[serde(skip_serializing_if = "Option::is_none")]
    pub error: Option<HubError>,
    #[serde(skip_serializing_if = "Option::is_none")]
    pub resolved: Option<String>,
    #[serde(skip_serializing_if = "Vec::is_empty")]
    pub warnings: Vec<Warning>,
}

#[derive(Debug, Serialize)]
pub struct EventFrame {
    pub event: String,
    pub sub: String,
    pub seq: u64,
    pub source: String,
    pub at: String, // RFC 3339 UTC
    pub data: Value,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
#[serde(rename_all = "camelCase")]
pub struct HubError {
    pub code: ErrorCode,
    pub message: String,
    pub origin: Origin,
    pub retryable: bool,
    pub ran: Ran,
    #[serde(skip_serializing_if = "Option::is_none")]
    pub app: Option<String>,
    #[serde(skip_serializing_if = "Option::is_none")]
    pub target: Option<String>,
    #[serde(skip_serializing_if = "Option::is_none")]
    pub app_error: Option<Value>,
    #[serde(skip_serializing_if = "Option::is_none")]
    pub data: Option<Value>,
}

#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
#[serde(rename_all = "snake_case")]
pub enum ErrorCode {
    BadFrame, FrameTooLarge, BadRequest, HelloRequired, Unauthenticated, ProtocolVersion,
    UnknownMethod, BadTarget, UnknownApp, AppUnavailable, AmbiguousTarget, NoCapableApp,
    UnknownCommand, ParamsInvalid, ConfirmationRequired, Forbidden, PathDenied, Busy,
    ResultTooLarge, TimeoutNotRun, TimeoutUnknown, AppDisconnected, AppAuthFailed,
    AdapterProtocol, Disabled, BadParams, NoDocument, NotFound, AppBusy, NotPersisted,
    Cancelled, AppInternal, AppError, JobStillRunning, Internal,
    /// A code from a newer hub minor version (rule D-1). Clients fall back to origin/retryable.
    #[serde(other)]
    Unknown,
}

#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
#[serde(rename_all = "snake_case")]
pub enum Origin { Hub, Adapter, App }

#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
#[serde(rename_all = "snake_case")]
pub enum Ran { No, Maybe, Yes }

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Warning {
    pub code: String, // "folded_id", "deprecated", "advisory_params", "profile_stale"
    pub message: String,
}

// ---- Command effects ----

#[derive(Debug, Clone, Copy, PartialEq, Eq, PartialOrd, Ord, Hash, Serialize, Deserialize)]
#[serde(rename_all = "kebab-case")]
pub enum EffectClass {
    Read,
    WriteAppState,
    WriteUserFile,
    ExternalNetwork,
    /// A class from a newer minor version (rule D-2): always needs confirmation.
    #[serde(other)]
    Unknown,
}

#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
#[serde(rename_all = "snake_case")]
pub enum EffectSource { Declared, Curated, Inferred, Default }

#[derive(Debug, Clone, PartialEq, Eq, Serialize, Deserialize)]
#[serde(rename_all = "camelCase")]
pub struct Effects {
    pub classes: BTreeSet<EffectClass>,
    pub destructive: bool,
    pub undoable: Option<bool>,
    pub idempotent: bool,
    pub confirm: bool, // derived by `Effects::derive_confirm`, never trusted from input
    pub source: EffectSource,
    #[serde(default, skip_serializing_if = "Option::is_none")]
    pub note: Option<String>,
}

impl Effects {
    /// §6.4.2: the worst case for a command nobody has described.
    pub fn unknown(network_capable: bool) -> Self {
        let mut classes: BTreeSet<EffectClass> =
            [EffectClass::WriteAppState, EffectClass::WriteUserFile].into_iter().collect();
        if network_capable {
            classes.insert(EffectClass::ExternalNetwork);
        }
        Self { classes, destructive: true, undoable: None, idempotent: false, confirm: true,
               source: EffectSource::Default, note: None }
    }

    /// §6.4.1 `confirm` rule. Policy layers (§10.3, §11.4) may only make this stricter.
    pub fn derive_confirm(&mut self) {
        let beyond_app_state = self.classes.iter().any(|c| {
            !matches!(c, EffectClass::Read | EffectClass::WriteAppState)
        });
        self.confirm = self.destructive || beyond_app_state || self.source == EffectSource::Default;
    }

    /// Rule E2: a curated override may add, never remove.
    pub fn tighten(&mut self, other: &Effects) {
        self.classes.extend(other.classes.iter().copied());
        self.destructive |= other.destructive;
        self.idempotent &= other.idempotent;
        self.derive_confirm();
    }

    /// Union for a batch or recipe (§6.2.4). An empty batch is `read`.
    pub fn union<'a>(all: impl IntoIterator<Item = &'a Effects>) -> Effects {
        let mut out = Effects { classes: BTreeSet::from([EffectClass::Read]), destructive: false, undoable: Some(true),
                                idempotent: true, confirm: false, source: EffectSource::Declared, note: None };
        for e in all {
            out.classes.remove(&EffectClass::Read);
            out.undoable = match (out.undoable, e.undoable) { (Some(a), Some(b)) => Some(a && b), _ => None };
            out.source = out.source.weakest(e.source);
            out.tighten(e);
        }
        if out.classes.is_empty() { out.classes.insert(EffectClass::Read); }
        out.derive_confirm();
        out
    }
}

// ---- Addressing ----

#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
#[serde(rename_all = "snake_case")]
pub enum Kind { Cmd, Method, Tool }

#[derive(Debug, Clone, PartialEq, Eq)]
pub enum Instance { Auto, Id(String), Pid(u32), Gui, Headless, New }

#[derive(Debug, Clone, PartialEq, Eq)]
pub enum Target {
    Address { app: String, instance: Instance, kind: Kind, name: String },
    Capability(String),
}

// `impl FromStr for Target` parses the §6.2.2 grammar with `split_once('/')`, `split_once('@')`
// and `split_once(':')`, checks every character class and length, and returns
// `HubError { code: BadTarget, .. }` on any mismatch. It never indexes or unwraps.

/// §6.2.3 rule N3. Lowercase, drop `_` and `-` inside segments, keep dots.
pub fn fold_key(id: &str) -> String {
    id.split('.')
        .map(|seg| seg.chars().filter(|c| *c != '_' && *c != '-').flat_map(char::to_lowercase).collect::<String>())
        .collect::<Vec<_>>()
        .join(".")
}
```

`EffectSource::weakest` (not shown) orders `Default` < `Inferred` < `Curated` < `Declared` and returns the lower one, so a batch is only as trusted as its least-described step. The `effects.toml` loader deserializes into a `#[serde(deny_unknown_fields)]` struct with `BTreeMap<String, EffectRow>` for `[command."id"]` and `Vec<RuleRow>` for `[[rule]]`.

---

## Findings and recommendations from this section

| ID    | Scope                | Opportunity                                                                                                                                         | Suggested fix                                                                                                                                                                                                     | Owner                                                                                | Status |
|-------|----------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------|--------|
| A4-R1 | §6.1 transport       | Choose the hub socket                                                                                                                               | Loopback TCP, OS-chosen port in the hub manifest, in v1; AF_UNIX via `uds_windows` (PdfCraft precedent) in v1.x behind the same codec                                                                             | Claude (hub); A8 confirms                                                            | open   |
| A4-R2 | §6.4 command effects | A1 open question 3: five classes or a flag                                                                                                          | Four classes (`read`, `write-app-state`, `write-user-file`, `external-network`) plus `destructive`, `undoable`, `idempotent`, derived `confirm` flags                                                             | Paul (decide); A7, A8 apply                                                          | open   |
| A4-R3 | §6.1, D3             | Protocol shape                                                                                                                                      | Mirror the suite envelope; enrich `error` to an object; keep `engine.execute`/`engine.commands` as compatibility methods with string errors                                                                       | Paul (D3); A10 records                                                               | open   |
| A4-F1 | PdfCraft adapter, R2 | After `auth`, PdfCraft's control channel reads lines of any length (`max = usize::MAX`, `pdfcraft` `crates/ui-egui/src/control.rs:808` @ `ac89a11`) | Hub: the adapter never sends more than its configured cap, and caps its own reads of replies. Upstream: a post-auth cap in line with the suite's 4 MiB                                                            | Claude (adapter); PdfCraft (upstream)                                                | open   |
| A4-F2 | §6.5 errors          | Apps return free-text errors; the hub must classify by prefix, which breaks if wording changes                                                      | Hub: the prefix table in §6.5.3 as data in each profile, with contract tests (Not verified #1–#2). Upstream: add an `errorCode` field next to `error` (backward compatible), part of O-B1's `craft-control-proto` | Claude (hub); craft-libs / each app                                                  | open   |
| A4-F3 | §6.5 retry safety    | Six apps reply bare `"timeout"`, so the hub can't tell "never ran" from "may have applied"                                                          | Hub: treat as `ran: maybe`, retry only reads and idempotent calls. Upstream: adopt VectorCraft's withdrawal (`control_server.rs:36-40`) and PhotoCraft's two timeout texts                                        | Claude (hub); LightCraft, FilmCraft, DesignCraft, EffectCraft, SoundCraft, WordCraft | open   |
| A4-F4 | §6.4 command effects | Spawning programs (AssetCraft `file.openIn`, `file.run`) fits no class                                                                              | Interim: curate as `write-app-state` + `confirm: true` + note. Decide on an `external-process` class for protocol 1.1 (an additive change, rule D-2)                                                              | Paul (decide); Claude (AssetCraft row)                                               | open   |
| A4-F5 | §6.4.4 inference     | `journal`, `undoable` and `view` exist only in source, so the hub can't use them at runtime                                                         | `craftctl profile seed <app> --from-source <repo>`: a one-off static scan that writes candidate `effects.toml` rows for review. Retire per app when O4 lands                                                      | Claude (tool); Paul (review rows)                                                    | open   |
| A4-F6 | §6.4.4 WordCraft     | `mutates: false` means "doesn't change the document", and covers `file.save`/`saveAs`/`open`                                                        | Inference rule in §6.4.4 plus curated rows for every WordCraft `file.*`. Upstream: none needed if O-B5 adds real effects                                                                                          | Claude (profile)                                                                     | open   |
| A4-F7 | §6.3 schemas         | No typed params outside PdfCraft                                                                                                                    | Start small: overlays for the ~15 AC1–AC3 commands only (rule S5); EffectCraft's derived schema in advisory mode                                                                                                  | Claude (profiles)                                                                    | open   |
| A4-F8 | §6.2.3 ids           | Two id styles across apps                                                                                                                           | Fold rule N3 with per-app collision check N4; aliases only in profiles (N5); capabilities for cross-app sameness (N6)                                                                                             | Claude (hub)                                                                         | open   |
| A4-F9 | §6.2.2 addressing    | Useful handoff calls are control methods, not registry commands                                                                                     | `kind` in the address (`cmd`/`method`/`tool`). Upstream later: expose `app.open`-style methods as registry commands so `cmd` covers them                                                                          | Claude (hub); each app (later)                                                       | open   |

## Not verified

Everything above is a static read; nothing was built or run. These checks need Paul's Windows machine. Start each GUI app in its own PowerShell window first; ports match 03a/03b.

**Next action (Paul):**

1. Error-string prefixes, unknown and disabled (feeds §6.5.3 and A4-F2). Start FilmCraft with no project: `Set-Location F:\Script-Projects\ArtCrafting\filmcraft; cargo run -q --release -p filmcraft -- --control 9876`. Then:
   ```powershell
   Set-Location F:\Script-Projects\ArtCrafting\filmcraft
   $c = [Net.Sockets.TcpClient]::new('127.0.0.1', 9876); $s = $c.GetStream()
   $w = [IO.StreamWriter]::new($s); $r = [IO.StreamReader]::new($s); $w.AutoFlush = $true
   $w.WriteLine('{"id":1,"method":"engine.execute","params":{"command":"nope.nope","params":{}}}'); $r.ReadLine()
   $w.WriteLine('{"id":2,"method":"engine.execute","params":{"command":"file.exportOtio","params":{"path":"C:\\Temp\\x.otio"}}}'); $r.ReadLine()
   $c.Close()
   ```
   Expected: `{"error":"unknown command `nope.nope`","id":1,"ok":false}` and `{"error":"command `file.exportOtio` is not available right now: no sequence is open","id":2,"ok":false}` (key order may differ).
2. Same two calls against LightCraft (`--control 7980`) and SoundCraft (`--control 0`, read the printed port), with SoundCraft's disabled command chosen from `soundcraft-cli commands`. Expected: the same `unknown command` text in both; SoundCraft's disabled text starts with `` `<id>` is not available: ``.
3. FilmCraft catalog keys (Example 1): `cargo run -q --release -p filmcraft-cli -- commands --json | ConvertFrom-Json | Select-Object -First 1 | ConvertTo-Json`. Expected keys: `id, label, menu, shortcut, shortcuts, defaultShortcut, params, enabled`, and **no** `journal`/`readOnly` (confirms A4-F5).
4. SoundCraft real id (Example 5): `Set-Location F:\Script-Projects\ArtCrafting\soundcraft; cargo run -q --release -p soundcraft-cli -- describe session.save_as`. Expected: a JSON entry with params `{path, replace?: false}`.
5. FilmCraft OTIO overwrite (the `destructive = true` guess in §6.4.3): export a sequence twice to the same `.otio` path with `filmcraft-cli --bridge 127.0.0.1:9876 exec file.exportOtio path=C:\Temp\x.otio`. Expected: either the second run overwrites silently (keep `destructive = true`) or it errors (change the row to `destructive = false`).
6. PdfCraft headless `doc_open` result (Example 4):
   ```powershell
   Set-Location F:\Script-Projects\ArtCrafting\pdfcraft
   $in = @(
     '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-06-18","capabilities":{},"clientInfo":{"name":"probe","version":"0"}}}',
     '{"jsonrpc":"2.0","method":"notifications/initialized"}',
     '{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"doc_open","arguments":{"path":"F:\\Script-Projects\\ArtCrafting\\assetcraft\\test_data\\library\\<any>.pdf"}}}'
   ) -join "`n"
   $in | cargo run -q --release -p pdfcraft-cli -- mcp
   ```
   Expected: the `id: 2` reply has `isError: false` and a `structuredContent` with `doc`, `pages` and `editable`. If `structuredContent` is missing, the adapter falls back to text content.
7. PdfCraft post-auth line cap (A4-F1). Start `pdfcraft.exe --control "$env:LOCALAPPDATA\PipelineCraft\pdf.json"`, authenticate as in 03a Not verified #8, then send one 8 MiB line of `x`. Expected per its test (`control.rs:1008-1011`): a `-32700` parse error and the connection **stays open**, which confirms there is no cap above 4 KiB after auth.
8. PhotoCraft filter id spelling (Example 8): `Set-Location F:\Script-Projects\ArtCrafting\photocraft; cargo run -q --release -p photocraft-cli -- commands --json --filter blur`. Expected: a Gaussian blur id; replace `filter.blur.gaussianBlur` in Example 8 with the real one.

## Open questions

1. **Effects model (A4-R2).** Four classes plus a `destructive` flag, as recommended, or the outline's five classes? And does protocol 1.1 add `external-process` (A4-F4)? (Paul)
2. **Id folding (N3).** Auto-run a unique folded match with a warning, as recommended, or only suggest it in `unknown_command`? Auto-run helps MCP clients that guess camelCase; suggest-only is stricter. (Paul)
3. **Error shape on hub methods.** `error` as an object on `hub.*` (recommended) while the compatibility `engine.execute` keeps a string. Is two shapes on one socket acceptable? (Paul, with A7 for craftctl output)
4. **Socket (A4-R1).** TCP loopback in v1 with AF_UNIX later, or AF_UNIX from day one to remove the browser-reachability class of threats entirely? (A8 proposes, Paul decides)
5. **Confirmation tokens.** Tokens are bound to connection + target + params hash and expire after 2 min. Is a dry run allowed to issue the token for the real run (Example 7)? (A7, A8)

## Inputs for downstream agents

- **A3 (§4–5):** profiles live in `profiles/<app>/{profile.toml, effects.toml, schemas.json}`; `profile.toml` holds your adapter quirks plus `aliases`, `capabilities` and `network_capable`. Adapters number their own wire ids, keep a pool of 2–4 connections per instance, and need a `path_transform` hook (PhotoCraft root-relative paths, O13). Instance ids are short hub-assigned strings (`p1`); the target grammar accepts `@p1`, `@pid=N`, `@gui`, `@headless`, `@new`. Assumed: the hub manifest at `%LOCALAPPDATA%\PipelineCraft\hub.json` with `{port, token, pid, protocol}`; `hub.app.launch`/`hub.app.stop` are yours to define. `hub.apps` should report `profile: ok|stale|missing`.
- **A5 (§7):** asset refs travel inside `params`; the hub doesn't change params except through an adapter's `path_transform`. Example 2 depends on your path rules. Results over 32 MiB must go by file path, not inline.
- **A6 (§8–9):** event frame `{event, sub, seq, source, at, data}` with per-subscription `seq`; method names `hub.subscribe`, `hub.unsubscribe`, `hub.job.list|get|wait|cancel`; hub job ids are hub-scoped strings (`j7`), mapped from app job numbers. Batch steps reserve `as` and `from` for chaining. `ran` (`no|maybe|yes`) is what makes a recipe step safe to retry.
- **A7 (§10, §12):** MCP annotations from command effects: `readOnlyHint` = classes is exactly `{read}`; `destructiveHint` = `destructive`; `idempotentHint` = `idempotent`; `openWorldHint` = contains `external-network`. `confirmation_required` + `confirm` token is the protocol mechanism for §10.3. Tool names derive from addresses; `kind` must survive in the name (e.g. `pdfcraft.tool.doc_open`). `craftctl profile check|seed` belong in §12.
- **A8 (§11):** `hub.hello` carries `auth.token`; pre-hello lines are capped at 4 KiB; non-JSON first lines close the connection (HTTP guard); `forbidden` is the code for effect-class permissions; the confirm token binding is in Example 3. Policy may only tighten `Effects.confirm`.
- **A9 (§13):** contract tests: error-prefix table per app (Not verified #1–#2 as fixtures), fold-key collision check (N4), overlay ids exist and `required` keys appear in the params doc (S4), unknown commands get `source: default`.
- **A10 (§15):** D3 recommendation text is A4-R3. A4-R2 (effects model) and open questions 2–4 are candidate decisions.
