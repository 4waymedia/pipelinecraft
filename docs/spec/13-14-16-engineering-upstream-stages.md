# PipelineCraft (working name): §13 Engineering rules and quality gate · §14 Upstream and fork strategy · §16 Stages

_2026-10-15 · spec draft v0 · status: draft (agent A9)_

- **Scope:** how AssetCraft's code rules and `cargo xtask ci` gate apply to the hub crates, the test strategy (fake app, contract snapshots, golden transcripts, fuzzing, fixtures), how each P/R target is measured, one consolidated Windows verification script for every "Not verified" step in the spec so far (§13); the ranked list of upstream PR candidates with hub-side fallbacks, the first three PRs, and the fork workflow (§14); and the hub stage plan 0–5 plus the missing AssetCraft Stage 1.5 text (§16).
- **Project goal it serves:** a suite of local-first creative tools that hand work to each other (AssetCraft → StoryCraft → FilmCraft, later PrepressCraft) without the cloud, built in small, checkable steps that never wait on upstream.
- **Immediate area of improvement:** the spec has six sections of design and six separate "Not verified" scripts, but no single gate, no single list of upstream asks, and no step order. This section gives each of those one home, so Paul can run one script, send one small PR, and start one stage.

**Evidence basis.** Static read only; nothing was built or run. Sibling facts come from the earlier sections and are cited by their IDs: `01` = `01-goals-glossary.md` (A1), `03a`/`03b` = inventories (O1–O14, O-B1–O-B13), `A3` = `04-05-architecture-discovery.md`, `A4` = `06-protocol.md`, `A5` = `07-asset-refs.md`. Where this section needed a detail those docs lack, it re-read source and cites `repo` `path:line` @ commit: `assetcraft` `3a41b2b`, `photocraft` `fd28da5`, `lightcraft` `38adc17`, `filmcraft` `9ed9650`, `pdfcraft` `ac89a11`, `wordcraft` `3292ba0`, `designcraft` `7ed5697`, `effectcraft` `2c08f67`, `vectorcraft` `0946bc5`, `soundcraft` `3a28a65`. Crate licences in §13.2.2 come from crates.io metadata (`cargo info`, 2026-10-15), not from a `cargo deny` run. A6 (§8–9), A7 (§10, §12) and A8 (§11) were drafting in parallel and their docs were not on disk when this was written; their items are referenced by category and must be merged by the integrator. Terms follow `01` §2: **connection** (never "session") for a hub client link, **command effect** in prose, field `effects`, address `<app>/<command.id>`.

**New facts found in this pass (not in earlier sections):**

| ID     | Fact                                                                                                                                                                                                                                              | Source                                                                                                                                                                                                                                                                                                                                                                                     |
|--------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| A9-N1  | Every control server in the suite is `std::net::TcpListener` plus `std::thread` plus `std::sync::mpsc` into the UI thread. None uses tokio                                                                                                        | `photocraft` `apps/photocraft/src/control_server.rs:5,7,20,28,40`; `lightcraft` `apps/lightcraft/src/control_server.rs:11,14,28,36`; `filmcraft` `…/control_server.rs:10,13,27,35`; `designcraft` `…:11,14,28,36`; `effectcraft` `…:10,13,27,35`; `vectorcraft` `…:11,14,62,70`; `soundcraft` `…:10,13,28,38`; `wordcraft` `…:9,12,40,109`; `pdfcraft` `crates/ui-egui/src/control.rs:791` |
| A9-N2  | tokio appears only under the rmcp MCP servers of PhotoCraft and FilmCraft (and in ArtCraft, which is not a hub app)                                                                                                                               | `photocraft` `crates/automation/Cargo.toml:23-24`, `apps/photocraft-cli/Cargo.toml:32`; `filmcraft` `crates/automation/Cargo.toml:18-19`; `artcraft` `Cargo.toml:118`                                                                                                                                                                                                                      |
| A9-N3  | The suite already uses BLAKE3, as `default-features = false, features = ["pure"]` (no C or assembly build step)                                                                                                                                   | `photocraft` `crates/engine/Cargo.toml:34`                                                                                                                                                                                                                                                                                                                                                 |
| A9-N4  | Fuzzing precedent: standalone `cargo-fuzz` crates **excluded from the workspace** (`[workspace]` in their own `Cargo.toml`), libFuzzer, nightly, run by a manually dispatched CI job                                                              | `photocraft` `crates/psd/fuzz/Cargo.toml:8-9,15`; `book/src/security/fuzzing.md:3-17`                                                                                                                                                                                                                                                                                                      |
| A9-N5  | `proptest = "1"` is a workspace dependency in all nine apps                                                                                                                                                                                       | e.g. `designcraft` `Cargo.toml:42`, `filmcraft` `Cargo.toml:71`, `pdfcraft` `Cargo.toml:69`                                                                                                                                                                                                                                                                                                |
| A9-N6  | DesignCraft's `commands --json` prints `[]` because `commands` takes its first argument as a FILTER and `--json` matches nothing; `commands` always prints JSON                                                                                   | `designcraft` `apps/designcraft-cli/src/main.rs:6` (usage), `:104-115`                                                                                                                                                                                                                                                                                                                     |
| A9-N7  | The "7979 clash" in `01` Not verified #3 is between **client defaults**, not server binds: `vectorcraft` MCP's `DEFAULT_ADDR` and `soundcraft-cli app`'s default `--port`. Starting both apps can't fail a bind; a client can reach the wrong app | `vectorcraft` `crates/mcp/src/lib.rs:32-33`; `soundcraft` `apps/soundcraft-cli/src/main.rs:362`                                                                                                                                                                                                                                                                                            |
| A9-N8  | Upstream commit rules differ from AssetCraft's `Stage N step M:`: FilmCraft wants a task id or an area prefix (`M9.9: …`, `docs: …`), and AI agents end commit messages with their attribution line                                               | `filmcraft` `docs/contributing.md:140-148`; PdfCraft PR template `.github/pull_request_template.md:1-76`                                                                                                                                                                                                                                                                                   |
| A9-N9  | EffectCraft's `command.list` reply has `why` and optional `schema`, but not the `journal` flag the spec already carries                                                                                                                           | `effectcraft` `crates/engine/src/commands/query.rs:9-30` (reply object at `:21`); `crates/engine/src/commands/mod.rs:114-115`                                                                                                                                                                                                                                                              |
| A9-N10 | PdfCraft's control auth frame is JSON-RPC `{"method":"auth","params":{"token":…}}` → `{"result":{"ok":true}}`; after auth a line has no length cap and an unparsable line gets `-32700` without closing                                           | `pdfcraft` `crates/ui-egui/src/control.rs:806-846`                                                                                                                                                                                                                                                                                                                                         |

---

## 13. Engineering rules and quality gate

### 13.1 How AssetCraft's `AGENTS.md` applies to hub crates

The hub incubates in the AssetCraft workspace (A3-R1, decision D2), so it inherits the workspace lints, the layer check and the deny config **as they are**. The table says what each non-negotiable means for hub code, and what the hub has to add.

| Rule (`assetcraft` `AGENTS.md` @ `3a41b2b`) | Applies to hub crates as                                                                                                                                                                                                                                        | Already enforced by                                                                                                                                                                  | Hub addition                                                                                                                                                                                             |
|---------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| 1. Rust only                                | Daemon, `craftctl`, MCP shim and status panel are Rust. The panel is egui at L6 only. MCP is stdio (later loopback HTTP from Rust), never a web UI                                                                                                              | Review                                                                                                                                                                               | None                                                                                                                                                                                                     |
| 2. Layering                                 | Every hub crate gets a `TABLE` row (A3 §4.3) before its first line of code                                                                                                                                                                                      | `xtask/src/layers.rs:19-33`, `check()` `:91-120`                                                                                                                                     | Rows in §13.2.1, the `ISLANDS` rule (A3-O2), and a row for the test kit                                                                                                                                  |
| 3. Everything is a command                  | Every hub method is a command in the hub's own registry (`hub.*`, A4 §6.2.1) and runs through one dispatcher, `Hub::execute(method, params)`. `craftctl`, the MCP shim and the panel only call hub methods. The dispatcher is not called `Session` (`01` A1-O3) | Review; contract test that every `hub.*` method has an `effects` row                                                                                                                 | Test `hub_methods_have_effects`                                                                                                                                                                          |
| 4. Never crash                              | Every byte from an app, a manifest, a config file, a token file or a client is hostile. Bounded reads everywhere (A4 §6.1.4, A3 §5.2 reading rules). A poisoned lock is recovered, not unwrapped. A panic in a dependency is caught at the worker boundary      | Workspace lints `Cargo.toml:59-68` (`unsafe_code = "forbid"`, `unwrap_used`/`expect_used`/`panic` = deny), `[lints] workspace = true` per crate (e.g. `crates/engine/Cargo.toml:11`) | Patterns to copy: `lock().unwrap_or_else(\|p\| p.into_inner())` (`crates/thumbs/src/pool.rs:43`, `disk.rs:330`); `catch_unwind` around third-party work (`crates/thumbs/src/model.rs:174`); fuzz §13.3.5 |
| 5. Read before write                        | The hub writes only its own state dir until a hub stage allows more, and gates **which command effects it will forward** per stage (§16.0). Its own writes (manifests, tokens, ledger, hash index) are stage + rename, like PdfCraft's control file (A3 §5.2)   | Review                                                                                                                                                                               | Router stage gate (§16.0) with a test per stage; atomic writer shared with AssetCraft (`craft-atomic` pattern from craft-libs, `03b` §3.6b)                                                              |
| 6. Clean room                               | No Adobe or Microsoft code or assets; no GPL/LGPL/AGPL deps; code adapted from a sibling gets a `NOTICE` line; bundled fixtures get `ATTRIBUTION.md` rows                                                                                                       | `deny.toml:19-40` (licence allow list), `:50-55` (sources: crates.io + `storytold` git only), `cargo xtask deny` in `ci` (`xtask/src/main.rs:57`)                                    | `NOTICE` lines listed in §13.2.3; `ATTRIBUTION.md` rows for snapshots and transcripts (§13.3.3); fuzz crates stay outside the workspace (§13.3.5)                                                        |
| 7. AI optional, secrets in the keyring      | The hub has no AI. It holds app tokens and its own client token; where they live is A8's (§11). Elo stays out                                                                                                                                                   | Review                                                                                                                                                                               | Candidate crate `keyring` (§13.2.2), decided by A8                                                                                                                                                       |
| Before handing back                         | `cargo xtask ci` passes; tests run against `test_data/library`; `STATUS.md` updated; no git writes                                                                                                                                                              | `xtask/src/main.rs:33-60`                                                                                                                                                            | Hub tests must pass with **no Craft app installed**: CI uses only the fake app (§13.3.2). Live checks are separate xtask tasks (§13.2.4)                                                                 |

### 13.2 Additions to the gate

#### 13.2.1 Layer table rows

A3's rows (A3 §4.3) stand, with two additions from this section:

```rust
// xtask/src/layers.rs, TABLE: additions on top of A3 §4.3
    ("craft-assetref", Class::Layer(0)),       // A5's `assetref` crate, renamed (A9-F2)
    ("craft-hub-testkit", Class::Exempt),      // fake app + snapshots + transcripts; dev-dependency only
```

- `short_name` strips only `assetcraft-` (`xtask/src/layers.rs:73-75`), so hub rows use full package names, as A3 wrote them.
- **A9-F2.** A5 proposes a pure L0 crate named `assetref` (A5 "Inputs for downstream agents"). A3's island rule (`craft-` crates may depend only on `craft-` crates, A3-O2) would then forbid every hub crate from using it. Rename it `craft-assetref` so it travels with the hub and the control crates at promotion time.
- `craft-hub-testkit` is `Exempt` because it is tooling. Layered crates use it only as a dev-dependency, which `check()` skips (`layers.rs:99`, `DepKind::Dev` filtered).
- The `ISLANDS` change to `check()` (A3-O2) gets two unit tests next to the existing ones: a `craft-` crate depending on `thumbs` fails; `engine` depending on `craft-control-client` passes.

#### 13.2.2 Concurrency and dependencies

**Decision proposal A9-R1: no tokio in the hub v1.** Std threads, `std::sync::mpsc` channels and socket deadlines (`set_read_timeout` / `set_write_timeout`), the way every app's control server is written (A9-N1) and the way AssetCraft's pools are written (`crates/thumbs/src/pool.rs`). Answers A3 open question 5.

| Option                     | For                                                                                                                                                                                                                                                                                   | Against                                                                                                                                                              |
|----------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **Std threads (proposed)** | Same model as all nine control servers (A9-N1) and AssetCraft; no runtime; blocking code with deadlines is easy to test with fakes; thread count is small (≤ 10 apps × ≤ 4 adapter connections + ≤ 32 client connections + pollers + child readers ≈ 100 threads at the very top end) | One thread per connection and per child pipe; a future streamable-HTTP MCP endpoint (§10.5) has to be a small hand-written loopback HTTP/1.1 server or wait for v1.x |
| tokio                      | rmcp is tokio (A9-N2); async scales to many connections                                                                                                                                                                                                                               | Adds a runtime to every hub crate's tests; the hub's MCP is hand-written JSON-RPC anyway (A3 §4.3 `craft-hub-mcp`); no app talks async on its control channel        |

Revisit trigger: more than 64 concurrent client connections in practice, or a streamable-HTTP front door that needs SSE fan-out. Owner: Paul (decide), Claude (implement).

**Allowed new dependencies (proposal; each added to `[workspace.dependencies]` with a comment naming its user, like the existing ones at `Cargo.toml:23-47`).** Licence and MSRV from crates.io metadata on 2026-10-15; all are on the `deny.toml:20-32` allow list.

| Crate           | Version | Licence                                                 | MSRV | Used by                                        | Note                                                                                                                                                                                       |
|-----------------|---------|---------------------------------------------------------|------|------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `blake3`        | 1.8.7   | CC0-1.0 OR Apache-2.0 OR Apache-2.0 WITH LLVM-exception | n/a  | `craft-assetref` (A5 §7.1.3)                   | `default-features = false, features = ["pure"]` as PhotoCraft does (A9-N3): Rust SIMD paths stay, C/asm (AVX-512) goes, so no C toolchain on Windows. Re-measure `full` hashing (§13.4 P6) |
| `sysinfo`       | 0.39.6  | MIT                                                     | 1.95 | `craft-hub-discovery` (process probe, A3 §5.1) | MSRV equals the workspace's `rust-version = "1.95"` (`Cargo.toml:9`): a newer `sysinfo` may force a toolchain bump. Pin the minor                                                          |
| `toml`          | 1.1.8   | MIT OR Apache-2.0                                       | 1.85 | `craft-hub-discovery`, `craft-hub-adapters`    | Adapter config (A3 §4.4) and effects tables (A4 §6.4.3)                                                                                                                                    |
| `semver`        | 1.0.28  | MIT OR Apache-2.0                                       | 1.68 | `craft-hub-adapters`                           | Version-ranged overrides (A3-O9)                                                                                                                                                           |
| `command-group` | 5.0.1   | Apache-2.0 OR MIT                                       | 1.68 | `craft-hub-core` (headless children)           | Job object with kill-on-close on Windows, process group on Unix, for R4. Candidate only (A3 §5.3); `cfg(windows)` behaviour needs Paul's build                                             |
| `keyring`       | 4.2.0   | MIT OR Apache-2.0                                       | 1.88 | Hub token storage, if A8 chooses the keyring   | A8 decides (§11)                                                                                                                                                                           |
| `uds_windows`   | 1.2.1   | MIT                                                     | 1.85 | v1.x AF_UNIX listener (A4-R1)                  | Not in v1                                                                                                                                                                                  |
| `proptest`      | 1.12.0  | MIT OR Apache-2.0                                       | 1.88 | dev-dependency, every hub crate                | Suite standard (A9-N5)                                                                                                                                                                     |
| `libfuzzer-sys` | 0.4.13  | (MIT OR Apache-2.0) AND NCSA                            | n/a  | standalone fuzz crates only                    | **NCSA is not on the allow list.** It never enters the workspace graph: fuzz crates are excluded like PhotoCraft's (A9-N4), so `cargo deny` never sees it                                  |

`deny.toml` needs **no change** for the workspace: `[graph] all-features = true` and the five targets (`deny.toml:8-17`) already cover the hub crates once they are members, and no new git source is needed. Opportunity A9-O1: add a one-line comment above `allow = [` naming the hub's licences, so the next reviewer knows why `CC0-1.0` and `BSL-1.0` stay.

#### 13.2.3 `NOTICE` lines the hub will need

Each is "Copy" in the suite's Depend / Copy / Call / Build terms, so `NOTICE` gets a provenance line (repo, path, revision) when the code lands:

| Hub code                                     | Adapted from                                                            | Cite                                                                            |
|----------------------------------------------|-------------------------------------------------------------------------|---------------------------------------------------------------------------------|
| Guarded NDJSON server loop with deadlines    | VectorCraft control server (line guard, deadlines, request withdrawal)  | `vectorcraft` `apps/vectorcraft/src/control_server.rs` @ `0946bc5` (`03b` O-B1) |
| Token handshake and token file handling      | PhotoCraft automation security                                          | `photocraft` `crates/automation/src/security.rs:74-250` @ `fd28da5` (`03a` O2)  |
| Control/manifest file write (stage + rename) | PdfCraft control file publisher                                         | `pdfcraft` `apps/pdfcraft/src/main.rs:432-499` @ `ac89a11` (A3 §5.2)            |
| `hub.lock` single instance                   | LightCraft catalog lock                                                 | `lightcraft` `crates/catalog/src/lock.rs:1-19` @ `38adc17` (A3 §5.4)            |
| Installed-app locator                        | AssetCraft's own `open_in.rs` (same repo, no NOTICE line; moved, A3-O8) | `assetcraft` `crates/engine/src/open_in.rs:166-174`                             |

#### 13.2.4 xtask tasks

`cargo xtask ci` keeps its exact sequence (`xtask/src/main.rs:33-60`); new hub crates join `fmt`, `clippy` and `test` automatically through the `crates/*` / `apps/*` globs (`Cargo.toml:3`). Three **opt-in** tasks are added, none of them part of `ci`, because each needs something CI doesn't have:

| Task                                        | What it does                                                                                                                                                                       | Needs                                           | Owner                              |
|---------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------|------------------------------------|
| `cargo xtask contract check [--app <id>]`   | Runs each installed app's catalog argv (adapter config, `03b` O-B3), normalizes it, diffs it against the stored snapshot, prints added / removed / changed ids. Exit 1 on a change | Installed apps (found by the hub's discovery)   | Claude (task), Paul (runs)         |
| `cargo xtask contract refresh [--app <id>]` | Same, then rewrites the snapshot and its `.meta.toml` (§13.3.3)                                                                                                                    | Same                                            | Paul runs, Claude reviews the diff |
| `cargo xtask fuzz <target> [--time <s>]`    | `cargo +nightly fuzz run` in a standalone fuzz crate (§13.3.5)                                                                                                                     | Nightly, `cargo-fuzz`; Linux or WSL recommended | Claude                             |

Perf numbers are `#[ignore]` tests (§13.4), run with `cargo test --release -p craft-hub-core -- --ignored perf_ --nocapture`, so no `bench` task is needed. Budget proposal: the hub's tests add at most 30 s to `cargo xtask ci` on Paul's machine (A9-O2, owner Claude; measure at hub Stage 0).

### 13.3 Testing strategy

#### 13.3.1 Levels

| Level                | What                                                                                                                                                                           | Where                                                              | In `xtask ci`                     |
|----------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------|-----------------------------------|
| Unit                 | Pure functions: target grammar (A4 §6.2.2), id folding (A4 N3/N4), effects resolution (A4 §6.4), error-prefix mapping (A4 §6.5.3), manifest validation, `WirePath::parse` (A5) | Each crate's `src/*_tests.rs`, AssetCraft style                    | yes                               |
| Property             | Codecs and readers never panic and respect limits; round trips (`encode ∘ decode`)                                                                                             | `proptest` in each codec crate                                     | yes (1,000 cases)                 |
| Fake-app integration | Each adapter kind against the fake app over real loopback sockets and real child processes (§13.3.2)                                                                           | `crates/craft-hub-adapters/tests/`, `crates/craft-hub-core/tests/` | yes                               |
| Contract             | Adapter configs, effects tables, overlays and aliases checked against recorded catalogs (§13.3.3)                                                                              | `crates/craft-hub-adapters/tests/contract.rs`                      | yes (offline)                     |
| Golden transcript    | Recorded wire conversations replayed byte for byte (§13.3.4)                                                                                                                   | `crates/craft-hub-testkit/transcripts/`                            | yes                               |
| Fuzz                 | libFuzzer on framing and parsers (§13.3.5)                                                                                                                                     | `fuzz/` crates outside the workspace                               | no (manual)                       |
| Live                 | Real apps on Paul's Windows machine (§13.5) and `cargo xtask contract check`                                                                                                   | PowerShell                                                         | no                                |
| Perf / soak          | P1–P6, R1–R5 (§13.4)                                                                                                                                                           | `#[ignore]` tests, PowerShell soak                                 | no (R1–R3 yes, with loose bounds) |

#### 13.3.2 The fake app (`craft-hub-testkit`)

One binary, `craft-fake-app`, plus a library with an in-process variant for fast unit tests (the `FakeLocator` / `ToolRunner` pattern AssetCraft already uses, `crates/thumbs/src/decoders/video.rs:108`). It serves a **recorded real catalog** (§13.3.3), so a test against the fake sees the same ids, keys and quirks as the real app at that version.

| Mode (`--kind`)   | Emulates                                                                                                  | Auth variants (`--auth`)                                                                                                                                                                                                                              | Must reproduce (quirk flags)                                                                                                                                                                                                                                                                                                                                                                                                       |
|-------------------|-----------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `control-jsonl`   | PhotoCraft, LightCraft, FilmCraft, WordCraft, DesignCraft, EffectCraft, VectorCraft, SoundCraft (A3 §4.4) | `none`; `token-frame` (first frame `auth {token}`, token read from `--control-token-file`, created if missing, `03a` §3a.2); `key-field` (key file `control-key.local-<port>` in `--profile-dir`, `key` on every request or `auth` once, `03a` §3a.6) | `--port-report requested\|stdout\|none` (A3-F1: logs the requested port, SoundCraft prints the real one); `--bind-fail continue\|exit` (A3-F2); `--junk close\|continue` (`03b` O-B2); `--unknown-method error\|runs-as-command` (A3-O6); `--max-line <bytes>`; `--timeout-text bare\|photocraft` (A4-F3); `--disabled-key why\|disabled_reason\|disabledReason` (`03a` O9); `--idle-drop <s>` (LightCraft 5 s, VectorCraft 5 min) |
| `control-jsonrpc` | PdfCraft (A3 §4.4)                                                                                        | Token in a control file the caller names; JSON-RPC `auth` first (A9-N10)                                                                                                                                                                              | `--control <file>` writes `{port,token,pid}` and **does not remove it on exit** (A3-F4); pre-auth 4 KiB cap, no cap after (A4-F1); `ui.command {id}` takes no params (`03a` O6)                                                                                                                                                                                                                                                    |
| `cli-headless`    | Every `<app>-cli` (A3 §4.4)                                                                               | n/a                                                                                                                                                                                                                                                   | `--catalog-argv` styles: `commands --json`, `commands` only, `--json` treated as a filter (A9-N6), text lines only (SoundCraft), `tools` (PdfCraft); `exec <id> <json>`; `run <script.jsonl>`; exit codes 0/1/2; `--hang` (for timeout kill); `--flood <bytes>` on stdout (bounded reads)                                                                                                                                          |
| `mcp-stdio`       | `<app>-cli mcp` (A3 §4.4)                                                                                 | n/a                                                                                                                                                                                                                                                   | Tool styles `suite-28`, `generic-execute`, `pdfcraft-tools`; `notifications/progress` every N ms; honours `notifications/cancelled`; `--die-after <n>` requests                                                                                                                                                                                                                                                                    |

Shared flags for every mode: `--latency-ms <n>` (fixed reply delay, for P1), `--jobs poll` (a `jobs.list` whose entries advance on a timer and carry the timestamp of each state change, for P4), `--hang-on <command id>` (R1), `--app <id> --version <semver>` (identity and version gates, A3 §5.6), `--catalog <file>`.

Tests that use it, at minimum (one per row of A3 §5.3 and A4 §6.5.2 that a fake can trigger):

| Test                                | Fake setup                               | Asserts                                                                    | Spec item      |
|-------------------------------------|------------------------------------------|----------------------------------------------------------------------------|----------------|
| `attach_none_auth`                  | `control-jsonl --auth none`              | `app.up`, catalog merged, identity probe passed                            | A3 §5.3        |
| `attach_token_frame` / `_bad_token` | `--auth token-frame`                     | Good token works; bad token gives `app_auth_failed`, connection closed     | A4 §6.5.2      |
| `attach_key_field`                  | `--auth key-field`                       | Key read from the key file **for that port**                               | A3 §5.3 step 3 |
| `identity_mismatch_other_listener`  | Two fakes, second `--bind-fail continue` | The hub detects the wrong app on the port and retries once on another port | A3-F2, §5.3    |
| `port_zero_lost`                    | `--port-report requested` with port 0    | The hub never passes 0 to that app kind                                    | A3-F1          |
| `junk_continue_never_attached`      | `--junk continue` (SoundCraft)           | The hub's own client never sends non-JSON; config marks the app risky      | `03b` O-B2     |
| `wordcraft_unknown_method`          | `--unknown-method runs-as-command`       | The hub sends no invented methods (no `ping`)                              | A3-O6          |
| `timeout_ran_maybe`                 | `--hang-on x --timeout-text bare`        | `timeout_unknown`, `ran: maybe`, no auto-retry of a write                  | A4-F3, §6.4.2  |
| `kill_mid_call`                     | `--hang-on x`, then kill the process     | Typed error within 2 s; hub answers `hub.ping` after                       | R1             |
| `stale_control_file`                | `control-jsonrpc`, exit                  | File trusted only with a live `pdfcraft`-stem pid                          | A3-F4          |
| `headless_children_die_with_hub`    | Three `mcp-stdio` children, kill the hub | Zero children after 2 s                                                    | R4             |
| `cli_flood_bounded`                 | `cli-headless --flood 64MiB`             | Read stops at the configured cap, child killed, error typed                | A3 §5.3        |
| `stage_gate_refuses_writes`         | Any fake, a `write-user-file` command    | `forbidden` at hub Stage 0–1 (§16.0)                                       | §16.0          |

#### 13.3.3 Contract tests against recorded catalogs

**Where snapshots live.** `crates/craft-hub-testkit/snapshots/<app>/<app_version>.catalog.json` plus `<app_version>.meta.toml`. Not in `test_data/`: that folder is CC0 and script-generated (`test_data/README.md:3-5`), while a catalog is output of an MIT OR Apache-2.0 app, so it gets its own `ATTRIBUTION.md` row: `crates/craft-hub-testkit/snapshots/` · storytold apps · output of `<app>-cli` at the commit in each `.meta.toml` · MIT OR Apache-2.0.

```toml
# crates/craft-hub-testkit/snapshots/filmcraft/0.8.0.meta.toml
app          = "filmcraft"
app_version  = "0.8.0"
commit       = "9ed9650"            # from `git -C <repo> log -1 --format=%h`, or "installed" for an MSI build
argv         = ["commands", "--json"]
captured_at  = "2026-10-15"
captured_by  = "paul"
sha256       = "<of the normalized file>"
count        = 702
```

The JSON is **normalized** before it is stored (sorted by `id`, pretty-printed, key order fixed) so a refresh shows a reviewable diff. Size estimate: about 2–3 MB for all ten apps (the largest catalogs have 700–860 entries, `03a` §3a.1, `03b` §3.0); text, so no `.gitattributes` change.

**How they are refreshed.** Paul runs `cargo xtask contract refresh` on Windows (needs the apps installed or built), or the first time, step 3 of the §13.5 script, which writes the raw catalogs to `%LOCALAPPDATA%\PipelineCraft\snapshots\` for Claude to import. Old versions stay as long as an adapter config's `tested` range names them (A3 §5.6), so version-ranged overrides (A3-O9) stay tested.

**What the offline contract tests check** (every snapshot, every run of `xtask ci`):

1. The adapter parses the snapshot into `CommandInfo` (`03a` O9) with no error and no dropped entry.
2. Every id named in that app's profile exists in the snapshot for each version in `tested`: effects rows (A4 §6.4.3), aliases (A4 N5), capability mappings (`03b` O-B8), schema overlays (A4 S4), `identity_probe` ids (A3 §5.3 step 4).
3. Fold-key collisions (A4 N4) are reported.
4. A command with no row and no inferred effects resolves to `source: default` with the worst-case effects (A4 §6.4.2).
5. The error-prefix table classifies the recorded error strings (A4 Not verified #1–#2, kept as fixtures in `snapshots/<app>/errors.json`) to the expected codes.
6. Every overlay's `required` keys appear in the command's `params` text (A4 S4).

#### 13.3.4 Golden protocol transcripts

`crates/craft-hub-testkit/transcripts/<adapter-kind>/<case>.jsonl`, one JSON object per line, so junk bytes and exact line endings survive:

```json
{"dir":"c2s","raw":"{\"id\":\"auth\",\"method\":\"auth\",\"params\":{\"token\":\"<TOKEN>\"}}"}
{"dir":"s2c","raw":"{\"id\":\"auth\",\"ok\":true,\"result\":{\"authenticated\":true}}"}
{"dir":"c2s","raw":"{\"id\":2,\"method\":\"engine.commands\"}"}
{"dir":"s2c","raw":"<SNAPSHOT:photocraft/0.9.0>","delay_ms":3}
```

- **Two sources.** (a) Written by hand from the static read, for every A4 §6.7 worked example and every handshake in `03a`/`03b` (status `static`). (b) Recorded on Paul's machine by a `--record <dir>` flag on the hub's adapters (status `live`), which replaces each secret with `<TOKEN>` / `<KEY>` before writing. A transcript goes from `static` to `live` when a recording matches it.
- **Two uses.** The fake app replays the `s2c` side as a scripted server (adapter tests), and the hub listener is driven with the `c2s` side of hub-protocol transcripts (A4 §6.7 examples) to assert byte-exact replies apart from fields marked volatile (`connection`, timestamps).
- Transcripts are hub-authored test data: `ATTRIBUTION.md` row as AssetCraft contributors, CC0-1.0, except `live` ones that embed app output, which take the snapshot row's licence.

#### 13.3.5 Fuzzing the framing

| Target                 | Input                                        | Property                                                                                     | Run as                    |
|------------------------|----------------------------------------------|----------------------------------------------------------------------------------------------|---------------------------|
| `line_reader`          | arbitrary bytes, random chunking             | Never more than `max` bytes buffered; `TooLong` exactly when a line exceeds `max`; no panic  | proptest (CI) + libFuzzer |
| `suite_jsonl_decode`   | arbitrary line                               | Ok or a typed `bad_frame` / `bad_request`; unknown top-level key rejected (A4 §6.1.4)        | proptest (CI) + libFuzzer |
| `jsonrpc2_decode`      | arbitrary line                               | Same, for the PdfCraft and MCP codecs                                                        | proptest (CI) + libFuzzer |
| `app_reply_decode`     | arbitrary reply line for a known request     | Never panics; mismatched id → `adapter_protocol`; huge `result` → refused at the adapter cap | proptest (CI) + libFuzzer |
| `manifest_parse`       | arbitrary file ≤ 64 KiB                      | Ignored-and-reported or valid; 16 KiB limit holds (A3 §5.2)                                  | proptest (CI) + libFuzzer |
| `adapter_config_parse` | arbitrary TOML                               | Typed error, never a partial config                                                          | proptest (CI)             |
| `target_parse`         | arbitrary string                             | Ok or `bad_target` (A4 §6.2.2)                                                               | proptest (CI) + libFuzzer |
| `wire_path_parse`      | arbitrary UTF-16-ish strings, `windows` flag | Refuses device paths, volume GUIDs, ADS, `..`, unpaired surrogates, NUL (A5 "Inputs" for A8) | proptest (CI) + libFuzzer |

- **CI (property tests):** 1,000 cases per target (`PROPTEST_CASES` default); 10,000 under `--ignored` to satisfy R2 (§13.4).
- **libFuzzer:** one standalone crate per target crate, `crates/<crate>/fuzz/`, with its own `[workspace]` so it is not a member and `cargo deny` never sees `libfuzzer-sys`'s NCSA licence (A9-N4). Nightly only; Linux or WSL is the supported place to run it (owner Claude). Every crash becomes a permanent, minimized regression test in the normal suite before the fix counts (PhotoCraft's corpus policy, `book/src/security/fuzzing.md:35-39`).
- **Socket level:** the corpus in `test_data/hub/frames/` (below) is replayed against the hub listener and, through the fake app, against every adapter's reader. That is R2's "per adapter" half.

#### 13.3.6 `test_data` fixtures

New generator `test_data/make_hub_fixtures.py` (Python standard library only, seeded, CC0, like `make_model_fixtures.py`) writes `test_data/hub/`, **outside** `library/` so the folder-count tests are untouched (`test_data/README.md:34-35`: a new folder changes them). One `ATTRIBUTION.md` row: `test_data/hub/` · AssetCraft contributors · generated by `test_data/make_hub_fixtures.py` · CC0-1.0.

| Folder           | Contents                                                                                                                                                                                                         | Exercises                 |
|------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------|
| `hub/manifests/` | valid v1 (`app` and `hub` publisher); unknown `manifest: 2`; 17 KiB file; bad pid; `token_file` outside allowed dirs; unknown fields; invalid UTF-8; truncated JSON                                              | A3 §5.2 reading rules, R3 |
| `hub/frames/`    | HTTP preamble (`POST / HTTP/1.1` …); UTF-8 BOM; invalid UTF-8; a line of exactly 4 MiB and one of 4 MiB + 1; 10,000-deep nested arrays; NUL bytes; CRLF and bare CR; two objects on one line; a JSON array frame | A4 §6.1.4, R2             |
| `hub/config/`    | adapter-config overrides: good, unknown key, wrong types, version range that overlaps                                                                                                                            | A3 §4.4, A3-O9            |
| `hub/recipes/`   | a 3-step recipe with a dry run; one with a step whose output feeds the next; one that names a missing app                                                                                                        | §8.4–8.5, §9.4 (A6)       |
| `hub/otio/`      | a hand-written 3-clip OTIO timeline referencing `library/environments/*.mp4` by absolute `file://` URL placeholder rewritten at test time                                                                        | AC3 (Stage 5), A5 §7.2.5  |

Reused from `library/` as A5 asked: `environments/dupes/snow_clearing_plate_copy.jpg` (same bytes, different asset) and `edge-cases/Réaper ünïcode name.png` (NFC name). The > 260-character path is made at test time in a temp dir (and on Paul's machine by §13.5 step 25), not committed.

### 13.4 Performance and robustness targets

A1's draft numbers (`01` §1.4) are kept unchanged, plus one new target P6 from A5. **None is a gate until Paul's Windows baseline exists**; CI checks a loose bound (about 10× the target) so a gross regression still fails, as AssetCraft's exit tests do ("limit 5 s, generous for Windows Defender", `assetcraft` `STATUS.md:122`).

| ID  | Measure                                        | Target (draft, A1)                                     | How measured                                                                                                                                                                                                                                                                                         | CI bound                | Becomes a gate        |
|-----|------------------------------------------------|--------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------|-----------------------|
| P1  | Hub-added latency, read call, warm connection  | p50 ≤ 2 ms, p95 ≤ 10 ms                                | `perf_p1_call_overhead` (`#[ignore]`): in-process fake app (`control-jsonl`, `--auth none`, `--latency-ms 0`) on loopback. 100 warm-up calls, then 1,000 pairs of the same read sent **directly** and **through the hub**; report p50/p95 of the paired difference. Real-app floor from §13.5 step 8 | p95 < 50 ms             | Hub Stage 1 exit      |
| P2  | `craftctl apps` with 10 known apps             | ≤ 1 s cold, ≤ 100 ms warm                              | `perf_p2_apps`: 10 fake apps (mixed kinds) registered in a temp `PIPELINECRAFT_HOME`. Cold = no daemon running and no `hub.json`, so the time includes daemon start; warm = second call. Windows: `Measure-Command { craftctl apps --json }` twice                                                   | cold < 5 s              | Hub Stage 0 exit      |
| P3  | Attach to a running app with control on        | ≤ 500 ms incl. handshake                               | Hub log lines `attach.start` and `app.up` carry monotonic timestamps; the test reads the delta for a `token-frame` fake; Windows: same log for real apps                                                                                                                                             | < 2 s                   | Hub Stage 0 exit      |
| P4  | Event delivery, state change → subscriber      | p95 ≤ 100 ms push; ≤ poll interval + 50 ms when polled | Fake app `--jobs poll` stamps each job state change; a subscriber records arrival; same clock (one machine); 200 events. With the 250 ms poll interval (A3 §4.1) the polled bound is 300 ms                                                                                                          | p95 < 1 s               | Hub Stage 3 exit      |
| P5  | AC1 Open In into a running app                 | ≤ 1 s command → app reply                              | Hub log (request in → app reply out), plus end to end on Windows: `Measure-Command { assetcraft-cli exec file.openIn '{"app":"photocraft"}' }` with one selected file and PhotoCraft running with control                                                                                            | none (needs a real app) | Hub Stage 2 exit      |
| P6  | `q1` quick hash per asset ref (new, A5 §7.1.3) | p95 ≤ 5 ms on SSD, any file size                       | `perf_p6_q1` over `test_data/library` plus, on Windows, Paul's largest local files; `blake3` `pure` build (§13.2.2). Also record `full` throughput with A5 Not verified #5 (§13.5 step 27)                                                                                                           | functional only         | Hub Stage 2 exit      |
| R1  | App killed mid-call                            | Typed error ≤ 2 s; hub stays up                        | `kill_mid_call` (§13.3.2), deterministic                                                                                                                                                                                                                                                             | ≤ 5 s                   | Now (Stage 1)         |
| R2  | Hostile frames                                 | Rejected, connection closed, no panic, hub up          | Property tests per codec (1,000 in CI, 10,000 `--ignored`), plus the `test_data/hub/frames/` corpus against the listener and every adapter reader (§13.3.5)                                                                                                                                          | 1,000 cases             | Now (Stage 0)         |
| R3  | Stale manifest                                 | Cleaned within one discovery pass                      | Test spawns a child, records its pid, lets it exit, writes a manifest with that pid, runs one `discovery.pass()`: `publisher: hub` file deleted, `publisher: app` file ignored (A3 §5.4)                                                                                                             | yes                     | Now (Stage 0)         |
| R4  | No orphans                                     | Zero hub-owned children after hub exit or crash        | `headless_children_die_with_hub` on Linux (process group); on Windows the job-object path is `cfg(windows)`, checked by §13.5 step 30 (`Get-Process craft-fake-app`)                                                                                                                                 | yes (Linux)             | Hub Stage 2 exit      |
| R5  | 24 h soak                                      | RSS growth ≤ 10 % after warm-up                        | PowerShell loop: one `craftctl call` per second plus a `craftctl watch` subscriber, sampling `(Get-Process pipelinecraft).WorkingSet64` every 5 min to CSV; compare hour 1 to hour 24                                                                                                                | no                      | Hub Stage 5 exit (v1) |
| R6  | Never-crash rule                               | `cargo xtask ci` clean                                 | Workspace lints (`Cargo.toml:59-68`)                                                                                                                                                                                                                                                                 | yes                     | Always                |

### 13.5 Windows verification checklist (consolidated)

Every "Not verified" step in `01`, `03a`, `03b`, A3, A4 and A5, deduplicated and grouped by app. When A6, A7 and A8 land, their steps append as groups W13+ (integrator). Each step says what it verifies and what to expect.

**Deduplication notes.**
- Command counts (`01` #1, `03a` #1–2, `03b` #1–4) are one step (3), which also saves the first contract snapshots.
- PdfCraft control file (`01` #2, `03a` #8, A3 #4) is one step (16).
- "No auth" checks (`01` #3, `03a` #4) fold into each app's group.
- **A9-F1 (port clash in the earlier scripts):** `03a` #5 uses 7981 for WordCraft and `03b` #5 uses 7981 for DesignCraft. This script uses one port plan so every app can stay open at once: PhotoCraft 47601, LightCraft 47602, FilmCraft 47603, WordCraft 47605, DesignCraft 47606, EffectCraft 47607, VectorCraft 47608, SoundCraft `0` (prints its port), bind-failure test 47650, PdfCraft a control file. These sit in A3's draft `port_range` neighbourhood (A3 §5.2 example uses 47601).
- **A9-F3 (wrong expectation in `01` #3):** the "VectorCraft and SoundCraft both bind 7979, the second fails" expectation can't happen: 7979 is a client default in both (A9-N7). Step 20 tests what can happen instead.

Run everything in **one** PowerShell window, in order (the helpers in step 2 live in that window). Use throwaway files only. Paths assume repos under `F:\Script-Projects\ArtCrafting\`; AssetCraft may still be at `F:\Script-Projects\assetcraft` until the move (project context §1), so adjust `$AC` in step 2 and the `Set-Location` lines of steps 13, 25 and 26 if so.

**W0. Setup (all apps)**

1. Clone and build (verifies nothing yet; read-only clones of upstream are enough for this script; §14.4 turns the PR targets into forks).
   ```powershell
   Set-Location F:\Script-Projects\ArtCrafting
   $apps = 'photocraft','lightcraft','filmcraft','pdfcraft','wordcraft','designcraft','effectcraft','vectorcraft','soundcraft'
   foreach ($a in $apps + 'artcraft') { if (-not (Test-Path ".\$a")) { git clone -c core.autocrlf=false "https://github.com/storytold/$a.git" $a } }
   foreach ($a in $apps) { Push-Location ".\$a"; cargo build --release -p $a -p "$a-cli"; Pop-Location }
   ```
   Expected: nine release builds. If a package name differs from `<app>` / `<app>-cli`, note it (the earlier sections assumed these names).

2. Helpers for this window.
   ```powershell
   Set-Location F:\Script-Projects\ArtCrafting
   $R  = 'F:\Script-Projects\ArtCrafting'
   $AC = "$R\assetcraft"                      # or F:\Script-Projects\assetcraft before the move
   $H  = "$env:LOCALAPPDATA\PipelineCraft"
   $LIB = "$AC\test_data\library"
   New-Item -ItemType Directory -Force "$H\tokens", "$H\handoff", "$H\snapshots", "$env:LOCALAPPDATA\CraftApps\run" | Out-Null
   function Exe([string]$a) { "$R\$a\target\release\$a.exe" }
   function Cli([string]$a) { "$R\$a\target\release\$a-cli.exe" }
   function Q([string]$s) { '"' + $s + '"' }
   function J($o) { $o | ConvertTo-Json -Compress -Depth 8 }
   function Send-Craft([int]$Port, [string[]]$Lines, [int]$Read = 1, [int]$Max = 240) {
     $c = [Net.Sockets.TcpClient]::new('127.0.0.1', $Port)
     try {
       $c.ReceiveTimeout = 15000
       $s = $c.GetStream(); $w = [IO.StreamWriter]::new($s); $w.NewLine = "`n"; $w.AutoFlush = $true
       $r = [IO.StreamReader]::new($s)
       foreach ($l in $Lines) { $w.WriteLine($l) }
       for ($i = 0; $i -lt $Read; $i++) {
         try { $x = $r.ReadLine() } catch { $x = "<no reply: $($_.Exception.Message)>" }
         if ($null -eq $x) { '<closed>'; break }
         if ($x.Length -gt $Max) { $x.Substring(0, $Max) + " …($($x.Length) chars)" } else { $x }
       }
     } finally { $c.Close() }
   }
   ```
   Expected: no output. `J` builds JSON with correctly escaped Windows paths.

3. Live catalogs, counts and first snapshots. Verifies `01` #1, `03a` #1–2, `03b` #1–4, O-B3, A9-N6, §13.3.3.
   ```powershell
   Set-Location F:\Script-Projects\ArtCrafting
   foreach ($a in 'photocraft','lightcraft','filmcraft','wordcraft','effectcraft') {
     & (Cli $a) commands --json | Out-File "$H\snapshots\$a.catalog.json" -Encoding utf8
     '{0,-12} {1}' -f $a, (Get-Content "$H\snapshots\$a.catalog.json" -Raw | ConvertFrom-Json).Count
   }
   foreach ($a in 'designcraft','vectorcraft') {
     & (Cli $a) commands | Out-File "$H\snapshots\$a.catalog.json" -Encoding utf8
     '{0,-12} {1}' -f $a, (Get-Content "$H\snapshots\$a.catalog.json" -Raw | ConvertFrom-Json).Count
   }
   'designcraft commands --json: ' + ((& (Cli designcraft) commands --json) -join '')
   'soundcraft commands lines:   ' + ((& (Cli soundcraft) commands | Measure-Object -Line).Lines)
   'soundcraft commands --json:  [' + ((& (Cli soundcraft) commands --json) -join '') + ']'
   & (Cli soundcraft) commands | Out-File "$H\snapshots\soundcraft.catalog.txt" -Encoding utf8
   & (Cli pdfcraft) tools | Out-File "$H\snapshots\pdfcraft.tools.json" -Encoding utf8
   'pdfcraft tools: ' + (Get-Content "$H\snapshots\pdfcraft.tools.json" -Raw | ConvertFrom-Json).Count
   & (Cli effectcraft) info --json
   foreach ($a in $apps) { '{0,-12} {1}' -f $a, (& (Cli $a) --version) }
   ```
   Expected: photocraft ≈ 863, lightcraft ≈ 313, filmcraft ≈ 702, wordcraft ≈ 629, effectcraft ≥ 728 (equal to `info`'s `commands`), designcraft ≈ 463 and `--json` gives `[]`, vectorcraft ≈ 743, soundcraft ≈ 478 lines and `[]` (nothing printed), pdfcraft 163; one version per CLI. Keep `$H\snapshots\` and the version lines: Claude imports them as the first contract snapshots.

**W1. PhotoCraft**

4. Token handshake, read root and path forms. Verifies `03a` #3, O13, A5 #6, A5-O3.
   ```powershell
   Set-Location F:\Script-Projects\ArtCrafting\photocraft
   $pcTok = "$H\tokens\photocraft-47601.token"
   $pc = Start-Process (Exe photocraft) -ArgumentList '--control','47601','--control-token-file',(Q $pcTok),'--automation-read-root',(Q $LIB) -PassThru
   Start-Sleep 8
   $auth = J @{ id = 'auth'; method = 'auth'; params = @{ token = (Get-Content $pcTok -Raw).Trim() } }
   Send-Craft 47601 @($auth, '{"id":2,"method":"engine.commands"}') -Read 2
   Send-Craft 47601 @($auth,
     (J @{ id = 3; method = 'app.open'; params = @{ path = 'environments/snow_clearing_plate.jpg' } }),
     (J @{ id = 4; method = 'app.open'; params = @{ path = "$LIB\environments\ruins_vertical.png" } }),
     (J @{ id = 5; method = 'app.open'; params = @{ path = ("$LIB\environments\ruins_vertical.png").ToLower() } })) -Read 4
   ```
   Expected: `{"id":"auth","ok":true,"result":{"authenticated":true}}`, then a long `engine.commands` reply; ids 3 and 4 `ok`; id 5 refused with "absolute paths must be inside the automation root" (A5-O3: case-sensitive past the drive). Leave PhotoCraft open for step 31.

5. Port 0 loses the port. Verifies `03a` #7, O7, A3-F1, A3-O7 (piped output from a release GUI build).
   ```powershell
   Set-Location F:\Script-Projects\ArtCrafting\photocraft
   $p = Start-Process (Exe photocraft) -ArgumentList '--control','0','--control-token-file',(Q "$H\tokens\pc0.token") -RedirectStandardError "$H\pc0-err.txt" -PassThru
   Start-Sleep 8; Get-Content "$H\pc0-err.txt"; Stop-Process -Id $p.Id
   Remove-Item "$H\pc0-err.txt", "$H\tokens\pc0.token" -ErrorAction SilentlyContinue
   ```
   Expected: `photocraft: control server listening on 127.0.0.1:0` (`photocraft` `apps/photocraft/src/control_server.rs:27`). An **empty** file means piped stderr doesn't work for release GUI builds (A3-O7), which changes how the hub reads launch errors.

6. Real filter id spelling. Verifies A4 #8 (Example 8).
   ```powershell
   Set-Location F:\Script-Projects\ArtCrafting\photocraft
   (Get-Content "$H\snapshots\photocraft.catalog.json" -Raw | ConvertFrom-Json) | Where-Object { $_.id -like '*blur*' } | Select-Object id, label
   ```
   Expected: a Gaussian blur id; it replaces `filter.blur.gaussianBlur` in A4 §6.7 Example 8.

**W2. LightCraft**

7. No handshake, error text, port log. Verifies `01` #3, `03a` #4, O2, A4 #2.
   ```powershell
   Set-Location F:\Script-Projects\ArtCrafting\lightcraft
   $lc = Start-Process (Exe lightcraft) -ArgumentList '--control','47602' -RedirectStandardError "$H\lc-err.txt" -PassThru
   Start-Sleep 8
   Send-Craft 47602 @('{"id":1,"method":"engine.commands","params":{}}')
   Send-Craft 47602 @('{"id":2,"method":"engine.execute","params":{"command":"nope.nope","params":{}}}')
   Get-Content "$H\lc-err.txt"
   ```
   Expected: `{"id":1,"ok":true,…` with no `auth` frame; `unknown command `nope.nope`` in id 2; stderr `listening on 127.0.0.1:47602`.

8. Real-app floor for P1. Verifies `01` #5.
   ```powershell
   Set-Location F:\Script-Projects\ArtCrafting\lightcraft
   $t = 1..20 | ForEach-Object { (Measure-Command { Send-Craft 47602 @('{"id":1,"method":"engine.commands","params":{}}') | Out-Null }).TotalMilliseconds }
   ($t | Measure-Object -Average -Minimum -Maximum) | Format-List Average, Minimum, Maximum
   ```
   Expected: well under 50 ms average, including PowerShell's connect cost. Record it next to P1.

9. Headless MCP `--compact` still has `command_run`. Verifies A3 #9. LightCraft holds one process per library (`catalog.lock`), so close the window first.
   ```powershell
   Set-Location F:\Script-Projects\ArtCrafting\lightcraft
   Stop-Process -Id $lc.Id; Start-Sleep 2
   $in = @(
     '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-06-18","capabilities":{},"clientInfo":{"name":"probe","version":"0"}}}',
     '{"jsonrpc":"2.0","method":"notifications/initialized"}',
     '{"jsonrpc":"2.0","id":2,"method":"tools/list","params":{}}') -join "`n"
   (($in | & (Cli lightcraft) mcp --compact) -match 'command_run').Count
   ```
   Expected: 1 or more.

**W3. FilmCraft**

10. No handshake and error texts. Verifies `03a` #4, A4 #1.
    ```powershell
    Set-Location F:\Script-Projects\ArtCrafting\filmcraft
    $fc = Start-Process (Exe filmcraft) -ArgumentList '--control','47603' -PassThru
    Start-Sleep 8
    Send-Craft 47603 @('{"id":1,"method":"engine.commands"}')
    Send-Craft 47603 @('{"id":2,"method":"engine.execute","params":{"command":"nope.nope","params":{}}}')
    Send-Craft 47603 @((J @{ id = 3; method = 'engine.execute'; params = @{ command = 'file.exportOtio'; params = @{ path = "$H\handoff\x.otio" } } }))
    ```
    Expected: catalog reply with no handshake; ``unknown command `nope.nope` ``; ``command `file.exportOtio` is not available right now: no sequence is open``.

11. Catalog keys and `file.import`. Verifies A4 #3, A4-F5, `01` #4 (first half).
    ```powershell
    Set-Location F:\Script-Projects\ArtCrafting\filmcraft
    (Get-Content "$H\snapshots\filmcraft.catalog.json" -Raw | ConvertFrom-Json)[0].PSObject.Properties.Name -join ', '
    & (Cli filmcraft) describe file.import
    ```
    Expected: `id, label, menu, shortcut, shortcuts, defaultShortcut, params, enabled`, with no `journal` or `readOnly`; `file.import` params `{"paths":[str],…}`.

12. OTIO export twice, then re-import. Verifies A4 #5, `01` #4 (second half), A5 #8. First, in the FilmCraft window: import `$LIB\environments\snow_clearing_flythrough.mp4` and put it on a sequence.
    ```powershell
    Set-Location F:\Script-Projects\ArtCrafting\filmcraft
    $o = "$H\handoff\t1\a.otio"; New-Item -ItemType Directory -Force (Split-Path $o) | Out-Null
    $exp = J @{ id = 4; method = 'engine.execute'; params = @{ command = 'file.exportOtio'; params = @{ path = $o } } }
    Send-Craft 47603 @($exp); Send-Craft 47603 @($exp)
    ```
    Expected: the first export is `ok`; the second either overwrites silently (keep `destructive = true` in FilmCraft's effects table) or errors (change the row). Then File → New project in the window, and:
    ```powershell
    Set-Location F:\Script-Projects\ArtCrafting\filmcraft
    Send-Craft 47603 @((J @{ id = 5; method = 'engine.execute'; params = @{ command = 'file.import'; params = @{ paths = @($o) } } }))
    ```
    Expected: same clip count, media online (absolute `file://` URLs, A5 §7.2.5).

13. Sidecar overwrite (FilmCraft + LightCraft). Verifies A5 #4, A5-O4. On a copy only.
    ```powershell
    Set-Location F:\Script-Projects\ArtCrafting\assetcraft
    New-Item -ItemType Directory -Force "$env:TEMP\xmp-test" | Out-Null
    Copy-Item "$LIB\environments\snow_clearing_flythrough.mp4" "$env:TEMP\xmp-test\"
    ```
    Then (a) FilmCraft: import it, set a Description, Clip ▸ Update Metadata; (b) LightCraft: add `%TEMP%\xmp-test` in place, give the clip 4 stars; (c) FilmCraft: Update Metadata again; (d):
    ```powershell
    Set-Location F:\Script-Projects\ArtCrafting\assetcraft
    Select-String -Path "$env:TEMP\xmp-test\snow_clearing_flythrough.xmp" -Pattern 'Rating', 'CreatorTool'
    ```
    Expected (static read): after (b) `xmp:Rating` and `CreatorTool="FilmCraft"`; after (c) the rating is gone. If it survives, close A5-O4 (and U22 in §14.1).

**W4. PdfCraft**

14. Control file, auth, post-auth line cap, stale file. Verifies `01` #2, `03a` #8, A3 #4, A3-F4, A4 #7, A4-F1, A9-N10.
    ```powershell
    Set-Location F:\Script-Projects\ArtCrafting\pdfcraft
    $cf = "$H\pdf.json"
    $pd = Start-Process (Exe pdfcraft) -ArgumentList '--control',(Q $cf) -PassThru
    Start-Sleep 8
    $j = Get-Content $cf -Raw | ConvertFrom-Json; $j
    $pauth = J @{ jsonrpc = '2.0'; id = 1; method = 'auth'; params = @{ token = $j.token } }
    Send-Craft $j.port @($pauth, '{"jsonrpc":"2.0","id":2,"method":"ui.commands","params":{}}') -Read 2
    Send-Craft $j.port @($pauth, ('x' * 8MB), '{"jsonrpc":"2.0","id":3,"method":"ui.state","params":{}}') -Read 3
    Stop-Process -Id $pd.Id; Start-Sleep 2; Test-Path $cf
    ```
    Expected: `{port, token (32 hex), pid}`; `{"result":{"ok":true}}`; a `ui.commands` list; for the 8 MiB line a `-32700` parse error and **then** an answer to id 3 (no cap after auth, A4-F1); finally `True` (file left behind, A3-F4).

15. Headless `doc_open` result shape. Verifies A4 #6.
    ```powershell
    Set-Location F:\Script-Projects\ArtCrafting\pdfcraft
    $in = @(
      '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-06-18","capabilities":{},"clientInfo":{"name":"probe","version":"0"}}}',
      '{"jsonrpc":"2.0","method":"notifications/initialized"}',
      (J @{ jsonrpc = '2.0'; id = 2; method = 'tools/call'; params = @{ name = 'doc_open'; arguments = @{ path = "$LIB\environments\location_breakdown.pdf" } } })) -join "`n"
    $in | & (Cli pdfcraft) mcp
    ```
    Expected: the `id: 2` reply has `isError: false` and `structuredContent` with `doc`, `pages`, `editable`.

**W5. WordCraft**

16. Key file, key auth, unknown method, dirty-document open, key file ACL and removal. Verifies `03a` #5–6, O5, O12, A3 #5–6, A3-O6.
    ```powershell
    Set-Location F:\Script-Projects\ArtCrafting\wordcraft
    $wc = Start-Process (Exe wordcraft) -ArgumentList '--control','47605' -PassThru
    Start-Sleep 8
    $kf = "$env:APPDATA\WordCraft\control-key.local-47605"; $k = (Get-Content $kf -Raw).Trim(); $k.Length
    icacls $kf
    Send-Craft 47605 @((J @{ id = 1; key = $k; method = 'engine.commands'; params = @{} }))
    Send-Craft 47605 @((J @{ id = 2; key = $k; method = 'app.state'; params = @{} }))
    Set-Content "$H\o5.md" '# O5 test'
    ```
    Expected: `64`; ACL lists only you, SYSTEM and Administrators (anything else is an O12 finding); a catalog reply; for `app.state` an "unknown command" style registry error, not "unknown method" (A3-O6). Now type a few words into the open WordCraft document, then:
    ```powershell
    Set-Location F:\Script-Projects\ArtCrafting\wordcraft
    Send-Craft 47605 @((J @{ id = 3; key = $k; method = 'engine.execute'; params = @{ command = 'file.open'; params = @{ path = "$H\o5.md" } } }))
    ```
    Expected (O5): the typed words are gone with no prompt. Close the WordCraft window **by hand**, then `Test-Path $kf` → `False`.

**W6. DesignCraft**

17. Control round trip, no handshake, UI ids. Verifies `03b` #5, `01` #3.
    ```powershell
    Set-Location F:\Script-Projects\ArtCrafting\designcraft
    $dc = Start-Process (Exe designcraft) -ArgumentList '--control','47606' -PassThru
    Start-Sleep 8
    $r = Send-Craft 47606 @('{"id":1,"method":"engine.commands"}') -Max 200000000
    $r.Substring(0, 60); $r -match '"ui":true'; $r -match 'file.exportIdml'
    ```
    Expected: starts `{"id":1,"ok":true,"result":[`; `True`; `True`.

18. Bind failure leaves an app running without control. Verifies A3 #2, A3-F2. Uses VectorCraft as the second app.
    ```powershell
    Set-Location F:\Script-Projects\ArtCrafting\vectorcraft
    $vcx = Start-Process (Exe vectorcraft) -ArgumentList '--control','47606' -RedirectStandardError "$H\vc-err.txt" -PassThru
    Start-Sleep 8; Get-Content "$H\vc-err.txt"; $vcx.HasExited
    (Send-Craft 47606 @('{"id":1,"method":"engine.commands"}') -Max 200000000) -match 'file.exportIdml'
    Stop-Process -Id $vcx.Id
    ```
    Expected: `vectorcraft: control server failed to bind 127.0.0.1:47606: …`; `False` (still running); `True` (the port still answers as DesignCraft).

**W7. EffectCraft**

19. Version over control, bad port, catalog keys. Verifies A3-F6, `03b` bad-port row, A9-N9.
    ```powershell
    Set-Location F:\Script-Projects\ArtCrafting\effectcraft
    $ec = Start-Process (Exe effectcraft) -ArgumentList '--control','47607' -PassThru
    Start-Sleep 8
    Send-Craft 47607 @('{"id":1,"method":"engine.execute","params":{"command":"app.capabilities","params":{}}}') -Max 600
    $e0 = Start-Process (Exe effectcraft) -ArgumentList '--control','0' -PassThru; Start-Sleep 5; $e0.HasExited; if ($e0.HasExited) { $e0.ExitCode } else { Stop-Process -Id $e0.Id }
    (Get-Content "$H\snapshots\effectcraft.catalog.json" -Raw | ConvertFrom-Json)[0].PSObject.Properties.Name -join ', '
    ```
    Expected: a reply that includes a version; `True` and exit code `1`; keys include `why`, not `journal` (U4).

**W8. VectorCraft and the 7979 client default**

20. A client default reaches the wrong app. Verifies A9-N7 (replaces `01` #3's bind expectation), O-B7, A3-F2.
    ```powershell
    Set-Location F:\Script-Projects\ArtCrafting\designcraft
    Stop-Process -Id $dc.Id; Start-Sleep 2
    $dc = Start-Process (Exe designcraft) -ArgumentList '--control','7979' -PassThru
    Start-Sleep 8
    Set-Location F:\Script-Projects\ArtCrafting\soundcraft
    ((& (Cli soundcraft) app engine.commands) -join '') -match 'file.exportIdml'
    ```
    Expected: `True`: SoundCraft's CLI, with no port given, talked to DesignCraft. This is why the hub never trusts a port without the identity check (A3 §5.3 step 4).

21. VectorCraft MCP falls back to headless with nothing on its default address. Verifies A3-F7.
    ```powershell
    Set-Location F:\Script-Projects\ArtCrafting\vectorcraft
    Stop-Process -Id $dc.Id; Start-Sleep 2
    $in = @(
      '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-06-18","capabilities":{},"clientInfo":{"name":"probe","version":"0"}}}',
      '{"jsonrpc":"2.0","method":"notifications/initialized"}',
      '{"jsonrpc":"2.0","id":2,"method":"tools/list","params":{}}') -join "`n"
    (($in | & (Cli vectorcraft) mcp) -match '"tools"').Count
    ```
    Expected: 1 (a tools list from the headless fallback). Restart DesignCraft on 47606 for step 22: `$dc = Start-Process (Exe designcraft) -ArgumentList '--control','47606' -PassThru`.

**W9. SoundCraft**

22. Port 0 through piped stdout, the junk guard, error texts, a real id. Verifies A3 #1, A3-O7, `03b` #6, O-B2, A4 #2, A4 #4.
    ```powershell
    Set-Location F:\Script-Projects\ArtCrafting\soundcraft
    $sc = Start-Process (Exe soundcraft) -ArgumentList '--control','0','--no-audio' -RedirectStandardOutput "$H\sc-out.txt" -PassThru
    Start-Sleep 8; Get-Content "$H\sc-out.txt"
    $port = [int]((Select-String -Path "$H\sc-out.txt" -Pattern 'SOUNDCRAFT_CONTROL_PORT=(\d+)').Matches[0].Groups[1].Value)
    Send-Craft $port @('POST / HTTP/1.1', 'Host: 127.0.0.1', '', '{"id":7,"method":"engine.parity"}') -Read 4
    Send-Craft 47606 @('POST / HTTP/1.1', 'Host: 127.0.0.1', '', '{"id":7,"method":"engine.commands"}') -Read 4
    Send-Craft $port @('{"id":2,"method":"engine.execute","params":{"command":"nope.nope","params":{}}}')
    & (Cli soundcraft) describe session.save_as
    ```
    Expected: a `SOUNDCRAFT_CONTROL_PORT=<n>` line (empty file → A3-O7); SoundCraft: two or more `bad JSON` errors and then `{"id":7,"ok":true,…}` (O-B2 is real; U5 in §14.1); DesignCraft: one error then `<closed>`; ``unknown command `nope.nope` ``; a `session.save_as` entry with params `{path, replace?: false}`. For the disabled-command text, pick a disabled id from `$H\snapshots\soundcraft.catalog.txt` and send it the same way; expected prefix `` `<id>` is not available: ``.

**W10. ArtCraft**

23. No CLI binary. Verifies `03b` #7, O-B11.
    ```powershell
    Set-Location F:\Script-Projects\ArtCrafting\artcraft
    cargo metadata --no-deps --format-version 1 | ConvertFrom-Json | ForEach-Object { $_.packages.targets | Where-Object { $_.kind -contains 'bin' } | Select-Object name }
    ```
    Expected: only `artcraft`, `artcraft-native` and test or library targets.

**W11. AssetCraft and the file system**

24. Long paths enabled? Verifies A5 #1.
    ```powershell
    Set-Location F:\Script-Projects\ArtCrafting
    Get-ItemProperty HKLM:\SYSTEM\CurrentControlSet\Control\FileSystem -Name LongPathsEnabled
    ```
    Expected: `1` or `0`; note it, step 25 depends on it.

25. Long-path fixture and who opens it. Verifies A5 #2–3, A5-O8.
    ```powershell
    Set-Location F:\Script-Projects\ArtCrafting\assetcraft
    $deep = "\\?\F:\Script-Projects\ArtCrafting\lp\" + (("a" * 50 + "\") * 5) + "Réaper ünïcode name.png"
    New-Item -ItemType Directory -Force -Path (Split-Path $deep) | Out-Null
    Copy-Item -LiteralPath "$LIB\edge-cases\Réaper ünïcode name.png" -Destination $deep
    $plain = $deep.Substring(4); $plain.Length
    cargo run -q -p assetcraft-cli -- probe "$plain" --json
    & (Cli lightcraft) render "$plain" -o "$env:TEMP\lp-lightcraft.png" --size 256
    & (Cli filmcraft) probe "$plain"
    ```
    Expected: a length over 300; all three succeed (static guess). Any failure → `long_paths: refuse` for that app (A5-O8, U25).

26. Two spellings of one file. Verifies A5 #7.
    ```powershell
    Set-Location F:\Script-Projects\ArtCrafting\assetcraft
    subst X: "$AC\test_data"
    cargo run -q -p assetcraft-cli -- probe X:\library\environments\snow_clearing_plate.jpg --json
    subst X: /D
    ```
    Expected: the probe succeeds (the hub will see `X:\…` and `F:\…` as two path keys; hash resolution reconciles them).

27. BLAKE3 cost baseline. Verifies A5 #5, feeds P6.
    ```powershell
    Set-Location F:\Script-Projects\ArtCrafting
    cargo install b3sum --locked
    $big = '<path to your largest local video>'
    (Get-Item $big).Length / 1GB
    Measure-Command { b3sum --num-threads 1 $big } | Select-Object TotalSeconds
    Measure-Command { b3sum $big } | Select-Object TotalSeconds
    ```
    Expected: single-thread time ≈ size ÷ disk speed. (`b3sum` is the default, SIMD build; the hub's `pure` build may be a little slower on CPU-bound runs, A9-F7.)

**W12. Cross-app, while apps are running**

28. Process probe sees argv. Verifies A3 #3, A3-R2. PhotoCraft (step 4), FilmCraft (10), DesignCraft (21), EffectCraft (19) and SoundCraft (22) should still be open.
    ```powershell
    Set-Location F:\Script-Projects\ArtCrafting
    Get-CimInstance Win32_Process -Filter "Name like '%craft.exe'" | Select-Object ProcessId, Name, CommandLine | Format-Table -Wrap
    ```
    Expected: each command line includes its `--control …` arguments, PhotoCraft's `--control-token-file <path>` too.

29. Folder and token ACLs. Verifies `03a` #6, O12, A3 #7.
    ```powershell
    Set-Location F:\Script-Projects\ArtCrafting
    icacls "$env:LOCALAPPDATA\CraftApps\run"
    icacls "$H\tokens\photocraft-47601.token"
    ```
    Expected: only your user, SYSTEM and Administrators.

30. CLI version equals GUI version in one install. Verifies A3 #8. Only for apps installed from their MSI.
    ```powershell
    Set-Location 'C:\Program Files'
    foreach ($a in 'PhotoCraft','LightCraft','FilmCraft','PdfCraft','WordCraft','DesignCraft','EffectCraft','VectorCraft','SoundCraft') {
      $d = "C:\Program Files\$a"; $l = $a.ToLower()
      if (Test-Path "$d\$l-cli.exe") { '{0,-12} cli {1}  gui {2}' -f $a, (& "$d\$l-cli.exe" --version), (Get-Item "$d\$l.exe").VersionInfo.ProductVersion }
    }
    ```
    Expected: the same version twice per app. (R4's Windows check, `Get-Process craft-fake-app` after killing the hub, joins this group at hub Stage 2, when the fake app exists.)

31. Clean up.
    ```powershell
    Set-Location F:\Script-Projects\ArtCrafting
    Get-Process photocraft, filmcraft, designcraft, effectcraft, soundcraft -ErrorAction SilentlyContinue | Stop-Process
    Remove-Item "$H\tokens\*", "$H\pdf.json", "$H\o5.md", "$H\*-err.txt", "$H\sc-out.txt" -ErrorAction SilentlyContinue
    ```
    Expected: no craft windows left; `$H\snapshots\` kept. Send Claude the output of steps 3–30 (or just the lines that differ from "Expected").

---

## 14. Upstream and fork strategy

**Rule (from A3 §4.2 and `01` NG8):** the hub never waits on upstream. Every row below has a hub-side fallback that already exists in the spec; a PR only makes a quirk go away, and the adapter config's version-ranged override (A3-O9) switches the quirk off for app versions that include the fix.

### 14.1 Consolidated PR candidates

All upstream asks from `01`, `03a`, `03b`, A3, A4 and A5, merged where two IDs ask for the same change, renumbered `U#`. A8's `S#` items were not available; the security rows here (U5, U6, U28) are placeholders for that category and the integrator should merge A8's IDs into them. **Value:** H = removes a whole adapter kind of quirk or a safety risk for every hub user; M = removes one app's quirk; L = nice to have. **Size:** XS ≤ 20 lines + a test; S ≤ 150; M ≤ 600; L larger or cross-repo. Ranked by value, then size.

| Rank | ID  | Source IDs                  | Target repo(s)                                                                                                                                                            | PR title (one line)                                                                      | Value           | Size       | Depends on                                               | Hub fallback if rejected                                                                                |
|------|-----|-----------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------|-----------------|------------|----------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| 1    | U1  | O7, A3-O4, A3-F1/F2         | photocraft, lightcraft, filmcraft, designcraft, vectorcraft (+ exit rule in effectcraft, wordcraft)                                                                       | control: log the bound address and exit non-zero when `--control` can't bind             | H               | XS per app | none                                                     | Explicit ports from `port_range`, identity check, one retry (A3 §5.3)                                   |
| 2    | U2  | O-B3, A9-N6                 | designcraft (`apps/designcraft-cli/src/main.rs:104-115`), soundcraft, vectorcraft                                                                                         | cli: `commands --json` prints the full catalog (accept `--json`)                         | H               | XS         | none                                                     | Per-app `catalog_argv` in adapter config (A3 §4.4)                                                      |
| 3    | U4  | O4, A1-O4, A9-N9            | effectcraft (`crates/engine/src/commands/query.rs:21`), photocraft (`crates/engine/src/commands.rs:146-149`), lightcraft, filmcraft, designcraft, vectorcraft, soundcraft | engine: include each command's `journal` flag in `command.list` / `engine.commands`      | H               | XS per app | none                                                     | Curated `effects.toml` + `craftctl profile seed` from source (A4-F5)                                    |
| 4    | U3  | O3, O-B7, A1-O1, A3-O1      | designcraft first, then every app                                                                                                                                         | control: publish a manifest file `<run dir>/<app>-<pid>.json` while `--control` is on    | H               | S          | Run-dir name agreed (A3 open question 1); U1 (real port) | Launch-and-remember + process probe + app artefacts (A3-R2)                                             |
| 5    | U5  | O-B2 (+ A8 S# category)     | soundcraft (`apps/soundcraft/src/control_server.rs:97-137`)                                                                                                               | control: close the connection on a non-request line, like the other apps                 | H               | S          | Paul confirms with §13.5 step 22                         | Hub launches SoundCraft only headless (`mcp-stdio`) and never with `--control`; warn in `craftctl apps` |
| 6    | U6  | O2 (+ A8 S# category)       | lightcraft, filmcraft first; then designcraft, effectcraft, vectorcraft, soundcraft                                                                                       | control: optional token handshake (`--control-token-file`, `auth` first frame)           | H               | S–M        | Best after U3 (manifest carries `token_file`)            | Attach policy `probed-or-launched` (A3 §5.3)                                                            |
| 7    | U7  | O5                          | wordcraft                                                                                                                                                                 | control: route `file.open` through the unsaved-changes guard, as `file.close` is         | M               | XS–S       | none                                                     | Effects row `destructive`; refuse while `document.inspect` reports dirty unless confirmed               |
| 8    | U8  | A3-O6                       | wordcraft                                                                                                                                                                 | control: answer `unknown method` for names outside the method table                      | M               | XS         | none                                                     | Config `unknown_method = "runs-as-command"`; never send invented methods                                |
| 9    | U9  | A3-O5                       | pdfcraft                                                                                                                                                                  | control: remove the control file on clean exit                                           | M               | XS         | none                                                     | Trust the file only with a live `pdfcraft` pid (A3 §5.4)                                                |
| 10   | U10 | A4-F1                       | pdfcraft (`crates/ui-egui/src/control.rs:808`)                                                                                                                            | control: cap request lines after auth (4 MiB, as the suite does)                         | M               | XS         | none                                                     | Adapter never sends more than its cap and caps its reads                                                |
| 11   | U11 | O10                         | pdfcraft (`apps/pdfcraft-cli/src/main.rs:743`)                                                                                                                            | cli: include `destructive` in `tools` output                                             | M               | XS         | none                                                     | Read hints from MCP `command_list` annotations                                                          |
| 12   | U21 | A5-O3                       | photocraft (`workspace.rs:157-172`)                                                                                                                                       | automation: compare root paths case-insensitively on Windows and accept `\\?\`           | M               | XS–S       | none                                                     | Adapter always sends the root-relative forward-slash form                                               |
| 13   | U15 | A3-O3                       | every app (EffectCraft `app.capabilities` is the model)                                                                                                                   | control: read-only `app.hello` → `{app, version, pid, instance, protocol, capabilities}` | M               | S          | Best with U3                                             | Identity-probe ids + `<cli> --version` (A3 §5.6)                                                        |
| 14   | U14 | A4-F3                       | lightcraft, filmcraft, designcraft, effectcraft, soundcraft, wordcraft                                                                                                    | control: say whether a timed-out request ran (withdrawal, two timeout texts)             | M               | S          | none                                                     | Treat bare `timeout` as `ran: maybe`; retry reads only                                                  |
| 15   | U13 | A4-F2                       | every app (via `craft-control-proto` if U16 lands)                                                                                                                        | control: add a stable `errorCode` next to `error`                                        | M               | S per app  | Easier after U16                                         | Prefix table per profile + contract tests                                                               |
| 16   | U22 | A5-O4                       | filmcraft                                                                                                                                                                 | metadata: merge sidecars with `lightcraft-meta::merge_xmp` instead of replacing them     | M               | S–M        | Paul reproduces (§13.5 step 13)                          | Effects row `write-user-file` + `destructive`, always confirm                                           |
| 17   | U12 | O6                          | pdfcraft                                                                                                                                                                  | control: `engine.execute {command, params}` alias that runs the linked tool              | M               | M          | none                                                     | Route PdfCraft writes through `pdfcraft-cli mcp` (A3 §4.4)                                              |
| 18   | U19 | O-B4                        | designcraft, soundcraft (wordcraft)                                                                                                                                       | mcp: add the suite tool set (`command_list`, `command_run`, `command_batch`, …)          | M               | M          | none                                                     | `generic-execute` tool style in adapter config                                                          |
| 19   | U20 | O14                         | vectorcraft or effectcraft first                                                                                                                                          | engine: suite command `thumbnail.render {path, maxSide, out}`                            | M               | S–M        | AssetCraft defines it first (Stage 1.5 step 6)           | Per-app thumbnail recipe in adapter config                                                              |
| 20   | U23 | A5-O5                       | filmcraft (`otio.rs:678`)                                                                                                                                                 | interchange: keep unknown OTIO `metadata.*` namespaces on import and write them back     | M               | M          | none                                                     | Handoff ledger keeps clip → AssetRef (A5 §7.4.6)                                                        |
| 21   | U25 | A5-O8                       | any app that fails §13.5 step 25                                                                                                                                          | windows: declare `longPathAware` in the app manifest                                     | M               | XS         | Paul's long-path run                                     | Adapter `long_paths: refuse` → typed `path_too_long_for_app`                                            |
| 22   | U28 | O12 (+ A8 S# category)      | photocraft, pdfcraft, wordcraft                                                                                                                                           | security: user-only ACL on token/key files on Windows                                    | M               | S–M        | Paul's ACL run (§13.5 steps 16, 29)                      | Hub keeps its own secrets in its user-only dir or the keyring (A8)                                      |
| 23   | U17 | O-B5                        | every app                                                                                                                                                                 | engine: an `effects` field on `CommandSpec`                                              | H               | M per app  | U4 accepted first; AssetCraft Stage 1.5 is the reference | Curated tables (A4 §6.4.3)                                                                              |
| 24   | U16 | O-B1, A3-R1 promotion 1     | craft-libs (+ one adopting app)                                                                                                                                           | craft-control: shared envelope, guarded server, client and manifest writer               | H               | L          | AssetCraft Stage 1.5 and hub Stage 0 prove it            | Crates stay in AssetCraft / the hub repo; apps keep their copies                                        |
| 25   | U18 | O8, O-B13, A1-O5            | every app                                                                                                                                                                 | control: opt-in `events.subscribe`                                                       | H               | M–L        | Hub Stage 3 proves the event shape                       | Polling bridges (A3 §4.1)                                                                               |
| 26   | U27 | A4-F9                       | every app                                                                                                                                                                 | engine: expose `app.open`-style control methods as registry commands                     | L               | M          | none                                                     | `kind` in the address (`method:`)                                                                       |
| 27   | U24 | A5-O6                       | filmcraft (`otio.rs:696`)                                                                                                                                                 | interchange: de-duplicate media by a case-folded path on Windows                         | L               | XS         | none                                                     | AssetCraft's OTIO writer emits one spelling per file                                                    |
| 28   | U26 | `assetcraft` `STATUS.md:63` | lightcraft                                                                                                                                                                | deps: bump `quick-xml` to 0.41 (RUSTSEC-2026-0194, -0195)                                | L (for the hub) | XS         | none                                                     | AssetCraft keeps its pin; exposure is a slow thumbnail on a hostile file                                |

Not PR candidates (hub-side only, by design): `03a` O1 (two codecs), O9 (catalog normalization), O11 (bridge flag names), O13 (PhotoCraft path transform), `03b` O-B6 (typed params, long term), O-B8 (capability aliases), O-B11/O-B12 (ArtCraft stays out). Later suite standard, no ask in v1: A5-O11 (`craft-assetref` hash recipe to craft-libs).

### 14.2 What stays hub-side if upstream says no

Nothing in v1 depends on a PR. The table's last column is the permanent answer if a PR is declined; the cost is only more adapter config and more contract-test rows. The two rows where a "no" leaves a real gap are both safety items, and the hub handles them by **not using** the risky surface:

- **U5 declined:** SoundCraft is only driven headless. Paul is told in `craftctl apps` that `--control` on SoundCraft accepts lines after junk.
- **U6 declined:** unauthenticated apps are attached only when the hub launched them or the process probe saw them (A3 §5.3), never from a port in user config. A8 owns the final policy.

### 14.3 The first three PRs

Project note (D6): test the waters with one small PR first, such as the manifest file. **Recommendation A9-R2:** keep that intent, but send the manifest as the **second** PR, because it carries a naming decision upstream has to make (the run-dir name, A3 open question 1) while the first PR should carry none. All three go to the two forks already planned (DesignCraft, EffectCraft, project context §1), whose control code is near-identical (`03b` §3.6b), so one review style is learned once.

| Order | PR                                                                      | Repo        | Size | Why this one now                                                                                                                                                                                                                                                                 | What "accepted" teaches                                          |
|-------|-------------------------------------------------------------------------|-------------|------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------|
| 1     | U2 `cli: accept --json on commands (suite convention)`                  | designcraft | XS   | A pure convention fix with no design debate: `commands` already prints JSON, and `--json` is being read as a filter (A9-N6). It unblocks the contract snapshot for DesignCraft. Send it with an **issue** proposing U3 so the name discussion starts in parallel                 | Review latency, CI expectations, whether outside PRs are welcome |
| 2     | U3 + U1 `control: report the bound port and publish a control manifest` | designcraft | S    | The D6 test proper. Scoped small: write `{manifest, app, app_version, pid, port, codec, protocol}` (A3 §5.2 subset, no auth fields since DesignCraft has none) by stage + rename when the listener is up, log `local_addr()`, delete on clean exit. One test with a temp run dir | Appetite for suite standards; who decides the run-dir name       |
| 3     | U4 `engine: include journal in command.list`                            | effectcraft | XS   | One field in one `json!` (`crates/engine/src/commands/query.rs:21`); it is the first step toward real command effects (U17) and removes guesswork from the effects table                                                                                                         | Whether upstream wants effects metadata at all (sets U17)        |

If PR 1 gets no response in two weeks, stop and ask in the issue rather than sending PR 2 (owner Paul). If PR 2 is declined, nothing changes in the hub (fallback row U3).

### 14.4 Fork workflow

**Remotes.** `origin` = `github.com/4waymedia/<repo>` (Paul's fork), `upstream` = `github.com/storytold/<repo>` with **push disabled** (the same guard as the superseded ArtCraft clone, `assetcraft` `STATUS.md:203-205`). `main` is only ever fast-forwarded from `upstream/main`; nothing is committed on it.

**Branches.**

| Pattern             | Use                                                                                                                                | Goes upstream? |
|---------------------|------------------------------------------------------------------------------------------------------------------------------------|----------------|
| `main`              | Mirror of `upstream/main`                                                                                                          | n/a            |
| `pr/<area>-<topic>` | One PR, one feature, branched from `upstream/main` (e.g. `pr/cli-commands-json`, `pr/control-manifest`, `pr/command-list-journal`) | yes            |
| `paul/<topic>`      | Personal experiments and notes                                                                                                     | never          |

**Following upstream first.** Each fork's own `AGENTS.md` / `CLAUDE.md` / `CONTRIBUTING.md` wins inside that repo: its `cargo xtask ci` (FilmCraft also has `assets` and `wasm` gates, `filmcraft` `docs/contributing.md:113-123`), its commit style (task id or area prefix, A9-N8, **not** AssetCraft's `Stage N step M:`), and its PR template where one exists (PdfCraft, VectorCraft `.github/pull_request_template.md`). Claude reads those files before writing the patch (owner Claude). Upstream asks AI agents to end commit messages with their attribution line (`filmcraft` `docs/contributing.md:148`); the blocks below add only the `Co-Authored-By` line, because a session link is personal and stays out of upstream history.

**Keeping personal notes out.** Notes about the hub live in the AssetCraft repo (`docs/pipelinecraft/`, `docs/upstream/`), never in a fork. In each fork, local-only files are excluded through `.git/info/exclude` (never committed), and every PR branch is checked with `git diff --stat upstream/main...HEAD` before pushing: only the files the PR is about may appear.

**Set up the two forks for the first three PRs** (DesignCraft, EffectCraft). If step 1 of §13.5 already cloned them from storytold, the second block converts those clones instead.

```powershell
Set-Location F:\Script-Projects\ArtCrafting
gh auth status
foreach ($r in 'designcraft', 'effectcraft') {
  gh repo fork "storytold/$r" --clone=false
  if (-not (Test-Path ".\$r")) { git clone -c core.autocrlf=false "https://github.com/4waymedia/$r.git" $r }
  Push-Location ".\$r"
  if (-not ((git remote) -contains 'upstream')) { git remote add upstream "https://github.com/storytold/$r.git" }
  git remote set-url --push upstream DISABLED
  git fetch upstream
  git switch main
  git merge --ff-only upstream/main
  git push origin main
  Add-Content .git\info\exclude "`n# personal, never committed`n/NOTES.local.md`n/.paul/"
  git remote -v
  Pop-Location
}
```

Expected: `gh` reports each fork (created or already existing); `git remote -v` shows `origin` → `4waymedia/<repo>` (fetch and push) and `upstream` → `storytold/<repo>` (fetch) / `DISABLED` (push).

Convert an existing storytold clone (only if `git remote -v` shows `origin` → `storytold`):

```powershell
Set-Location F:\Script-Projects\ArtCrafting\designcraft
git remote rename origin upstream
git remote add origin https://github.com/4waymedia/designcraft.git
git remote set-url --push upstream DISABLED
git fetch --all
git switch main
git branch --set-upstream-to=origin/main main
git merge --ff-only upstream/main
git push origin main
```

Every other PR target later (same block, other names):

```powershell
Set-Location F:\Script-Projects\ArtCrafting
foreach ($r in 'photocraft', 'lightcraft', 'filmcraft', 'pdfcraft', 'wordcraft', 'vectorcraft', 'soundcraft', 'craft-libs') {
  gh repo fork "storytold/$r" --clone=false
  if (-not (Test-Path ".\$r")) { git clone -c core.autocrlf=false "https://github.com/4waymedia/$r.git" $r }
  Push-Location ".\$r"
  if (-not ((git remote) -contains 'upstream')) { git remote add upstream "https://github.com/storytold/$r.git" }
  git remote set-url --push upstream DISABLED
  git fetch upstream; git switch main; git merge --ff-only upstream/main; git push origin main
  Add-Content .git\info\exclude "`n# personal, never committed`n/NOTES.local.md`n/.paul/"
  Pop-Location
}
```

**One PR, start to finish** (PR 1 as the example; Claude supplies the patch as a file, Paul applies and commits):

```powershell
Set-Location F:\Script-Projects\ArtCrafting\designcraft
git fetch upstream
git switch -c pr/cli-commands-json upstream/main
git apply F:\Script-Projects\ArtCrafting\assetcraft\docs\upstream\designcraft-cli-commands-json.patch
cargo xtask ci
git diff --stat upstream/main...HEAD
git add apps/designcraft-cli/src/main.rs
git commit -m "cli: accept --json on commands (suite convention)" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
git push -u origin pr/cli-commands-json
gh pr create --repo storytold/designcraft --base main --head 4waymedia:pr/cli-commands-json --fill
```

Expected: `cargo xtask ci` passes; the diff stat lists only `apps/designcraft-cli/src/main.rs` (and its test file, if the patch adds one, which then also goes on the `git add` line); `gh` prints the PR URL. Keep the branch current with `git fetch upstream; git rebase upstream/main; git push --force-with-lease` when upstream asks (PdfCraft's template warns that `main` moves fast, `.github/pull_request_template.md:4-6`).

Weekly sync of every fork:

```powershell
Set-Location F:\Script-Projects\ArtCrafting
Get-ChildItem -Directory | Where-Object { Test-Path "$($_.FullName)\.git" } | ForEach-Object {
  Push-Location $_.FullName
  if ((git remote) -contains 'upstream') { git fetch upstream; git switch main; git merge --ff-only upstream/main; git push origin main }
  Pop-Location
}
```

---

## 16. Stages

### 16.0 Rules across all hub stages

**Commit messages.** AssetCraft's convention is `Stage N step M: <what changed>` (project context §4; `assetcraft` `CLAUDE.md:20` shows the shorter `Stage N: <what changed>`). **A9-F4:** while the hub incubates in the AssetCraft workspace, "Stage 1 step 2" would be ambiguous between AssetCraft and the hub. Proposal: hub work uses `Hub Stage N step M: <what changed>`; AssetCraft keeps `Stage N step M:`. After promotion to its own repo (A3-R1, at hub Stage 2), the hub repo drops the `Hub` prefix. Fork PRs follow the upstream repo's convention instead (§14.4).

```powershell
Set-Location F:\Script-Projects\ArtCrafting\assetcraft
git add -A
git commit -m "Hub Stage 0 step 4: craft-hub-discovery config list and run-dir manifests"
git push
```

**Stage gate on command effects (AGENTS rule 5 for the hub).** The router refuses, with `forbidden` (A4 §6.5.2), any call whose resolved command effects go beyond what the current hub stage allows. The gate is a constant in `craft-hub-core` with one test per stage, so a later stage can't loosen it by accident.

| Hub stage  | Command effects the hub will forward                                                                                                                                                   | Hub's own writes                                                |
|------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------|
| 0 See      | catalog reads only (`engine.commands`, `commands --json`, `--version`)                                                                                                                 | Its state dir: `hub.json`, `hub.lock`, logs, manifests it wrote |
| 1 Ask      | `read`                                                                                                                                                                                 | Same                                                            |
| 2 Hand off | `read`, `write-app-state` (opening files in an app); `write-user-file` only when every output path is inside the hub's `derived\`, `handoff\` or `tmp\` dirs (A5 §7.5)                 | Plus `tokens\`, `handoff\`, `derived\`, hash index              |
| 3 Chain    | Plus `write-user-file` anywhere, only after a dry run of the same batch on this connection and a confirmation token (A4 §6.5.2 `confirmation_required`); `destructive` always confirms | Plus audit log (A8 §11.5)                                       |
| 4 MCP      | Same as 3, with MCP-side confirmation (§10.3, A7)                                                                                                                                      | Same                                                            |
| 5 Produce  | Same as 3                                                                                                                                                                              | Same                                                            |

`external-network` is never forwarded in v1 (no local app needs it, A4 §6.4.1).

**Dependencies on AssetCraft stages.**

```text
AssetCraft:  1 ✅ ──► 1.5 API ──► 2 Organize ──► 3 Structure ──► 3.5 ──► 4 Plan A+B ──► 5 ──► 6 Execute
                       │  ▲            │                                  │                    ▲
                       │  │            │ (write rules proven)             │ (OTIO writer,      │ (recipes use
                       ▼  │            ▼                                  ▼  boards)            │  the hub)
Hub:                 0 See ─► 1 Ask ─► 2 Hand off ─► 3 Chain ─► 4 MCP ─► 5 Produce ──────────┘
                     (shares craft-control-* with 1.5; Open In v2 lands in AssetCraft at hub 2)
```

| Hub stage  | Needs from AssetCraft                                                                                     | Gives AssetCraft                                                                      |
|------------|-----------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------|
| 0 See      | Stage 1.5 steps 1–3 (`effects` on `CommandSpec`, full `commands --json`, `craft-control-proto`/`-server`) | One app discovery shared with Open In (A3-O8)                                         |
| 1 Ask      | Stage 1.5 complete (AssetCraft is a hub app with `--control`)                                             | Read calls into siblings from the L4 `bridge`                                         |
| 2 Hand off | Stage 1.5 `asset.ref`, `thumb.render`; L4 `bridge` crate                                                  | Open In v2 into running windows (O-B10); thumbnails for formats it can't decode (AC2) |
| 3 Chain    | Stage 2 done (its atomic write, dry-run and undo rules are the model the hub's write gate follows)        | Jobs and events Stage 6 recipes build on                                              |
| 4 MCP      | none new                                                                                                  | One MCP server for the whole suite                                                    |
| 5 Produce  | Stage 4 (Storyline A + B, L3 `boards` + `interchange`, OTIO export)                                       | AC3: storyline → FilmCraft timeline                                                   |

**A9-F5 (schedule tension).** The roadmap puts the hub after AssetCraft and StoryCraft (project context §7), while A3-R1 starts the hub's crates together with AssetCraft Stage 1.5 because the control code is the same. This plan resolves it by keeping hub Stages 0–1 **small and read-only**: they fall out of the Stage 1.5 crates in weeks, not months, and give Open In and recipes one discovery. Hub Stages 2+ wait for their AssetCraft prerequisite in the table above. Paul decides (open question 2).

### 16.1 Hub Stage 0: See

Discovery, health and `craftctl apps`, read-only. Exit of this stage is G1 (`01` §1.2) on the fake apps and on Paul's real apps.

| Step | Work                                                                                                                                                                                                             | Crates touched                               |
|------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------|
| 1    | `TABLE` rows (A3 §4.3 + §13.2.1) and the `ISLANDS` rule with its tests (A3-O2)                                                                                                                                   | `xtask`                                      |
| 2    | `craft-control-proto`: add `Manifest` (A3 §5.2 schema v1), `AppDescriptor`, normalized `CommandInfo` (`03a` O9), bounded line reader (on top of what AssetCraft Stage 1.5 put there)                             | `craft-control-proto`                        |
| 3    | `craft-hub-testkit`: fake app `control-jsonl` (three auth modes), `control-jsonrpc`, `cli-headless`; snapshot store with the catalogs from §13.5 step 3; `test_data/make_hub_fixtures.py`                        | `craft-hub-testkit`, `test_data`             |
| 4    | `craft-hub-discovery`: config list (locator lifted from `open_in.rs`, A3-O8, with AssetCraft's engine switched to it), run-dir manifests, WordCraft key files, PdfCraft control files, process probe (`sysinfo`) | `craft-hub-discovery`, `engine` (AssetCraft) |
| 5    | `craft-hub-adapters`: catalog fetch per adapter kind, identity probe, version read (A3 §5.6)                                                                                                                     | `craft-hub-adapters`, `craft-control-client` |
| 6    | `craft-hub-core`: attach (no GUI launch yet), registry merge, health (1 s pid, 5 s pass), stale cleanup (A3 §5.4), `hub.lock`, stage gate constant (§16.0)                                                       | `craft-hub-core`                             |
| 7    | `pipelinecraft serve`: listener through `craft-control-server`, `hub.json`, methods below                                                                                                                        | `pipelinecraft`, `craft-control-server`      |
| 8    | `craftctl apps [--json] [--refresh]`, `craftctl commands <app>` (A7 owns the output design)                                                                                                                      | `craftctl`                                   |
| 9    | Tests, perf P2/P3 baselines, `STATUS.md`                                                                                                                                                                         | all of the above                             |

- **Methods delivered:** `hub.hello`, `hub.ping`, `hub.apps`, `hub.commands`, compatibility `engine.commands` (A4 §6.2.1).
- **Acceptance:** fake-app tests for attach (three auth kinds + JSON-RPC), identity mismatch, port-0 kinds, stale manifest (R3) and hostile frames at the listener (R2) pass in `cargo xtask ci`; on Windows, `craftctl apps` lists every installed app with its state (`ok` / `untested` / `unsupported` / `unreachable` / `running, not controllable`, A3 "Inputs" for A7), finds a LightCraft that Paul started by hand with `--control 47602`, and shows a PdfCraft started with a control file in the hub dir; P2 and P3 recorded; nothing written outside `%LOCALAPPDATA%\PipelineCraft\`.
- **Depends on:** AssetCraft Stage 1.5 steps 1–3.

### 16.2 Hub Stage 1: Ask

Read-only calls into any app through the hub.

| Step | Work                                                                                                                                                   | Crates touched                               |
|------|--------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------|
| 1    | Both codecs and all auth strategies on the call path (`03a` O1, O2); per-instance connection pool 2–4 (A4 §6.1.4); deadlines and `ran` (A4 §6.5)       | `craft-control-client`, `craft-hub-adapters` |
| 2    | Target grammar, id folding and collision check (A4 §6.2.2–6.2.3)                                                                                       | `craft-hub-core`                             |
| 3    | Profiles: `profiles/<app>/{profile.toml, effects.toml}` for all nine apps + AssetCraft, seeded by `craftctl profile seed` (A4-F5) and reviewed by Paul | `craft-hub-adapters`                         |
| 4    | Error mapping tables (A4 §6.5.3) with the recorded error strings as contract fixtures                                                                  | `craft-hub-adapters`                         |
| 5    | `cli-headless` and `mcp-stdio` call paths for reads (PdfCraft typed tools, `03a` O6); fake app `mcp-stdio` mode                                        | `craft-hub-adapters`, `craft-hub-testkit`    |
| 6    | Router with the stage gate set to `read`                                                                                                               | `craft-hub-core`                             |
| 7    | `craftctl call`, `craftctl describe`, `craftctl profile seed\|check`                                                                                   | `craftctl`                                   |
| 8    | Golden transcripts for A4 §6.7 Examples 1, 4, 5, 6                                                                                                     | `craft-hub-testkit`                          |

- **Methods delivered:** `hub.call` and `hub.batch` (reads only, `dryRun` accepted), `hub.describe`, `hub.route`, compatibility `engine.execute`.
- **Acceptance:** one real read per app on Windows through `craftctl call` (e.g. `filmcraft/command.list`, `photocraft/method:jobs.list`, `pdfcraft/tool:doc_info`, `soundcraft/transport.status`); a write command returns `forbidden`; an unknown command returns `unknown_command` with suggestions; R1 (`kill_mid_call`) passes; P1 recorded from the fake and compared to §13.5 step 8; contract tests green for every snapshot.
- **Depends on:** AssetCraft Stage 1.5 complete.

### 16.3 Hub Stage 2: Hand off

Asset refs, Open In and thumbnails (§9.1–9.2, AC1, AC2). **Deviation from the outline (A9-F6):** AC2 needs a cancellable job ("Cancelling the job stops the app-side work", `01` AC2b), so a one-step job manager moves here from Stage 3; Stage 3 adds events, pollers and chaining on top.

| Step | Work                                                                                                                                                    | Crates touched                                                               |
|------|---------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------|
| 1    | `craft-assetref` (L0): `WirePath`, `PathKey`, `AssetRef`, `q1`/`full` BLAKE3 (`pure`), hash index (A5 §7.1, §7.6)                                       | `craft-assetref`                                                             |
| 2    | GUI launch (A3 §5.3): port pick, token files, PdfCraft control file, piped output, identity retry; headless children in a job object (R4)               | `craft-hub-core`                                                             |
| 3    | Per-adapter path transforms and roots (`03a` O13, A5 §7.4.4)                                                                                            | `craft-hub-adapters`                                                         |
| 4    | Capability aliases `asset.open`, `asset.thumbnail` (`03b` O-B8) and per-app thumbnail recipes (`03a` O14)                                               | `craft-hub-adapters`                                                         |
| 5    | One-step jobs: `hub.job.get`, `hub.job.wait`, `hub.job.cancel` (A4 §6.2.1) over app jobs, blocking calls and CLI children                               | `craft-hub-core`                                                             |
| 6    | Stage gate: `write-app-state` and hub-dir `write-user-file` (§16.0); handoff ledger (A5 §7.4.6); derived/temp GC as a hub job (A5 §7.5)                 | `craft-hub-core`                                                             |
| 7    | AssetCraft L4 `bridge`: Open In v2 through the hub when it runs, today's launch when it doesn't (O-B10); thumbnails for formats AssetCraft can't decode | AssetCraft `bridge`, `engine`, `thumbs`                                      |
| 8    | Promotion 2 (A3-R1): move `craft-hub-*`, `pipelinecraft`, `craftctl` to their own repo named after D1; AssetCraft depends on it by pinned rev           | new repo; AssetCraft `Cargo.toml`, `deny.toml` `allow-org` (add `4waymedia`) |

- **Methods delivered:** `hub.app.launch`, `hub.app.stop`, `hub.job.get|wait|cancel`; capability routing in `hub.call`.
- **Acceptance:** AC1 a–d and AC2 a–d (`01` §1.4) on Windows; P5 and P6 recorded; R4 checked on Windows (`Get-Process craft-fake-app` after killing the hub); `cargo xtask ci` green in both repos after the move.
- **Depends on:** AssetCraft Stage 1.5 `asset.ref` and `thumb.render`.

### 16.4 Hub Stage 3: Chain

Jobs with progress, events and recipes with a dry run (§8, §9.4).

| Step | Work                                                                                                                            | Crates touched                         |
|------|---------------------------------------------------------------------------------------------------------------------------------|----------------------------------------|
| 1    | Event bus, `hub.subscribe` / `hub.unsubscribe`, event frames with `seq`, bounded per-subscriber queues (A4 §6.2.6; payloads A6) | `craft-hub-core`                       |
| 2    | Polling bridges per app (A3 §4.1 table), MCP progress from `mcp-stdio` children                                                 | `craft-hub-adapters`, `craft-hub-core` |
| 3    | Full job manager: `hub.job.list`, progress, timeouts, app job id mapping (A6 §8.3)                                              | `craft-hub-core`                       |
| 4    | `hub.batch` chaining (`as` / `from`), dry run, confirmation tokens; stage gate opens `write-user-file` behind them              | `craft-hub-core`                       |
| 5    | Recipe file format (A1-O6 superset of PhotoCraft's `{command, params}`) and runner                                              | `craft-hub-core`                       |
| 6    | Audit log for write and destructive calls (A8 §11.5)                                                                            | `craft-hub-core`                       |
| 7    | `craftctl watch`, `craftctl run <recipe> --dry-run` (A7)                                                                        | `craftctl`                             |
| 8    | Start the R5 soak                                                                                                               | none                                   |

- **Methods delivered:** `hub.subscribe`, `hub.unsubscribe`, `hub.job.list`; `dryRun` and `confirm` on `hub.call` / `hub.batch` for writes.
- **Acceptance:** P4 on the fake; a three-step recipe from `test_data/hub/recipes/` runs dry (writes nothing: the test hashes the output folder before and after) and then for real after confirmation; a polled FilmCraft export reports `job.progress` and `job.done`; every write appears in the audit log.
- **Depends on:** AssetCraft Stage 2.

### 16.5 Hub Stage 4: MCP

The MCP front door (§10, A7).

| Step | Work                                                                                                                            | Crates touched      |
|------|---------------------------------------------------------------------------------------------------------------------------------|---------------------|
| 1    | `craft-hub-mcp` stdio shim (`pipelinecraft mcp`), a hub client with `client.kind = "mcp"` (A4 V2)                               | `craft-hub-mcp`     |
| 2    | Tool generation, naming and grouping from the merged registry (A7 §10.1); annotations from command effects (A4 "Inputs" for A7) | `craft-hub-mcp`     |
| 3    | Confirmation flow for write and destructive tools (A7 §10.3, A4 `confirmation_required`)                                        | `craft-hub-mcp`     |
| 4    | Resources: running apps, selection, recent assets as `handoff.selection` / AssetRef JSON (A5)                                   | `craft-hub-mcp`     |
| 5    | Per-connection permissions for MCP clients (A8 §11.4)                                                                           | `craft-hub-core`    |
| 6    | Fake MCP client tests; then Claude Code and the local llama.cpp stack on Windows                                                | `craft-hub-testkit` |

- **Methods delivered:** none new on the hub socket; an MCP server over stdio.
- **Acceptance:** AC4 for AC1 and AC2 from Claude and from the llama.cpp stack; read-only tools run without a prompt; every write tool asks; the tool list stays within A7's count target.
- **Depends on:** nothing new in AssetCraft.

### 16.6 Hub Stage 5: Produce

Plan → FilmCraft end to end (§9.3, AC3) and the v1 exit.

| Step | Work                                                                                                                          | Crates touched                      |
|------|-------------------------------------------------------------------------------------------------------------------------------|-------------------------------------|
| 1    | `handoff.timeline` payload from AssetCraft's OTIO writer (A5 §7.2.5)                                                          | AssetCraft `interchange`, `bridge`  |
| 2    | FilmCraft `file.import` through the hub, ledger records clip ids, `expect` check of clip count, order and durations ± 1 frame | `craft-hub-core`, FilmCraft profile |
| 3    | Recipe `plan-to-filmcraft` with dry run, in `test_data/hub/recipes/`, plus the `test_data/hub/otio/` fixture                  | `craft-hub-testkit`, `test_data`    |
| 4    | AC4 for AC3 (from MCP)                                                                                                        | `craft-hub-mcp`                     |
| 5    | R5 24 h soak result; all P/R targets compared to baselines; tag the hub v1                                                    | none                                |

- **Acceptance:** AC3 a–d; AC1–AC4 all pass from `craftctl` and MCP on Windows; R5 passes; P1–P6 within the gates Paul set after the baselines.
- **Depends on:** AssetCraft Stage 4.

### 16.7 AssetCraft Stage 1.5: text to add to `docs/assetcraft-plan.md` (O-B9)

Matches the plan's existing format (stage table row; a `### Stage N: Name` section with a lead paragraph, Scope / Goal / Immediate-area bullets, **Features**, **Commands**, **CLI**, **Building blocks**, **Steps**, **Exit**, as Stage 0 and Stage 3.5 do at `docs/assetcraft-plan.md:67-132`, `:168-216`). Owner: Paul approves; Claude applies it in AssetCraft and archives nothing (it is an insertion).

**1. Stage table** (`docs/assetcraft-plan.md:55-65`): insert after the Stage 1 row.

```markdown
| 1.5   | API: control, MCP, effects   | Command effects on every command; `--control` channel with a manifest; `assetcraft-cli serve` and `mcp`; thumbnails and asset refs as commands |
```

**2. Section**: insert after the Stage 1 section (after `docs/assetcraft-plan.md:147`, before `### Stage 2: Organize`).

```markdown
### Stage 1.5: API (control channel, MCP, command effects)

Everything AssetCraft can do becomes reachable from outside its window, the way the sibling apps
are, and every command says what it touches. PipelineCraft (working name), "Open In v2" and
Stage 6 recipes all build on this. No new user-facing features, and still no writes to user files.

- **Scope:** command effects and `enabled` on every command; `assetcraft-cli commands --json` in the
  full suite shape; an opt-in loopback control channel (`--control`) with a token and a manifest
  file; `assetcraft-cli serve` (headless control) and `assetcraft-cli mcp`; `thumb.render` and
  `asset.ref` as commands.
- **Goal it serves:** AssetCraft becomes the first app that declares command effects natively and
  publishes where it listens, so it is the reference for the hub contract and for the small upstream
  PRs (manifest file, effects field).
- **Immediate area of improvement:** today AssetCraft can only be driven through `assetcraft-cli exec`
  (one process per call), its catalog has no `enabled` or effects, and nothing outside it can ask for
  a thumbnail or an asset's identity.

**Features**
- **Command effects:** each `CommandSpec` declares classes (`read`, `write-app-state`,
  `write-user-file`, `external-network`) plus `destructive` / `undoable` / `idempotent` flags
  (PipelineCraft spec §6.4). `app.commands` lists them with `enabled` and `disabledReason`.
- **Control channel:** `assetcraft --control <port|0> [--control-token-file F]`, loopback only,
  JSON lines `{id, method, params}` → `{id, ok, result|error}`, methods `engine.execute
  {command, params}` and `engine.commands`, `auth {token}` as the first frame. The real bound port
  is logged, a failed bind exits non-zero, and a manifest is written to
  `%LOCALAPPDATA%\CraftApps\run\assetcraft-<pid>.json` (removed on exit).
- **Headless:** `assetcraft-cli serve --control <port|0>` runs the same server over a headless
  `Session`; `assetcraft-cli mcp` serves the suite MCP tool set over stdio.
- **Hostile input:** request-only line guard, 4 MiB line cap, 4 KiB before auth, 16 connections,
  per-request deadline (VectorCraft's server is the model).

**Commands** (new)
- `thumb.render {path, maxSide?, out?}`: PNG thumbnail through the existing pipeline. Without `out`
  it returns base64 PNG (≤ 1 MiB); with `out` it writes a **new** file only, never overwrites
  (the CLI `thumb` rule).
- `asset.ref {path}` → `{path, size, mtime, q1}` (PipelineCraft spec §7.1, BLAKE3 `q1`).
- `app.commands` gains `effects`, `enabled`, `disabledReason`.

**CLI**
- `assetcraft-cli commands --json` (full shape) and `assetcraft-cli describe <id>`
- `assetcraft-cli serve --control <port|0> [--control-token-file F]`
- `assetcraft-cli mcp`

**Building blocks (reuse before build)**
| Need                                    | From                                                                     | Notes                                                       |
|-----------------------------------------|--------------------------------------------------------------------------|-------------------------------------------------------------|
| Guarded NDJSON server, deadlines        | VectorCraft `apps/vectorcraft/src/control_server.rs` (Copy)              | Into `crates/craft-control-server` (L1, no UI); NOTICE line |
| Token handshake                         | PhotoCraft `crates/automation/src/security.rs` (Copy)                    | Same frame as PhotoCraft; NOTICE line                       |
| Manifest write (stage + rename)         | PdfCraft control file, `apps/pdfcraft/src/main.rs` (Copy pattern)        | Adds `app`, `app_version`, `codec`, `protocol`              |
| Envelope, `CommandInfo`, manifest types | New `crates/craft-control-proto` (L0)                                    | Shared with PipelineCraft from day one                      |
| MCP tool names                          | Suite convention `command_list` / `command_run` / `command_batch` (Call) | Hand-written JSON-RPC, no tokio                             |

**Steps**
1. `CommandSpec` gains `effects` and `enabled`; every existing command declares them (`cmd!` and a
   `query!` form for reads); `file.openIn` and `file.run` are `write-app-state` with
   `confirm: true` until the spec decides on an `external-process` class.
2. `app.commands` and `assetcraft-cli commands --json` emit the full shape; `describe <id>`.
3. `crates/craft-control-proto` (L0) and `crates/craft-control-server` (L1); `TABLE` rows and the
   island rule; server thread → UI thread over a channel, like the siblings.
4. `assetcraft --control` and `assetcraft-cli serve`: token, manifest, real port, exit on bind failure.
5. `assetcraft-cli mcp` with the suite tool set; tool hints from command effects.
6. `thumb.render` and `asset.ref` commands (`blake3`, `pure` feature).
7. Optional: add EffectCraft, VectorCraft and SoundCraft to the Open In catalog.
8. Tests: fake client over a real socket; hostile frames; manifest written and removed;
   `engine.commands` equals `commands --json`; exit test.

**Exit**
- `engine.commands` over `--control` equals `assetcraft-cli commands --json` (same ids, effects and
  `enabled`).
- A PowerShell `TcpClient` script authenticates, runs `nav.open` and `app.state`, and a second
  script without the token is refused; an HTTP request line closes the connection.
- The manifest appears in `%LOCALAPPDATA%\CraftApps\run\` while the app runs and is gone after a
  normal exit.
- `assetcraft-cli mcp` lists its tools in Claude Code, and read-only tools are marked read-only.
- `cargo xtask ci` passes, and nothing writes to user files except a new `thumb.render` `out` file.
```

**3. Feature backlog** (`docs/assetcraft-plan.md:229-248`): insert after the Folder watcher row.

```markdown
| Control channel, MCP, effects   | 1.5   | M    |
```

**4. Open decisions** (`docs/assetcraft-plan.md:252-258`): add one row.

```markdown
| Where `craft-control-*` lives                | Incubate here; offer to craft-libs after one fork PR builds on it | Stage 1.5 |
```

Commit block for when the text is applied:

```powershell
Set-Location F:\Script-Projects\ArtCrafting\assetcraft
git add docs/assetcraft-plan.md STATUS.md
git commit -m "Stage 1.5 step 0: add Stage 1.5 (API) to the plan"
git push
```

---

## Findings and opportunities from this section

| ID    | Scope                  | Opportunity                                                                                 | Suggested fix                                                                                                                               | Owner                           | Status |
|-------|------------------------|---------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------|--------|
| A9-R1 | §13.2.2 concurrency    | A3 open question 5: tokio or threads                                                        | Std threads, mpsc and socket deadlines in v1; revisit at > 64 connections or an SSE front door                                              | Paul (decide); Claude           | open   |
| A9-R2 | §14.3 upstream, D6     | "One small PR first, e.g. the manifest" puts a naming decision in the first PR              | PR 1 = U2 (DesignCraft `--json`, XS) with an issue proposing U3; PR 2 = U3 + U1 manifest in DesignCraft; PR 3 = U4 `journal` in EffectCraft | Paul (decide); Claude (patches) | open   |
| A9-F1 | §13.5, earlier scripts | `03a` #5 (WordCraft) and `03b` #5 (DesignCraft) both use port 7981                          | One port plan in §13.5; integrator points the earlier sections at it                                                                        | Claude (integrator)             | open   |
| A9-F2 | §13.2.1 layers         | A5's `assetref` crate name falls outside A3's `craft-` island rule                          | Rename to `craft-assetref`, row L0                                                                                                          | Claude (integrator, A5 text)    | open   |
| A9-F3 | `01` Not verified #3   | The 7979 "second bind fails" expectation can't happen: both are client defaults (A9-N7)     | Replace with §13.5 step 20 (a client default reaches the wrong app)                                                                         | Claude (integrator)             | open   |
| A9-F4 | §16.0 commits          | `Stage N step M:` is ambiguous while hub and AssetCraft share a repo                        | `Hub Stage N step M:` for hub work until promotion; forks use upstream's style                                                              | Paul (confirm)                  | open   |
| A9-F5 | §16.0 schedule         | Roadmap: hub after StoryCraft; A3-R1: hub crates with AssetCraft 1.5                        | Hub Stages 0–1 small and read-only alongside 1.5; Stages 2+ gated on their AssetCraft prerequisite                                          | Paul (decide)                   | open   |
| A9-F6 | §16.3 stage content    | AC2 needs cancellable jobs, but the outline puts jobs in Stage 3                            | One-step jobs in hub Stage 2; events, pollers and chaining stay in Stage 3                                                                  | Paul (confirm)                  | open   |
| A9-F7 | §13.2.2 `blake3`       | `pure` drops the C/asm AVX-512 path; `full` hashing of 50 GB files may become CPU-bound     | Keep `pure` (no C toolchain); compare the hub's `full` against `b3sum` from §13.5 step 27 at hub Stage 2; drop `pure` only if it matters    | Claude                          | open   |
| A9-F8 | §13.2.2 `sysinfo`      | Its MSRV (1.95) equals the workspace MSRV, so an unpinned bump can force a toolchain change | Pin `sysinfo = "0.39"` and bump deliberately                                                                                                | Claude                          | open   |
| A9-F9 | §13.3.3 licensing      | Catalog snapshots are app output under MIT OR Apache-2.0, not CC0 like `test_data/`         | Store under `crates/craft-hub-testkit/snapshots/` with their own `ATTRIBUTION.md` row                                                       | Claude                          | open   |
| A9-O1 | `deny.toml`            | The allow list has no note on which crates need which licence                               | One comment line naming the hub's licences (blake3 CC0/Apache, sysinfo MIT)                                                                 | Claude                          | open   |
| A9-O2 | §13.2.4 CI time        | Hub integration tests spawn processes and sockets                                           | Budget ≤ 30 s added to `cargo xtask ci`; in-process fakes for unit tests; measure at hub Stage 0                                            | Claude                          | open   |
| A9-O3 | §14.4 fork hygiene     | Personal notes could leak into a PR branch                                                  | `.git/info/exclude` per fork, `git diff --stat upstream/main...HEAD` before every push, explicit `git add <paths>` in PR blocks             | Paul (runs); Claude (blocks)    | open   |
| A9-O4 | §13.5 snapshots        | The first contract snapshots can come from the verification run itself                      | Step 3 saves raw catalogs to `%LOCALAPPDATA%\PipelineCraft\snapshots\`; Claude imports and normalizes them at hub Stage 0 step 3            | Paul (run); Claude (import)     | open   |

## Not verified

Everything above is a static read; nothing was built or run. The consolidated Windows script is §13.5 (steps 1–31). Items that are specific to this section and not covered there:

**Next action (Paul):**

1. Run §13.5 in one PowerShell window and send back the output of steps 3–30 (or only the lines that differ from "Expected").
2. Confirm the package names used by step 1 (`-p <app> -p <app>-cli`). Expected: every build succeeds; any rename goes into the adapter config.
   ```powershell
   Set-Location F:\Script-Projects\ArtCrafting\designcraft
   cargo metadata --no-deps --format-version 1 | ConvertFrom-Json | ForEach-Object { $_.packages.name }
   ```
   Expected: `designcraft` and `designcraft-cli` among the names (repeat per app if a build failed).
3. After the hub's dependencies are added (hub Stage 0), run the licence gate with cargo-deny installed. Expected: `licenses ok, sources ok`, as AssetCraft prints today (`STATUS.md:80`).
   ```powershell
   Set-Location F:\Script-Projects\ArtCrafting\assetcraft
   cargo install cargo-deny --locked
   cargo xtask deny
   ```
4. `command-group` job objects on Windows (R4): at hub Stage 2, kill the daemon with `Stop-Process -Force` while three headless children run, then `Get-Process craft-fake-app -ErrorAction SilentlyContinue`. Expected: nothing.
5. `gh` is installed and signed in as `4waymedia` before §14.4: `Set-Location F:\Script-Projects\ArtCrafting; gh auth status`. Expected: "Logged in to github.com account 4waymedia".

## Open questions

1. **First PR (A9-R2, D6).** Send the tiny `--json` fix first with a manifest issue alongside it, as recommended, or the manifest PR first as the project note suggests? (Paul)
2. **When hub Stage 0 starts (A9-F5).** Alongside AssetCraft Stage 1.5 (shared crates, read-only hub), or after StoryCraft as the roadmap says? (Paul)
3. **Snapshots in the repo (A9-F9).** About 2–3 MB of normalized catalog JSON under `crates/craft-hub-testkit/snapshots/`, refreshed by Paul. Acceptable, or keep only ids and keys (smaller, but contract tests then can't check params text)? (Paul)
4. **Perf gates.** Keep A1's numbers as drafts until the Windows baseline, then set each gate at the baseline plus a margin? Or adopt A1's numbers as gates now? (Paul)
5. **Commit prefix (A9-F4).** `Hub Stage N step M:` while incubating: acceptable? (Paul)
6. **One-step jobs in hub Stage 2 (A9-F6).** Acceptable deviation from the outline? (Paul)

## Inputs for downstream agents

- **A6 (§8–9):** Stage 2 carries one-step jobs (`hub.job.get|wait|cancel`); Stage 3 adds `hub.job.list`, events, pollers and chaining. P4 needs each event to carry the app-side change time (`at`) so the harness can measure delivery. Recipes for tests live in `test_data/hub/recipes/` (generated by `make_hub_fixtures.py`). Write-user-file calls need a dry run and a confirmation token from Stage 3 (§16.0).
- **A7 (§10, §12):** `craftctl` commands per stage: Stage 0 `apps`, `commands`; Stage 1 `call`, `describe`, `profile seed|check`; Stage 3 `watch`, `run --dry-run`. P2 is measured with `craftctl apps --json`. `bench` is not a `craftctl` command (perf lives in `#[ignore]` tests). MCP is hub Stage 4 and depends on Stage 3's confirmation tokens.
- **A8 (§11):** security PR rows U5 (SoundCraft guard), U6 (token handshake for unauthenticated apps) and U28 (Windows ACLs) are placeholders for your `S#` items; merge your IDs into them. The stage gate (§16.0) never forwards `external-network` in v1 and opens `write-user-file` outside hub dirs only at Stage 3 with dry run + confirmation. `keyring` 4.2.0 (MIT OR Apache-2.0) is pre-cleared if you choose it.
- **A10 (§15):** D2 is consistent with A3-R1 and this stage plan (promotion 2 at hub Stage 2 step 8). D6 recommendation text is A9-R2. New candidate decisions: A9-R1 (no tokio), A9-F5 (when hub Stage 0 starts), A9-F6 (jobs in Stage 2).
- **Integrator:** apply A9-F1 (port plan), A9-F2 (`craft-assetref`), A9-F3 (fix `01` Not verified #3); renumber `U#` alongside `O#`/`O-B#` if the merged doc uses one ID series; merge A6/A7/A8 Not verified steps into §13.5 as W13+.
