# PipelineCraft (working name): §15 Open decisions

_2026-10-15 · spec draft v0 · status: draft (agent A10)_

- **Scope:** every decision Paul has to make that the eleven section drafts (`00` to `13-14-16`) raised, deduplicated into 31 decision pages (D1–D6 from the outline, D7–D31 new), each with options, a recommendation, what it blocks and the default if Paul doesn't decide. Also every place two sections disagree, with a proposed resolution for the integration pass.
- **Project goal it serves:** a suite of local-first creative tools that hand work to each other (AssetCraft → StoryCraft → FilmCraft, later PrepressCraft) without the cloud. Decisions here are what stands between the spec and the first small step of code (AssetCraft Stage 1.5 step 1, hub Stage 0 step 1).
- **Immediate area of improvement:** decisions are scattered across eleven "Open questions" lists, some duplicated with different wording, and a few sections contradict each other because they were drafted in parallel. This memo gives Paul one screen to decide from, five items to decide first, and gives the integrator one conflict list to clear.

**Evidence basis.** This section reads the spec drafts only; it re-derives nothing from source. Every decision cites the doc and ID it came from: `00` outline, `01` (A1), `03a` (A2a, O1–O14), `03b` (A2b, O-B1–O-B13), `04-05` (A3), `06` (A4), `07` (A5), `08-09` (A6), `10-12` (A7), `11` (A8), `13-14-16` (A9). Terms follow `01` §2: **connection** (never "session") for a hub client link, **command effect** in prose, JSON field `effects`, address `app[@instance]/[kind:]name`. "PipelineCraft" stays the working name until D1.

**Stage shorthand.** "Hub S*n*" = hub Stage *n* (`13-14-16` §16.1–16.6: 0 See, 1 Ask, 2 Hand off, 3 Chain, 4 MCP, 5 Produce). "AssetCraft 1.5" = AssetCraft Stage 1.5 as drafted in `13-14-16` §16.7, steps 1–8.

**ID note for the integrator.** The outline's D1–D6 are kept. `06` uses D-1…D-6 for its deprecation rules and `10-12` uses D01…D13 for `craftctl doctor` checks; neither are decisions. See conflict A10-X1.

---

## 15.1 Summary (one screen)

Ordered by urgency. **★ decide first** = the five that block hub Stage 0–1 or AssetCraft Stage 1.5 and are expensive to reverse once code or an upstream PR depends on them.

| ID  | Decision                                        | Recommendation                                                                                    | Blocks                                    | Urgency                   |
|-----|-------------------------------------------------|---------------------------------------------------------------------------------------------------|-------------------------------------------|---------------------------|
| D9  | Add AssetCraft Stage 1.5; when hub S0 starts    | Add 1.5 to the plan now; run hub S0–S1 (read-only) alongside it; hub S2+ waits on prerequisites   | AssetCraft 1.5 (all); hub S0              | ★ decide first (now)      |
| D2  | Separate repo vs crates in AssetCraft           | Incubate in AssetCraft; `craft-control-*` → craft-libs after one fork PR; hub repo at S2          | AssetCraft 1.5 step 3; hub S0 step 1      | ★ decide first (now)      |
| D8  | Command effects model                           | Four classes + `destructive`/`undoable`/`idempotent` flags + derived `confirm`                    | AssetCraft 1.5 step 1                     | ★ decide first (now)      |
| D3  | Hub protocol shape                              | Suite JSON lines, `error` as an object, `engine.*` compat methods; loopback TCP in v1             | AssetCraft 1.5 steps 3–4; hub S0 step 7   | ★ decide first (now)      |
| D10 | Suite run-dir name ("CraftApps")                | `%LOCALAPPDATA%\CraftApps\run\`, proposed upstream in the PR-1 issue                              | AssetCraft 1.5 step 4; D6 PR 2            | ★ decide first (now)      |
| D7  | Crate and binary naming                         | Crates `craft-control-*`, `craft-hub-*`, `craft-assetref`; binaries follow D1                     | Hub S0 step 1; AssetCraft 1.5 step 3      | now                       |
| D6  | First upstream PR                               | PR 1 DesignCraft `--json` + manifest issue; PR 2 manifest; PR 3 EffectCraft `journal`             | Nothing in the hub (NG8)                  | now (runs in parallel)    |
| D14 | Threads or tokio                                | Std threads, mpsc, socket deadlines; revisit at > 64 connections or SSE                           | Hub S0 (all crates)                       | before hub S0             |
| D11 | Hub token storage (A8 D-S1)                     | Inline in `hub.json` with an owner-only DACL; keyring for persistent secrets only                 | Hub S0 step 7                             | before hub S0             |
| D5  | Daemon on demand vs always on                   | On demand, started by the first client; browsing in AssetCraft never starts it                    | Hub S0 step 7; `craftctl`, MCP shim       | before hub S0             |
| D12 | Attach to hand-started unauthenticated apps     | Trust tiers, `attach.min_tier = "C"`; MCP one step tighter on tier C                              | Hub S0 step 6                             | before hub S0             |
| D23 | Contract snapshots in the repo                  | Full normalized catalogs (~2–3 MB) under `craft-hub-testkit/snapshots/`                           | Hub S0 step 3                             | before hub S0             |
| D26 | Commit prefix while incubating                  | `Hub Stage N step M:`                                                                             | First hub commit                          | before hub S0             |
| D25 | Performance targets: drafts or gates            | Drafts until Paul's Windows baseline; then baseline + margin; CI checks 10× bounds                | Hub S0 exit (P2, P3)                      | before hub S0 exit        |
| D4  | Canonical asset id                              | Composite AssetRef: path + size/mtime + BLAKE3 `q1`; `full`, XMP ids, `ac:<ULID>` optional        | AssetCraft 1.5 step 6; hub S2 step 1      | before AssetCraft 1.5 s6  |
| D20 | Id folding                                      | Auto-run a unique folded match for `read` only; writes get `unknown_command` + suggestion         | Hub S1 step 2                             | before hub S1             |
| D21 | PdfCraft writes through headless `pdfcraft-cli` | Yes for batch and recipes; the GUI only for Open In                                               | Hub S1 step 5; hub S2                     | before hub S1             |
| D1  | Final name                                      | RelayCraft (runner-up HubCraft); `craftctl` unchanged                                             | Hub S2 step 8 (repo name); state-dir name | before hub S2 (soft: S0)  |
| D13 | May the hub launch GUI apps                     | Yes, for Open In only (`launch: "if-needed"`); it never closes a GUI window                       | Hub S2 step 2; AC1(c)                     | before hub S2             |
| D27 | One-step jobs in hub S2                         | Yes: `hub.job.get/wait/cancel` in S2; events, pollers, chaining stay in S3                        | Hub S2 step 5; AC2(b)                     | before hub S2             |
| D28 | Protocol 1.0 method set                         | Put `hub.handoff`, `hub.recipe.*`, `hub.approval.*`, `readOnly`, ranked search in 1.0             | Hub S2 (`hub.handoff`), S3, S4            | before hub S2             |
| D15 | Which calls need confirmation                   | A4 derived `confirm` + A8 matrix; MCP `write-app-state` runs free on tier A/B, human on tier C    | Hub S2 (writes open); hub S4; AC4 text    | before hub S2             |
| D24 | Temp lifetimes and proxies root                 | Keep 24 h / 7 d / 512 MiB as drafts; recipes pass an explicit proxies root Paul picks             | Hub S2 step 6 (GC); hub S5 proxies        | before hub S2             |
| D29 | AC1 target apps                                 | Acceptance on PhotoCraft, LightCraft, VectorCraft; catalog adds EffectCraft, SoundCraft           | Hub S2 acceptance; AssetCraft 1.5 step 7  | before hub S2             |
| D31 | NAS and UNC paths                               | Empty UNC allowlist for scripts and MCP; mapped-drive → UNC resolution only if Paul needs it      | Hub S2 step 3 (path keys, roots)          | before hub S2             |
| D19 | Recipe format and dry-run kinds                 | JSON `*.recipe.json`; dry run `plan` (dispatches nothing) by default, `probe` opt-in              | Hub S3 step 5                             | before hub S3             |
| D16 | Human approval channel                          | MRTR elicitation → legacy elicitation → `craftctl approve` / panel; the model never holds a token | Hub S3 step 4; hub S4 step 3              | before hub S3             |
| D30 | Audit content and "watch all"                   | Keep script code, hash document text; `scope: "all"` only for interactive `craftctl` and panel    | Hub S3 steps 6–7                          | before hub S3             |
| D17 | MCP tool surface                                | 14 static tools, `[a-z0-9_]` names, three workflow tools; no `recipe_run` tool yet                | Hub S4 step 2                             | before hub S4             |
| D18 | MCP transport and the local llama.cpp stack     | stdio only in v1; the Python stack speaks the hub protocol directly; Streamable HTTP in v1.x      | Hub S4; AC4                               | before hub S4             |
| D22 | ArtCraft's role                                 | Not a hub app; Storyteller MCP sessions stay out of scope; files arrive via AssetCraft's watcher  | Hub S5 (generated-asset ingest)           | before hub S5 (cheap now) |

---

## 15.2 Decision pages

Each page: question; scope, goal and immediate improvement; options; **Recommendation**; what it blocks; the default if Paul doesn't decide; sources. Owner of every decision: **Paul** (decides). Owner of the follow-up: **Claude** unless the page says otherwise. Status of every page: **open**.

### D9 · Add AssetCraft Stage 1.5, and when hub Stage 0 starts · ★ decide first

- **Question:** does AssetCraft's plan get a Stage 1.5 (command effects, `--control` with a manifest, `serve`, `mcp`, `thumb.render`, `asset.ref`) before Stage 2, and do hub Stages 0–1 run alongside it or wait until after StoryCraft as the roadmap says?
- **Scope / goal / improvement:** AssetCraft plan and roadmap order · AssetCraft becomes the first app that declares command effects and publishes where it listens, the reference for the hub and for upstream PRs · today Stage 1.5 is in the brief but not in `docs/assetcraft-plan.md` or `STATUS.md` (`03b` O-B9).

| Option                                             | For                                                                                                       | Against                                                                                    |
|----------------------------------------------------|-----------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------|
| A. Add 1.5 now; hub S0–S1 alongside it (read-only) | The control code is the same code (A3-R1); one discovery for Open In and the hub (A3-O8); S0–S1 are small | Hub work starts before StoryCraft, against the roadmap order                               |
| B. Add 1.5 now; hub only after StoryCraft          | Roadmap order kept; AssetCraft still gains `--control`, MCP and effects                                   | `craft-control-*` is written without its second user, so its shape isn't tested by the hub |
| C. Fold 1.5 into AssetCraft Stage 6 (recipes)      | Nothing changes now                                                                                       | No reference app for effects or the manifest; PR 3 of D6 has no model to point to          |

- **Recommendation: A.** Approve the `13-14-16` §16.7 text as written (stage row, section, backlog row, open-decision row). Hub S0–S1 never write anything outside `%LOCALAPPDATA%\PipelineCraft\` and forward only reads (§16.0 stage gate), so they can't disturb AssetCraft's own stages. Hub S2+ wait for their AssetCraft prerequisite (§16.0 dependency table).
- **Blocks:** all of AssetCraft 1.5; hub S0 (depends on AssetCraft 1.5 steps 1–3).
- **Default if undecided:** Claude doesn't edit the AssetCraft plan; AssetCraft Stage 2 proceeds as planned; the hub stays a spec.
- **Sources:** `03b` O-B9, open question 1; `13-14-16` A9-F5, §16.0, §16.7, open question 2; `04-05` A3-R1.

### D2 · Separate repo vs crates inside AssetCraft · ★ decide first

- **Question:** where does the hub's code live first, and where does the shared control code (`craft-control-*`) end up?
- **Scope / goal / improvement:** repo layout and layer table · write the control client/server once for AssetCraft and the hub · today the suite's control transport is copied ×3 and diverging (`03b` O-B1).

| Option                                          | For                                                                                       | Against                                                                                          |
|-------------------------------------------------|-------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------|
| A. Separate hub repo now                        | Clean boundary; own release cadence                                                       | `craft-control-*` needed in two repos at once: copied (the O-B1 problem) or published unproven   |
| B. Crates in AssetCraft permanently             | One workspace, one `cargo xtask ci`                                                       | StoryCraft and others would depend on AssetCraft's repo; AssetCraft CI grows forever             |
| C. Incubate in AssetCraft, promote twice (rec.) | Starts small; one copy of the control code from day one; layers enforced by today's xtask | Needs the `ISLANDS` rule (`craft-` crates depend only on `craft-` crates) so promotion is a move |

- **Recommendation: C (A3-R1).** Promotion 1: `craft-control-*` → craft-libs when one sibling fork (DesignCraft or EffectCraft) builds against it in a PR. Promotion 2: `craft-hub-*`, `craft-assetref`, the daemon and `craftctl` → their own repo at hub S2 step 8, named after D1. Add `ISLANDS` (A3-O2) in hub S0 step 1. This also answers `03b` open question 2 (incubate here, offer to craft-libs later).
- **Blocks:** AssetCraft 1.5 step 3 (`crates/craft-control-proto`, `-server`); hub S0 step 1 (`TABLE` rows).
- **Default if undecided:** C.
- **Sources:** `00` D2; `04-05` §4.2, A3-R1, A3-O2; `03b` O-B1, open question 2; `13-14-16` §16.3 step 8, §16.7 open-decision row.

### D8 · Command effects model · ★ decide first

- **Question:** keep the outline's five effect classes, or four classes plus a `destructive` flag? Add an `external-process` class? Where does the effects table come from?
- **Scope / goal / improvement:** `CommandSpec`, `CommandInfo`, MCP hints, permissions · one honest "what does this touch" answer per command, which confirmations, dry runs and MCP annotations all read · today no registry declares effects; hints are name lists (`01` A1-O4, `03a` O4).

| Option                              | For                                                                                                            | Against                                                                                  |
|-------------------------------------|----------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------|
| A. Five classes (outline)           | Matches the outline text                                                                                       | A `destructive` class loses whether the write hit app state or a user file               |
| B. Four classes + flags (rec.)      | Maps 1:1 to PdfCraft's two bools and MCP's `readOnlyHint`/`destructiveHint`; classes stay a set, flags scalars | Spawning programs (`file.openIn`, `file.run`) fits no class; curated with a note for now |
| C. B plus `external-process` in 1.0 | Names what `file.openIn` really does                                                                           | One more class for every app profile to curate; nothing but AssetCraft needs it yet      |

- **Recommendation: B (A4-R2)**: classes `read`, `write-app-state`, `write-user-file`, `external-network`; flags `destructive`, `undoable`, `idempotent`, derived `confirm`, `source`. Unknown commands get the worst case (`06` §6.4.2). `external-process` is deferred to protocol 1.1; adding a class is additive under `06` rule D-2, and older clients must treat an unknown class as needing confirmation. Effects source: curate now (with `craftctl profile seed`, A4-F5) and replace rows as upstream PRs land (answers `03a` open question 1). Add the three `external-network` rows A8 found (A8-N6, conflict A10-X12).
- **Blocks:** AssetCraft 1.5 step 1 (every `CommandSpec` declares effects).
- **Default if undecided:** B, with `file.openIn`/`file.run` curated as `write-app-state` + `confirm: true`.
- **Sources:** `00` §6.4; `01` open question 3, A1-O2, A1-O4; `03a` O4, open question 1; `03b` O-B5; `06` A4-R2, A4-F4, A4-F5, open question 1; `11` A8-N6, A8-F1.

### D3 · Hub protocol shape · ★ decide first

- **Question:** does the hub's client socket mirror the suite's `{id, method, params}` → `{id, ok, result|error}` envelope, or adopt JSON-RPC 2.0? Which socket in v1? Are two error shapes on one socket acceptable?
- **Scope / goal / improvement:** the wire contract (`craft-control-proto`) · one way to call any command in any app, usable from a PowerShell `TcpClient` · today seven of eight control channels already share the envelope (`06` §6.1.1).

| Option                                                               | For                                                                                                   | Against                                                                   |
|----------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------|
| A. Suite JSON lines, `error` object, `engine.*` compat (rec., A4-R3) | Every existing script and `designcraft-cli app --port` works against the hub; `hub.batch` has dry run | Two error shapes on one socket (object on `hub.*`, string on `engine.*`)  |
| B. JSON-RPC 2.0                                                      | Standard; PdfCraft and every MCP server already speak it                                              | Only PdfCraft's control speaks it; array batches have no order or dry run |
| C. Suite JSON lines, string errors everywhere                        | One error shape                                                                                       | No stable code, `retryable` or `ran`, which recipes and retries need      |

- **Recommendation: A.** Two error shapes are acceptable because they're split by method namespace and the compat methods also carry `errorCode` (answers `06` open question 3). Socket: loopback TCP with an OS-chosen port in `hub.json` in v1; AF_UNIX (`uds_windows`) in v1.x behind the same codec (A4-R1, confirmed by A8 B1; answers `06` open question 4).
- **Blocks:** AssetCraft 1.5 steps 3–4 (envelope types, control server); hub S0 step 7 (listener).
- **Default if undecided:** A.
- **Sources:** `00` D3; `03a` O1; `06` §6.1, A4-R1, A4-R3, open questions 3–4; `11` §11.1, inputs for A4.

### D10 · Suite run-dir name ("CraftApps") · ★ decide first

- **Question:** what is the suite-wide folder where every app (not only the hub) writes its control manifest?
- **Scope / goal / improvement:** discovery (§5.2) and the first suite-standard PR · apps publish where they listen, so discovery stops depending on argv parsing · today only PdfCraft writes a control file, and nothing is suite-wide (A3-O1).

| Option                                    | For                                                                           | Against                                                                |
|-------------------------------------------|-------------------------------------------------------------------------------|------------------------------------------------------------------------|
| A. `%LOCALAPPDATA%\CraftApps\run\` (rec.) | Names the suite, not the hub, so D1 never moves it; reads as a product family | A new word upstream has to accept                                      |
| B. `%LOCALAPPDATA%\storytold\run\`        | The upstream org's own name; upstream may prefer it                           | Ties Paul's own apps (AssetCraft, PrepressCraft) to another org's name |
| C. `%LOCALAPPDATA%\Crafting\run\`         | Matches Paul's folder `F:\Script-Projects\ArtCrafting\`                       | Ambiguous with ArtCraft; personal naming in an upstream standard       |

- **Recommendation: A**, proposed in the issue that accompanies D6 PR 1, so upstream can counter-propose before any app writes there. Keep the `CRAFT_RUN_DIR` override (`04-05` §5.2). If upstream picks another name, the hub reads both for one version (version-ranged override, A3-O9) and AssetCraft moves.
- **Why first:** AssetCraft 1.5 step 4 writes its manifest there, and once one upstream app does too, a rename costs a migration in every app.
- **Blocks:** AssetCraft 1.5 step 4 (manifest path); D6 PR 2 (U3).
- **Default if undecided:** A, flagged "pending upstream" in the AssetCraft plan.
- **Sources:** `04-05` §5.2, A3-O1, open question 1; `13-14-16` U3, A9-R2, §16.7; `11` M2 (the run dir must pass the DACL check).

### D7 · Crate and binary naming

- **Question:** do hub crates carry a product prefix (`relaycraft-*`, `pipelinecraft-*`) or a rename-proof prefix (`craft-hub-*`)? What is A5's asset-ref crate called?
- **Scope / goal / improvement:** crate list and `TABLE` rows · a rename (D1) never touches code · today A1 and A3 disagree (`01` A1-N1 vs `04-05` open question 4) and A5's `assetref` falls outside the island rule (A9-F2).

| Option                                                       | For                                                                 | Against                                               |
|--------------------------------------------------------------|---------------------------------------------------------------------|-------------------------------------------------------|
| A. `craft-control-*`, `craft-hub-*`, `craft-assetref` (rec.) | Rename-proof; matches the `craft-` island rule; reads as suite code | Doesn't advertise the product name                    |
| B. `relaycraft-*`                                            | Product name visible                                                | Every crate renames if D1 changes; outside the island |

- **Recommendation: A.** Binaries follow D1 (`pipelinecraft` until then, `relaycraft` if adopted); `craftctl` never changes. A5's crate is `craft-assetref` (L0).
- **Blocks:** hub S0 step 1 (`TABLE` rows, `ISLANDS`); AssetCraft 1.5 step 3 (crate names).
- **Default if undecided:** A.
- **Sources:** `01` A1-N1; `04-05` §4.3, open question 4; `07` inputs for A3; `13-14-16` A9-F2.

### D6 · First upstream PR

- **Question:** which small PR tests upstream's appetite first, and when are the FilmCraft issues filed?
- **Scope / goal / improvement:** §14 upstream strategy · learn how storytold reviews outside PRs before asking for a standard · today the outline suggests the manifest PR first, which carries a naming decision (D10).

| Option                                                                      | For                                                            | Against                                                                   |
|-----------------------------------------------------------------------------|----------------------------------------------------------------|---------------------------------------------------------------------------|
| A. Manifest PR first (outline)                                              | The real standardization test                                  | Bundles a naming debate (D10) into the first contact                      |
| B. DesignCraft `commands --json` fix first, manifest issue alongside (rec.) | XS, no design debate; unblocks DesignCraft's contract snapshot | Tests review latency more than appetite for standards                     |
| C. A security fix first (SoundCraft line guard, S1/U5)                      | High value                                                     | Needs Paul's live check (§13.5 step 22) first; a "you have a hole" opener |

- **Recommendation: B (A9-R2).** PR 1 = U2 in DesignCraft with an issue proposing U3 and the D10 name; PR 2 = U3 + U1 (manifest + real port) in DesignCraft; PR 3 = U4 (`journal` in `command.list`) in EffectCraft. If PR 1 has no response in two weeks, ask in the issue before sending PR 2. FilmCraft issues (sidecar overwrite A5-O4, foreign OTIO metadata A5-O5): file after Paul reproduces them (§13.5 step 13, A5 Not verified step 8), so the issue carries evidence (answers `07` open question 3).
- **Blocks:** nothing in the hub (NG8, `13-14-16` §14.2); PR 2 needs D10.
- **Default if undecided:** no PRs sent; every quirk stays in adapter config.
- **Sources:** `00` D6; `13-14-16` §14.1, §14.3, A9-R2, open question 1; `07` open question 3; `11` S1–S12.

### D14 · Threads or tokio

- **Question:** is the daemon blocking std threads with deadlines, or async (tokio)?
- **Scope / goal / improvement:** every hub crate · same model as all nine control servers and AssetCraft's pools, easy to test with fakes · answers `04-05` open question 5.

| Option                | For                                                                                        | Against                                                                         |
|-----------------------|--------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------|
| A. Std threads (rec.) | Matches A9-N1 (no app uses tokio on control); ~100 threads at the very top end; no runtime | A Streamable HTTP endpoint needs a small hand-written HTTP/1.1 loop             |
| B. tokio              | rmcp is tokio; scales to many connections                                                  | A runtime in every crate's tests; the hub's MCP is hand-written JSON-RPC anyway |

- **Recommendation: A (A9-R1).** Revisit trigger: more than 64 concurrent client connections, or an SSE front door.
- **Blocks:** hub S0 (all crates).
- **Default if undecided:** A.
- **Sources:** `04-05` open question 5; `13-14-16` §13.2.2, A9-R1, A9-N1, A9-N2.

### D11 · Hub token storage (A8 D-S1)

- **Question:** does the hub's per-run token live inline in `hub.json` with an owner-only DACL, or in Windows Credential Manager?
- **Scope / goal / improvement:** `hub.json` and AssetCraft rule 7 ("secrets in the OS keyring, never in files") · one discovery file that PowerShell tests can read · today PhotoCraft, PdfCraft and WordCraft already keep their own secrets in files.

| Option                                     | For                                                                            | Against                                                                                      |
|--------------------------------------------|--------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------|
| A. Inline in `hub.json`, owner-only (rec.) | PdfCraft precedent; one file; no keyring service needed; dies with the process | Bends rule 7 for a per-run secret                                                            |
| B. Credential Manager entry                | Honours rule 7 literally; DPAPI-protected (general knowledge)                  | Keyring code in every client; stale entry after a crash; no gain against same-user processes |

- **Recommendation: A**, with rule 7 read as "persistent secrets live in the keyring; per-process loopback tokens live in owner-only files that die with the process". Claude records that reading in AssetCraft `AGENTS.md` once accepted (A8-O1).
- **Blocks:** hub S0 step 7 (`hub.json`).
- **Default if undecided:** A, and the `AGENTS.md` note is not written.
- **Sources:** `11` §11.2.2, D-S1, A8-O1, open question 1; `04-05` §5.2; `06` inputs for A3.

### D5 · Daemon on demand vs always on

- **Question:** is the hub a background service, or started by the first client that needs it? Does browsing a folder in AssetCraft start it?
- **Scope / goal / improvement:** hub lifetime · no background process Paul didn't ask for · the outline recommends on demand; `08-09` open question 3 asks about browsing.

| Option                                               | For                                                              | Against                                                                 |
|------------------------------------------------------|------------------------------------------------------------------|-------------------------------------------------------------------------|
| A. On demand, first client starts it (rec.)          | Nothing runs unless used; token rotates every start (A8 §11.2.2) | First call pays the start cost (P2 cold ≤ 1 s)                          |
| B. Always on (login task)                            | Warm every time                                                  | A background process and an open port all day; another thing to install |
| C. On demand, and AssetCraft starts it when browsing | Thumbnails for every format without asking                       | A background process appears just from looking at a folder              |

- **Recommendation: A.** `craftctl`, the MCP shim and explicit AssetCraft actions (Open In, "render previews with owning apps") start it; plain browsing uses the hub only if it's already running, else AssetCraft's type icon (AC2(c) fallback). Proposal from this section (not in any draft yet): the daemon exits after 10 minutes with no connections and no jobs; Paul confirms the number.
- **Blocks:** hub S0 step 7; `craftctl` and the MCP shim's start logic (`10-12` §10.5.1).
- **Default if undecided:** A, with no idle exit.
- **Sources:** `00` D5; `04-05` §4.1, A10 inputs; `08-09` open question 3; `11` §11.2.2.

### D12 · Attaching to hand-started unauthenticated apps

- **Question:** may the hub attach to a LightCraft, FilmCraft, DesignCraft, EffectCraft, VectorCraft or SoundCraft window Paul started himself with `--control`, before upstream token auth (S2/U6) exists?
- **Scope / goal / improvement:** lifecycle attach and §11 trust · Open In reaches the window Paul already has open (AC1(b)) without letting an impostor or an MCP client ride an open port · three drafts gave three answers (`03a` open question 3, `04-05` §5.3, `11` §11.3.5).

| Option                                                        | For                                                                           | Against                                                                         |
|---------------------------------------------------------------|-------------------------------------------------------------------------------|---------------------------------------------------------------------------------|
| A. Only instances the hub launched (`03a`)                    | Simplest proof of identity                                                    | AC1(b) fails for windows Paul opened himself                                    |
| B. Probed or launched (`04-05`)                               | AC1(b) works                                                                  | Identity probe can be replayed by an impostor on the port (A8-R3)               |
| C. Trust tiers A/B/C/C-read/D, `min_tier = "C"` (rec., A8-R4) | AC1(b) works; listener-owner check stops impostors; MCP one step tighter on C | Needs a port → pid crate (`netstat2` or `listeners`, A8-O3); without it, C-read |

- **Recommendation: C.** `hub.apps` reports `trust` per instance; every attach is audited with its tier.
- **Blocks:** hub S0 step 6 (attach); A8-O3 crate choice.
- **Default if undecided:** C.
- **Sources:** `03a` open question 3; `04-05` §5.3, open question 2; `11` §11.3.5, A8-R3, A8-R4, A8-F6, open question 2.

### D23 · Contract snapshots in the repo

- **Question:** store full normalized `commands --json` catalogs (~2–3 MB for ten apps) in the repo, or only ids and keys?
- **Scope / goal / improvement:** §13.3.3 contract tests · offline tests that catch upstream drift on every `cargo xtask ci` · nothing records catalogs today.

| Option                         | For                                                                        | Against                                                 |
|--------------------------------|----------------------------------------------------------------------------|---------------------------------------------------------|
| A. Full normalized JSON (rec.) | Tests can check params text (A4 S4) and error prefixes; diffs are readable | ~2–3 MB of text; refreshed by Paul on Windows           |
| B. Ids and keys only           | Small                                                                      | Can't check overlay `required` keys against params docs |

- **Recommendation: A**, under `crates/craft-hub-testkit/snapshots/` with its own `ATTRIBUTION.md` row (app output is MIT OR Apache-2.0, not CC0; A9-F9).
- **Blocks:** hub S0 step 3.
- **Default if undecided:** A.
- **Sources:** `13-14-16` §13.3.3, A9-F9, A9-O4, open question 3.

### D26 · Commit prefix while incubating

- **Question:** how are hub commits labelled while they share AssetCraft's repo?
- **Scope / goal / improvement:** commit history · "Stage 1 step 2" stays unambiguous between AssetCraft and the hub · A9-F4.
- **Options:** A. `Hub Stage N step M:` until promotion, then plain `Stage N step M:` in the hub repo (rec.). B. Plain `Stage N step M:` for both (ambiguous). C. Area prefixes (`hub: …`), FilmCraft style (loses the stage/step link STATUS.md uses).
- **Recommendation: A.** Fork PRs follow the upstream repo's own convention (`13-14-16` §14.4, A9-N8).
- **Blocks:** the first hub commit.
- **Default if undecided:** A.
- **Sources:** `13-14-16` §16.0, A9-F4, open question 5.

### D25 · Performance targets: drafts or gates

- **Question:** are A1's P1–P5 / R1–R6 numbers gates now, or drafts until a baseline on Paul's machine?
- **Scope / goal / improvement:** §13.4 · targets that mean something on an RTX 5090 Windows box with Defender · `01` open question 4, `13-14-16` open question 4.
- **Options:** A. Drafts until the Windows baseline, then baseline + margin per target; CI checks a loose ~10× bound meanwhile (rec.). B. Adopt A1's numbers as gates now (may fail for reasons unrelated to the hub, such as Defender scans).
- **Recommendation: A.** Each target names the stage whose exit makes it a gate (`13-14-16` §13.4 table). Paul also answers whether ≤ 1 s for Open In into a running app feels right (P5).
- **Blocks:** hub S0 exit (P2, P3 become gates there).
- **Default if undecided:** A.
- **Sources:** `01` §1.4, open question 4; `13-14-16` §13.4, open question 4.

### D4 · Canonical asset id

- **Question:** how do two apps agree they mean the same asset: path, content hash, XMP id or AssetCraft id? Which hash?
- **Scope / goal / improvement:** §7 AssetRef and AssetCraft's planned content hash · handoffs that survive renames and copies, with one hash the hub and AssetCraft share · today four apps fingerprint files four incompatible ways (`07` §7.0.2).

| Option                                                                                             | For                                                                                                    | Against                                                          |
|----------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------|------------------------------------------------------------------|
| A. Composite: path + size/mtime + BLAKE3 `q1`; optional `full`, XMP ids, `catalogId` (rec., A5-R1) | Degrades to a plain path for every existing command; `q1` ≤ 2 MiB read; catalog id wins once it exists | More fields; resolution rules R0–R6 to implement                 |
| B. Path + size/mtime only                                                                          | Free                                                                                                   | A move or rename loses the asset                                 |
| C. Full content hash as identity                                                                   | Strongest                                                                                              | Minutes on a 50 GB file; merges copies that are different assets |

- **Recommendation: A**, with BLAKE3 (`pure` feature, no C toolchain), `q1` = 1 MiB head + 1 MiB tail + size, `full` only on demand as a cancellable job. AssetCraft adopts BLAKE3 `full` as its Stage 2/3 content hash so its catalog doubles as the hub's index (A5-O1). Catalog ids are `ac:<ULID>` (sortable by creation time, no content dependency); no reason found to prefer UUIDs (answers `07` open question 4).
- **Blocks:** AssetCraft 1.5 step 6 (`asset.ref` returns `q1`); hub S2 step 1 (`craft-assetref`).
- **Default if undecided:** A.
- **Sources:** `00` D4; `07` §7.1, A5-R1, A5-O1, A5-O2, open questions 1 and 4; `13-14-16` A9-F7, P6.

### D20 · Id folding: auto-run or suggest

- **Question:** when a caller writes `session.saveAs` and the app's id is `session.save_as`, does the hub run the unique folded match or only suggest it?
- **Scope / goal / improvement:** §6.2.3 addressing · MCP clients that guess camelCase still work without letting a guess run a write · `06` open question 2.

| Option                                                           | For                                                            | Against                                                                        |
|------------------------------------------------------------------|----------------------------------------------------------------|--------------------------------------------------------------------------------|
| A. Auto-run with a `folded_id` warning (A4)                      | Fewest round trips                                             | A guessed id runs a write; confirmation still applies (N8) but only if flagged |
| B. Suggest only (`unknown_command` + suggestion)                 | Strict; the caller always names the real id                    | One more round trip for every guess                                            |
| C. Auto-run for `read` targets only; suggest for the rest (rec.) | Reads stay frictionless; writes always run under their real id | Two behaviours to document                                                     |

- **Recommendation: C**, a narrowing of A4's rule from this section. It keeps N1–N8 as written and adds one condition to N3 ("and the resolved command's effects are exactly `{read}`").
- **Blocks:** hub S1 step 2.
- **Default if undecided:** C.
- **Sources:** `06` §6.2.3, A4-F8, open question 2.

### D21 · PdfCraft writes through headless `pdfcraft-cli`

- **Question:** may the hub drive PdfCraft writes through a separate headless `pdfcraft-cli mcp` process instead of the open window?
- **Scope / goal / improvement:** PdfCraft adapter · PdfCraft's typed tools usable from recipes and PrepressCraft (§9.5.1) · PdfCraft control can't pass params to registry commands (`03a` O6).
- **Options:** A. Headless for batch and recipes; the GUI only for Open In (`ui.open`) (rec.). B. GUI only, so only `ui.command {id}` without params (most writes impossible). C. Wait for upstream U12 (`engine.execute` alias).
- **Recommendation: A.** The headless process is a separate PdfCraft session: a recipe never edits the document Paul has open, which is the safer side.
- **Blocks:** hub S1 step 5 (reads via typed tools); hub S2+ for writes.
- **Default if undecided:** A.
- **Sources:** `03a` O6, open question 2; `04-05` §4.4; `13-14-16` U12.

### D1 · Final name

- **Question:** what replaces the working name "PipelineCraft"?
- **Scope / goal / improvement:** binaries, the hub's state dir and the promoted repo · a short name that says "hands work between apps" · "PipelineCraft" reads as a render-farm pipeline and is long on a command line (`01` §1.5).

| Option                | For                                                            | Against                                     |
|-----------------------|----------------------------------------------------------------|---------------------------------------------|
| A. RelayCraft (rec.)  | 0 collisions in 13 repos; says "hand-off", not "control" (NG1) | Generic word (no trademark search was done) |
| B. HubCraft           | Literal; matches the glossary term "hub"                       | Plain; "hub" already has 10 unrelated hits  |
| C. Keep PipelineCraft | No change                                                      | The reasons in `01` §1.5                    |

- **Recommendation: A (A1-N1)**: binary `relaycraft`, state dir `%LOCALAPPDATA%\RelayCraft\`, repo `relaycraft` at promotion; `craftctl` unchanged; crate names unaffected (D7). Rejected in `01`: BridgeCraft and FlowCraft (clean-room echoes), MeshCraft (3D), CueCraft (marker clash), StageCraft (ILM).
- **Blocks:** hub S2 step 8 (repo name). Softer: hub S0 creates `%LOCALAPPDATA%\PipelineCraft\` and the `pipelinecraft` binary; deciding before S0 avoids a one-time state-dir move.
- **Default if undecided:** keep "PipelineCraft" everywhere; the rename stays one search-and-replace plus a state-dir move.
- **Sources:** `00` D1; `01` §1.5, A1-N1, open question 1.

### D13 · May the hub launch GUI apps

- **Question:** should the hub ever start a GUI app itself (AC1(c), Open In into a closed app), or only headless children, leaving GUI launches to AssetCraft's existing Open In?
- **Scope / goal / improvement:** lifecycle launch · Open In works the same whether the app is running or not · `04-05` open question 3.
- **Options:** A. The hub may launch a GUI with `--control` for Open In when the caller allows it (`hub.handoff {launch: "if-needed"}`), and never closes it (rec.). B. Headless only; GUI launches stay in AssetCraft's `file.openIn` (no `--control`, so later calls can't reach that window).
- **Recommendation: A.** A hub-launched window is tier B (D12) and belongs to Paul once open; `hub.app.stop` applies to headless instances only (conflict A10-X28).
- **Blocks:** hub S2 step 2; AC1(c).
- **Default if undecided:** A.
- **Sources:** `01` AC1; `04-05` §5.3, open question 3; `08-09` §9.0.1; `11` §11.3.5.

### D27 · One-step jobs in hub Stage 2

- **Question:** accept the deviation from the outline that moves a minimal job manager from Stage 3 to Stage 2?
- **Scope / goal / improvement:** stage plan · AC2(b) "cancelling the job stops the app-side work" can pass in the stage that delivers AC2 · A9-F6.
- **Options:** A. `hub.job.get/wait/cancel` in S2; `hub.job.list`, events, pollers and chaining stay in S3 (rec.). B. Keep jobs in S3 and move AC2's cancel clause to S3.
- **Recommendation: A.**
- **Blocks:** hub S2 step 5.
- **Default if undecided:** A.
- **Sources:** `13-14-16` §16.3, A9-F6, open question 6; `01` AC2.

### D28 · Protocol 1.0 method set

- **Question:** which methods proposed after A4's table go into protocol 1.0, and which wait for 1.1?
- **Scope / goal / improvement:** `06` §6.2.1 · one method table the integrator can freeze · A6, A7 and A8 each proposed additions (`08-09` open question 6).

| Proposed by | Additions                                                                                                                                  |
|-------------|--------------------------------------------------------------------------------------------------------------------------------------------|
| A6-R5       | `hub.handoff`, `hub.recipe.run`, `hub.recipe.undo`, `hub.recipe.validate`; error code `undo_conflict`                                      |
| A7-F2–F4    | `hub.call {readOnly}`; ranked `hub.commands {query, limit, cursor, capability}`; `hub.approval.list/accept/deny`                           |
| A8          | `hub.token.rotate`; codes `listener_mismatch`, `audit_unavailable`, `state_dir_insecure`, `policy_invalid`, `trust_insufficient`           |
| A5          | codes `asset_ref_invalid`, `path_*`, `case_ambiguous`, `asset_missing`, `asset_modified`, `asset_ambiguous`, `payload_version_unsupported` |

- **Options:** A. All in 1.0 (rec.). B. Only what hub S0–S2 needs in 1.0, the rest in 1.1.
- **Recommendation: A.** The protocol stays `"0.x"` (anything may break) until hub v1 at S5 (`06` §6.6.1), so "1.0" is just the frozen set at v1; there is no client in the field to stay compatible with. Each method still lands in the stage that needs it (`hub.handoff` S2, `hub.recipe.*` S3, `hub.approval.*` S3–S4).
- **Blocks:** hub S2 (`hub.handoff`), S3, S4.
- **Default if undecided:** A.
- **Sources:** `06` §6.2.1, §6.5.2, §6.6; `07` inputs for A4; `08-09` A6-R5, open question 6; `10-12` A7-F2, A7-F3, A7-F4; `11` A8-F3, §11.2.2.

### D15 · Which calls need confirmation

- **Question:** which command effects prompt, for which connection kind? In particular: does `write-app-state` from MCP (Open In, an import into a bin) ask? Do writes into the hub's own folders count as user-file writes? Does FilmCraft `file.import` count as a user-file write?
- **Scope / goal / improvement:** §6.4.1 derived `confirm`, §10.3, §11.4 · safe automation that isn't so chatty Paul starts approving blindly · today A7, A8 and the AC4 text give three different answers (conflict A10-X8).

| Option                                                     | For                                                                                   | Against                                                                                      |
|------------------------------------------------------------|---------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------|
| A. A4/A7: `write-app-state` never prompts (any client)     | AC1 is one call                                                                       | Ignores A8's tier C adjustment                                                               |
| B. A8 as drafted: MCP `write-app-state` → confirm (caller) | Stricter                                                                              | For MCP the "caller" is the model; a caller token returned to it confirms nothing (A7 §10.3) |
| C. A8's matrix, with MCP caller-kind replaced (rec.)       | Keeps A8's table for `craftctl` and apps; MCP gets allow on tier A/B, human on tier C | One more row to explain                                                                      |
| D. Every MCP non-read call needs a human                   | Simplest to reason about                                                              | AC4 becomes a dialog per Open In                                                             |

- **Recommendation: C.** For MCP connections there is no caller kind: `write-app-state` on tier A/B instances is allowed (the client's own tool approval still applies, advisory), on tier C it needs a human; `write-user-file`, `destructive`, `external-network` and `source: default` need a human. Paul can tighten any client in `policy.toml` (e.g. "human for every write from the llama.cpp client"). Hub-folder writes are decided as `write-app-state` (A8 P4, answers `10-12` open question 3). FilmCraft `file.import` is curated `write-app-state` + `write-user-file` until the ingest-preference check exists (A6-O9), so AC3 confirms once at commit (answers `08-09` open question 4). AC4's "every write tool asks" becomes "every write tool that §10.3 flags asks".
- **Blocks:** hub S2 (writes open for `craftctl` and apps); hub S4 (MCP); the AC4 text in `01` §1.4 and `08-09` §9.6.
- **Default if undecided:** D (strictest workable) until Paul decides.
- **Sources:** `06` §6.4.1; `08-09` §9.6, A6-O9, open questions 2 and 4; `10-12` §10.2–10.3, open questions 1 and 3; `11` §11.4.2, P2, P4, P8, open question 4; `13-14-16` §16.5 acceptance.

### D24 · Temp lifetimes and the proxies root

- **Question:** are 24 h for handoff folders, 7 days for unclaimed renders and 512 MiB for hub thumbnails right? Where do FilmCraft proxies go when the hub drives FilmCraft?
- **Scope / goal / improvement:** §7.5 GC rules · no disk creep in `%LOCALAPPDATA%`, and no proxies scattered next to media without Paul choosing it · `07` open question 2, A5-O9.
- **Options (lifetimes):** A. Keep A5's numbers as drafts, revisit after hub S2 use (rec.). B. Paul sets numbers now.
- **Options (proxies):** A. Hub recipes pass an explicit `destination` under one proxies root Paul picks (rec., A5-O9). B. FilmCraft's default `<media folder>\Proxies`, declared `write-user-file`.
- **Recommendation: A and A.** Paul names the proxies root when hub S5 starts (it's only needed by AC3-era recipes).
- **Blocks:** hub S2 step 6 (GC); hub S5 (proxy-producing recipes).
- **Default if undecided:** A5's numbers; proxies use FilmCraft's default and always confirm as `write-user-file`.
- **Sources:** `07` §7.5.3, A5-O9, open question 2.

### D29 · AC1 target apps

- **Question:** which apps must pass AC1 Open In for v1, and which get added to AssetCraft's Open In catalog?
- **Scope / goal / improvement:** acceptance scope · a v1 that is honest about what was tested · `01` open question 5; AssetCraft's catalog lacks EffectCraft, VectorCraft and SoundCraft (`03b` O-B10).
- **Options:** A. AC1 acceptance on PhotoCraft, LightCraft and VectorCraft as AC1 states; the catalog gains EffectCraft, VectorCraft and SoundCraft (AssetCraft 1.5 step 7); the other apps best-effort (rec.). B. AC1 acceptance on all nine apps.
- **Recommendation: A.** The three AC1 apps cover the three open verbs (`method:app.open`, `library.import`, `document.open`); the rest share code paths and get contract tests, not acceptance runs.
- **Blocks:** hub S2 acceptance; AssetCraft 1.5 step 7 (optional).
- **Default if undecided:** A.
- **Sources:** `01` AC1, open question 5; `03b` O-B10; `13-14-16` §16.7 step 7.

### D31 · NAS and UNC paths

- **Question:** which NAS hosts may scripts and MCP clients write to by default? Should mapped drives be resolved to UNC in the path key?
- **Scope / goal / improvement:** §7.4 path keys and §11.4 roots · the same file never looks like two assets, and a prompt-injected client can't write to a share · `07` open question 5, `11` open question 6 (T12).
- **Options:** A. Empty UNC allowlist for non-interactive profiles; no mapped-drive resolution in v1, hash resolution (R4) reconciles two spellings (rec.). B. Resolve mapped drives to UNC in the path key now (a Win32 call through a small crate).
- **Recommendation: A**, unless Paul's media is reached both by drive letter and by UNC (Not verified step 2); then B for the hub's path key.
- **Blocks:** hub S2 step 3 (path transforms, roots).
- **Default if undecided:** A.
- **Sources:** `07` §7.4, A5-O10, open question 5; `11` §11.4.2 roots, T12, open question 6.

### D19 · Recipe format and dry-run kinds

- **Question:** JSON or TOML recipes? May a dry run run reads and app preview commands (`probe`), or must it dispatch nothing?
- **Scope / goal / improvement:** §8.5 recipes · one cross-app chain format that pastes app params as they are · `08-09` open question 1.

| Option                                                                 | For                                                                                       | Against                                       |
|------------------------------------------------------------------------|-------------------------------------------------------------------------------------------|-----------------------------------------------|
| A. JSON `*.recipe.json`; dry run `plan` default, `probe` opt-in (rec.) | Params paste unchanged; matches PhotoCraft actions, FilmCraft `run`, DesignCraft `script` | No comments (a `note` field per step instead) |
| B. TOML recipes                                                        | Comments; hand-editable                                                                   | No `null`; nested params read poorly          |
| C. JSON; dry run always `plan` only                                    | Simplest guarantee                                                                        | Can't catch "no sequence open" before commit  |

- **Recommendation: A (A6-R4).** `plan` dispatches nothing (A4's batch rule); `probe` runs only `read` commands, curated shadow previews and writes into hub folders (G9). `hub.call`/`hub.batch` keep `dryRun: bool` (conflict A10-X15).
- **Blocks:** hub S3 step 5.
- **Default if undecided:** A.
- **Sources:** `01` A1-O6, NG7; `08-09` §8.5, §8.6, A6-R4, open question 1; `06` §6.2.4.

### D16 · Human approval channel

- **Question:** how does a human approve a call an MCP client (or a script) asked for? Can any client confirm on its own? Is a console `craftctl` enough until the status panel exists?
- **Scope / goal / improvement:** §10.3 and §11.4 P8 · a confirmation a prompt-injected model can't answer for itself · today A4's token is returned to the caller, which over MCP is the model (A8-F4).
- **Options:** A. A7-R2 channels in order: MRTR elicitation (2026-07-28) → legacy elicitation → out-of-band `craftctl approve` / status panel; the model never holds a redeemable token (rec.). B. Return A4's token to MCP clients and rely on the client's own tool approval (advisory by spec). C. A plus a Windows notification in v1.
- **Recommendation: A.** Self-confirming clients: only direct hub-protocol clients that hold the hub token in program code (Paul's Python harness, AssetCraft dialogs) use caller tokens; never MCP (answers `10-12` open question 2). v1 approval surface is the console `craftctl approve`; a Windows toast is a v1.x opportunity (answers `11` open question 3). Verb and timeouts are unified in conflicts A10-X6 and A10-X7.
- **Blocks:** hub S3 step 4 (confirmation tokens); hub S4 step 3 (MCP confirmation).
- **Default if undecided:** A.
- **Sources:** `06` Example 3, open question 5; `10-12` §10.3, A7-R2, A7-F4, A7-F9, open question 2; `11` P8, P9, A8-F4, open question 3.

### D30 · Audit content and "watch all"

- **Question:** does the audit log keep script code and hash document text? Who may watch jobs started by other connections (`filter.scope: "all"`)?
- **Scope / goal / improvement:** §11.5 audit and §8.1 subscriptions · enough to reconstruct what a client did, without copying Paul's documents into a log · `11` open question 5, `08-09` open question 5.
- **Options:** A. Keep script code, hash document text, redact per A8's table; `scope: "all"` only for interactive `craftctl` and the panel (rec.). B. Hash everything (less forensic value). C. `scope: "all"` for any connection (an MCP client could watch AssetCraft's jobs).
- **Recommendation: A.** The `scope: "all"` rule is this section's proposal for `11` §11.4.2 (a row in the defaults table).
- **Blocks:** hub S3 step 6 (audit); hub S3 step 7 (`craftctl watch --scope all`).
- **Default if undecided:** A.
- **Sources:** `08-09` B8, open question 5; `11` §11.5, open question 5.

### D17 · MCP tool surface

- **Question:** one tool per command, per-app suite tools, or a small fixed set of hub tools? Which workflow tools are first-class?
- **Scope / goal / improvement:** §10.1 · Claude and a 32k-context local model can both use the whole suite · ~5,800 commands can't be tools (`10-12` §10.1.1).
- **Options:** A. 14 static tools (11 generic + `open_in`, `thumbnail_render`, `plan_to_filmcraft`), `[a-z0-9_]` names, catalogs as `craft://` resources (rec., A7-R1). B. Per-app suite tools (`<app>_command_run` × 10 apps). C. A plus a `recipe_run` tool.
- **Recommendation: A.** Recipes run through `hub_plan`/`hub_batch {recipe}`; add `recipe_run` only if AC4 runs show models struggle (answers `10-12` open question 4).
- **Blocks:** hub S4 step 2.
- **Default if undecided:** A.
- **Sources:** `03b` O-B4; `10-12` §10.1, A7-R1, A7-F1, open question 4.

### D18 · MCP transport and the local llama.cpp stack

- **Question:** does v1 ship Streamable HTTP on loopback, or stdio only? How does Paul's FastAPI/Instructor stack reach the suite?
- **Scope / goal / improvement:** §10.5 and AC4 · both MCP actors (Claude, local stack) work in v1 · `01` open question 2, `10-12` open question 5.
- **Options:** A. stdio only in v1; the Python stack speaks the hub protocol directly (`10-12` §10.5.4) or spawns `pipelinecraft mcp` (rec.). B. Streamable HTTP at hub S4 with A7's Origin/Host/bearer rules. C. Both at S4.
- **Recommendation: A**, with B moved to v1.x. It keeps the threat surface to one socket in v1 and needs no HTTP server (D14). If Paul's stack can only speak MCP over HTTP, B moves into S4.
- **Blocks:** hub S4; AC4 for the local stack.
- **Default if undecided:** A.
- **Sources:** `01` open question 2; `10-12` §10.5, A7-R3, open question 5; `11` §11.1.

### D22 · ArtCraft's role

- **Question:** confirm ArtCraft is not a hub-routed app in v1, and that the Storyteller server-side MCP sessions stay out of scope?
- **Scope / goal / improvement:** suite boundary · generated assets still enter the pipeline, through AssetCraft · ArtCraft has no local control, CLI or MCP (`03b` O-B11).
- **Options:** A. Not a hub app; AssetCraft's ArtCraft provider downloads into a watched folder and AssetCraft's watcher ingests (rec.). B. A hub adapter over the Storyteller web API (against NG2, the two-APIs rule).
- **Recommendation: A.**
- **Blocks:** hub S5 (generated-asset ingest); nothing earlier.
- **Default if undecided:** A.
- **Sources:** `00` scope note; `01` NG2; `03b` O-B11, O-B12, open question 3.

---

## 15.3 Cross-section conflicts for the integrator

Two drafts disagree in each row. IDs are `A10-X#`. "Owner" is who applies the fix; Paul only where a decision page owns it. Several drafts (A9 in particular) were written before A6–A8 were on disk, which explains most rows.

| ID      | Where                                                                                                                                                      | Disagreement                                                                                                                                   | Proposed resolution                                                                                                                                                                                                                                                  | Owner                         | Status |
|---------|------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------|--------|
| A10-X1  | `06` §6.6.2; `10-12` §12.5; this §15                                                                                                                       | `06` deprecation rules are D-1…D-6 and `10-12` doctor checks are D01…D13; both collide with decision IDs D1–D31                                | Rename `06` rules to DEP-1…DEP-6 and `10-12` checks to DR01…DR13                                                                                                                                                                                                     | Claude (integrator)           | open   |
| A10-X2  | `01` A1-N1 vs `04-05` §4.3, open question 4                                                                                                                | Crate prefix `relaycraft-*` vs `craft-hub-*`                                                                                                   | `craft-control-*` / `craft-hub-*` (D7); binaries follow D1; edit `01` A1-N1 text                                                                                                                                                                                     | Claude; Paul (D7)             | open   |
| A10-X3  | `07` inputs for A3 vs `13-14-16` A9-F2                                                                                                                     | Crate `assetref` vs `craft-assetref`                                                                                                           | `craft-assetref` (L0), inside the `craft-` island (D7)                                                                                                                                                                                                               | Claude (integrator)           | open   |
| A10-X4  | `06` inputs for A7 (`pdfcraft.tool.doc_open`), `03b` O-B4 (`<app>.command_run`), `08-09` §9.6 (`assetcraft.cmd.selection_handoff`) vs `10-12` A7-F1        | Dotted, address-derived MCP tool names vs `[a-z0-9_]` only                                                                                     | `[a-z0-9_]` names (A7-F1); addresses travel in the `target` argument; rewrite the `08-09` §9.6 case table with A7's 14 tools                                                                                                                                         | Claude (integrator)           | open   |
| A10-X5  | `06` Example 3 and `08-09` §9.6 sequence vs `10-12` A7-R2 and `11` P8                                                                                      | `06`/`08-09` return the confirm token to the caller, including an MCP model, which then retries with it; A7 and A8 forbid that                 | Over MCP the model never holds a redeemable token (D16). Redraw the `08-09` §9.6 diagram with an approval id; `06` Example 3 stays as the `craftctl`/app (caller-kind) example                                                                                       | Claude (integrator)           | open   |
| A10-X6  | `10-12` §10.3, §12.3 vs `11` P8, §11.4                                                                                                                     | Approval verb `craftctl confirm <ap-id>` + `hub.approval.list/accept/deny` vs `craftctl approve <id>` + retry with `confirm: "<approval id>"`  | Verb `craftctl approve [list \| <id> \| --deny <id>]` (it approves an approval id; `confirm` is already the token field). Methods `hub.approval.*` (A7-F4). The model retries the **same** call with no id: the approval is keyed server-side by params hash (A7 C1) | Claude (integrator)           | open   |
| A10-X7  | `06` A4 confirm token (2 min), `10-12` C2 (2 min) vs `11` P9 (10 min) and `08-09` §8.6.3 (plan hash)                                                       | Token lifetimes and bindings differ                                                                                                            | Three kinds: call token 2 min bound to connection + target + params hash (A4); plan token 10 min bound to A6's plan hash incl. `registryHash` (P9 + §8.6.3); human approval 2 min (C2)                                                                               | Claude (integrator)           | open   |
| A10-X8  | `10-12` §10.2–10.3 vs `11` §11.4.2 vs `01` AC4 / `08-09` §9.6 / `13-14-16` §16.5                                                                           | MCP `write-app-state`: no prompt (A7) vs confirm (caller) (A8) vs "every write tool asks" (AC4, A9)                                            | Decided by D15 (rec. C). Edit AC4 and §16.5 acceptance text to "every write tool that §10.3 flags asks"; A8's `mcp` column loses the caller kind                                                                                                                     | Paul (D15); Claude            | open   |
| A10-X9  | `04-05` §4.1 poller table vs `08-09` §8.2, A6-R7                                                                                                           | A3's `open.done` event for PdfCraft pending opens vs A6's job events                                                                           | Drop `open.done`; a PdfCraft `ui.open` is part of the Open In job and ends in `job.progress`/`job.done`                                                                                                                                                              | Claude (integrator)           | open   |
| A10-X10 | `04-05` §4.1 (fixed 250 ms) vs `08-09` A6-R2 (adaptive 250 ms → 1 s → 2 s) and `13-14-16` P4                                                               | Poll interval                                                                                                                                  | Adopt A6's P-1–P-8; P4's polled bound is measured in the first second (250 ms tier) and stated as such in §13.4                                                                                                                                                      | Claude (integrator)           | open   |
| A10-X11 | `07` Not verified step 8 vs `08-09` A6-F8, A6-O10                                                                                                          | A5 expects FilmCraft's own OTIO export to write absolute `file://` URLs; FilmCraft's engine sets `relative_to` to the `.otio` folder           | Correct step 8's expected result (relative URLs, media online only beside the export). AC3 unaffected: AssetCraft's writer emits absolute URLs (`07` §7.2.5 rule 1)                                                                                                  | Claude (integrator)           | open   |
| A10-X12 | `06` §6.4.1 ("none found"), `13-14-16` §16.0 ("no local app needs it") vs `11` A8-N6, A8-F1                                                                | A8 found three `external-network` commands: EffectCraft `roto.model.download`, `face.model.download`; LightCraft `metadata.downloadPlaceNames` | Add curated rows to `profiles/effectcraft/effects.toml` and `profiles/lightcraft/effects.toml`; fix `06` text. Keep A9's stage gate closing `external-network` in v1 (none is needed for AC1–AC4); A8's grant rows apply from v1.x                                   | Claude (profiles, integrator) | open   |
| A10-X13 | `13-14-16` A9-F1                                                                                                                                           | `03a` Not verified #5 (WordCraft) and `03b` #5 (DesignCraft) both use port 7981                                                                | Point both at the `13-14-16` §13.5 port plan                                                                                                                                                                                                                         | Claude (integrator)           | open   |
| A10-X14 | `13-14-16` A9-F3, A9-N7 vs `01` Not verified #3 and `03b` O-B7                                                                                             | "Second app fails to bind 7979" can't happen: 7979 is a **client** default (VectorCraft MCP, `soundcraft-cli app`), not a server bind          | Replace `01` step 3's expectation with `13-14-16` §13.5 step 20 ("a client default reaches the wrong app"); reword O-B7                                                                                                                                              | Claude (integrator)           | open   |
| A10-X15 | `06` §6.2.1 (`dryRun?: bool`) vs `08-09` §8.5–8.6, §9.6 (`dryRun: "probe"`) and `10-12` (`--dry-run plan\|probe`)                                          | Type of `dryRun`                                                                                                                               | `hub.call`/`hub.batch` keep `dryRun: bool` (dispatch nothing). `hub.recipe.run` takes `dryRun: true \| "plan" \| "probe"`, where `true` = `"plan"` (D19)                                                                                                             | Claude (integrator)           | open   |
| A10-X16 | `01` A1-O6 (`{"app","command","params"}`) vs `06` §6.2.4, `08-09` §8.5.2, `10-12` (`{target, params, as, from}`)                                           | Recipe step shape                                                                                                                              | `{target, params?, as?, from?}`; edit A1-O6; `craftctl recipe import --from photocraft-action` fills `target`                                                                                                                                                        | Claude (integrator)           | open   |
| A10-X17 | `06` §6.2.5 capability row, `10-12` §10.1.2 `thumbnail_render` vs `08-09` inputs for A4                                                                    | `effectcraft/method:render.frame {path, max_side}` as a thumbnailer: in that method `path` is the **output** and it renders the open comp      | `asset.thumbnail` for EffectCraft = headless `effectcraft-cli render-frame` (`08-09` §9.2); fix the A4 row and A7's table                                                                                                                                            | Claude (integrator)           | open   |
| A10-X18 | `06` §6.2.1 vs `08-09` A6-R5 vs `10-12` tool 8                                                                                                             | `hub.handoff` isn't in A4's table; A7's `hub_handoff` accepts three payload types where A6 accepts five                                        | Add `hub.handoff` to §6.2.1 (D28); A7's tool accepts all five `handoff.*` types                                                                                                                                                                                      | Claude (integrator)           | open   |
| A10-X19 | `04-05` §5.3 (`attach_unauthenticated = "probed-or-launched"`), `03a` open question 3, `13-14-16` U6 fallback vs `11` A8-R4, A8-F6                         | Three attach policies                                                                                                                          | `[attach] min_tier = "C"` and `trust` in `hub.apps` (D12); update A3 §5.3 and A9's U6 fallback text                                                                                                                                                                  | Claude; Paul (D12)            | open   |
| A10-X20 | `13-14-16` §14.1 (U5, U6, U28 placeholders) vs `11` S1–S12                                                                                                 | Two upstream lists; A9 couldn't see A8's                                                                                                       | Map: S1→U5, S2→U6, S3→U28, S6→U1, S7→U10, S8→U9, S10→U7+U8, S12→U4/U15/U17. Add new U rows for S4 (PhotoCraft token to file), S5 (bridge `localhost` → `127.0.0.1`), S9 (EffectCraft `scripting.*` prefs), S11 (WordCraft key to `%LOCALAPPDATA%`)                   | Claude (integrator)           | open   |
| A10-X21 | `13-14-16` §16.1 vs `11` inputs for A9                                                                                                                     | A8 puts DACL checks, listener-owner check and token work in Stage 0; A9's Stage 0 steps don't list them                                        | Add to hub S0 step 6 (listener-owner check, tier assignment) and step 7 (DACL check DG1–DG5 before `hub.json`, token generation)                                                                                                                                     | Claude (integrator)           | open   |
| A10-X22 | `04-05` §5.2 manifest (`control.protocol` integer) vs `06` §6.6.1 (`"MAJOR.MINOR"`) and `06` inputs for A3 (flat `hub.json {port, token, pid, protocol}`)  | Protocol field type and `hub.json` shape                                                                                                       | `hub.json` uses A3's manifest schema (`"app":"pipelinecraft"`, nested `control`, `auth`) with the integer major; `hub.hello` negotiates `"MAJOR.MINOR"`; edit `06`'s inputs text                                                                                     | Claude (integrator)           | open   |
| A10-X23 | `03a` O2 (`token-frame`, `token-file-jsonrpc`, `key-file`, `none`) vs `04-05` §5.2 (`none`, `token-frame`, `key-field`)                                    | Auth kind names                                                                                                                                | Use A3's enum; PdfCraft is `token-frame` with codec `jsonrpc2`; retire O2's names in the merged §3                                                                                                                                                                   | Claude (integrator)           | open   |
| A10-X24 | `04-05` §4.1 (at most 2 hub connections per instance) vs `06` inputs for A3 (pool of 2–4)                                                                  | Connection pool size                                                                                                                           | Default 2 per instance; a profile may raise it to 4                                                                                                                                                                                                                  | Claude (integrator)           | open   |
| A10-X25 | `03b` O-B5 (`writes-app-state`, `writes-user-file`, `network`) vs `06` §6.4.1                                                                              | Effect class spellings                                                                                                                         | A4's spellings (`write-app-state`, `write-user-file`, `external-network`); edit O-B5                                                                                                                                                                                 | Claude (integrator)           | open   |
| A10-X26 | `10-12` `open_in` (reads selection via `assetcraft/app.state`) vs `10-12` A7-F6 and `08-09` S15-1                                                          | Selection source                                                                                                                               | `selection.get` → `handoff.selection` in AssetCraft 1.5 (add to §16.7 Commands); `open_in` uses it                                                                                                                                                                   | Claude (integrator)           | open   |
| A10-X27 | `13-14-16` §16.0 (hub-dir writes allowed as limited `write-user-file`) vs `11` P4 (decided as `write-app-state`) vs `10-12` (hub folders "not user files") | Three phrasings of one rule                                                                                                                    | One rule, `11` P4; §16.0's stage-gate row and A7's `plan_to_filmcraft` row cite it                                                                                                                                                                                   | Claude (integrator)           | open   |
| A10-X28 | `04-05` §5.3 (the hub never closes a GUI window) vs `06` §6.2.1 (`hub.app.stop {instance}`)                                                                | `hub.app.stop` on any instance                                                                                                                 | `hub.app.stop` applies to headless instances only; a GUI quit goes through the app's own `app.quit` with human confirmation (`11` target rules)                                                                                                                      | Claude (integrator)           | open   |
| A10-X29 | `01` A1-N1 (binary `relaycraft`) vs `04-05`, `06`, `10-12` (binary `pipelinecraft`)                                                                        | Binary name before D1                                                                                                                          | `pipelinecraft` until D1; `01` states the RelayCraft names as conditional on D1                                                                                                                                                                                      | Claude (integrator)           | open   |

---

## Not verified

This section is a reading of the drafts; nothing was built or run, and no source was re-read. A few recommendations rest on facts about Paul's machine that only a live run settles.

**Next action (Paul):**

1. Check the default ACL on `%LOCALAPPDATA%` (feeds D10, D11: the run dir and `hub.json` must be user-only).
   ```powershell
   Set-Location F:\Script-Projects\ArtCrafting
   icacls $env:LOCALAPPDATA
   ```
   Expected: entries for your user, `NT AUTHORITY\SYSTEM` and `BUILTIN\Administrators` only, each `(OI)(CI)(F)`. Anything else (for example `Everyone` or `Users`) means the hub must repair the folder DACL (`11` DG1–DG5) before D11 option A is safe.
2. Check whether any media is reachable both by drive letter and by UNC (feeds D31).
   ```powershell
   Set-Location F:\Script-Projects\ArtCrafting
   Get-PSDrive -PSProvider FileSystem | Where-Object DisplayRoot | Select-Object Name, DisplayRoot
   net use
   ```
   Expected: an empty list if all media is local. Any row means D31 option B is worth it; send the hosts for the UNC allowlist.
3. Confirm `gh` is ready before D6 PR 1.
   ```powershell
   Set-Location F:\Script-Projects\ArtCrafting
   gh auth status
   ```
   Expected: "Logged in to github.com account 4waymedia".
4. Run the consolidated `13-14-16` §13.5 script before D25 turns any target into a gate. Expected: the per-step results listed there.

## Open questions

1. **One sitting for the five.** Can Paul decide D9, D2, D8, D3 and D10 together? They are coupled: D9 starts the work, D2 and D7 say where the code goes, D8 and D3 shape AssetCraft 1.5 steps 1–4, and D10 fixes the manifest path in step 4. (Paul)
2. **Defaults in the merge.** Should the integrator write each undecided item into the merged spec using its "Default if undecided", marked "default, not decided", or leave it as an open question? Recommendation: write the default, marked. (Paul)
3. **D1 timing.** Decide the name before hub S0 to skip a state-dir move later, or keep "PipelineCraft" through S1? (Paul)
4. **Local stack facts for D18.** Does Paul's FastAPI/Instructor stack already include an MCP client, and can it spawn a stdio child process? (Paul)

## Inputs for downstream agents

- **Integrator:** apply A10-X1 to A10-X29 (§15.3). After Paul decides, replace each §15.2 page's "Default if undecided" with the decision, its date and "decided by Paul", and update the source sections the page names. The outline's §15 table becomes §15.1 of this file.
- **Integrator, renumbering:** the merged spec needs one ID series for upstream asks: keep A9's `U#` and fold A8's `S#` per A10-X20; fold `03a` O1–O14 and `03b` O-B1–O-B13 as A9 suggests.
- **Peer reviewers:** the protocol/security lens should check D15 option C (MCP has no caller-kind confirmation) against `11` §11.6 T-rows; the upstream-fit lens should check D10's name before the D6 issue is written.
- **Claude, after decisions:** D9 → apply `13-14-16` §16.7 to `docs/assetcraft-plan.md` and hand Paul the commit block; D11 (if A) → add the rule 7 reading to AssetCraft `AGENTS.md`; D7 → add `TABLE` rows and `ISLANDS` at hub S0 step 1; D8 → AssetCraft 1.5 step 1 effects on every `CommandSpec`.
- **Every section:** keep writing "PipelineCraft (working name)" until D1 is decided.
