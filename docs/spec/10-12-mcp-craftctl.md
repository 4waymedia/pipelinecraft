# PipelineCraft (working name): §10 MCP front door · §12 `craftctl` CLI

_2026-10-15 · spec draft v0 · status: draft (agent A7)_

- **Scope:** the hub's single MCP server (§10: tool surface, command effects → MCP annotations, confirmation over MCP, resources and URI scheme, transports, client configs for Claude and Paul's llama.cpp stack, tool-layer prompt-injection guardrails) and the hub's command-line client `craftctl` (§12: command tree, flags, exit codes, `--json` shapes, PowerShell examples, the `craftctl doctor` spec).
- **Project goal it serves:** G5 Automate and acceptance case AC4 (§1.4): AC1 Open In, AC2 Thumbnails and AC3 Plan → FilmCraft run from Claude and from the local llama.cpp stack through one MCP server, and from PowerShell through `craftctl`, with every write confirmed by a human and no cloud in the path.
- **Immediate area of improvement:** today an MCP client has to be wired to nine separate per-app MCP servers with nine tool styles (25, 28 + one per command, 17, 163, 16, 25, 36, 32 and 16 tools; 03a §3a.1, 03b §3.1b), whose read-only hints are hand-kept name lists. This section replaces that with 14 tools over the merged registry, annotations derived from command effects, and a confirmation step that a prompt-injected model can't answer for itself.

**Evidence basis.** Static read of the shallow clones in `/home/claude/src` at the commits in `01-goals-glossary.md` (`filmcraft` `9ed9650`, `photocraft` `fd28da5`, `effectcraft` `2c08f67`, `vectorcraft` `0946bc5`, `lightcraft` `38adc17`, `assetcraft` `3a41b2b`, others as listed there). Nothing was built or run. Wave 1–2 facts are cited by ID: `01` (A1-*), `03a` (O1–O14), `03b` (O-B1–O-B13), `04-05` (A3-*), `06` (A4-*, §6 method and error names), `07` (A5-*, payload types). Terms follow §2: **connection** (never "session") for a hub client link, **command effect** in prose with JSON field `effects`, addresses `app[@instance]/[kind:]name`, capabilities `cap:…`. MCP spec facts were checked on the web on 2026-10-15 and are cited with the revision date; three spec pages could not be fetched (listed in **Not verified**).

---

## 10. MCP front door

### 10.0 What the suite already agreed on (the base to build on)

The suite has an MCP convention, and the hub should extend it rather than invent one (03b O-B4).

| Fact                                                                                                     | Where it is defined (static read)                                                                                                                                                                                                                                                  |
|----------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **Origin:** FilmCraft PR #28, "MCP annotations, resources, export progress and cancellation", landed 2026-10-06 | `filmcraft` `ROADMAP.md:121` @ `9ed9650`                                                                                                                                                                                                                                       |
| **Reference implementation** of the five tools with all four hints                                      | `filmcraft` `crates/automation/src/server.rs:306-311` (`command_list`, read-only, idempotent, closed-world), `:331-336` (`command_run`, destructive), `:340-345` (`command_batch`, `stop_on_error` default true), `:367-372` (`doc_inspect`), `:376-381` (`render_preview`) @ `9ed9650` |
| Resources paired with the tools                                                                          | `filmcraft` `crates/automation/src/server.rs:589-591`: `filmcraft://document` (= `doc_inspect`), `filmcraft://commands` (= `command_list`) @ `9ed9650`                                                                                                                              |
| Adopters say so in their docs                                                                            | `photocraft` `docs/mcp.md:3` @ `fd28da5`; `effectcraft` `docs/mcp.md:19-31` @ `2c08f67`; `vectorcraft` `docs/mcp.md:255-266` @ `0946bc5` (adds `$N.path` result references and `atomic` batches); `lightcraft` `docs/mcp.md:126-136` @ `38adc17`                                   |
| Not adopted                                                                                              | DesignCraft, SoundCraft, WordCraft (03b O-B4; 03a §3a.6)                                                                                                                                                                                                                            |
| Apps already emit 2026-07-28 MCP fields                                                                  | PhotoCraft `tools/list` returns `resultType: "complete"`, `ttlMs: 600000`, `cacheScope: "private"` (`photocraft` `crates/automation/src/server/conventions.rs:35`, resources `:64`, reads `ttlMs: 0` `:80` @ `fd28da5`); VectorCraft negotiates `2026-07-28` plus three older revisions (`vectorcraft` `crates/mcp/src/server.rs:14-15` @ `0946bc5`) |
| Hints today are name lists, not command effects                                                          | PhotoCraft `annotated()`: read-only for 7 names, destructive for the rest except 3 "additive" names (`photocraft` `conventions.rs:11-21`); LightCraft marks every generated `cmd_*` tool destructive (03a §3a.3)                                                                    |
| No sibling MCP server does elicitation, and VectorCraft explains why                                     | "Both are server→client requests, blocked by the same one-line-one-reply loop … which is what the `2026-07-28` revision's input-requests mechanism exists for" (`vectorcraft` `docs/mcp.md:242-244` @ `0946bc5`); grep for `elicit` finds no code in any repo                    |

### 10.0.1 MCP spec features this section relies on (verified 2026-10-15)

| Feature                                        | Revision                         | What it says                                                                                                                                                                                                                                       | Source                                                                                                                                           |
|------------------------------------------------|----------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------|
| Current revision                               | **2026-07-28** (previous 2025-11-25) | Stateless: no `initialize` handshake; version and client capabilities travel in each request's `_meta` (`io.modelcontextprotocol/protocolVersion`, `…/clientCapabilities`); servers MUST implement `server/discover`                          | [2026-07-28 changelog](https://modelcontextprotocol.io/specification/2026-07-28/changelog), major changes 2–3                                     |
| Multi round-trip requests (MRTR)               | 2026-07-28                       | Replaces server-initiated `elicitation/create`: a server returns `resultType: "input_required"` with `inputRequests`; the client retries the original request with `inputResponses` (and echoes `requestState`)                                | changelog major change 7, minor change 11                                                                                                        |
| `resultType` on every result                   | 2026-07-28                       | `"complete"` or `"input_required"`; results from older servers without it are treated as complete                                                                                                                                                 | changelog major change 8                                                                                                                         |
| Cache hints                                    | 2026-07-28                       | `ttlMs` and `cacheScope` (`public`/`private`) are **required** on `tools/list`, `resources/list`, `resources/read`, `resources/templates/list`, `prompts/list`; `tools/list` SHOULD be deterministic                                             | changelog minor changes 3, 5                                                                                                                     |
| Subscriptions                                  | 2026-07-28                       | `resources/subscribe` replaced by `subscriptions/listen`; `notifications/progress` still flows on the request's own response stream                                                                                                               | changelog major change 4                                                                                                                         |
| Streamable HTTP                                | 2026-07-28                       | No `Mcp-Session-Id`; `Mcp-Method` and `Mcp-Name` headers required on POST; no SSE resumability                                                                                                                                                   | changelog major changes 1, 9; minor change 4                                                                                                     |
| Deprecations                                   | 2026-07-28                       | Roots, Sampling and Logging deprecated (12-month window); `ping` and `logging/setLevel` removed                                                                                                                                                   | changelog "Deprecated" 1; major change 5                                                                                                          |
| Elicitation (form mode)                        | 2026-07-28                       | Client declares `elicitation: {form: {}}`; request `{method: "elicitation/create", params: {mode: "form", message, requestedSchema}}`; reply `action: accept \| decline \| cancel`; servers MUST NOT ask for passwords/keys in form mode; clients MUST show which server asks | [elicitation page (mirror)](https://modelcontextprotocol.net/specification/2026-07-28/client/elicitation.md)                                     |
| Tool annotations                               | current                          | `readOnlyHint` (default false), `destructiveHint` (default true), `idempotentHint` (default false), `openWorldHint` (default true); all are **hints, untrusted** unless the server is trusted                                                     | [MCP blog, 2026-03-16 "Tool annotations"](https://blog.modelcontextprotocol.io/posts/2026-03-16-tool-annotations/)                               |
| Tool names                                     | SEP-986, Final                   | 1–64 chars, case-sensitive, `A-Z a-z 0-9 _ - . /`                                                                                                                                                                                                 | [SEP-986](https://modelcontextprotocol.io/seps/986-specify-format-for-tool-names)                                                                 |
| stdio framing                                  | since 2024-11-05                 | Newline-delimited JSON-RPC, no embedded newlines, nothing but MCP on stdout, logs on stderr                                                                                                                                                      | [2024-11-05 transports](https://modelcontextprotocol.io/specification/2024-11-05/basic/transports)                                                |
| Local HTTP security                            | since 2024-11-05                 | Servers MUST validate `Origin` (DNS rebinding), SHOULD bind 127.0.0.1, SHOULD authenticate                                                                                                                                                        | same page, "Security Warning"; the 2026-07-28 Streamable HTTP page itself was not fetched (Not verified #9)                                       |
| Real-world reminder                            | 2026                             | `rmcp` before 1.4.0 did not validate `Host` on Streamable HTTP (CVE-2026-42559, DNS rebinding, 8.8). The suite pins rmcp 3.5 with stdio only (`photocraft` `crates/automation/Cargo.toml:23`), so it is not exposed today                       | [GitLab advisory](https://advisories.gitlab.com/cargo/rmcp/CVE-2026-42559/)                                                                       |

**Consequence for the design.** The front door must speak **two dialects**: 2026-07-28 (stateless, `server/discover`, MRTR elicitation) for new clients, and 2025-06-18 / 2025-11-25 (`initialize`, server→client `elicitation/create`) for clients that haven't moved yet. VectorCraft already serves four revisions from one hand-written server (`vectorcraft` `crates/mcp/src/server.rs:14-15`), so that's the pattern for `craft-hub-mcp` (A3 §4.3: hand-written JSON-RPC, L2).

### 10.1 Tool surface

#### 10.1.1 Why not one tool per command

| Approach                                         | Tools in `tools/list`                                                                                                                                                                                                                         | Verdict                                                                                                                                                         |
|--------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------|
| One tool per command (LightCraft's `cmd_*` style) | About **5,800**: PhotoCraft 863, LightCraft ~312 + ~179 UI, FilmCraft ~702, PdfCraft 194 + 163 tools, WordCraft 629, DesignCraft 463 + 134 UI, EffectCraft ≥ 728, VectorCraft 743 + 285 UI, SoundCraft ~478 + ~66 UI (03a §3a.1, 03b §3.1b; counts are static) | Unusable. At even 150 tokens per tool that's ~870k tokens of tool list, beyond any context window; a local model at 32k context would see nothing else. SEP-986's 64-char limit also collides with long ids |
| Per-app suite tools, namespaced (`<app>.command_run`, 03b O-B4) | 5 × 10 apps = 50, plus per-app curated tools                                                                                                                                                                                                  | Workable but repetitive; the model still has to pick an app before it can search, and capabilities (`cap:`) have no home                                         |
| **Hub tools over the merged registry** (this spec) | **14** (11 generic + 3 workflow)                                                                                                                                                                                                             | Recommended. Search finds commands across apps; the full catalogs live in resources, read on demand                                                             |

**Recommendation A7-R1: 14 tools, fixed at compile time, deterministic order.** The tool list is static, so `tools/list` can carry a long `ttlMs` and stays byte-identical between calls, which the 2026-07-28 revision asks for prompt-cache reasons (changelog minor change 3). The merged catalog changes as apps start and stop; it is served through `hub_commands` results and `craft://` resources, never through tool descriptions (also a guardrail, §10.6 G4). The count is justified by function: each tool maps to one hub method family in §6.2.1, plus one tool per acceptance workflow. The two that look redundant (`hub_read` beside `hub_call`, `hub_plan` beside `hub_batch`) exist so that a read-only annotation is **true and enforced**, which is what lets a client auto-approve them safely (§10.2).

#### 10.1.2 The 14 tools

Tool names use only `[a-z0-9_]`. SEP-986 also allows `.` and `/`, but the names travel on to LLM APIs (Claude Code's `mcp__<server>__<tool>`, OpenAI-compatible function names in llama.cpp's server) whose own name rules are stricter; underscore-only names pass everywhere (A7-F1).

| #  | Tool               | Maps to (§6.2.1)                                   | Arguments (JSON Schema properties; `additionalProperties: false`)                                                                                                                               | Result (`structuredContent`)                                                                                                     | Annotations (RO / D / I / OW) |
|----|--------------------|----------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------|-------------------------------|
| 1  | `hub_apps`         | `hub.apps`                                         | `running?: bool`, `installed?: bool`                                                                                                                                                            | `[{app, version, state, instances: [{instance, mode, pid?, health}]}]`                                                           | T / F / T / F                 |
| 2  | `hub_commands`     | `hub.commands` + search                            | `query?: str` (≤ 200 chars), `app?: str`, `effects?: [class]`, `enabledOnly?: bool`, `capability?: str`, `limit?: int (1–100, default 20)`, `cursor?: str`                                       | `{registryHash, total, nextCursor?, commands: [CommandRow]}` (row below)                                                         | T / F / T / F                 |
| 3  | `hub_describe`     | `hub.describe`, `hub.route`                        | `target: str` (address or `cap:…`)                                                                                                                                                              | full `CommandInfo` (§6.3.3), or for `cap:` the `candidates` list                                                                 | T / F / T / F                 |
| 4  | `hub_read`         | `hub.call`, **read class only**                    | `target`, `params?: object`, `timeoutMs?: int (≤ 60000)`                                                                                                                                        | the command result, plus `resolved`, `warnings`                                                                                  | T / F / T / F                 |
| 5  | `hub_call`         | `hub.call`                                         | `target`, `params?`, `wait?: bool (default true)`, `timeoutMs?`                                                                                                                                 | result, or `{job, pending: true}` when `wait: false`; confirmation per §10.3                                                     | F / T / F / T                 |
| 6  | `hub_plan`         | `hub.batch {dryRun: true}`; recipe dry run (A6)    | one of `steps: [{target, params?, as?, from?}]` (≤ 256) or `recipe: str` + `args?: object`                                                                                                     | the dry-run plan from §6.2.4 / Example 7: per-step `resolved`, `effects`, `validation`, and the union effects                     | T / F / T / F                 |
| 7  | `hub_batch`        | `hub.batch {dryRun: false}`; recipe run (A6)       | same as `hub_plan`, plus `stopOnError?: bool (default true)`                                                                                                                                    | `{completed, failed, steps: [{target, ok, result \| error}]}`, the suite `command_batch` shape plus `target`                     | F / T / F / T                 |
| 8  | `hub_handoff`      | hub handoff (§7.2) → `cap:asset.*`                 | `payload: object` (a §7.2 `handoff.asset \| handoff.selection \| handoff.timeline`), `to: str` (app id or `cap:…`), `instance?: str`                                                          | `{job?, resolved: [address], ledgerIds: [str]}`                                                                                  | F / F / F / F                 |
| 9  | `hub_job_status`   | `hub.job.get`, `hub.job.list`, `hub.job.wait`      | `job?: str` (omit to list), `app?`, `state?`, `waitMs?: int (0–30000, default 0)`                                                                                                              | one job or a list (A6 owns the job shape)                                                                                        | T / F / T / F                 |
| 10 | `hub_job_cancel`   | `hub.job.cancel`                                   | `job: str`                                                                                                                                                                                      | `{cancelled: bool}`                                                                                                              | F / F / T / F                 |
| 11 | `hub_resolve`      | AssetRef resolve (§7.1.4)                          | `ref: object` (AssetRef) or `path: str`                                                                                                                                                         | Resolution (`ok \| moved \| modified \| missing \| ambiguous`, A5)                                                               | T / F / T / F                 |
| 12 | `open_in`          | **AC1** workflow                                   | `app: "photocraft" \| "lightcraft" \| "vectorcraft" \| … `, `paths?: [str]` (omit = AssetCraft's current selection), `instance?: str`                                                        | `{opened: [{path, resolved, ok, error?}], launched: bool}`                                                                       | F / F / F / F                 |
| 13 | `thumbnail_render` | **AC2** workflow                                   | `paths: [str]` (≤ 64), `maxSide?: int (64–1024, default 256)`, `inline?: bool (default false)`                                                                                                  | `[{path, ok, thumb?: "craft://thumb/<q1>/<maxSide>", app?, fallback?: "type-icon"}]`; `inline: true` adds `image` content blocks | T / F / T / F                 |
| 14 | `plan_to_filmcraft`| **AC3** workflow                                   | `storyline: object` (`handoff.storyline`, §7.2.4) or `storylineId: str`, `dryRun?: bool (default **true**)`, `sequenceName?: str`                                                                | dry run: the plan (OTIO path, clip count, steps, effects); commit: `{otio, sequence, clips, durationTicks, checks: {clipCount, order, duration}}` | F / F / F / F                 |

`CommandRow` (compact, for search results; the full entry is `hub_describe`):

```json
{"address": "filmcraft/file.exportOtio", "label": "OpenTimelineIO…", "menuPath": ["File", "Export"],
 "enabled": false, "effects": "write-user-file!", "paramsDoc": "{\"path\":str}", "capabilities": ["timeline.export.otio"], "score": 0.92}
```

`effects` here is a short string (`read`, `write-app-state`, `write-user-file!`, where `!` means `destructive`, and `?` means `source: default`) so the model can see risk at a glance; `paramsDoc` is truncated to 200 characters.

**Search (`hub_commands.query`).** Lexical, in the hub, no model: split the query into lowercase tokens; score each command by token hits on `id` segments (weight 3, with fold keys from §6.2.3 rule N3), `label` (2), `menuPath` (1), `capabilities` (3) and `paramsDoc` (0.5); boost `enabled` entries ×1.2; tie-break by address. The same function backs `craftctl commands --search` (§12), so a search Paul runs in PowerShell returns the same ranking the model sees. Embeddings are out of scope for v1 (start small).

**Tool descriptions are short and static.** Each says what it does, its cost class, and one example. `hub_commands` says: "Search ~5,800 commands across running Craft apps. Use before `hub_read`/`hub_call`. Example: `{"query":"export otio"}`." Server `instructions` (the FilmCraft precedent at `server.rs:587`) adds the addressing grammar in three lines and says "read effects before calling; writes ask the user."

**Why the three workflow tools are first-class.** Each one wraps a sequence that a model would otherwise have to assemble from 4–10 generic calls with exact per-app quirks (PhotoCraft root-relative paths O13, one `app.open` per item 07 §7.2.2, FilmCraft `file.import` of an OTIO file 07 §7.2.5, VectorCraft `file.place.info` thumbnails 03b). They are thin: each is a hub-side recipe (A6 §8.4) with a fixed shape, and each is also reachable as `craftctl run open-in|thumbnails|plan-to-filmcraft` (§12). If a workflow tool and the generic path disagree, that's a bug in the recipe, not a second implementation.

| Workflow tool       | Steps it runs (A6 owns the recipe format)                                                                                                                                                                                       | Confirmation                                                                                                   |
|---------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------|
| `open_in`           | `assetcraft/app.state` (selection, `commands.rs:142`, `:561` @ `3a41b2b`) if no `paths` → `hub_resolve` each → `cap:asset.open` or `cap:asset.import` on the chosen app (`photocraft/method:app.open`, `lightcraft/library.import`, `vectorcraft/document.open`, §6.2.5) → launch if needed (A3 §5.3) | `write-app-state`, not destructive: no prompt by default. **Exception:** WordCraft `file.open` (destructive, O5) and any `source: default` target prompt |
| `thumbnail_render`  | `cap:asset.thumbnail` per file (VectorCraft `file.place.info`, EffectCraft `render.frame`, others by O14 recipe) through headless instances → PNG into `derived\thumbs` (07 §7.5)                                               | None: the only write is to the hub's own cache, never a user file (07 G1)                                     |
| `plan_to_filmcraft` | AssetCraft `interchange` writes OTIO into `handoff\<job>\` (07 §7.2.5, planned) → `filmcraft/file.import {paths:[otio]}` → `hub_describe`-style check of clip count, order, duration (AC3c)                                      | Dry run by default (AC3d). Commit is `write-app-state` in FilmCraft; prompts because the recipe also writes the OTIO file (`write-user-file` into the hub folder counts as hub-owned, so **no**; see open question 3) |

### 10.2 Command effects → MCP annotations

Two layers, because `tools/list` is static but command effects are per target.

**Layer 1: static tool annotations** (the table in §10.1.2). Generic tools whose effects depend on the target carry the **worst case** of what they can do; `hub_read`, `hub_plan` and the read-only discovery tools carry `readOnlyHint: true`, and **the hub enforces it**: `hub_read` refuses any target whose resolved effects aren't exactly `{read}` with `forbidden` (§6.5.2) and the message "use hub_call". This is the difference from the per-app servers today, where the hint is a name list nobody checks (`photocraft` `conventions.rs:15-21`).

**Layer 2: per-call command effects.** Every `hub_commands`/`hub_describe` row, every `hub_plan` step and every `hub_call` result carries the target's `effects` object (§6.3.3), and the confirmation step (§10.3) uses the target's effects, never the tool's. Mapping, per A4's input to A7 (06 "Inputs for downstream agents"):

| Command effects (§6.4.1)                                                 | `readOnlyHint` | `destructiveHint` | `idempotentHint`       | `openWorldHint` | Hub confirmation (§10.3)              | Allowed through          |
|--------------------------------------------------------------------------|----------------|-------------------|------------------------|-----------------|----------------------------------------|--------------------------|
| `classes == {read}`                                                       | true           | false             | true                   | false           | none                                   | `hub_read`, `hub_call`   |
| `write-app-state`, not destructive                                        | false          | false             | `effects.idempotent`   | false           | none (client may still prompt)         | `hub_call`, `hub_batch`  |
| `write-app-state` + `destructive` (e.g. WordCraft `file.open`, O5)        | false          | true              | `effects.idempotent`   | false           | **required**                           | `hub_call`, `hub_batch`  |
| contains `write-user-file`                                                | false          | `effects.destructive` | `effects.idempotent` | false          | **required**                           | `hub_call`, `hub_batch`  |
| contains `external-network`                                               | false          | `effects.destructive` | `effects.idempotent` | **true**       | **required**                           | `hub_call`, `hub_batch`  |
| `source: default` (unknown command, §6.4.2: write + destructive)          | false          | true              | false                  | false (true if the app profile has `network_capable`) | **required**             | `hub_call`, `hub_batch`  |
| a batch or recipe                                                         | union of steps | any step destructive | all steps idempotent | any step network | required if any step requires it; **one** prompt listing every step | `hub_batch`, `plan_to_filmcraft` |

The per-call values are emitted in each result's `_meta` as `{"craft/effects": {…}}` (vendor-prefixed key, so it can't clash with `io.modelcontextprotocol/*`). Rule from A4 §6.4.1: confirmation can be tightened per connection (§11.4), never loosened below the derived `confirm`.

### 10.3 Confirmation policy for write and destructive calls

**The problem a token alone doesn't solve.** §6 already has a protocol mechanism: a call that needs confirmation fails with `confirmation_required` and `data.confirm` (a one-use token bound to connection, target and params hash, 2 min, 06 Example 3). That's enough for `craftctl`, where the hub's client is a person at a terminal. Over MCP, the hub's client is a **model**. If the token is returned to the model, a prompt-injected model simply calls again with it, and the "confirmation" confirms nothing. So over MCP the token must be redeemed by a **human channel**, never by the model.

**Recommendation A7-R2: three confirmation channels, tried in this order; the model never holds a redeemable token.**

| Order | Channel                                   | When it applies                                                                                                    | How it works                                                                                                                                                                                                                                                                                                                       |
|-------|-------------------------------------------|--------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| 1     | **MRTR elicitation** (2026-07-28)         | The request's `_meta` `clientCapabilities` declares `elicitation.form`                                            | The shim returns `resultType: "input_required"` with one `elicitation/create` form request (below). The client's UI shows it to the user. On a retry whose `inputResponses` carry `action: "accept"` and `approve: true`, the shim redeems the hub token itself and runs the call                                              |
| 1b    | **Legacy elicitation** (2025-06-18 / 2025-11-25) | The client sent `initialize` with an `elicitation` capability                                                 | The shim sends a server→client `elicitation/create` while the `tools/call` is pending, then continues. Requires the shim's read loop to keep reading during a call (the VectorCraft limitation, `docs/mcp.md:242-244`); `craft-hub-mcp` is designed with a reader thread from day one                                           |
| 2     | **Out-of-band approval**                  | No elicitation capability (Paul's llama.cpp stack today; any client that doesn't declare it)                       | The tool result is `isError: true`, code `confirmation_required`, with an **approval id** (`ap-…`) that is *not* the redeemable token, and the text "Waiting for approval: run `craftctl confirm ap-7c21` or approve in the status panel". A human approves via a `craftctl` connection or the panel (both are human-facing clients, `client.kind` `craftctl`/panel, 06 V2). The model then retries the same call, or polls `hub_job_status` |
| (3)   | Client-side tool approval                 | Always, but never relied on                                                                                         | Claude Code/Desktop ask before tool calls according to their own settings; annotations feed that. The hub treats it as advisory: annotations are untrusted hints by spec, and client settings can auto-approve                                                                                                                  |

**The elicitation the shim sends** (form mode; message text is built by the hub from the resolved target and effects, never from model-supplied text):

```json
{
  "resultType": "input_required",
  "inputRequests": {
    "approve": {
      "method": "elicitation/create",
      "params": {
        "mode": "form",
        "message": "PipelineCraft wants to run lightcraft@p1/app.export\nCommand effects: write-user-file (not destructive)\nWrites into: F:\\Exports\\Shoot-0412\\ (1 file per selected photo, 12 photos)\nRequested by MCP client \"claude-code\" on connection c-7f3a",
        "requestedSchema": {
          "type": "object",
          "properties": { "approve": { "type": "boolean", "title": "Run this export" } },
          "required": ["approve"]
        }
      }
    }
  },
  "requestState": "ap-7c21"
}
```

The exact nesting of `inputRequests` (map keyed by name vs list) must be checked against the 2026-07-28 `schema.ts`, which could not be fetched (Not verified #9). The changelog fixes the field names; one third-party article spells the result type `inputRequired`, which disagrees with the changelog's `input_required`, so the shim follows the changelog until the schema is read.

**Binding rules (A7-R2, hands to A8):**

| ID | Rule                                                                                                                                                                                                                                                                |
|----|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| C1 | `requestState` carries only the approval id. The hub keeps target, params hash, effects and connection server-side; a retry whose arguments hash differently from the approved call is refused (`confirmation_required` again), so a model can't swap params after approval |
| C2 | One approval = one call (or one batch). Expires after 2 min (A4's value). Declined or cancelled → the tool returns `isError: true`, code `cancelled`, `ran: "no"`                                                                                                   |
| C3 | Approval ids are random (128-bit, OS CSPRNG, as PdfCraft's token, 03a §3a.5) and useless without a human-facing connection: only `craftctl` (interactive TTY), the status panel, or the shim's own elicitation path can redeem them                                     |
| C4 | A dry run (`hub_plan`) may pre-create the approval for the identical commit (answers 06 open question 5: yes over `craftctl`, and over MCP only as a pending approval that still needs channel 1 or 2)                                                                |
| C5 | Batches and recipes get one approval listing every step that needs it (A4 §6.2.4)                                                                                                                                                                                  |
| C6 | Elicitation never asks for secrets (spec rule); the hub never needs one from the user over MCP                                                                                                                                                                      |
| C7 | Every approval, decline and timeout goes to the audit log with the channel used (A8 §11.5)                                                                                                                                                                          |

**What needs confirmation** is the `confirm` flag from §6.4.1 (`destructive`, or any class beyond `read`/`write-app-state`, or `source: default`). `write-app-state` alone (open a file, import into a bin) doesn't prompt, so AC1 is one call, not one call and a dialog. Paul can tighten that per client in §11.4 (for example "prompt for every write from the local llama.cpp client").

### 10.4 Resources

**URI scheme `craft://`.** Suite-neutral, like the run dir `%LOCALAPPDATA%\CraftApps\run\` (04-05 §5.2), so D1 never renames a URI, and it doesn't collide with the per-app schemes (`filmcraft://document`, `photocraft://document`, …). Instance addressing reuses §6.2.2 (`app@instance`).

| URI / template                            | Content (`mimeType`)                                                                                         | Source                                                                                         | `ttlMs` / `cacheScope` | Size rule                         |
|-------------------------------------------|--------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|------------------------|-----------------------------------|
| `craft://apps`                            | `hub.apps` result (`application/json`)                                                                       | discovery (A3)                                                                                 | 2000 / private         | —                                 |
| `craft://apps/{app}`                      | one app with instances, profile state (`ok \| stale \| missing`), version state                               | A3 §5.6                                                                                        | 2000 / private         | —                                 |
| `craft://commands/{app}`                  | the normalized catalog for that app, every `CommandInfo` (§6.3.3)                                            | registry merge                                                                                 | 600000 / private, refreshed when `registryHash` changes | up to ~1,000 rows; ≤ 2 MiB        |
| `craft://command/{app}/{id}`              | one `CommandInfo` with `paramsSchema`                                                                        | `hub.describe`                                                                                 | 600000 / private       | —                                 |
| `craft://doc/{app}@{instance}`            | the app's own document summary: proxied `doc_inspect` / `document.inspect` / `session.inspect`               | adapter (03a/03b method tables)                                                                | 0 / private            | > 1 MiB → refused with a hint to use `hub_read` with depth limits |
| `craft://selection/assetcraft`            | a `handoff.selection` (07 §7.2.2) built from AssetCraft's selection `{selected, focus}`                      | `assetcraft/app.state` (`crates/engine/src/commands.rs:142`, `:561` @ `3a41b2b`)               | 0 / private            | > 10,000 items → `handoff.file` envelope (07 §7.2) |
| `craft://recent/assets`                   | the last 50 AssetRefs handed off, newest first                                                               | handoff ledger (07 §7.4.6)                                                                     | 5000 / private         | —                                 |
| `craft://recipes`, `craft://recipes/{name}` | recipe list and one recipe's JSON with its effects union                                                    | recipe store (A6 §8.4)                                                                         | 60000 / private        | —                                 |
| `craft://jobs`, `craft://jobs/{job}`       | open and recent hub jobs                                                                                     | job manager (A6 §8.3)                                                                          | 0 / private            | —                                 |
| `craft://thumb/{q1}/{maxSide}`            | PNG blob (`image/png`) from the hub's thumbnail cache                                                        | `derived\thumbs` (07 §7.5.2)                                                                   | 3600000 / private      | ≤ 1 MiB (07 blob cap)             |

`resources/list` returns the fixed URIs plus one `craft://apps/{app}` and `craft://commands/{app}` per installed app; the `{…}` forms are `resources/templates/list` entries. Every list/read result carries `resultType: "complete"`, `ttlMs` and `cacheScope` (required by 2026-07-28); older clients ignore the extra fields, as PhotoCraft already relies on (`conventions.rs:35`).

**Subscriptions.** v1: none (the shim answers `subscriptions/listen` with no subscription types, and legacy `resources/subscribe` as unsupported). v1.1: `resourceSubscriptions` for `craft://selection/assetcraft` and `craft://jobs/{job}` fed by the hub's events (A6 §8.2), because AssetCraft can push selection changes once it is a native hub app. Long calls still report `notifications/progress` on their own response when the client sent a `progressToken` (the EffectCraft pattern, `effectcraft` `docs/mcp.md:44-60`), mapped from hub `job.progress` events.

**Two APIs reminder.** The Storyteller web API defines **remote** "MCP sessions" (`/v1/mcp/session/*`, 03b §3.1b watch-out). They have nothing to do with this front door, which is local stdio first, and `craft://` never names a Storyteller object.

### 10.5 Transports and client configuration

#### 10.5.1 stdio first: `pipelinecraft mcp`

Per A3 (§4.1, §4.3), the front door is a separate stdio process, `pipelinecraft mcp`, in crate `craft-hub-mcp` (L2). The MCP client spawns it; it reads `hub.json` (04-05 §5.2), starts the daemon if needed (D5), opens one hub connection with `hub.hello {client: {kind: "mcp", name: <client name>}}` (06 §6.6.1) and translates.

| Shim behaviour                        | Rule                                                                                                                                                                                                  |
|---------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Versions                              | `2026-07-28` (stateless; answers `server/discover`; reads `_meta` per request) plus `2025-11-25` and `2025-06-18` (`initialize`). Others → `UnsupportedProtocolVersion` (`-32022` in 2026-07-28) or the legacy init error |
| Client name                           | From `_meta` `io.modelcontextprotocol/clientInfo` or `initialize.clientInfo`, else `--client <name>` on the command line; it goes into `hub.hello` and the audit log, labelled *claimed* (not proof, 06 V2) |
| Reader thread                         | Reads stdin while a call runs, so `notifications/cancelled` maps to `hub.job.cancel`, and legacy elicitation works (§10.3 1b)                                                                       |
| stdout                                | MCP frames only; logs to stderr and the hub log (spec stdio rules)                                                                                                                                    |
| Limits                                | 4 MiB per incoming line (as the apps, 06 §6.1.4); tool results ≤ 1 MiB inline, larger results become a `resource_link` to `craft://jobs/{job}` or a hub file                                         |
| Hub down mid-call                     | The tool returns `isError: true`, code `app_disconnected`-style hub error, `ran: maybe`; the shim reconnects on the next call                                                                         |

#### 10.5.2 Streamable HTTP on loopback (later, inside the daemon)

For clients that can't spawn processes (or a FastAPI service that prefers HTTP). Opt-in: `pipelinecraft serve --mcp-http` binds `127.0.0.1:0`, path `/mcp`, and publishes the port in `hub.json` as `"mcpHttp": {"port": n}`.

| Rule (hand to A8 for §11)                                                                                                                                         | Basis                                                                       |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------|
| Bind 127.0.0.1 only, never `0.0.0.0`; NG3                                                                                                                          | spec Security Warning (2024-11-05 onward); 01 NG3                           |
| Reject any request with an `Origin` header that isn't on an explicit allow list (empty by default: browsers are refused outright)                                 | spec "MUST validate Origin"; 01 NG10                                        |
| Reject any `Host` other than `127.0.0.1:<port>` or `localhost:<port>`                                                                                              | CVE-2026-42559 lesson                                                       |
| `Authorization: Bearer <token>` with the hub token from `hub.json` (owner-only file, O12)                                                                          | spec "SHOULD authenticate"; A8 owns the token                               |
| Require `Mcp-Method` / `Mcp-Name` headers and check they match the body                                                                                            | 2026-07-28 minor change 4                                                   |
| No sessions, no SSE resumability; a dropped stream loses the request and the client re-issues it (so `ran: maybe` matters, §6.5)                                   | 2026-07-28 major changes 1, 9                                               |

Owner: Claude (hub), after stdio has shipped (hub Stage 4).

#### 10.5.3 Client configs

**Claude Desktop** (`%APPDATA%\Claude\claude_desktop_config.json`, the standard `mcpServers` shape; not re-checked against current docs, Not verified #10):

```json
{
  "mcpServers": {
    "craft": {
      "command": "C:\\Program Files\\PipelineCraft\\pipelinecraft.exe",
      "args": ["mcp", "--client", "claude-desktop"]
    }
  }
}
```

**Claude Code**, project-scoped `.mcp.json` at the repo root (same `mcpServers` shape with `"type": "stdio"`):

```json
{
  "mcpServers": {
    "craft": {
      "type": "stdio",
      "command": "F:\\Script-Projects\\ArtCrafting\\assetcraft\\target\\release\\pipelinecraft.exe",
      "args": ["mcp", "--client", "claude-code"]
    }
  }
}
```

or user-scoped from PowerShell (§12.3 example 11). Tools then appear as `mcp__craft__hub_read` and so on. Suggested Claude Code permission rules: allow `mcp__craft__hub_apps`, `hub_commands`, `hub_describe`, `hub_read`, `hub_plan`, `hub_job_status`, `hub_resolve`, `thumbnail_render` (all hub-enforced read-only); leave the rest on "ask". The hub's own confirmation still applies to writes either way.

**Generic local client (llama.cpp)** with any MCP host that takes a stdio command:

```json
{ "servers": { "craft": { "command": "pipelinecraft.exe", "args": ["mcp", "--client", "llama-local"] } } }
```

#### 10.5.4 Python example for Paul's stack (llama.cpp + FastAPI + Pydantic + Instructor)

Two ways in. **Option A (recommended for the FastAPI service): speak the hub protocol directly.** It's simpler than MCP (one TCP line per request, typed errors from §6.5, no MCP SDK version churn after 2026-07-28's stateless change), and the Python program, not the model, holds the confirm token and asks Paul, which is exactly the "human channel" §10.3 requires. **Option B: MCP over stdio** with the official Python SDK, when the stack should look like any other MCP host.

Option A, hub protocol (sketch; field names from §6; `hub.json` location from 04-05 §5.2):

```python
# craft_hub.py: minimal hub client for a llama.cpp tool loop. Python 3.11+, pydantic 2.
import json, os, socket, itertools
from pathlib import Path
from typing import Any, Literal
from pydantic import BaseModel

class HubError(BaseModel):
    code: str
    message: str
    origin: Literal["hub", "adapter", "app"]
    retryable: bool = False
    ran: Literal["no", "maybe", "yes"] = "no"
    data: dict[str, Any] = {}

class HubFailure(Exception):
    def __init__(self, err: HubError): super().__init__(err.message); self.err = err

class Hub:
    def __init__(self, client_name: str = "llama-local"):
        m = json.loads((Path(os.environ["LOCALAPPDATA"]) / "PipelineCraft" / "hub.json").read_text("utf-8"))
        self._sock = socket.create_connection(("127.0.0.1", m["control"]["port"]), timeout=70)
        self._r, self._w = self._sock.makefile("r", encoding="utf-8"), self._sock.makefile("w", encoding="utf-8")
        self._ids = itertools.count(1)
        token = m["auth"]["token"] if "token" in m["auth"] else Path(m["auth"]["token_file"]).read_text().strip()
        self.request("hub.hello", {"protocol": "1.0", "auth": {"token": token},
                                   "client": {"name": client_name, "version": "0.1", "kind": "mcp"}})

    def request(self, method: str, params: dict[str, Any]) -> Any:
        rid = next(self._ids)
        self._w.write(json.dumps({"id": rid, "method": method, "params": params}) + "\n"); self._w.flush()
        while True:
            frame = json.loads(self._r.readline())
            if "event" in frame:            # pushed events (A6); this sketch ignores them
                continue
            if frame.get("id") == rid:
                if frame["ok"]:
                    return frame["result"]
                raise HubFailure(HubError.model_validate(frame["error"]))

    def call(self, target: str, params: dict[str, Any] | None = None, ask=input) -> Any:
        try:
            return self.request("hub.call", {"target": target, "params": params or {}})
        except HubFailure as f:
            if f.err.code != "confirmation_required":
                raise
            # The program asks the human; the model never sees the token.
            print(f"\n{f.err.message}\neffects: {f.err.data.get('effects')}")
            if ask("Run it? [y/N] ").strip().lower() != "y":
                raise
            return self.request("hub.call", {"target": target, "params": params or {},
                                             "confirm": f.err.data["confirm"]})
```

Tool loop against llama.cpp's OpenAI-compatible server (`llama-server --jinja`), exposing three tools; Instructor (or plain Pydantic) validates the model's arguments before anything reaches the hub:

```python
# loop.py
from openai import OpenAI
from pydantic import BaseModel, Field
import json
from craft_hub import Hub, HubFailure

class Search(BaseModel):
    query: str = Field(max_length=200)
class Call(BaseModel):
    target: str = Field(pattern=r"^(cap:[a-z0-9.-]+|[a-z][a-z0-9-]*(@[A-Za-z0-9=]+)?/([a-z]+:)?[A-Za-z0-9._-]+)$")
    params: dict = {}

TOOLS = [{"type": "function", "function": {"name": n, "description": d, "parameters": m.model_json_schema()}}
         for n, d, m in [("hub_commands", "Search Craft app commands", Search),
                         ("hub_read", "Run a read-only command", Call),
                         ("hub_call", "Run a command; writes ask the user", Call)]]

hub, llm = Hub(), OpenAI(base_url="http://127.0.0.1:8080/v1", api_key="local")
msgs = [{"role": "user", "content": "Which FilmCraft commands export OTIO?"}]
for _ in range(8):
    out = llm.chat.completions.create(model="local", messages=msgs, tools=TOOLS).choices[0].message
    msgs.append(out.model_dump(exclude_none=True))
    if not out.tool_calls:
        print(out.content); break
    for tc in out.tool_calls:
        args = json.loads(tc.function.arguments)
        try:
            if tc.function.name == "hub_commands":
                res = hub.request("hub.commands", {"filter": Search(**args).query})
            elif tc.function.name == "hub_read":
                c = Call(**args); res = hub.request("hub.call", {"target": c.target, "params": c.params, "readOnly": True})
            else:
                c = Call(**args); res = hub.call(c.target, c.params)
            content = json.dumps(res)[:20000]
        except HubFailure as f:
            content = json.dumps({"error": f.err.model_dump()})
        # App text is data, not instructions (§10.6 G4).
        msgs.append({"role": "tool", "tool_call_id": tc.id, "content": "UNTRUSTED APP DATA:\n" + content})
```

`readOnly: true` on `hub.call` is the hub-side switch behind `hub_read` (A7-F2: a new optional param for A4's `hub.call`; the hub answers `forbidden` for non-read targets). Note the sketch's `hub.commands` uses `filter`; the ranked `query` search is part of A7-F3.

Option B, MCP stdio with the official Python SDK (API as of the 1.x SDK; a 2026-07-28 SDK may differ, Not verified #11):

```python
import asyncio
from mcp import ClientSession, StdioServerParameters
from mcp.client.stdio import stdio_client

async def main():
    srv = StdioServerParameters(command=r"C:\Program Files\PipelineCraft\pipelinecraft.exe", args=["mcp", "--client", "llama-local"])
    async with stdio_client(srv) as (r, w), ClientSession(r, w) as s:
        await s.initialize()
        hits = await s.call_tool("hub_commands", {"query": "export otio"})
        print(hits.structuredContent)
asyncio.run(main())
```

This client declares no elicitation, so writes come back as `confirmation_required` with an approval id, approved with `craftctl confirm` (§10.3 channel 2).

### 10.6 Prompt-injection guardrails at the tool layer

The threat: text the model reads (a PDF's words, an OTIO's metadata, a file name, a WordCraft document, a FilmCraft marker, a catalog label) tells it to do something destructive. The hub can't make a model resist that, so the tool layer makes sure a fooled model can't do damage without a person seeing exactly what will happen.

| ID  | Guardrail                                                                                                                                                                                                                                                                                                     | Where enforced                       |
|-----|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------|
| G1  | **Read-only tools are enforced, not hinted.** `hub_read`, `hub_plan`, discovery and `thumbnail_render` can't reach a non-read target (`forbidden`)                                                                                                                                                            | hub (`hub.call {readOnly}`)          |
| G2  | **Human-only confirmation.** The model never receives a redeemable token (§10.3, C1–C3)                                                                                                                                                                                                                       | shim + hub                           |
| G3  | **Hub-authored prompts.** The confirmation text comes from the resolved address, effects and the paths in params, rendered by the hub. Free text from the model (a "reason") is shown only under a "the assistant says:" label, never as the headline                                                         | shim                                 |
| G4  | **Static tool descriptions; app data only in results.** Tool names and descriptions are compiled in. Catalog labels, document text and file names reach the model only in results and resources, with control characters stripped and labels cut to 120 chars. Text results are prefixed `untrusted app data:`; `structuredContent` is the primary channel | shim                                 |
| G5  | **No UI scripting or script execution over MCP by default.** Deny list in the MCP policy: `*/method:ui.*`, `*/ui.*` (incl. `ui.dialog.confirm`, which would click through an app's own safety dialog), `*/method:app.quit`, `assetcraft/file.run`, EffectCraft `file.runScript` and `run_script`, DesignCraft/FilmCraft/PdfCraft script modes. Paul can allow ids per client (§11.4) | hub policy (A8)                      |
| G6  | **Paths stay in roots.** Write targets must resolve inside configured roots (07 §7.4; `path_denied`), and the confirmation lists every output folder                                                                                                                                                          | hub (A5 rules)                       |
| G7  | **Budgets.** Per MCP connection: ≤ 256 steps per batch (06 §6.2.4), ≤ 20 approved writes per 10 min, ≤ 4 concurrent jobs; over budget → `busy` with `retryable: false`                                                                                                                                         | hub (A8 sets numbers)                |
| G8  | **Capability resolution is shown.** A `cap:` call's prompt names the app that will run it (`resolved`), so a capability can't silently land on another app                                                                                                                                                    | shim                                 |
| G9  | **Unknown means worst case.** `source: default` commands prompt and are never auto-retried (06 §6.4.2)                                                                                                                                                                                                        | hub                                  |
| G10 | **Everything audited**, with the claimed client name, the confirmation channel and the approval id (A8 §11.5)                                                                                                                                                                                                  | hub                                  |

**Threat model rows handed to A8 (§11.6):**

| Threat                                                         | Path                                                                  | Tool-layer control                    | What A8 must still decide                                              |
|----------------------------------------------------------------|-----------------------------------------------------------------------|---------------------------------------|-------------------------------------------------------------------------|
| Injected text in a document asks the model to delete/overwrite | model → `hub_call` → app                                              | G2, G3, G6, G9                        | Whether any client may be trusted to self-confirm (recommend: none)    |
| Injected text asks for UI clicks that pass an app's own dialog | model → `hub_call ui.dialog.confirm`                                  | G5                                    | The per-client allow list format                                        |
| Model replays an approval with different params               | model → retry with altered args                                       | C1 params-hash binding                | Token lifetime (2 min default)                                          |
| Malicious tool description from another MCP server in the same client ("tool poisoning") tells the model to call `hub_call` | other server → model → hub | G2, G3 (the user sees the hub's own prompt) | Nothing hub-side beyond G2; document it for Paul                        |
| Local process spoofs `client.kind: "craftctl"` to self-approve | rogue process → hub socket                                            | C3 needs an **interactive** craftctl  | How the hub tells a real TTY approval from a script (A8: per-user token + console check, or panel-only approvals) |
| Browser page posts to the Streamable HTTP endpoint             | DNS rebinding                                                         | §10.5.2 Origin/Host/bearer rules      | Final transport rules; whether HTTP ships at all in v1                  |
| Oversized or hostile app replies flood the model               | app → shim                                                            | 1 MiB inline cap, truncation, G4      | Limits table in §11                                                     |

---

## 12. `craftctl` CLI

`craftctl` is the hub's human and script client (04-05 §4.1, crate `craftctl`, exempt app row). It starts the daemon on demand from `hub.json` (D5), connects with `hub.hello {client: {kind: "craftctl"}}`, and maps each subcommand to one hub method from §6.2.1, so it adds no behaviour the protocol doesn't have (everything is a command). The name stays `craftctl` whatever D1 decides (A1-N1).

### 12.1 Command tree

```text
craftctl [GLOBAL FLAGS] <command> …

  apps        [--running | --installed] [--refresh] [--app <app>]                        → hub.apps
  commands    [<app>] [--search <text>] [--filter <substr>] [--effects <class,…>]
              [--enabled] [--capability <cap>] [--limit <n>] [--all]                      → hub.commands
  describe    <target>                                                                    → hub.describe / hub.route
  call        <target> [--params <json> | --params-file <path> | key=value …]
              [--no-wait] [--timeout <ms>] [--dry-run] [--yes] [--read-only]              → hub.call
  batch       <file.jsonl | -> [--dry-run] [--yes] [--keep-going]                         → hub.batch
  run         <recipe> [--dry-run] [--yes] [--arg key=value …]                            → recipe run (A6)
  handoff     <app | cap:…> [<path> …] [--from-selection] [--intent open|import|thumbnail]
              [--instance <id>] [--yes]                                                   → hub handoff (§7.2)
  watch       [<topic-pattern> …] [--app <app>] [--since-now]                             → hub.subscribe
  jobs        [list] [--app <app>] [--state <s>]                                          → hub.job.list
  jobs        get <job> | wait <job> [--timeout <ms>] | cancel <job>                      → hub.job.get/wait/cancel
  confirm     [list] | <approval-id> | deny <approval-id>                                 → approvals (§10.3 channel 2)
  resolve     <path | ref.json>                                                           → AssetRef resolve (07 §7.1.4)
  hash        <path> [--full]                                                             → q1 / full BLAKE3 (07 §7.1.3)
  profile     check <app> | seed <app> --from-source <repo>                               → A4 §6.4.3, A4-F5
  doctor      [--app <app>] [--probe] [--only <id,…>]                                     → hub.doctor (§12.5)
  hub         status | start | stop | logs [--tail <n>]                                   → daemon lifecycle (A3)
  version
```

**Global flags**

| Flag                    | Meaning                                                                                                                  | Default                          |
|-------------------------|--------------------------------------------------------------------------------------------------------------------------|----------------------------------|
| `--json`                | Machine output on stdout (§12.4); nothing else on stdout. Progress and prompts go to stderr                             | off (human tables)               |
| `--hub <path>`          | Path of `hub.json`                                                                                                       | `%LOCALAPPDATA%\PipelineCraft\hub.json` (or `PIPELINECRAFT_HOME`) |
| `--no-start`            | Don't start the daemon if it isn't running; fail with exit 3                                                             | start on demand                  |
| `--timeout <ms>`        | Per-request deadline (passed as `timeoutMs`)                                                                             | 60000                            |
| `--quiet` / `-q`        | Human mode: results only, no headers                                                                                     | off                              |
| `--color auto\|never`   | ANSI colour in human mode                                                                                                | `auto` (off when not a TTY)      |
| `--yes` / `-y`          | Accept a confirmation prompt without asking. Still bound by per-connection permissions (§11.4); never accepted when stdin is not a TTY **and** `--yes` is absent | off                              |

**Params on the command line.** `--params '{"paths":["F:\\a.mov"]}'`, `--params-file p.json`, or `key=value` pairs where the value is parsed as JSON if it parses, else a string (FilmCraft `exec` and LightCraft `run` already take `k=v`, 03a §3a.3–3a.4). Mixing forms is a usage error (exit 2).

**Confirmations in `craftctl`.** On `confirmation_required`, an interactive `craftctl` prints the hub's prompt (target, effects, paths) to stderr and asks `Proceed? [y/N]`; on `y` it re-sends with the token (06 Example 3). Without a TTY and without `--yes`, it exits 4 and prints the approval id, so a script can't approve itself by accident.

### 12.2 Exit codes

Stable, documented, small numbers; 0/1/2 match `assetcraft-cli` (`apps/assetcraft-cli/src/main.rs:139-143`, `:149-175` @ `3a41b2b`).

| Exit | Meaning                                                                 | Hub error codes (06 §6.5.2) that produce it                                                                                         |
|------|-------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------|
| 0    | Success                                                                 | —                                                                                                                                   |
| 1    | The command ran (or was attempted) and failed in the app               | `disabled`, `bad_params`, `no_document`, `not_found`, `app_busy`, `not_persisted`, `app_internal`, `app_error`, `cancelled`, `result_too_large` |
| 2    | Usage error or invalid request                                          | (CLI parse errors), `bad_target`, `params_invalid`, `unknown_command`, `unknown_app`, `bad_request`                                 |
| 3    | Hub unreachable or refused the client                                   | (no `hub.json`, connect failed, `--no-start`), `unauthenticated`, `protocol_version`, `hello_required`, `internal`                  |
| 4    | Confirmation needed and not given, or forbidden by policy               | `confirmation_required` (non-interactive), declined prompt, `forbidden`, `path_denied`                                              |
| 5    | Target not reachable or not unique                                       | `app_unavailable`, `ambiguous_target`, `no_capable_app`, `busy`, `app_auth_failed`                                                   |
| 6    | Outcome unknown: the command **may** have applied                       | `timeout_unknown`, `app_disconnected`, `adapter_protocol` (any error with `ran: "maybe"`)                                           |
| 7    | Partial success: a batch or recipe had failed steps (`--keep-going`)    | per-step errors                                                                                                                     |
| 8    | `doctor`: warnings only                                                 | —                                                                                                                                   |
| 10   | `doctor`: at least one check failed                                     | —                                                                                                                                   |
| 124  | `jobs wait` / `--timeout` reached and the job is still running          | `job_still_running`, `timeout_not_run`                                                                                              |
| 130  | Interrupted (Ctrl+C); for `watch` this is the normal way out            | —                                                                                                                                   |

Rule: when `error.ran` is `maybe`, exit 6 wins over the code's own row, because "did it happen?" is what a script must branch on.

### 12.3 PowerShell examples

Each block assumes the workspace layout from A3-R1 (hub and `craftctl` incubated in the AssetCraft workspace) and a release build. Paths and ids are illustrative; nothing has been run.

**1. Build once, then see what's running (Stage 0 "See").**

```powershell
Set-Location F:\Script-Projects\ArtCrafting\assetcraft
cargo build --release -p pipelinecraft -p craftctl
$env:Path = "F:\Script-Projects\ArtCrafting\assetcraft\target\release;$env:Path"
craftctl apps
craftctl apps --json | ConvertFrom-Json | Where-Object { $_.instances.Count -gt 0 } | Select-Object app, version, state
```

Expected: a table with one row per installed app, `state` `ok`/`untested`/`unsupported`/`unreachable`/`running, not controllable` and the discovery source (A3 input to A7).

**2. Search commands across every app, then describe one.**

```powershell
Set-Location F:\Script-Projects\ArtCrafting\assetcraft
craftctl commands --search "export otio"
craftctl commands filmcraft --effects read --enabled --json | ConvertFrom-Json | Select-Object -ExpandProperty commands | Measure-Object
craftctl describe filmcraft/file.import --json | ConvertFrom-Json | Select-Object -ExpandProperty result | Select-Object address, effects, paramsSchemaSource
```

**3. A read-only call (no prompt), with `key=value` params.**

```powershell
Set-Location F:\Script-Projects\ArtCrafting\assetcraft
craftctl call lightcraft/library.importPreview paths='["F:\\Library\\shoots\\2026"]' --read-only --json |
  ConvertFrom-Json | Select-Object -ExpandProperty result
```

`--read-only` makes the hub refuse the call if the target isn't `read` (the same switch as MCP `hub_read`).

**4. Open In: a write that needs no prompt, then one that does.**

```powershell
Set-Location F:\Script-Projects\ArtCrafting\assetcraft
craftctl handoff photocraft "F:\Library\shoots\2026\IMG_0412.psd" --intent open
craftctl call lightcraft/app.export --params '{"dir":"F:\\Exports\\Shoot-0412","format":"jpeg","longEdge":2048}'
# prints the hub's prompt on stderr: lightcraft@p1/app.export · write-user-file · F:\Exports\Shoot-0412\  Proceed? [y/N]
```

**5. Dry-run a batch from a JSONL file, then commit it with one confirmation.**

```powershell
Set-Location F:\Script-Projects\ArtCrafting\assetcraft
@'
{"target":"filmcraft/file.import","params":{"paths":["F:\\Library\\clips\\A001.mov","F:\\Library\\clips\\A002.mov"]}}
{"target":"cap:timeline.export.otio","params":{"path":"F:\\Plan\\rough.otio"}}
'@ | Set-Content -Encoding utf8 $env:TEMP\rough.jsonl
craftctl batch $env:TEMP\rough.jsonl --dry-run
craftctl batch $env:TEMP\rough.jsonl
```

The dry run prints each step's `resolved` address and command effects and writes nothing (06 Example 7); the second run asks once for the whole batch.

**6. Watch events while a long job runs in another window.**

```powershell
Set-Location F:\Script-Projects\ArtCrafting\assetcraft
craftctl watch "job.*" "app.*" --app photocraft --json | ForEach-Object {
  $e = $_ | ConvertFrom-Json
  "{0}  {1,-14} {2}" -f $e.at, $e.event, ($e.data | ConvertTo-Json -Compress)
}
```

Each stdout line is one event frame (§12.4); Ctrl+C ends it with exit 130.

**7. Start a job without waiting, then wait or cancel it.**

```powershell
Set-Location F:\Script-Projects\ArtCrafting\assetcraft
$job = (craftctl call photocraft/filter.blur.gaussianBlur radius=4 --no-wait --json | ConvertFrom-Json).result.job
craftctl jobs wait $job --timeout 30000
if ($LASTEXITCODE -eq 124) { craftctl jobs cancel $job }
```

(The PhotoCraft filter id is unverified, 06 Not verified #8; check it with `craftctl commands photocraft --search blur`.)

**8. Run a recipe: dry run first, then commit (AC3 from the CLI).**

```powershell
Set-Location F:\Script-Projects\ArtCrafting\assetcraft
craftctl run plan-to-filmcraft --arg storylineId=sl_01JC --dry-run
craftctl run plan-to-filmcraft --arg storylineId=sl_01JC --json | ConvertFrom-Json | Select-Object -ExpandProperty result | Select-Object clips, durationTicks, checks
```

**9. Branch on exit codes in a script.**

```powershell
Set-Location F:\Script-Projects\ArtCrafting\assetcraft
craftctl call wordcraft/file.exportPdf --params '{"path":"F:\\Docs\\brief.pdf"}' --yes --json > $env:TEMP\r.json
switch ($LASTEXITCODE) {
  0 { "exported" }
  4 { "needs approval: " + (Get-Content $env:TEMP\r.json | ConvertFrom-Json).error.data.approval }
  5 { "WordCraft is not running with --control" }
  6 { "unknown outcome; check whether F:\Docs\brief.pdf appeared before retrying" }
  default { "failed: " + (Get-Content $env:TEMP\r.json | ConvertFrom-Json).error.message }
}
```

**10. Approve a write an MCP client asked for (channel 2 in §10.3).**

```powershell
Set-Location F:\Script-Projects\ArtCrafting\assetcraft
craftctl confirm list
craftctl confirm ap-7c21
```

`confirm list` shows each pending approval with client name, target, effects and paths; `confirm deny <id>` refuses one.

**11. Register the front door with Claude Code (user scope).**

```powershell
Set-Location F:\Script-Projects\ArtCrafting\assetcraft
claude mcp add --scope user craft -- "F:\Script-Projects\ArtCrafting\assetcraft\target\release\pipelinecraft.exe" mcp --client claude-code
claude mcp list
```

(`claude mcp add` syntax from general knowledge; Not verified #10.)

**12. Run the Windows checks and show only what needs attention.**

```powershell
Set-Location F:\Script-Projects\ArtCrafting\assetcraft
craftctl doctor --json | ConvertFrom-Json | Select-Object -ExpandProperty checks |
  Where-Object status -ne 'pass' | Format-Table id, status, title, fix -Wrap
"exit: $LASTEXITCODE"   # 0 all pass, 8 warnings, 10 failures
```

### 12.4 `--json` output shapes

One JSON document per invocation on stdout (UTF-8, no BOM, one trailing newline), except `watch`, which is NDJSON. Shapes are the hub reply without the `id`, so a script sees exactly what the protocol returns.

| Command                   | Success shape                                                                                                           | Failure shape                                       |
|---------------------------|-------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------|
| every non-streaming       | `{"ok": true, "result": <hub result>, "resolved"?: "<address>", "warnings": [ … ]}`                                    | `{"ok": false, "error": <§6.5.1 error object>, "exit": <n>}` |
| `apps`                    | `result` = `hub.apps` array; each instance adds `state` and `source` (`config \| manifest \| artefact \| probe`)        | as above                                            |
| `commands`                | `result` = `{registryHash, total, nextCursor?, commands: [CommandInfo]}` (full rows; `--search` adds `score`)           | as above                                            |
| `call` with `--no-wait`   | `result` = `{"job": "j7", "pending": true}`                                                                             | as above                                            |
| `call --dry-run`, `batch --dry-run`, `run --dry-run` | `result` = the plan: `{"dryRun": true, "steps": [ … ], "effects": { … }, "approval"?: "ap-…"}`                | as above                                            |
| `batch`, `run`            | `result` = `{"completed": n, "failed": n, "steps": [{"target", "resolved", "ok", "result" \| "error"}]}`                | top-level `ok: false` only if nothing ran; else exit 7 with `ok: true` and failed steps |
| `jobs …`                  | `result` = one job or `{"jobs": [ … ]}` (A6 shape)                                                                      | as above                                            |
| `confirm list`            | `result` = `{"pending": [{"approval", "client", "target", "effects", "paths", "expiresAt"}]}`                            | as above                                            |
| `watch`                   | one §6.2.6 event frame per line: `{"event", "sub", "seq", "source", "at", "data"}`                                     | a final `{"ok": false, "error": …}` line, then exit |
| `doctor`                  | §12.5 report                                                                                                            | same report; exit 8 or 10                           |
| `version`                 | `{"ok": true, "result": {"craftctl": "0.1.0", "hub": "0.1.0" \| null, "protocol": "1.0"}}`                               | —                                                   |

Human mode prints tables (space-padded, like the docs) and never prints secrets or tokens; approval ids are shown, confirm tokens never are.

### 12.5 `craftctl doctor`

**Purpose.** Turn the "Not verified" scripts in 01, 03a, 03b, 04-05, 06 and 07 into one repeatable command, so Paul runs one line instead of thirty, and the answers land in adapter config and STATUS.md. Doctor is itself a hub command, `hub.doctor` (everything is a command); when the daemon is down, `craftctl` runs the same checks in-process from the shared crate (A7-F5) so doctor works exactly when something is broken.

**Rules.**
- **Read-only by default.** No app is started, no socket is written to an app the hub didn't launch, nothing is deleted. Fix suggestions are printed, not applied.
- `--probe` adds **active** checks: it launches apps headless or with a throwaway profile (`--control` on a port from `port_range`, a temp folder under `%LOCALAPPDATA%\PipelineCraft\tmp\doctor\`), talks only to the processes it started, and ends them afterwards. It never touches a window Paul opened.
- No PowerShell dependency. Doctor shells out only to built-in executables (`icacls.exe`, `reg.exe`) with argv (no shell), parses their output, and reports `skip` if they're missing or blocked; PdfCraft's history with Constrained Language Mode policies (`pdfcraft` `ROADMAP.md:153` @ `ac89a11`) is the reason not to rely on `powershell.exe`. No `unsafe`, so no direct Win32 ACL calls in the workspace.
- Each check has an id, a source (the doc and step it automates), a status (`pass | warn | fail | skip`), evidence, a fix and an owner, in the house findings style.

**Checks.**

| ID  | Check                                                                                                 | Automates                                  | Mode     | Pass / warn / fail                                                                                                                         | Fix and owner on non-pass                                                                    |
|-----|-------------------------------------------------------------------------------------------------------|--------------------------------------------|----------|--------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------|
| D01 | `hub.json` exists, parses (manifest v1), pid alive, port answers `hub.hello`                          | 04-05 §5.2                                 | read     | pass: hub up · warn: hub not running (on-demand is normal) · fail: stale or invalid file                                                    | `craftctl hub start`; delete stale file (Claude, hub code)                                   |
| D02 | ACL of `%LOCALAPPDATA%\PipelineCraft\` and `tokens\` (`icacls`)                                       | 03a O12, Not verified #6                   | read     | pass: only the user, SYSTEM, Administrators · fail: any other principal                                                                     | reset ACL (Paul); hub creates with user-only ACL (Claude)                                    |
| D03 | ACL of the suite run dir `%LOCALAPPDATA%\CraftApps\run\`                                              | 04-05 Not verified #7                      | read     | same as D02                                                                                                                                | same                                                                                         |
| D04 | ACL of WordCraft key files and PhotoCraft token files the hub uses                                    | 03a O12, Not verified #5–6                 | read     | same as D02; warn if a key file has no live `wordcraft.exe` (stale)                                                                         | Paul (ACL); WordCraft removes stale files itself                                             |
| D05 | `LongPathsEnabled` (`reg query HKLM\SYSTEM\CurrentControlSet\Control\FileSystem`)                    | 07 Not verified #1                         | read     | pass: 1 · warn: 0 (apps without `longPathAware` may fail over 260 chars)                                                                   | Paul (policy); per-app `long_paths` config (Claude, A5-O8)                                  |
| D06 | Installed apps found by the locator; each CLI answers `--version`; version inside profile `tested`    | 04-05 §5.1, §5.6, A3-F6                    | read     | pass: inside · warn: `untested` · fail: CLI missing next to the GUI                                                                         | update profile range (Claude); reinstall (Paul)                                              |
| D07 | Catalog fetch per app through its configured source; count; detect O-B3 quirks (DesignCraft `--json` → `[]`, SoundCraft `--json` → nothing, PdfCraft has `tools` only) | 03a Not verified #1–2; 03b #1–4 | read (spawns CLIs, read-only) | pass: count within ±10 % of the profile's expected count · warn: outside · fail: zero or parse error                                       | update `catalog` quirk (Claude); upstream O-B3 (apps)                                        |
| D08 | `profile check` for every installed app: unknown TOML keys, missing evidence, live commands falling to `source: default` | 06 §6.4.3                       | read     | pass: 0 errors · warn: defaults exist (the curation to-do list) · fail: invalid profile                                                    | curate rows (Claude); review (Paul)                                                          |
| D09 | Running Craft processes and argv: who has `--control`, on which port, with or without auth            | 04-05 Not verified #3                      | read     | pass: none or all authenticated · warn: unauthenticated control ports open (LightCraft, FilmCraft, DesignCraft, EffectCraft, VectorCraft, SoundCraft) | close when unused (Paul); upstream O2 token handshake (apps)                                  |
| D10 | Port plan: how many of `port_range` (47600–47699) are free; who holds 7979                             | 03b O-B7, 04-05 §4.4                       | read     | pass: ≥ 20 free · warn: < 20 · fail: 0                                                                                                     | change `port_range` (Paul)                                                                   |
| D11 | Stale artefacts: run-dir manifests with dead pids; PdfCraft control files under the hub folder with dead pids | 04-05 §5.4, A3-F4, Not verified #4  | read     | pass: none · warn: stale files found (listed)                                                                                              | hub cleans its own on next pass (Claude); upstream A3-O5                                     |
| D12 | MCP client configs: does `%APPDATA%\Claude\claude_desktop_config.json` (and `.mcp.json` in the cwd) point at an existing `pipelinecraft.exe`? | §10.5.3                        | read     | pass: points at this build · warn: not configured · fail: points at a missing exe                                                          | Paul (config)                                                                                |
| D13 | Hub round trip: 100 × `hub.ping`, p50/p95                                                             | 01 P1                                      | read     | pass: p95 ≤ 10 ms · warn: above                                                                                                            | A9 (perf)                                                                                    |
| P01 | Piped stdout from a release GUI build: launch `soundcraft --control 0 --no-audio`, read `SOUNDCRAFT_CONTROL_PORT=` | 04-05 A3-O7, Not verified #1     | probe    | pass: port line read · fail: empty pipe (port discovery must use another source)                                                           | adapter `port_zero` change (Claude)                                                          |
| P02 | Port 0 loss: launch LightCraft with `--control 0`, check the logged address                          | 04-05 A3-F1, 03a O7                        | probe    | pass if real port printed (upstream fixed) · warn: `127.0.0.1:0` (expected today)                                                          | keep `port_zero = "lost"` (Claude); upstream A3-O4                                          |
| P03 | Bind failure: launch two apps on one port; the second must not be mistaken for the first              | 04-05 A3-F2, Not verified #2               | probe    | pass: identity probe rejects the wrong listener                                                                                           | Claude (identity probe ids)                                                                 |
| P04 | SoundCraft request-only guard: HTTP-shaped preamble, then a harmless read, against a doctor-launched SoundCraft | 03b O-B2, Not verified #6           | probe    | pass: connection closes after junk · **fail: the JSON line after junk runs** (expected today)                                              | upstream PR (SoundCraft; Paul files); keep `closes_on_junk = false` handling (Claude)        |
| P05 | Auth handshakes: PhotoCraft token-frame, WordCraft key-field, PdfCraft control file, against doctor-launched instances | 03a Not verified #3, #5, #8      | probe    | pass: each handshake succeeds and a wrong token is refused                                                                                 | adapter auth config (Claude)                                                                 |
| P06 | Error-string prefixes: `unknown command`, `not available` per app                                     | 06 Not verified #1–2                       | probe    | pass: every prefix in the profile table matches · warn: unmatched text (falls to `app_error`)                                              | profile error table (Claude, A4-F2)                                                          |
| P07 | Long path open: a generated > 300-char path under the doctor temp folder, opened headless per app      | 07 Not verified #2–3                       | probe    | pass / fail per app                                                                                                                        | `long_paths` per app (Claude); manifest PR (apps)                                           |
| P08 | FilmCraft OTIO round trip: export a tiny sequence to the temp folder, re-import, compare clip count   | 01 Not verified #4; 07 #8                  | probe    | pass: same count · fail: different or error                                                                                               | AC3 blocker (Claude / FilmCraft)                                                            |

**Report shape** (`--json`):

```json
{
  "ok": true,
  "result": {
    "doctor": 1,
    "host": {"os": "Windows 11 Pro 24H2", "user": "paul", "hub": "0.1.0", "protocol": "1.0"},
    "summary": {"pass": 14, "warn": 4, "fail": 1, "skip": 2},
    "checks": [
      {"id": "D05", "title": "Long paths enabled", "status": "warn", "mode": "read",
       "detail": "LongPathsEnabled = 0", "evidence": {"reg": "HKLM\\SYSTEM\\CurrentControlSet\\Control\\FileSystem", "value": 0},
       "fix": "Enable Win32 long paths, or keep per-app long_paths = refuse", "owner": "Paul", "source": "07 Not verified #1"},
      {"id": "P04", "title": "SoundCraft request-only guard", "status": "fail", "mode": "probe",
       "detail": "a JSON line after an HTTP preamble ran (engine.parity answered ok)",
       "fix": "Port DesignCraft's guard into soundcraft control_server.rs:97-137", "owner": "upstream (SoundCraft); Paul files", "source": "03b O-B2"}
    ]
  }
}
```

Exit code: 0 all pass or skip, 8 warnings only, 10 any failure (§12.2). Human mode prints the same rows as a padded table and ends with a ready-to-paste STATUS.md findings block.

---

## Findings and recommendations from this section

| ID    | Scope                       | Opportunity                                                                                                                                                         | Suggested fix                                                                                                                                                                                         | Owner                                  | Status |
|-------|-----------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------|--------|
| A7-R1 | §10.1 tool surface          | ~5,800 commands can't be tools                                                                                                                                      | 14 static tools (11 generic over the merged registry + 3 workflow tools for AC1–AC3), catalogs as `craft://` resources, lexical search shared with `craftctl`                                         | Paul (approve); Claude (build)         | open   |
| A7-R2 | §10.3 confirmation          | A confirm token returned to a model confirms nothing                                                                                                                | MRTR elicitation (2026-07-28) → legacy elicitation → out-of-band `craftctl confirm` / panel; approval bound to params hash; the model never holds a redeemable token                                 | Paul (approve); A8 (policy); Claude    | open   |
| A7-R3 | §10.5 transports            | Two MCP dialects in the field (stateless 2026-07-28 and `initialize`-based)                                                                                         | `craft-hub-mcp` serves 2026-07-28, 2025-11-25, 2025-06-18 from one hand-written server with a reader thread (VectorCraft's multi-version precedent); Streamable HTTP later with Origin/Host/bearer rules | Claude                                 | open   |
| A7-F1 | §10.1 naming; conflicts with 06 input to A7 | A4 suggested tool names like `pdfcraft.tool.doc_open`. Dots are legal in MCP (SEP-986) but not in every downstream LLM function-name rule              | Hub tool names use `[a-z0-9_]` only; addresses with dots and `kind:` travel as the `target` **argument**, not in tool names. Integrator: log as resolved against 06                                  | Claude (integrator)                    | open   |
| A7-F2 | §6.2.1 `hub.call`           | `hub_read`'s guarantee needs a hub-side switch                                                                                                                      | Add optional `readOnly: bool` to `hub.call` (refuse non-read targets with `forbidden`). Additive minor change per 06 D-1                                                                              | A4 (accept); Claude                    | open   |
| A7-F3 | §6.2.1 `hub.commands`       | `filter` is a substring; models and Paul need ranked search                                                                                                         | Add `query`, `limit`, `cursor`, `capability` to `hub.commands` with the scoring in §10.1.2; `craftctl commands --search` uses it                                                                     | A4 (accept); Claude                    | open   |
| A7-F4 | §6 approvals                | Out-of-band approval needs methods                                                                                                                                  | `hub.approval.list`, `hub.approval.accept {approval}`, `hub.approval.deny {approval}`, callable only from human-facing connections (A8 defines how that is proven)                                    | A4, A8; Claude                         | open   |
| A7-F5 | §4.3 crates                 | Doctor must work when the hub is down                                                                                                                               | New crate `craft-hub-doctor` (L3, depends on discovery + adapters + proto), used by `pipelinecraft` (`hub.doctor`) and `craftctl`; add a `TABLE` row and keep it inside the `craft-` island (A3-O2) | A3/A9 (layer row); Claude              | open   |
| A7-F6 | AssetCraft Stage 1.5 (O-B9) | `craft://selection/assetcraft` and `open_in` read the selection through `app.state`, which returns more than the selection                                          | Add `selection.get` → `handoff.selection` (07 §7.2.2) to AssetCraft's registry, next to A5-O2's `asset.ref`                                                                                           | Claude (AssetCraft)                    | open   |
| A7-F7 | §10.6 G5                    | Several apps expose dialog-confirming and script-running commands that would let a model pass an app's own safety prompt                                            | Default MCP deny list (G5) in hub policy; per-client allow list in §11.4                                                                                                                             | A8 (policy); Claude                    | open   |
| A7-F8 | Upstream (O-B4)             | No sibling MCP server can elicit, because each reads one line and replies before reading the next (`vectorcraft` `docs/mcp.md:242-244`)                            | When `craft-control` (O-B1) grows an MCP shim, give it a reader thread and MRTR support, so per-app servers can ask for confirmation too. Not needed for the hub, which confirms on its own side      | Claude (craft-control); upstream later | open   |
| A7-F9 | §12 `craftctl`              | A non-interactive script must never approve itself                                                                                                                  | Exit 4 without TTY and `--yes`; `--yes` still bound by §11.4; approvals for MCP calls only through `craftctl confirm` on a TTY                                                                       | Claude; A8                             | open   |

## Not verified

Everything above is a static read plus web checks of the MCP spec; nothing was built or run, and the hub doesn't exist yet. Steps 1–8 can run today against the apps; 9–11 are documentation checks.

**Next action (Paul):**

1. Confirm the suite convention tools in two apps (expected: `command_list`, `command_run`, `command_batch`, `doc_inspect`, `render_preview` in both `tools/list` replies, each with all four hints).
   ```powershell
   Set-Location F:\Script-Projects\ArtCrafting\filmcraft
   $in = @('{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-06-18","capabilities":{},"clientInfo":{"name":"probe","version":"0"}}}','{"jsonrpc":"2.0","method":"notifications/initialized"}','{"jsonrpc":"2.0","id":2,"method":"tools/list"}') -join "`n"
   $in | cargo run -q --release -p filmcraft-cli -- mcp --demo | Select-String '"command_list"|"render_preview"'
   Set-Location F:\Script-Projects\ArtCrafting\effectcraft
   $in | cargo run -q --release -p effectcraft-cli -- mcp | Select-String '"command_batch"|"doc_inspect"'
   ```
2. Confirm PhotoCraft already returns 2026-07-28 cache fields (expected: `"resultType":"complete"`, `"ttlMs":600000`, `"cacheScope":"private"` in the `tools/list` reply).
   ```powershell
   Set-Location F:\Script-Projects\ArtCrafting\photocraft
   $in = @('{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-06-18","capabilities":{},"clientInfo":{"name":"probe","version":"0"}}}','{"jsonrpc":"2.0","method":"notifications/initialized"}','{"jsonrpc":"2.0","id":2,"method":"tools/list"}') -join "`n"
   $in | cargo run -q --release -p photocraft-cli -- mcp | Select-String 'ttlMs'
   ```
3. Confirm VectorCraft accepts a 2026-07-28 request without `initialize` (expected: a `tools/list` reply with `resultType`; if it errors, the stateless form needs `server/discover` first, which tells us how far the sibling implementation goes).
   ```powershell
   Set-Location F:\Script-Projects\ArtCrafting\vectorcraft
   '{"jsonrpc":"2.0","id":1,"method":"tools/list","params":{"_meta":{"io.modelcontextprotocol/protocolVersion":"2026-07-28"}}}' | cargo run -q --release -p vectorcraft-cli -- mcp --headless
   ```
4. Count the live command total that §10.1.1 uses (expected: on the order of 5,000–6,000 across nine apps; record the number).
   ```powershell
   Set-Location F:\Script-Projects\ArtCrafting
   $n = 0; foreach ($a in 'photocraft','lightcraft','filmcraft','wordcraft','effectcraft') { Push-Location ".\$a"; $n += (cargo run -q --release -p "$a-cli" -- commands --json | ConvertFrom-Json).Count; Pop-Location }; "five apps: $n"
   ```
5. Check whether Claude Code or Claude Desktop declares elicitation (decides whether channel 1 in §10.3 is usable today). Point either client at a stand-in stdio server that logs its input, e.g. FilmCraft's `mcp --demo` wrapped with `Tee-Object`, and look for `"elicitation"` in the `initialize` capabilities or in `_meta` `clientCapabilities`.
   ```powershell
   Set-Location F:\Script-Projects\ArtCrafting\filmcraft
   Get-Content $env:TEMP\mcp-in.log | Select-String 'elicitation'
   ```
   Expected: present or absent; record per client and version.
6. Check that the llama.cpp server is reachable for the §10.5.4 loop (expected: a JSON model list).
   ```powershell
   Set-Location F:\Script-Projects\ArtCrafting
   Invoke-RestMethod http://127.0.0.1:8080/v1/models
   ```
7. Check that `icacls.exe` and `reg.exe` run under any local policy doctor would meet (expected: both print output; if either is blocked, D02–D05 report `skip`).
   ```powershell
   Set-Location F:\Script-Projects\ArtCrafting
   icacls "$env:LOCALAPPDATA"; reg query HKLM\SYSTEM\CurrentControlSet\Control\FileSystem /v LongPathsEnabled
   ```
8. Confirm `claude_desktop_config.json` location on this machine (expected: the file exists, or the folder does).
   ```powershell
   Set-Location F:\Script-Projects\ArtCrafting
   Test-Path "$env:APPDATA\Claude\claude_desktop_config.json"
   ```
9. Documentation (Claude, next pass): read the 2026-07-28 MRTR page, Streamable HTTP page and `schema.ts` (all three fetches were blocked in this session: `modelcontextprotocol.io/specification/2026-07-28/basic/patterns/mrtr`, `…/basic/transports/streamable-http`, `raw.githubusercontent.com/modelcontextprotocol/modelcontextprotocol/main/schema/2026-07-28/schema.ts`). Fix the exact `inputRequests` nesting in §10.3 and the HTTP security wording in §10.5.2.
10. Documentation (Claude): confirm the current Claude Code `.mcp.json` shape, `claude mcp add` syntax and elicitation support from the Claude Code MCP docs (fetch blocked in this session).
11. Documentation (Claude): confirm the Python MCP SDK's client API for 2026-07-28 (stateless) before Option B in §10.5.4 is copied into Paul's stack.

## Open questions

1. **Confirmation for `write-app-state`.** The draft prompts only for destructive, `write-user-file`, `external-network` and unknown commands, so Open In is one step. Should MCP clients (or just the local llama.cpp client) also prompt on every `write-app-state` call? (Paul; A8 for the policy shape)
2. **Self-confirming clients.** Should any client ever be trusted to confirm on its own (for example Paul's Python harness, which asks him at the console as in §10.5.4)? The draft allows it only for direct hub-protocol clients that hold the token in program code, and never for MCP. (Paul, A8)
3. **Hub-folder writes.** `plan_to_filmcraft` writes an OTIO file into the hub's own `handoff\` folder. The draft treats hub-owned folders as not user files (no prompt). Agree? (Paul; A4 for whether this needs a class note)
4. **Workflow tool set.** Are `open_in`, `thumbnail_render` and `plan_to_filmcraft` the right three first-class tools, or should a fourth (for example `recipe_run` by name) be first-class instead of going through `hub_plan`/`hub_batch {recipe}`? (Paul)
5. **Streamable HTTP in v1.** Ship it at hub Stage 4 with the rules in §10.5.2, or keep v1 stdio-only and let the FastAPI stack use the hub protocol directly (Option A)? (Paul, A8)

## Inputs for downstream agents

- **A4 (§6, integrator):** three additive asks: `hub.call {readOnly}` (A7-F2), ranked `hub.commands {query, limit, cursor, capability}` (A7-F3), and `hub.approval.list|accept|deny` (A7-F4). MCP tool names do not embed addresses (A7-F1 resolves the `pdfcraft.tool.doc_open` suggestion). Answer to your open question 5: a dry run may pre-create the approval, but over MCP it is redeemed only through a human channel (§10.3 C4).
- **A6 (§8–9):** assumed recipe step shape `{target, params?, as?, from?}` (A1-O6 + 06 §6.2.4); the three workflow tools are named recipes `open-in`, `thumbnails`, `plan-to-filmcraft`; `hub_job_status` needs `waitMs` ≤ 30 s; `notifications/progress` is fed from your `job.progress` events; v1.1 resource subscriptions need `selection.changed` and `job.*` topics.
- **A8 (§11):** the threat rows in §10.6, the binding rules C1–C7, the deny list G5, budgets G7, the Streamable HTTP rules in §10.5.2, and the open problem of proving a connection is a human-facing `craftctl` (C3). The audit log needs fields `client` (claimed), `channel` (`elicitation | legacy-elicitation | out-of-band | craftctl-tty`) and `approval`.
- **A9 (§13, §16):** new crate row `craft-hub-doctor` L3 (A7-F5); contract tests: `tools/list` is byte-stable across calls, every tool's annotations match §10.1.2, `hub_read` rejects a write target, an approval can't be redeemed with altered params, the shim serves all three MCP revisions. Stage 4 "MCP" exit = AC4 from Claude Code and from the §10.5.4 Python loop. `craftctl doctor` belongs in the Windows verification checklist (§13.3).
- **A3:** `pipelinecraft mcp --client <name>` flag; `hub.json` gains optional `mcpHttp {port}` when HTTP is on.
- **A10 (§15):** candidate decisions: A7-R1 (tool surface), A7-R2 (confirmation channels), open questions 1, 2 and 5.

## Sources (web, checked 2026-10-15)

- [MCP 2026-07-28 changelog](https://modelcontextprotocol.io/specification/2026-07-28/changelog)
- [MCP 2026-07-28 elicitation (mirror)](https://modelcontextprotocol.net/specification/2026-07-28/client/elicitation.md)
- [MCP blog: Tool annotations (2026-03-16)](https://blog.modelcontextprotocol.io/posts/2026-03-16-tool-annotations/)
- [SEP-986: tool name format](https://modelcontextprotocol.io/seps/986-specify-format-for-tool-names)
- [MCP 2024-11-05 transports (stdio, local HTTP security)](https://modelcontextprotocol.io/specification/2024-11-05/basic/transports)
- [CVE-2026-42559, rmcp Streamable HTTP DNS rebinding](https://advisories.gitlab.com/cargo/rmcp/CVE-2026-42559/)
- Secondary, not relied on for field names: [channel.tel on MRTR](https://www.channel.tel/blog/mcp-round-trip-requests-replace-elicitation)
