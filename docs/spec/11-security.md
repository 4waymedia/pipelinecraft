# PipelineCraft (working name): §11 Security and trust

_2026-10-15 · spec draft v0 · status: draft (agent A8)_

- **Scope:** the hub's trust boundary on Paul's Windows machine: what it binds to (§11.1), how it authenticates to apps and how clients authenticate to it (§11.2), how it resists browsers, DNS rebinding and port takeover (§11.3), which connection may run which command effect (§11.4), what it records (§11.5), and a threat table (§11.6). It also answers A3's open question on attaching to unauthenticated apps, and lists small upstream security PRs (S1–S12).
- **Project goal it serves:** local-first apps that hand work to each other (AssetCraft → StoryCraft → FilmCraft) and can be driven by `craftctl` and MCP clients without putting Paul's files at risk. The hub makes six unauthenticated control ports reachable through one front door. That front door has to be safer than the ports behind it, not a shortcut around them.
- **Immediate area of improvement:** today each app guards itself in its own way: a token (PhotoCraft), a control-file token (PdfCraft), a key file (WordCraft), and for the other six only a loopback bind and, in five of them, a line guard. Token files are owner-only on Unix and inherit folder ACLs on Windows. This section gives the hub one consistent model on top, and names the upstream fixes that would shrink what the hub has to compensate for.

**Evidence basis.** Static read only; nothing was built or run. Sibling facts come from wave 1 and wave 2 and are cited by their IDs: `03a` (O1–O14), `03b` (O-B1–O-B13), `04-05` (A3-F#, A3-O#, A3-R#), `06` (A4-R#, A4-F#, error codes), `07` (A5-O#, path rules N1–N8). Where this section needed a detail those docs lack, it re-read the source at HEAD and cites `repo` `path:line` @ commit. Commits: `assetcraft` `3a41b2b`, `photocraft` `fd28da5`, `lightcraft` `38adc17`, `filmcraft` `9ed9650`, `pdfcraft` `ac89a11`, `wordcraft` `3292ba0`, `designcraft` `7ed5697`, `effectcraft` `2c08f67`, `vectorcraft` `0946bc5`, `soundcraft` `3a28a65`. Terms follow `01` §2: **connection** (never "session") for a hub client link, **command effect** in prose, JSON field `effects`, address `app[@instance]/[kind:]name`. Statements about Windows behaviour and browsers that are not from the clones are marked "(general knowledge)" and appear again under **Not verified**.

**New facts found in this pass (not in waves 1–2):**

| ID     | Fact                                                                                                                                                                                                                                                                 | Source                                                                                                                                                                                                                                                                                    |
|--------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| A8-N1  | Every control server binds the IPv4 literal `("127.0.0.1", port)`. None binds `::1` or `localhost`                                                                                                                                                                   | `photocraft` `apps/photocraft/src/control_server.rs:20`; `lightcraft` `apps/lightcraft/src/control_server.rs:28`; `filmcraft` `:27`; `wordcraft` `:40`; `designcraft` `:28`; `effectcraft` `:27`; `vectorcraft` `:62`; `soundcraft` `:28`; `pdfcraft` `crates/ui-egui/src/control.rs:791` |
| A8-N2  | Three bridge clients accept `localhost`, `::1` and `[::1]` as targets while the servers listen on IPv4 only. PhotoCraft's bridge then sends the 256-bit token to whatever answers                                                                                    | `photocraft` `crates/automation/src/bridge.rs:46-53`; `filmcraft` `crates/automation/src/bridge.rs:35`; `effectcraft` `crates/automation/src/bridge.rs:26`                                                                                                                                |
| A8-N3  | WordCraft's MCP client already does the right thing: it sends the control key only to loopback (127.0.0.0/8, `::1`, `localhost`) and refuses otherwise                                                                                                               | `wordcraft` `crates/mcp/src/backend.rs:70`, `:102`                                                                                                                                                                                                                                        |
| A8-N4  | PhotoCraft's headless `serve` re-checks `local_addr().ip().is_loopback()` after binding and refuses otherwise; that's the strictest bind check in the suite                                                                                                          | `photocraft` `crates/automation/src/rpc.rs:305-309`                                                                                                                                                                                                                                       |
| A8-N5  | LightCraft `denoise-serve` binds whatever `--listen` says and only warns on `0.0.0.0` / `[::]` (it has its own token, 03a §3a.3). It's the one LAN listener in the suite, and it is not a control channel                                                            | `lightcraft` `apps/lightcraft-cli/src/main.rs:963-967`                                                                                                                                                                                                                                    |
| A8-N6  | `external-network` commands **do** exist at HEAD (A4 §6.4.1 found none): EffectCraft `roto.model.download` and `face.model.download` shell out to `curl`; LightCraft has `metadata.downloadPlaceNames`                                                               | `effectcraft` `crates/engine/src/models.rs:365-369`, `:424`, `:453`; `lightcraft` `crates/engine/src/cmd/geocode.rs:282`                                                                                                                                                                  |
| A8-N7  | EffectCraft runs JavaScript through `script.run {code}` and `file.runScript {path}`. Script sockets and file writes are gated by the pref `scripting.allowScriptsWriteFiles` (default false), and that pref is settable by `prefs.set`, an ordinary registry command | `effectcraft` `crates/engine/src/commands/file_more.rs:535`, `:1014`, `:1023`; `crates/script/src/socket.rs:1-5`, `:22-25`; `crates/engine/src/prefs.rs:302-303`; `crates/engine/src/commands/settings.rs:21-27`; key name in a test at `crates/engine/src/commands/app_more.rs:713`      |
| A8-N8  | DesignCraft `script.run {text}` runs a list of registry commands; its command effects are the union of whatever the text contains                                                                                                                                    | `designcraft` `crates/engine/src/cmd/scripts.rs:11-12`                                                                                                                                                                                                                                    |
| A8-N9  | Over control, LightCraft exposes `ui.clipboard` (read and write the system clipboard), and most apps expose `ui.screenshot` and synthetic input (`ui.key`, `ui.type`, `ui.click`)                                                                                    | `lightcraft` `crates/ui-egui/src/control.rs:306`, `:429`; `photocraft` `crates/ui-egui/src/control.rs:1200`; method lists in 03a §3a.2–3a.6, 03b §3.1b                                                                                                                                    |
| A8-N10 | PhotoCraft prints its control token to stderr when started with neither a token nor a token file. The hub pipes GUI stderr into its log (A3 §5.3 step 2), so that line would land in the hub log                                                                     | `photocraft` `apps/photocraft/src/main.rs:286-287`; `04-05` §5.3                                                                                                                                                                                                                          |
| A8-N11 | `pdfcraft-cli ui --control FILE` refuses a control file that is a symlink, and refuses one owned by another user **on Unix only**; on Windows the owner isn't checked                                                                                                | `pdfcraft` `apps/pdfcraft-cli/src/main.rs:868-905`                                                                                                                                                                                                                                        |
| A8-N12 | WordCraft's key file is replaced with `remove_file` + `create_new`, mode 0600 on Unix only, under `%APPDATA%\WordCraft`, which is the **roaming** profile                                                                                                            | `wordcraft` `apps/wordcraft/src/control_server.rs:84-97`; `crates/control-key/src/lib.rs:1-49` (03a §3a.6)                                                                                                                                                                                |

---

## 11.1 Loopback-only binding

### 11.1.1 What each surface binds today

| Surface                                  | Bind                                                          | Auth                        | Reachable from       | Cite                                                                                      |
|------------------------------------------|---------------------------------------------------------------|-----------------------------|----------------------|-------------------------------------------------------------------------------------------|
| PhotoCraft `--control`                   | `127.0.0.1:<port>`                                            | token, `auth` first frame   | any local process    | `photocraft` `apps/photocraft/src/control_server.rs:20`                                   |
| PhotoCraft `photocraft-cli serve --port` | `127.0.0.1:<port>`, re-checked loopback                       | token                       | any local process    | `photocraft` `crates/automation/src/rpc.rs:305-309`; `apps/photocraft-cli/src/lib.rs:562` |
| LightCraft `--control`                   | `127.0.0.1:<port>`                                            | none                        | any local process    | `lightcraft` `apps/lightcraft/src/control_server.rs:28`                                   |
| LightCraft `denoise-serve --listen`      | caller-chosen, may be `0.0.0.0`                               | own shared token            | **LAN** if so chosen | `lightcraft` `apps/lightcraft-cli/src/main.rs:963-967`                                    |
| FilmCraft `--control`                    | `127.0.0.1:<port>`                                            | none                        | any local process    | `filmcraft` `apps/filmcraft/src/control_server.rs:27`                                     |
| PdfCraft `--control <file>`              | `127.0.0.1:0`                                                 | token from the control file | any local process    | `pdfcraft` `crates/ui-egui/src/control.rs:791`                                            |
| PdfCraft single-instance hand-off        | AF_UNIX `%LOCALAPPDATA%\PdfCraft\instance-<SESSIONNAME>.sock` | folder ACL                  | same user (by ACL)   | 03a §3a.5                                                                                 |
| WordCraft `--control`                    | `127.0.0.1:<port>`                                            | key file, every request     | any local process    | `wordcraft` `apps/wordcraft/src/control_server.rs:40`                                     |
| DesignCraft `--control`                  | `127.0.0.1:<port>`                                            | none                        | any local process    | `designcraft` `apps/designcraft/src/control_server.rs:28`                                 |
| EffectCraft `--control`                  | `127.0.0.1:<port>`                                            | none                        | any local process    | `effectcraft` `apps/effectcraft/src/control_server.rs:27`                                 |
| EffectCraft script `Socket.listen`       | `127.0.0.1:<port>`                                            | none (script-owned)         | any local process    | `effectcraft` `crates/script/src/socket.rs:151`                                           |
| VectorCraft `--control`                  | `127.0.0.1:<port>`                                            | none                        | any local process    | `vectorcraft` `apps/vectorcraft/src/control_server.rs:62`                                 |
| SoundCraft `--control`                   | `127.0.0.1:<port>`                                            | none                        | any local process    | `soundcraft` `apps/soundcraft/src/control_server.rs:28`                                   |
| Every `<app>-cli mcp`                    | stdio (no socket)                                             | none: the parent owns it    | the parent process   | 03a §3a.1, 03b §3.1b                                                                      |
| AssetCraft                               | no listener at `3a41b2b`                                      | n/a                         | n/a                  | 03b §3.4                                                                                  |

"Any local process" includes processes of **other users** on the same machine (fast user switching, a second account, a service account) and, for the six unauthenticated apps, any web page that can reach loopback (§11.3). Loopback stops the LAN. It doesn't stop the machine.

### 11.1.2 Hub rules

| ID | Rule                                                                                                                                                                                                                                                                                              |
|----|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| L1 | The hub daemon listens on `127.0.0.1:0` only (OS-chosen port, A4-R1), and after binding checks `local_addr().ip()` is exactly `127.0.0.1`, as PhotoCraft's `serve` does (A8-N4). Any other result is a startup error                                                                              |
| L2 | No configuration key, environment variable or CLI flag can change the bind address in v1. NG3 (no LAN) is enforced by the absence of a setting, not by a default                                                                                                                                  |
| L3 | The hub **connects** to apps and to itself only through the literal `127.0.0.1`. It never resolves `localhost`. Manifests (`04-05` §5.2 `control.host` const `127.0.0.1`), adapter config and `hub.json` reject any other host, including `::1`, `[::ffff:127.0.0.1]` and other `127.x` addresses |
| L4 | `craftctl` and the MCP shim read the hub's port from `hub.json` and connect to `127.0.0.1:<port>`; they never accept a host argument in v1                                                                                                                                                        |
| L5 | The hub never starts LightCraft `denoise-serve` and doesn't model it as an app surface (03a §3a.3; `04-05` §4.4 "not modelled")                                                                                                                                                                   |
| L6 | A v1.x local socket (AF_UNIX via `uds_windows`, A4-R1) is **added** next to TCP, not instead of it, and then becomes the default for `craftctl` and the MCP shim. Browsers can't reach AF_UNIX at all, which removes T1 for the hub's own listener                                                |

### 11.1.3 IPv6: `::1` vs `127.0.0.1`

Windows keeps IPv4 and IPv6 listeners separate: an app bound on `127.0.0.1:7878` doesn't own `[::1]:7878`, so any local process, including another user's, can bind `[::1]:7878` while the app runs (general knowledge; Not verified step 3). On Windows, `localhost` normally resolves to `::1` first (general knowledge). Put together:

- A client that dials `localhost:7878` tries `::1` first. With nothing there it falls back to `127.0.0.1` and works, which is why the issue is invisible day to day.
- With a squatter on `[::1]:7878`, the client talks to the squatter. PhotoCraft's bridge would send its token there as the first frame (A8-N2; `photocraft` `crates/automation/src/bridge.rs:46-53`). WordCraft's MCP client would send its key, because it treats `::1` and `localhost` as safe loopback (A8-N3).
- The hub avoids all of this with rule L3 (dial the literal). The upstream fix is S5.

**Recommendation A8-R1.** Keep v1 IPv4-only (L1, L3). If a future app binds dual-stack, its adapter config must list both addresses and the hub must apply the listener-owner check (§11.3.4) to the family it dials. Owner: Claude (hub). Status: open.

---

## 11.2 Auth normalization

### 11.2.1 One credential per hop

```
 client ──(hub token, once, in hub.hello)──▶ hub ──(per-app secret, per adapter connection)──▶ app
```

Clients never see an app's secret, and apps never see the hub token. The hub keeps a `SecretRef` per instance and only the adapter that dials the app reads it.

| App                                                                      | What the hub presents                                                           | Where the hub gets it                                                                                                                                                                                                                                                 | Who creates it       | Lifetime                                                | Cite                                                           |
|--------------------------------------------------------------------------|---------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------|---------------------------------------------------------|----------------------------------------------------------------|
| PhotoCraft                                                               | first frame `{"method":"auth","params":{"token":"<64 hex>"}}`                   | hub-launched: a fresh token the hub writes to `tokens\photocraft-<port>.token` **before** launch and passes as `--control-token-file`. Hand-started: the file named by `--control-token-file` on argv, only if it lies under the hub state dir (A3 §5.2 reading rule) | hub (launch) or Paul | one app process                                         | 03a §3a.2 Auth; `04-05` §5.3 step 1                            |
| PdfCraft                                                                 | first request `{"jsonrpc":"2.0","method":"auth","params":{"token":"<32 hex>"}}` | the control file the hub named at launch (`<state>\control\pdfcraft-<n>.json`), trusted only with a live `pdfcraft.exe` pid (A3-F4)                                                                                                                                   | PdfCraft             | until the file is replaced (it is never removed, A3-F4) | 03a §3a.5 Auth; `pdfcraft` `apps/pdfcraft/src/main.rs:455-500` |
| WordCraft                                                                | top-level `"key"` on every request, or `auth` once                              | `%APPDATA%\WordCraft\control-key.local-<port>` (or `WORDCRAFT_PROFILE_DIR`), re-read on every reconnect like WordCraft's own MCP client                                                                                                                               | WordCraft            | while the window is open                                | 03a §3a.6 Auth; `04-05` §4.4 `key-field`                       |
| LightCraft, FilmCraft, DesignCraft, EffectCraft, VectorCraft, SoundCraft | nothing                                                                         | n/a                                                                                                                                                                                                                                                                   | n/a                  | n/a                                                     | 03a §3a.3–3a.4; 03b §3.2b                                      |
| Any `<app>-cli` (headless, MCP stdio)                                    | nothing; the hub is the parent process                                          | n/a                                                                                                                                                                                                                                                                   | n/a                  | the child process                                       | `04-05` §4.4                                                   |

Rules for these secrets:

| ID | Rule                                                                                                                                                                                                                     |
|----|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| K1 | Never on argv, never in an environment variable the hub sets for a child, never in a log line, never in an event or error `data`. PhotoCraft gets a **file** path, never `--control-token` (`04-05` §5.3 step 1)         |
| K2 | Before the adapter sends a secret, the listener-owner check (§11.3.4) must pass for that port. A secret is never sent to a listener the hub hasn't tied to the expected pid                                              |
| K3 | Every secret file the hub reads must be a regular file (no link, `symlink_metadata`), at most 4 KiB, owned by the current user, and in a folder whose DACL passes §11.2.3. PdfCraft's CLI already refuses links (A8-N11) |
| K4 | Files the hub creates (PhotoCraft token files, PdfCraft control-file paths) live under `%LOCALAPPDATA%\PipelineCraft\tokens\` and `...\control\`, and are deleted when the instance ends (`04-05` §5.4)                  |
| K5 | In memory, secrets are held in a small `Secret(String)` newtype whose `Debug`/`Display` print `[redacted]`, and compared in constant time, as PhotoCraft and PdfCraft do (03a §3a.2, §3a.5)                              |

### 11.2.2 The hub's own token

| Aspect      | Specification                                                                                                                                                                                                                                                                                                                                                                                                                             |
|-------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Generation  | 256 bits from the OS CSPRNG via `getrandom` (MIT/Apache, already used by PdfCraft and WordCraft, 03a §3a.5–3a.6), encoded as 64 lowercase hex characters. Same size and encoding as PhotoCraft's (03a §3a.2)                                                                                                                                                                                                                              |
| When        | At every `pipelinecraft serve` start. The daemon is on demand (D5), so the token naturally changes often                                                                                                                                                                                                                                                                                                                                  |
| Storage     | Inline in `hub.json` (`auth.kind = "token-frame"`, `auth.token`), the manifest A3 defines (`04-05` §5.2, `"app":"pipelinecraft"`), in `%LOCALAPPDATA%\PipelineCraft\`. Written by stage-and-rename in the same folder (PdfCraft's `write_control_file` pattern, `pdfcraft` `apps/pdfcraft/src/main.rs:455-500`, or `craft-atomic`), into a folder that has passed the DACL check in §11.2.3 **before** the file is written                |
| Presented   | Once per connection, as `hub.hello` `params.auth.token` (A4 §6.6.1, rule V3), within 5 s and under the 4 KiB pre-hello cap (A4 §6.1.4). A wrong token gets `unauthenticated` and the connection closes. Compare in constant time                                                                                                                                                                                                          |
| Client side | Before sending it, a client checks that the `hub.json` pid is alive and is `pipelinecraft.exe`, and (when available) that the listener on `port` belongs to that pid (§11.3.4). That stops a stale `hub.json` from leaking the token to a process that grabbed the old port (T8)                                                                                                                                                          |
| Rotation    | (1) Automatic at every daemon start. (2) On demand with hub command `hub.token.rotate {dropConnections?: bool}` (proposed name, command effect `write-app-state`, allowed only from an interactive `craftctl` connection, §11.4): writes a new `hub.json`, refuses new `hub.hello` with the old token at once, and keeps or drops open connections. (3) No timed rotation in v1 (start small); open connections are already authenticated |
| Clean exit  | `hub.json` is deleted on clean exit. After a crash it is stale; clients detect that by the pid check and start a new daemon, which overwrites it                                                                                                                                                                                                                                                                                          |
| Not used    | No environment variable, no argv, no clipboard, no keyring for this token (see D-S1 below)                                                                                                                                                                                                                                                                                                                                                |

**Keyring.** The hub uses the OS keyring (Windows Credential Manager through the `keyring` crate, MIT OR Apache-2.0; licence to confirm at add time) only for **persistent** secrets. v1 has none: every secret above lives for one process. The first persistent ones are expected in v1.x: (a) named long-lived client tokens for a scheduled task, if Paul wants one (`craftctl token issue --profile <p>`); (b) a stable PhotoCraft token for hand-started windows, if per-launch files prove awkward. Entries are named `PipelineCraft/<purpose>/<name>`.

**Decision D-S1 (for A10 §15): ephemeral loopback tokens in owner-only files.** AssetCraft's rule 7 says "Secrets live in the OS keyring, never in files" (`assetcraft` `AGENTS.md:25-26` @ `3a41b2b`). The hub can't meet that literally for app secrets, because PhotoCraft, PdfCraft and WordCraft themselves read and write their secrets as files. For the hub's own token there is a choice:

| Option                                               | For                                                                                                                                              | Against                                                                                                                                                                  |
|------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **A. Inline in `hub.json`, owner-only DACL** (rec.)  | Matches PdfCraft's control file and A4's assumption; PowerShell `TcpClient` tests can read it; one discovery file; works with no keyring service | Bends rule 7 for a per-run secret                                                                                                                                        |
| B. Credential Manager entry, `hub.json` has no token | Honours rule 7 literally; Credential Manager is per user and DPAPI-protected (general knowledge)                                                 | Every client needs keyring code; PowerShell needs `craftctl hub token` to test; a crash leaves a stale entry; no gain against same-user processes, which can read either |

Recommendation: **A**, with rule 7 read as "persistent secrets live in the keyring; per-process loopback tokens live in owner-only files that die with the process". Owner: Paul (decide); Claude (record in AGENTS.md if accepted). Status: open.

### 11.2.3 Windows file protection (DACL)

Wave 1 found that every token file is 0600 on Unix only, and inherits the folder's ACL on Windows (03a O12; `photocraft` `crates/automation/src/security.rs:140-153`; `pdfcraft` `apps/pdfcraft/src/main.rs:474-478`; `wordcraft` `apps/wordcraft/src/control_server.rs:93-94`). The hub's approach is to protect the **folders** once and let files inherit, rather than set an ACL per file.

**Target DACL** for `%LOCALAPPDATA%\PipelineCraft\` and everything under it, and the check (not the fix) for the suite run dir `%LOCALAPPDATA%\CraftApps\run\` (`04-05` §5.2):

```text
D:PAI(A;OICI;FA;;;<user SID>)(A;OICI;FA;;;SY)
```

That is: protected (no inherited entries), full access for Paul's user SID and for SYSTEM, inherited by sub-folders and files (`OICI`). Administrators are left out on purpose. They can take ownership anyway, so listing them adds nothing, and leaving them out makes the check simple: exactly two entries.

**Procedure (DG1–DG5), run at daemon start and when creating the folder:**

| Step | Action                                                                                                                                                                                                                                                                                                                                                                       |
|------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| DG1  | Create the folder if it is missing (`create_dir_all`) and refuse to continue if it is a link or junction (`symlink_metadata`)                                                                                                                                                                                                                                                |
| DG2  | Read its security descriptor as SDDL. v1 mechanism: spawn `%SystemRoot%\System32\icacls.exe <dir> /save <tmpfile>`, which writes SDDL (locale-independent, unlike the names `icacls` prints). Get the user SID from `%SystemRoot%\System32\whoami.exe /user /fo csv /nh`                                                                                                     |
| DG3  | Pass when the DACL grants access only to the user SID, `SY` and optionally `BA`, with no `Everyone` (`WD`), `Users` (`BU`), `Authenticated Users` (`AU`) or other user SIDs. The default `%LOCALAPPDATA%` inheritance is expected to pass (03a Not verified #6)                                                                                                              |
| DG4  | If the check fails, apply the target with `icacls.exe <dir> /inheritance:r /grant:r *<SID>:(OI)(CI)F *S-1-5-18:(OI)(CI)F`, then re-check. Still failing → the hub runs with **no secrets written** (it refuses to launch PhotoCraft/PdfCraft with control and doesn't write `hub.json` with a token), reports `state_dir_insecure` and tells Paul the exact `icacls` command |
| DG5  | Files are created with `create_new` inside the checked folder, never somewhere else and moved in. A token file is checked once after creation (DG2 on the file); extra entries → delete it and fail the launch                                                                                                                                                               |

Why spawn `icacls` rather than call Win32: the house rule forbids `unsafe` in the hub's crates (`assetcraft` `AGENTS.md` non-negotiable 4), and the Win32 security APIs need it. A3 already accepts a permissive crate that hides `unsafe` for job objects (`04-05` §5.3, `command-group`). A crate with a safe ACL API is the cleaner long-term choice (opportunity A8-O2). Until one is picked and its licence checked, `icacls` is built into Windows, needs no dependency, and its result is easy to test.

**Recommendation A8-R2.** Folder-level DACL check and repair (DG1–DG5) for the hub state dir; check-only for the suite run dir (an app may own it); a warning for `%APPDATA%\WordCraft` if WordCraft's key file folder fails DG3. Owner: Claude (hub). Status: open.

---

## 11.3 Browser origin, DNS rebinding, CSRF and port takeover

### 11.3.1 How a web page reaches a loopback port

A page can't open raw TCP. It can send HTTP to `http://127.0.0.1:<port>/`:

- **Cross-origin "simple" request** (CSRF style): `fetch(url, {method: "POST", mode: "no-cors", body: "<text>"})` with a `text/plain` body. There's no preflight, the bytes go out, and the page can't read the reply (general knowledge).
- **DNS rebinding:** a host name the attacker controls first resolves to the attacker's server, then to `127.0.0.1`. The page becomes "same origin" with the loopback port and **can** read replies (general knowledge).
- **Port scanning:** a page can try thousands of ports in minutes by timing `fetch` failures, so a non-default port is no defence (general knowledge).
- Some browsers are adding prompts or blocks for public pages that call local addresses ("private/local network access"). Coverage varies by browser and version, so the hub doesn't rely on it (general knowledge).

Either way the bytes on the wire are an HTTP/1.1 request:

```text
POST / HTTP/1.1\r\n
Host: 127.0.0.1:7979\r\n
Content-Type: text/plain;charset=UTF-8\r\n
Origin: https://evil.example\r\n
...more headers...\r\n
\r\n
{"id":1,"method":"engine.execute","params":{"command":"session.save_as","params":{"path":"C:\\x"}}}\n
```

The body is the attacker's, so it can end in `\n` and contain several lines. The first line is always `<METHOD> <target> HTTP/1.1`, which never parses as JSON. That's the property the line guards rely on.

### 11.3.2 Each app's exposure, from source

| App         | First non-JSON line                               | Pre-auth line cap | Can a browser run a command? (static read)                                                                             | Cite                                                                                           |
|-------------|---------------------------------------------------|-------------------|------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| PhotoCraft  | not `auth` → `authentication required`, **close** | 1 MiB             | **No.** The request line fails auth; the token isn't knowable                                                          | `photocraft` `apps/photocraft/src/control_server.rs:74-81` (`authentication_reply`, `:75`)     |
| PdfCraft    | `-32700` parse error, **close before auth**       | 4 KiB             | **No**                                                                                                                 | `pdfcraft` `crates/ui-egui/src/control.rs:824-826`, `:808`                                     |
| WordCraft   | `bad JSON …; closing the connection`, **close**   | 1 MiB             | **No.** The comment names this case ("a web page's `fetch` to this port")                                              | `wordcraft` `apps/wordcraft/src/control_server.rs:200-208`                                     |
| LightCraft  | `Line::Reject`, **close**                         | 4 MiB             | **No.** No auth, but the guard closes on the request line                                                              | `lightcraft` `apps/lightcraft/src/control_server.rs:129-145`, `:181`                           |
| FilmCraft   | `Line::Reject`, **close**                         | 4 MiB             | **No**, same guard                                                                                                     | `filmcraft` `apps/filmcraft/src/control_server.rs:93-109`, `:141`                              |
| DesignCraft | `Line::Reject`, **close**                         | 4 MiB             | **No**, same guard                                                                                                     | `designcraft` `apps/designcraft/src/control_server.rs:80-103`, `:132-157`                      |
| EffectCraft | `Line::Reject`, **close**                         | 4 MiB             | **No**, same guard                                                                                                     | `effectcraft` `apps/effectcraft/src/control_server.rs:79-89`, `:127`                           |
| VectorCraft | `Line::Reject`, **close**                         | 4 MiB             | **No**, same guard plus deadlines                                                                                      | `vectorcraft` `apps/vectorcraft/src/control_server.rs:162-172`, `:226-230`                     |
| SoundCraft  | `bad JSON: …` reply, **connection stays open**    | ~86 MiB           | **Yes (O-B2).** Each header line gets an error; the body line runs as a command. DNS rebinding can also read the reply | `soundcraft` `apps/soundcraft/src/control_server.rs:97-137` (`:132` is the non-closing branch) |

Two more details from the SoundCraft loop that make O-B2 worse than "one stray command":

1. A JSON line with no `method` still dispatches, with method `""` (`soundcraft` `apps/soundcraft/src/control_server.rs:119-121`), so the server never sends a "not a request" refusal at all.
2. The read loop treats a final line without `\n` at end of stream as a full line (the `read_bounded_line` contract, `:75-95`), so even a body without a trailing newline runs when the browser closes the connection.

What SoundCraft over `engine.execute` could do to Paul from a web page (static): `session.save_as {path}` writes a file at an attacker-chosen path, `file.bounce_mix {path}` writes audio anywhere, and `session.open` replaces the open session without a save prompt (03b §3.1b). Those are `write-user-file` and `destructive`. **Needs the live check in Not verified step 1 before anyone files it.**

The **close-on-first-non-JSON-line** guard (A4's framing rule; A4 inputs to A8, "non-JSON first lines close the connection") is sufficient against both CSRF and rebinding for the eight guarded apps, because no browser API can make the first line JSON (general knowledge). Being unauthenticated still exposes those six apps to any local process (T2, T3).

### 11.3.3 Hub-side mitigations

| ID | Mitigation                                                                                                                                                                                                                                                                                                                                                                                                                                | Covers           |
|----|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------|
| B1 | The hub's listener applies A4's framing exactly: the first line must be a JSON object that is `hub.hello` with a valid token within 5 s and 4 KiB, else one error and close (A4 §6.1.4). A rebinding page can't learn the token, so it can't pass the first frame                                                                                                                                                                         | hub listener, T1 |
| B2 | A first line that starts with an HTTP method token (`GET `, `POST `, `PUT `, `OPTIONS `, `HEAD `, `PRI * HTTP/2.0`) is still answered with `bad_frame`, and also counted as a `browser_probe` audit record (rate-limited to one summary a minute, §11.5)                                                                                                                                                                                  | visibility of T1 |
| B3 | The hub never answers in HTTP: no status line, no CORS headers. A future streamable-HTTP MCP endpoint in the daemon (`04-05` §4.1, "later") must validate `Host` (`127.0.0.1:<port>` only), reject any request carrying an `Origin` header that isn't on an allowlist, and require the hub token as a bearer token. This follows the MCP transport guidance on DNS rebinding (general knowledge; re-check the spec when that work starts) | future HTTP, T1  |
| B4 | **SoundCraft adapter, until S1 lands:** `control_policy = "headless-preferred"`. Hub work on SoundCraft goes to `soundcraft-cli` headless / MCP stdio, which a browser can't reach. The hub launches the GUI with `--control` only for an explicit Open In (`hub.app.launch {mode: gui}`), on a `port_range` port, and `craftctl apps` and `hub.apps` show `exposure: "browser-reachable"` for that instance while it runs                | SoundCraft O-B2  |
| B5 | When the hub **attaches** to a SoundCraft window Paul started with `--control`, it attaches (§11.3.5 tier C) but the warning B4 shows; the hub can't close a port it didn't open                                                                                                                                                                                                                                                          | SoundCraft O-B2  |
| B6 | Socket transport later (L6): AF_UNIX for hub clients removes the hub listener from the browser's reach entirely. Answer to A4 open question 4: TCP loopback in v1 is acceptable **with** B1; add AF_UNIX in v1.x, not first                                                                                                                                                                                                               | hub listener     |

### 11.3.4 Listener-owner check (port takeover)

A3 found that a failed bind leaves most apps running with **no** control channel (A3-F2), so a port the hub chose may be answered by someone else. A3's identity check (catalog contains `identity_probe` ids) proves only that the listener *says* it is the app. A process that replays a recorded catalog passes it.

**Recommendation A8-R3: tie the port to the pid before trusting it.** Before an adapter sends a secret (K2) or treats an unauthenticated instance as attached, it asks the OS which process owns the listening socket on `127.0.0.1:<port>` and requires that it is the instance's pid (and that pid's exe matches, `04-05` §5.3 step 1).

| Aspect           | Detail                                                                                                                                                                                                                         |
|------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Windows data     | The TCP table with owning pid (`GetExtendedTcpTable`; what `Get-NetTCPConnection … \| Select OwningProcess` and `netstat -ano` show; general knowledge)                                                                        |
| How, no `unsafe` | A permissive crate with a safe API (candidates `netstat2` or `listeners`; licence and maintenance to confirm, A8-O3), or, as a fallback, spawn `netstat.exe -ano -p TCP` and parse the `LISTENING` rows for `127.0.0.1:<port>` |
| When             | At attach, after every reconnect, and in the client before sending the hub token (T8)                                                                                                                                          |
| On mismatch      | Don't send anything. Report the new code `listener_mismatch` (origin `adapter`, `ran: no`; input to A4 §6.5.2), mark the instance `unreachable`, and write an audit record                                                     |
| If unavailable   | Fall back to A3's identity probe and cap the instance at trust tier C-read (§11.3.5)                                                                                                                                           |

It also covers peer identity for clients in v1.x: the same table maps a client's local port to its pid, so the hub can see which exe opened a connection (§11.4.3).

### 11.3.5 Answer to A3's open question: attaching to unauthenticated apps Paul started

The question (`04-05` §5.3, open question 2; 03a open question 3): may the hub attach to a LightCraft, FilmCraft, DesignCraft, EffectCraft, VectorCraft or SoundCraft window Paul started himself with `--control`, before upstream auth (S2) exists?

**Reasoning.** Attaching doesn't open anything new: the port is already open to every local process. What the hub adds is (1) the chance of talking to an **impostor** on that port (A3-F2) and (2) **amplification**: an MCP client, which can't reach the port directly, can reach the app through the hub. (1) is solved by the listener-owner check. (2) is solved by §11.4 policy, which is stricter than the bare port. So attaching is safe, provided the identity is proven.

**Recommendation A8-R4: yes, with trust tiers.** Replaces A3's `attach_unauthenticated = "probed-or-launched"` with `attach.min_tier` (default `"C"`).

| Tier   | Name                   | How the instance qualifies                                                                                                                                                                                             | What may run through it                                                                                                                                                        |
|--------|------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| A      | Secret-proven          | The app accepted a secret only it could know: PhotoCraft token, PdfCraft control-file token, WordCraft key for that port (A3 `IdentityProof::SecretFromAppFile` / token), **and** the listener-owner check passed (K2) | Everything §11.4 allows                                                                                                                                                        |
| B      | Hub-launched           | Unauthenticated app the hub spawned itself: the hub knows the child pid; listener owner == child pid; identity probe passes                                                                                            | Everything §11.4 allows                                                                                                                                                        |
| C      | Probed, owner-verified | Paul started it. Process probe shows `<app>.exe` (exe path in a configured install dir or a known build output), same user, `--control <port>` on argv, listener owner == that pid, identity probe passes              | Everything §11.4 allows for `craftctl` and app clients. MCP connections get **one step tighter**: `write-app-state` moves from confirm-by-caller to confirm-by-human (§11.4.2) |
| C-read | Probed, owner unknown  | As C, but the listener-owner check isn't available (crate missing, call failed)                                                                                                                                        | `read` command effects only, for every client                                                                                                                                  |
| D      | Unverified             | A port that only appears in user config or in a manifest whose pid doesn't own the port, or whose exe doesn't match                                                                                                    | Nothing. Listed by `craftctl apps` as `unverified` with the reason. `craftctl attach <app> --port N` (interactive only) re-runs the checks; it never skips them                |

Notes:

- The identity requirement from A3 stays: A3-F2 makes the identity check mandatory, and every tier above D includes it. The listener-owner check is added on top, not instead.
- Tiers A and B can both apply (a hub-launched PhotoCraft is A).
- `hub.apps` reports `trust: "A" | "B" | "C" | "C-read" | "D"` per instance (input to A3 and A7).
- Every attach writes an audit record with the tier and the evidence (§11.5).

Owner: Paul (decide); Claude (implement). Status: open.

### 11.3.6 Stale manifests

A3's stale rules hold (`04-05` §5.4: dead pid or wrong exe = stale; PdfCraft files trusted only with a live `pdfcraft.exe`). Added here:

| ID | Rule                                                                                                                                                                                                                                  |
|----|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| M1 | A manifest's pid being alive is necessary, not sufficient. Pid reuse plus a port grabbed after a crash is exactly the takeover case, so the listener-owner check (A8-R3) runs before any secret from that manifest is used            |
| M2 | The suite run dir is read only if it passes the DG3 check. If it grants write to anyone but Paul and SYSTEM, the hub ignores its manifests and says why. Manifests in it are plain files, so anyone who can write there can plant one |
| M3 | An inline `auth.token` is accepted only from a manifest file that is owned by the current user and not a link (K3). A3's rule that `token_file` must resolve inside the hub state dir or the app's own settings dir stays             |
| M4 | `hub.json` follows the same rules on the client side (T8)                                                                                                                                                                             |

---

## 11.4 Per-connection permissions

### 11.4.1 Inputs to a decision

Every call is decided from four things, all already defined elsewhere:

1. **Command effects** of the resolved target (A4 §6.4): classes `read`, `write-app-state`, `write-user-file`, `external-network`, plus the flags `destructive`, `source`, and the derived `confirm`.
2. **The connection's profile**, picked at `hub.hello` from `client.kind` (`app`, `craftctl`, `mcp`; A4 rule V2), `client.name`, and two facts the client reports: `tty` (craftctl) and `mcpClient` (the MCP client's `clientInfo.name`, which the shim copies from MCP `initialize`).
3. **The instance's trust tier** (§11.3.5).
4. **Paths** in the params, after A5's normalization (N1–N8), checked against the profile's roots.

The decision is one of `allow`, `confirm` (caller), `confirm` (human) or `deny`, and the **strictest** input wins.

| ID  | Rule                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   |
|-----|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| P1  | Ordering: `allow` < `confirm` (caller) < `confirm` (human) < `deny`. The decision is the max over all classes in `effects.classes`, the `destructive` flag, the tier adjustment and any matching target rule                                                                                                                                                                                                                                                                                                                                                                                           |
| P2  | Floor: A4's derived `effects.confirm = true` is never lowered to `allow` (A4 §6.1 rule "policy may only tighten"). A policy `allow` for such a call becomes `confirm` (caller)                                                                                                                                                                                                                                                                                                                                                                                                                         |
| P3  | `source: default` (unknown command effects, A4 §6.4.2) counts as `destructive` + `write-user-file`, so MCP connections need human confirmation for every uncurated command                                                                                                                                                                                                                                                                                                                                                                                                                             |
| P4  | **Hub-root writes.** A `write-user-file` call whose every output path (from params, after A5 normalization) lies under `%LOCALAPPDATA%\PipelineCraft\{handoff,derived,renders,tmp}\` is decided as `write-app-state`. Thumbnails (AC2) and handoff files (AC3) would otherwise prompt on every call. If the hub can't tell where a command writes, P4 doesn't apply                                                                                                                                                                                                                                    |
| P5  | **Meta-commands** run other commands: DesignCraft `script.run` (A8-N8), EffectCraft `script.run` / `file.runScript` (A8-N7), PhotoCraft `command_batch`, any `*_batch` MCP tool. The hub either expands the steps it can parse (DesignCraft text, PhotoCraft steps) and decides on the union, or treats the call as `source: default`. EffectCraft JavaScript can't be expanded, so it's always `default`                                                                                                                                                                                              |
| P6  | `hub.batch` and recipes are decided on the union of their steps' effects (A4 §6.2.4), and need one confirmation for the plan                                                                                                                                                                                                                                                                                                                                                                                                                                                                           |
| P7  | Policy is **never writable through the hub**. There is no `hub.policy.*` command. The policy file is edited by Paul and reloaded on change or on `craftctl policy reload`. A prompt-injected MCP client can't loosen it                                                                                                                                                                                                                                                                                                                                                                                |
| P8  | Confirmations come in two kinds. **Caller:** A4's `confirmation_required` returns `data.confirm`, a single-use token bound to connection + target + params hash, and the caller asks its human (AssetCraft dialog, `craftctl` y/N prompt, the MCP client's own tool approval, §10.3). **Human:** `data.confirm` is **not** returned. Instead `data.approval` is an id that only Paul can approve, from an interactive `craftctl approve <id>` or the status panel. The caller waits on `hub.job.wait`-style polling or retries with `confirm: "<approval id>"` once it's approved (input to A4 and A7) |
| P9  | Dry runs (A4 open question 5): a dry run may return a caller-kind confirm token for **exactly** that plan (plan hash), valid 10 minutes, single use. It never returns a human-kind approval; the human approves the plan the dry run printed                                                                                                                                                                                                                                                                                                                                                           |
| P10 | **Grants** are pre-confirmations Paul writes in the policy file: a target pattern plus roots (and optionally a recipe hash) that count as a confirmation for one profile. They're the only way an unattended script does `write-user-file`. The audit log records `confirmedBy: "grant:<name>"`                                                                                                                                                                                                                                                                                                        |

### 11.4.2 Default profiles

| Effect / flag                         | `craftctl` (interactive, `tty: true`) | `craftctl` (script, `tty: false`) | `app: assetcraft` | `app: <other>`      | `mcp` (any client)                  |
|---------------------------------------|---------------------------------------|-----------------------------------|-------------------|---------------------|-------------------------------------|
| `read`                                | allow                                 | allow                             | allow             | allow               | allow                               |
| `write-app-state`                     | allow                                 | allow                             | allow             | allow               | confirm (caller)                    |
| `write-user-file`                     | confirm (caller)                      | deny unless a grant               | confirm (caller)  | deny unless a grant | confirm (human)                     |
| `external-network`                    | confirm (caller)                      | deny unless a grant               | confirm (caller)  | deny                | deny unless a grant                 |
| `destructive`                         | confirm (caller)                      | deny unless a grant               | confirm (caller)  | deny                | confirm (human)                     |
| `source: default` (P3)                | confirm (caller)                      | deny                              | confirm (caller)  | deny                | confirm (human)                     |
| Tier C instance                       | as above                              | as above                          | as above          | as above            | `write-app-state` → confirm (human) |
| Tier C-read instance                  | read only                             | read only                         | read only         | read only           | read only                           |
| `hub.token.rotate`, `hub.app.stop`    | confirm (caller)                      | deny                              | deny              | deny                | deny                                |
| Writes outside profile roots          | allowed (Paul's own console)          | deny                              | confirm (caller)  | deny                | deny                                |
| Writes to the deny-paths list (below) | confirm (caller), shown in red        | deny                              | deny              | deny                | deny                                |

**Target rules that ship in the defaults** (the evidence is in the A8-N rows):

| Pattern (A4 target grammar)                                                                                                 | `mcp`               | `craftctl` script | Why                                                                                                               |
|-----------------------------------------------------------------------------------------------------------------------------|---------------------|-------------------|-------------------------------------------------------------------------------------------------------------------|
| `*/method:ui.clipboard`                                                                                                     | deny                | deny              | Reads and writes the system clipboard, which can hold passwords (A8-N9)                                           |
| `*/method:ui.key`, `ui.type`, `ui.text`, `ui.click`, `ui.drag`, `ui.pointer`, `ui.press`, `ui.release`, `ui.dialog.confirm` | deny                | deny              | Synthetic input can click through an app's own confirmation dialogs, which would bypass P8 (NG6: no UI scripting) |
| `*/method:ui.screenshot`, `ui.render`                                                                                       | allow               | allow             | Reads; useful to Claude. The image shows only that app's window                                                   |
| `effectcraft/prefs.set` with `key`/`values` under `scripting.*`                                                             | deny                | deny              | Turning on `scripting.allowScriptsWriteFiles` gives scripts network and file access (A8-N7)                       |
| `effectcraft/script.run`, `effectcraft/file.runScript`                                                                      | confirm (human)     | deny              | Arbitrary JavaScript; can't be expanded (P5)                                                                      |
| `effectcraft/roto.model.download`, `effectcraft/face.model.download`, `lightcraft/metadata.downloadPlaceNames`              | deny unless a grant | deny              | `external-network` (A8-N6). Curated rows for A4's `effects.toml`                                                  |
| `*/app.quit`, `*/method:app.quit`, `*/method:ui.quit`                                                                       | confirm (human)     | deny              | Can lose unsaved work (A4: `destructive`)                                                                         |
| `wordcraft/file.open`, `soundcraft/session.open`                                                                            | confirm (human)     | deny              | Replace a dirty document with no prompt over control (03a O5; 03b §3.1b)                                          |

**Deny-paths list** (writes refused, matched on A5's path key, case-insensitive): `%WINDIR%`, `%ProgramFiles%`, `%ProgramFiles(x86)%`, `%ProgramData%`, `%APPDATA%\Microsoft\Windows\Start Menu`, the Startup folders, `%USERPROFILE%\.ssh`, the hub's own `policy.toml`, `hub.json` and `tokens\`, and any output whose extension is in `{exe, dll, bat, cmd, ps1, psm1, vbs, js, jse, wsf, lnk, url, scr, msi, reg, hta}`. A prompt-injected export into a Startup folder would otherwise be persistence.

**Roots.** `mcp` and `craftctl` script profiles have `roots.read` and `roots.write`. Defaults: read = AssetCraft's library roots (once AssetCraft reports them) + the hub state dir; write = the hub's `renders\` + any folders Paul lists. Root checks use A5's path key (component-wise, case-insensitive) and, for writes, walk every component from the root to the parent with `symlink_metadata` and refuse a link or junction on the way (§11.6 T6). The hub never sets an app's automation root to a drive root or to `%USERPROFILE%` itself (input to A5 §7.4.4, whose headless rule is "smallest common folder of the inputs").

**Rate limits** (MCP only, per connection): 30 non-read calls a minute, 5 destructive a minute, 32 steps per batch. Over the limit → `busy` with `data.reason: "rate"`.

### 11.4.3 Policy file format

Compiled defaults (`include_str!`, like A3's `apps.default.toml`) plus a user file `%LOCALAPPDATA%\PipelineCraft\policy.toml` merged on top. Unknown keys are an error (`deny_unknown_fields`); a broken file keeps the previous policy and reports `policy_invalid` with the line number. The hub never crashes on it.

```toml
# %LOCALAPPDATA%\PipelineCraft\policy.toml (schema 1). Edited by Paul only; never writable through the hub (P7).
schema = 1

[attach]
min_tier = "C"                      # A | B | C | C-read | D (D = attach to nothing unverified)

[audit]
reads = false                       # log read calls too (§11.5)
retention_days = 30
max_total_mib = 256

# ------------------------------------------------------------------ profiles
# Matched at hub.hello, first match wins. `kind` and `name` are self-declared (A4 rule V2);
# `peer_exe` is checked only once the peer-process check exists (v1.x, A8-O3).
[[profile]]
name  = "craftctl-interactive"
match = { kind = "craftctl", tty = true }
decide = { read = "allow", write-app-state = "allow", write-user-file = "confirm",
           external-network = "confirm", destructive = "confirm", default = "confirm" }
confirm_via = "caller"

[[profile]]
name  = "craftctl-script"
match = { kind = "craftctl", tty = false }
decide = { read = "allow", write-app-state = "allow", write-user-file = "deny",
           external-network = "deny", destructive = "deny", default = "deny" }
roots = { read = ['F:\Library'], write = ['%LOCALAPPDATA%\PipelineCraft\renders'] }

[[profile]]
name  = "assetcraft"
match = { kind = "app", name = "assetcraft" }
decide = { read = "allow", write-app-state = "allow", write-user-file = "confirm",
           external-network = "confirm", destructive = "confirm", default = "confirm" }
confirm_via = "caller"                # AssetCraft shows its own dialog

[[profile]]
name  = "mcp-claude"
match = { kind = "mcp", mcp_client = "claude-*" }
decide = { read = "allow", write-app-state = "confirm", write-user-file = "confirm-human",
           external-network = "deny", destructive = "confirm-human", default = "confirm-human" }
confirm_via = "caller"                # for write-app-state; "-human" values override per class
roots = { read = ['F:\Library', 'F:\Script-Projects\ArtCrafting\assetcraft\test_data\library'],
          write = ['%LOCALAPPDATA%\PipelineCraft\renders', 'F:\Exports'] }
rate  = { non_read_per_min = 30, destructive_per_min = 5, max_batch_steps = 32 }

[[profile]]
name  = "mcp-local"                   # Paul's llama.cpp stack (01 open question 2)
match = { kind = "mcp" }
inherit = "mcp-claude"

[[profile]]
name  = "fallback"
match = {}                            # anything else: read only
decide = { read = "allow", write-app-state = "deny", write-user-file = "deny",
           external-network = "deny", destructive = "deny", default = "deny" }

# ------------------------------------------------------------------ target rules (tighten or deny)
[[rule]]
profiles = ["mcp-*", "craftctl-script"]
target   = "*/method:ui.clipboard"
decide   = "deny"
reason   = "system clipboard (A8-N9)"

[[rule]]
profiles = ["*"]
target   = "effectcraft/prefs.set"
when     = { param_prefix = { key = "scripting.", values = "scripting." } }
decide   = "deny"
reason   = "enables script network/file access (A8-N7)"

# ------------------------------------------------------------------ grants (pre-confirmations, P10)
[[grant]]
name     = "nightly-otio-export"
profiles = ["craftctl-script"]
target   = "filmcraft/file.exportOtio"
roots    = { write = ['F:\Exports\otio'] }
expires  = "2027-01-01"
```

Profile matching for `mcp_client` uses the name the MCP client sends in `initialize` (`clientInfo.name`), copied by the shim. The shim is hub code, so the model can't change what `kind` says. It can change what it *calls*, which is what the profile limits. The limit of this design is stated in T2: a process running as Paul can claim any `kind`.

**`tty` and Claude Code.** An MCP client with a shell (Claude Code's Bash tool) can run `craftctl` directly, which skips the `mcp` profile. `craftctl` reports `tty: false` when stdin isn't a console, so it gets `craftctl-script`: no user-file writes, no destructive calls, no network without a grant. That's why the script profile is the strict one, and why "human" confirmations (P8) can only be approved from a console `craftctl` (input to A7 §12).

---

## 11.5 Audit log

### 11.5.1 Where and how

| Aspect      | Specification                                                                                                                                                                                                                                                                                      |
|-------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Location    | `%LOCALAPPDATA%\PipelineCraft\logs\audit\audit-YYYY-MM-DD.jsonl` (local date), inside the DACL-checked state dir. Separate from the diagnostic log `logs\hub-*.log`                                                                                                                                |
| Format      | JSON Lines, UTF-8, one object per line, ≤ 16 KiB per line (longer values are digested, below)                                                                                                                                                                                                      |
| Writes      | Append-only, one `write` per record. Records for `write-user-file`, `external-network` and `destructive` calls are written and flushed **before** the call is dispatched (`phase: "start"`) and again after (`phase: "end"`)                                                                       |
| Fail-closed | If the start record can't be written (disk full, permission), the call is refused with a new code `audit_unavailable` (origin `hub`, `ran: no`; input to A4 §6.5.2). Read calls and `write-app-state` continue and the failure goes to the diagnostic log. Never a crash (G8 in A5 §7.5.3 applies) |
| Retention   | 30 days and 256 MiB total (policy `[audit]`). GC deletes only files named `audit-YYYY-MM-DD.jsonl` in that folder, oldest first, and never follows links (A5 G1)                                                                                                                                   |
| Reading     | `craftctl audit [--since] [--app] [--decision] [--json]` (input to A7 §12). `craftctl audit export --redact-paths` replaces `%USERPROFILE%` and library roots with variables for sharing                                                                                                           |
| Integrity   | v1: each record carries `seq` (per file, gapless) so a deleted line shows. A per-record BLAKE3 hash chain is an opportunity (A8-O4); it shows accidental damage, not tampering by a process running as Paul                                                                                        |

### 11.5.2 What is recorded

| Event (`ev`)      | When                                                                                                                                  |
|-------------------|---------------------------------------------------------------------------------------------------------------------------------------|
| `hello`           | Every `hub.hello`, accepted or refused (`unauthenticated`, `protocol_version`)                                                        |
| `call`            | Every non-read `hub.call`, `hub.batch` step, `engine.execute` compatibility call, and recipe step; reads only if `audit.reads = true` |
| `decision`        | `deny`, `forbidden`, `path_denied`, `confirmation_required` issued, approval granted/expired                                          |
| `attach`          | Every attach and detach, with tier and evidence (§11.3.5)                                                                             |
| `launch` / `stop` | `hub.app.launch`, `hub.app.stop`, headless child start/exit                                                                           |
| `security`        | `listener_mismatch`, `browser_probe` (rate-limited), `state_dir_insecure`, `policy_invalid`, `policy_reload`, `token_rotate`          |

### 11.5.3 Record shape

```json
{"v":1,"seq":184,"at":"2026-10-15T17:22:41.031Z","ev":"call","phase":"end",
 "conn":"c-7f3a","client":{"kind":"mcp","name":"pipelinecraft-mcp","mcpClient":"claude-code","profile":"mcp-claude","peerPid":18320},
 "method":"hub.call","target":"lightcraft/app.export","resolved":"lightcraft@p1/app.export","tier":"C",
 "effects":{"classes":["write-user-file"],"destructive":false,"source":"curated"},
 "decision":"confirm-human","confirmedBy":"approval:ap-19","dryRun":false,
 "params":{"dir":"F:\\Exports\\Shoot-0412","format":"jpeg","longEdge":2048},
 "paramsDigest":"b3:5c1e…(32 hex)",
 "result":{"ok":true,"ms":1840,"files":["F:\\Exports\\Shoot-0412\\IMG_0412.jpg"]},
 "error":null}
```

On failure `result` is null and `error` holds `{code, origin, ran}` from A4 §6.5 (no `message` text from the app beyond 256 characters).

### 11.5.4 Redaction

| Data                                                                                                                                                                             | Rule                                                                                                                                                     |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------|
| Hub token, app tokens, WordCraft `key`, `auth` params, PdfCraft control-file contents                                                                                            | Never recorded. The `hello` record has `auth: "ok"` / `"bad"` only                                                                                       |
| Any params key at any depth matching `token`, `key` (top level or inside `auth`), `password`, `passphrase`, `secret`, `credential`, `cookie`, `authorization` (case-insensitive) | `"[redacted]"`. Exception: `ui.key {"key": "Enter"}` is denied for MCP anyway; for other profiles a one-character or named-key value is kept             |
| Base64 and binary-ish fields (`data`, `base64`, `thumbnailBase64`, PNG results, WordCraft `file.open` `data`, DesignCraft `file.place` base64)                                   | `{"len": n, "b3": "<32 hex>"}`                                                                                                                           |
| Document text (`text`, `story`, `content`, `type_text`, `set_story_text`)                                                                                                        | `{"len": n, "b3": "…"}`. Private writing doesn't belong in a security log                                                                                |
| Script code (`script.run` `code`, DesignCraft `script.run` `text`)                                                                                                               | **Kept**, up to 16 KiB, then digested. It *is* the action, and the first thing to read after an incident                                                 |
| Any other string over 1 KiB                                                                                                                                                      | First 256 characters + `{len, b3}`                                                                                                                       |
| Paths                                                                                                                                                                            | Kept as normalized wire paths (A5 §7.4.2); they're what the log is for. `--redact-paths` on export                                                       |
| App stderr captured by the hub (A3 §5.3)                                                                                                                                         | Not in the audit log. In the diagnostic log, lines matching `control token: [0-9a-f]{32,}` (A8-N10) or any 32–128 hex run after `token`/`key` are masked |

---

## 11.6 Threat model

STRIDE letters: **S**poofing, **T**ampering, **R**epudiation, **I**nformation disclosure, **D**enial of service, **E**levation of privilege. Stages are §16's: 0 See, 1 Ask, 2 Hand off, 3 Chain, 4 MCP, 5 Produce.

| ID  | Threat                                                                                                                                                        | STRIDE     | Today (static)                                                                                                                                                                                                                                                  | Mitigation                                                                                                                                                                                                                                                                                                                                                                                                                      | Owner                                                 | Stage |
|-----|---------------------------------------------------------------------------------------------------------------------------------------------------------------|------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------|-------|
| T1  | Malicious web page: CSRF `fetch` or DNS rebinding to a loopback port                                                                                          | S, T, E    | Eight apps close on the HTTP request line; SoundCraft runs the body line (O-B2, §11.3.2)                                                                                                                                                                        | Hub: B1–B3 on its own listener; B4–B5 for SoundCraft; AF_UNIX in v1.x (L6). Upstream: S1                                                                                                                                                                                                                                                                                                                                        | Claude (hub); SoundCraft (S1)                         | 0     |
| T2  | Rogue local process, same user: reads `hub.json` and token files, calls apps directly, claims any `kind`                                                      | S, T, I, E | Same as today without a hub: it can already reach every app and read every file Paul can                                                                                                                                                                        | **Out of scope for prevention** (the user account is the boundary). The hub limits blast radius for *honest* clients: policy per profile, `tty: false` → script profile, human approvals only from a console, audit log. Peer-exe check (A8-O3) raises the bar                                                                                                                                                                  | Paul (accepts); Claude                                | 0–4   |
| T3  | Other local user (second account, service account) connects to loopback ports                                                                                 | S, I, E    | Six apps have no auth, so another user can drive them (§11.1.1). PhotoCraft/PdfCraft/WordCraft tokens sit in files that inherit folder ACLs (O12)                                                                                                               | Hub: DACL check and repair DG1–DG5; all hub secrets under the checked dir; tier rules never send secrets without K2. Upstream: S2 (auth for the six), S3 (Windows ACLs)                                                                                                                                                                                                                                                         | Claude (hub); upstream (S2, S3)                       | 0     |
| T4  | Prompt-injected MCP client: an asset filename, XMP description, PDF/DOCX text, OTIO metadata or app reply says "export everything to …" or "enable scripting" | T, I, E    | n/a (no hub). Each app's MCP has hand-kept hints; LightCraft marks every generated tool destructive (03a §3a.3)                                                                                                                                                 | `mcp` profile defaults (§11.4.2): human approval for user-file and destructive writes, network denied, target denies (`ui.*` input, `ui.clipboard`, `prefs.set scripting.*`), roots + deny-paths, rate limits, P7 (policy not writable). A7: mark app-returned text as untrusted data in tool results                                                                                                                           | Claude (hub); A7 (§10)                                | 4     |
| T5  | Malicious asset file reaches an app via handoff (crafted PSD/PDF/EPS/IDML/OTIO, or a script file)                                                             | T, D, E    | Apps parse hostile formats in-process (never-crash rule per app). VectorCraft has an EPS interpreter (`vectorcraft` `crates/eps/src/import/interp.rs`); EffectCraft runs `.jsx`                                                                                 | Hub: hands over paths only and parses no asset content beyond hashing (A5 §7.1.3); `asset.open` / Open In refuse script types (`.jsx`, `.js`, `.jsonl` scripts) unless confirm (human); headless thumbnail children run with read root = the asset folder, write root = `derived\`, killed on timeout (A3 job object); no GUI auto-restart, headless restart ≤ 3 per 60 s (A3 §5.4); results ≤ 32 MiB, never decoded by the hub | Claude (hub); each app (parsers)                      | 2     |
| T6  | Path traversal through AssetRef or confinement differences between apps                                                                                       | T, E       | PhotoCraft's root check is textual and case-sensitive past the drive (A5-O3) but opens through cap-std; VectorCraft canonicalizes; FilmCraft and LightCraft have no confinement; DesignCraft/WordCraft/SoundCraft/EffectCraft confine headless only (A5 §7.0.2) | The hub is the gate for apps without confinement: A5 N1–N8 (refuse `..`, ADS, device paths, verbatim forms stripped) plus profile roots; write checks walk components for links/junctions (§11.4.2 Roots); never a drive-root automation root; the strictest of hub roots and app roots applies; fuzz the validator (A5, §13.2)                                                                                                 | Claude (hub); A5; PhotoCraft (A5-O3)                  | 2     |
| T7  | Stale manifest or port takeover: another process holds the port the app failed to bind (A3-F2), or a pid was reused                                           | S, I       | Most apps keep running without control after a bind failure; PdfCraft never removes its control file (A3-F4)                                                                                                                                                    | A3 identity probe + listener-owner check (A8-R3) before any secret or trust; tiers (§11.3.5); M1–M3. Upstream: exit on bind failure (A3-O4), remove control file on exit (A3-O5)                                                                                                                                                                                                                                                | Claude (hub); upstream                                | 0     |
| T8  | Fake hub: a process listens on the port from a stale `hub.json` and collects the hub token                                                                    | S, I       | n/a                                                                                                                                                                                                                                                             | Clients check pid alive + exe + listener owner before `hub.hello` (§11.2.2); token rotates at each start; challenge-response hello in protocol 1.1 (A8-O5)                                                                                                                                                                                                                                                                      | Claude (hub, craftctl, shim)                          | 0     |
| T9  | `localhost` / `::1` confusion leaks an app secret to a squatter on `[::1]:<port>`                                                                             | S, I       | PhotoCraft bridge sends its token to whatever answers `localhost`/`::1` (A8-N2)                                                                                                                                                                                 | Hub: L3 (literal `127.0.0.1` only). Upstream: S5                                                                                                                                                                                                                                                                                                                                                                                | Claude (hub); PhotoCraft, FilmCraft, EffectCraft (S5) | 0     |
| T10 | Secrets leak through logs, stderr, argv or events                                                                                                             | I          | PhotoCraft prints its token to stderr without a token file (A8-N10)                                                                                                                                                                                             | K1, K5; hub always passes `--control-token-file`; stderr masking (§11.5.4); audit redaction. Upstream: S4                                                                                                                                                                                                                                                                                                                       | Claude (hub); PhotoCraft (S4)                         | 0     |
| T11 | Command effects misclassified: an uncurated command, a meta-command (`script.run`), or a pref that unlocks more (EffectCraft scripting)                       | E          | No app publishes complete command effects (O4, O-B5)                                                                                                                                                                                                            | A4's worst-case default; P3, P5; curated rows for A8-N6/N7/N8; `craftctl profile check` lists defaults. Upstream: O4/O-B5 (effects field), S9                                                                                                                                                                                                                                                                                   | Claude (profiles); A4; upstream                       | 1–4   |
| T12 | UNC path in a handoff (from an imported OTIO, a document, or an MCP call) makes an app connect to `\\host\share` and send Windows credentials                 | I          | A5 allows UNC as-is (§7.4.1); FilmCraft resolves `file://server/share` URLs (A5 §7.2.5)                                                                                                                                                                         | Hub policy `unc_hosts = [...]` allowlist (Paul's NAS). A UNC path to any other host is `path_denied` for `mcp` and script profiles and confirm (caller) for interactive ones; OTIO the hub writes never carries a non-allowlisted host                                                                                                                                                                                          | Claude (hub); A5; Paul (list hosts)                   | 2     |
| T13 | Flooding: many connections, huge lines, endless batches, crash loops                                                                                          | D          | Apps cap lines (1–4 MiB, SoundCraft ~86 MiB) and connections (16); PdfCraft has no cap after auth (A4-F1)                                                                                                                                                       | A4 framing caps (4 KiB pre-hello, 4 MiB, 32 connections, 32 in flight); MCP rate limits; adapters never send more than the app's cap; restart limits (A3)                                                                                                                                                                                                                                                                       | Claude (hub); PdfCraft (A4-F1)                        | 0     |
| T14 | Repudiation: "who exported over my file?"                                                                                                                     | R          | No app keeps a control-channel log                                                                                                                                                                                                                              | §11.5 audit log with connection, profile, peer pid, decision and `confirmedBy`                                                                                                                                                                                                                                                                                                                                                  | Claude (hub)                                          | 1     |

---

## Upstream security opportunities (S1–S12)

Each is phrased as one small PR to the storytold app (or Paul's fork first, per D6). The hub works without any of them; each one removes a hub-side workaround. IDs from other sections are cross-referenced, not re-numbered.

| ID  | App(s)                                                                   | Small PR                                                                                                                                                                                                                                                                                                                                                                                                           | Removes hub workaround                    | Owner (PR)                      | Status |
|-----|--------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------|---------------------------------|--------|
| S1  | SoundCraft                                                               | Port DesignCraft's `parse_line` + `serve_lines` guard (`designcraft` `apps/designcraft/src/control_server.rs:80-157`) into `soundcraft` `apps/soundcraft/src/control_server.rs:97-137`, keeping SoundCraft's 86 MiB `MAX_LINE`. Add the same two tests LightCraft has (`lightcraft` `control_server.rs:288`, `:359`). Same as O-B2                                                                                 | B4, B5                                    | SoundCraft upstream; Paul files | open   |
| S2  | LightCraft, FilmCraft, DesignCraft, EffectCraft, VectorCraft, SoundCraft | Opt-in `--control-token-file <path>` + `auth` first frame, copied from PhotoCraft (`photocraft` `crates/automation/src/security.rs:74-250`), one PR per app, LightCraft and FilmCraft first (03a O2). Better: through `craft-control-server` (O-B1) so the code exists once                                                                                                                                        | Tiers B/C become A                        | each app; Claude drafts         | open   |
| S3  | PhotoCraft, PdfCraft, WordCraft                                          | On Windows, create token/control/key files with an owner-only DACL (or in a folder that has one), and check the owner when reading them (the Unix branch of `pdfcraft` `apps/pdfcraft-cli/src/main.rs:868-905` on Windows). Sites: `photocraft` `crates/automation/src/security.rs:140-153`; `pdfcraft` `apps/pdfcraft/src/main.rs:474-478`; `wordcraft` `apps/wordcraft/src/control_server.rs:93-94`. Same as O12 | DG checks on app folders                  | each app                        | open   |
| S4  | PhotoCraft                                                               | With neither token nor token file, write the token to a default owner-only file and print its **path**, not the token (`photocraft` `apps/photocraft/src/main.rs:286-287`)                                                                                                                                                                                                                                         | stderr masking                            | PhotoCraft                      | open   |
| S5  | PhotoCraft, FilmCraft, EffectCraft                                       | Bridge clients: map `localhost` to `127.0.0.1` and drop `::1`/`[::1]`, because the servers bind IPv4 only (`photocraft` `crates/automation/src/bridge.rs:46-53`; `filmcraft` `crates/automation/src/bridge.rs:35`; `effectcraft` `crates/automation/src/bridge.rs:26`). WordCraft's MCP client could do the same (`wordcraft` `crates/mcp/src/backend.rs:70`)                                                      | L3 (protects Paul's direct use)           | each app                        | open   |
| S6  | five apps                                                                | Exit non-zero (or refuse to open) when `--control` was asked for and the bind failed, and print `local_addr()`. Same as A3-O4 / O7; listed here for its security effect (no silent takeover)                                                                                                                                                                                                                       | listener-owner check stays, tiers simpler | each app                        | open   |
| S7  | PdfCraft                                                                 | Cap post-auth lines at 4 MiB (A4-F1, `pdfcraft` `crates/ui-egui/src/control.rs:808`)                                                                                                                                                                                                                                                                                                                               | adapter-side cap                          | PdfCraft                        | open   |
| S8  | PdfCraft                                                                 | Remove the control file on clean exit (A3-O5)                                                                                                                                                                                                                                                                                                                                                                      | M1 stale handling                         | PdfCraft                        | open   |
| S9  | EffectCraft                                                              | Refuse `prefs.set` for `scripting.*` keys when the request comes over control or MCP, or require a UI confirmation, so automation can't turn on script network/file access (A8-N7; `effectcraft` `crates/engine/src/commands/settings.rs:21-27`)                                                                                                                                                                   | target deny rule                          | EffectCraft                     | open   |
| S10 | WordCraft                                                                | Return "unknown method" for names outside the method table (A3-O6), and guard control `file.open` on a dirty document (O5)                                                                                                                                                                                                                                                                                         | adapter quirk, curated row                | WordCraft                       | open   |
| S11 | WordCraft                                                                | Move `control-key.<instance>` from roaming `%APPDATA%` to `%LOCALAPPDATA%` (A8-N12): a per-process secret shouldn't roam                                                                                                                                                                                                                                                                                           | DG warning                                | WordCraft                       | open   |
| S12 | all                                                                      | Publish command effects in `command.list` (O4, O-B5) and an `app.hello` with a per-process `instance` nonce (A3-O3). The nonce, also written to the manifest, is a cheap identity proof for unauthenticated apps until S2 lands                                                                                                                                                                                    | P3 defaults, identity probe               | each app; craft-control-proto   | open   |

## Findings and recommendations from this section

| ID    | Scope                  | Opportunity                                                                                                | Suggested fix                                                                                                                                                                      | Owner                          | Status |
|-------|------------------------|------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------|--------|
| A8-R1 | §11.1 binding          | Apps bind IPv4; clients that dial `localhost` can reach a squatter on `::1`                                | L1–L6: bind and dial the literal `127.0.0.1`; reject other hosts in config and manifests                                                                                           | Claude (hub)                   | open   |
| A8-R2 | §11.2.3 DACL           | Token files on Windows inherit folder ACLs (O12)                                                           | Folder-level DACL check and repair DG1–DG5 with `icacls` (SDDL via `/save`), fail with no secrets written                                                                          | Claude (hub); Paul (verify)    | open   |
| A8-R3 | §11.3.4 identity       | A3's identity probe can be replayed by an impostor                                                         | Listener-owner check (port → pid) before any secret and for tiers A–C                                                                                                              | Claude (hub)                   | open   |
| A8-R4 | §11.3.5 attach policy  | A3's open question 2                                                                                       | Attach to hand-started unauthenticated apps at tier C (owner-verified); MCP gets one step tighter there; C-read without the owner check; D refused                                 | Paul (decide); Claude          | open   |
| A8-R5 | §11.4 permissions      | No caller-specific limits exist anywhere today                                                             | Profiles + P1–P10 + the defaults table; policy file only editable by Paul                                                                                                          | Paul (review defaults); Claude | open   |
| A8-R6 | §11.5 audit            | No app logs control-channel writes                                                                         | JSONL audit log, fail-closed for user-file/network/destructive calls, 30 days / 256 MiB, redaction table                                                                           | Claude (hub)                   | open   |
| A8-F1 | A4 §6.4.1 (integrator) | A4 says no local `external-network` command exists; three do (A8-N6)                                       | Add curated rows to `profiles/effectcraft/effects.toml` and `profiles/lightcraft/effects.toml`; fix the A4 table text                                                              | Claude (profiles); integrator  | open   |
| A8-F2 | A4 §6.4 (integrator)   | Meta-commands (`script.run`, `file.runScript`, batch tools) have the effects of what they run              | Rule P5: expand where parseable, else `source: default`                                                                                                                            | A4; Claude                     | open   |
| A8-F3 | A4 §6.5.2 (integrator) | New codes this section needs                                                                               | Add `listener_mismatch` (adapter, no), `audit_unavailable` (hub, no), `state_dir_insecure` (hub, no), `policy_invalid` (hub, no); `trust_insufficient` (hub, no) for tier refusals | A4; integrator                 | open   |
| A8-F4 | A4 §6.5 / A7 §10.3     | A4's `confirmation_required` always returns the token to the caller, so a model could confirm its own call | Rule P8 "human" kind: `data.approval` instead of `data.confirm`, approved only from a console `craftctl` or the panel                                                              | A4, A7; Paul                   | open   |
| A8-F5 | A4 §6.4 / A5 §7.5      | Writes into hub-owned folders would prompt like user-file writes                                           | Rule P4 (hub-root writes decided as `write-app-state`)                                                                                                                             | A4, A5; Claude                 | open   |
| A8-F6 | A3 §5.3 (integrator)   | `attach_unauthenticated = "probed-or-launched"` is superseded                                              | Replace with `[attach] min_tier` (A8-R4) and `trust` in `hub.apps`                                                                                                                 | A3; integrator                 | open   |
| A8-O1 | §11.2.2 / D-S1         | Rule 7 ("never in files") vs per-process tokens                                                            | Decide D-S1; record the reading in AssetCraft `AGENTS.md`                                                                                                                          | Paul (D-S1)                    | open   |
| A8-O2 | §11.2.3                | `icacls` spawning works but is clumsy                                                                      | Evaluate a permissive crate with a safe ACL API; switch when one passes licence and maintenance review                                                                             | Claude                         | open   |
| A8-O3 | §11.3.4, §11.4.3       | Port → pid and peer pid need a TCP-table source                                                            | Pick `netstat2` or `listeners` (licence check), fallback `netstat -ano`; then add `peer_exe` to profile matching                                                                   | Claude                         | open   |
| A8-O4 | §11.5.1                | `seq` shows a deleted line but not an edited one                                                           | BLAKE3 chain per record (`prev`), checked by `craftctl audit verify`                                                                                                               | Claude                         | open   |
| A8-O5 | §11.2.2, T8            | The hub token travels in clear on loopback in `hub.hello`                                                  | Protocol 1.1 (additive, A4 rule D-1): `auth: {scheme: "hmac"}` challenge-response so the token never crosses the socket                                                            | A4; Claude                     | open   |

## Not verified

Everything above is a static read; nothing was built or run. The Windows and browser behaviour marked "(general knowledge)" needs these checks on Paul's machine. Use a scratch profile and throwaway files.

**Next action (Paul):**

1. **SoundCraft guard (O-B2, S1), with a browser-shaped request.** Start SoundCraft and note the port it prints (03b Not verified step 6 shows how), then send exactly what a browser's `no-cors` POST sends, with a harmless read in the body.
   ```powershell
   Set-Location F:\Script-Projects\ArtCrafting\soundcraft
   $p = Start-Process .\target\release\soundcraft.exe -ArgumentList '--control','0','--no-audio' -RedirectStandardOutput "$env:TEMP\sc-out.txt" -PassThru
   Start-Sleep 5; $port = [int]((Get-Content "$env:TEMP\sc-out.txt" | Select-String 'SOUNDCRAFT_CONTROL_PORT=(\d+)').Matches[0].Groups[1].Value)
   $c = [Net.Sockets.TcpClient]::new('127.0.0.1', $port); $s = $c.GetStream()
   $w = [IO.StreamWriter]::new($s); $r = [IO.StreamReader]::new($s); $w.NewLine = "`r`n"
   $w.WriteLine('POST / HTTP/1.1'); $w.WriteLine("Host: 127.0.0.1:$port"); $w.WriteLine('Content-Type: text/plain;charset=UTF-8')
   $w.WriteLine('Origin: https://example.com'); $w.WriteLine('')
   $w.Write("{`"id`":7,`"method`":`"engine.parity`"}`n"); $w.Flush()
   1..5 | ForEach-Object { $r.ReadLine() }; $c.Close(); Stop-Process $p.Id
   ```
   Expected if O-B2 is real: four `bad JSON` lines (one per header; the blank line is skipped), then `{"id":7,"ok":true,…}`. Repeat against DesignCraft (`designcraft.exe --control 47650`): expected one error line, then the connection closes (the next `ReadLine` returns nothing).
2. **Same check from a real browser (optional).** With SoundCraft on `$port`, open any `https://` page, press F12, and run `fetch("http://127.0.0.1:<port>/",{method:"POST",mode:"no-cors",body:'{"id":1,"method":"engine.parity"}\n'})`. Expected: the console shows an opaque or failed response either way. The proof is in step 1; this step only shows whether your browser version blocks or prompts for public → local requests.
3. **IPv6 squatting (§11.1.3, T9).** Start LightCraft on 47651, then in a second window:
   ```powershell
   Set-Location F:\Script-Projects\ArtCrafting\lightcraft
   $l = [Net.Sockets.TcpListener]::new([Net.IPAddress]::IPv6Loopback, 47651); $l.Start()
   $c = [Net.Sockets.TcpClient]::new('localhost', 47651); $c.Client.RemoteEndPoint; $c.Close(); $l.Stop()
   ```
   Expected: `$l.Start()` succeeds while LightCraft holds `127.0.0.1:47651`, and `RemoteEndPoint` shows `[::1]:47651`, meaning `localhost` reached the squatter. If `RemoteEndPoint` is `127.0.0.1:47651`, resolution order differs on your machine; L3 stays either way.
4. **Port reuse (T7).** With LightCraft still on 47651:
   ```powershell
   Set-Location F:\Script-Projects\ArtCrafting\lightcraft
   $s = [Net.Sockets.Socket]::new('InterNetwork','Stream','Tcp'); $s.SetSocketOption('Socket','ReuseAddress',$true)
   try { $s.Bind([Net.IPEndPoint]::new([Net.IPAddress]::Loopback, 47651)); 'BOUND (hijack possible)' } catch { $_.Exception.Message }; $s.Close()
   ```
   Expected: "Only one usage of each socket address…" (in use). `BOUND` means a same-user process can share the port, and the listener-owner check (A8-R3) becomes essential rather than defence in depth.
5. **Listener owner (A8-R3).** Expected: one row whose `OwningProcess` equals LightCraft's pid.
   ```powershell
   Set-Location F:\Script-Projects\ArtCrafting
   Get-NetTCPConnection -LocalAddress 127.0.0.1 -LocalPort 47651 -State Listen | Select-Object LocalPort, OwningProcess
   (Get-Process lightcraft).Id
   netstat -ano -p TCP | Select-String ':47651'
   ```
6. **DACL as SDDL (DG2–DG3).** Expected: one `D:` string whose entries name only `SY`, `BA` and your `S-1-5-21-…` SID, plus the same SID from `whoami`.
   ```powershell
   Set-Location F:\Script-Projects\ArtCrafting
   New-Item -ItemType Directory -Force "$env:LOCALAPPDATA\PipelineCraft" | Out-Null
   icacls "$env:LOCALAPPDATA\PipelineCraft" /save "$env:TEMP\pc-acl.txt" | Out-Null
   Get-Content "$env:TEMP\pc-acl.txt" -Encoding Unicode
   whoami /user /fo csv /nh
   icacls "$env:APPDATA\WordCraft" /save "$env:TEMP\wc-acl.txt" | Out-Null; Get-Content "$env:TEMP\wc-acl.txt" -Encoding Unicode
   ```
7. **DACL repair on a scratch folder (DG4).** Expected: after the second command, the SDDL starts `D:PAI` and lists exactly two entries.
   ```powershell
   Set-Location $env:TEMP
   New-Item -ItemType Directory -Force "$env:TEMP\pc-acl-test" | Out-Null
   $sid = (whoami /user /fo csv /nh).Split(',')[1].Trim('"')
   icacls "$env:TEMP\pc-acl-test" /inheritance:r /grant:r "*${sid}:(OI)(CI)F" "*S-1-5-18:(OI)(CI)F"
   icacls "$env:TEMP\pc-acl-test" /save "$env:TEMP\pc-acl-test.txt" | Out-Null; Get-Content "$env:TEMP\pc-acl-test.txt" -Encoding Unicode
   ```
8. **EffectCraft pref over control (A8-N7, S9).** Start `effectcraft.exe --control 47652`, then:
   ```powershell
   Set-Location F:\Script-Projects\ArtCrafting\effectcraft
   $c = [Net.Sockets.TcpClient]::new('127.0.0.1', 47652); $s = $c.GetStream()
   $w = [IO.StreamWriter]::new($s); $r = [IO.StreamReader]::new($s); $w.AutoFlush = $true
   $w.WriteLine('{"id":1,"method":"engine.execute","params":{"command":"prefs.set","params":{"key":"scripting.allowScriptsWriteFiles","value":true}}}'); $r.ReadLine()
   $w.WriteLine('{"id":2,"method":"engine.execute","params":{"command":"prefs.set","params":{"key":"scripting.allowScriptsWriteFiles","value":false}}}'); $r.ReadLine(); $c.Close()
   ```
   Expected: `"ok":true` for both, confirming that automation can unlock script network access. The second line turns it off again.
9. **Other local user (T3, optional).** From a second Windows account (`runas /user:<other> powershell`), connect to LightCraft's port with the `01` Not verified step 3 script, then run `Get-ChildItem C:\Users\<paul>\AppData\Local\PipelineCraft`. Expected: the LightCraft call **succeeds** (no auth, by design today) and the folder listing is **denied**.
10. **PhotoCraft root and junctions (T6).** Inside PhotoCraft's read root (A5 Not verified step 6 setup), `New-Item -ItemType Junction -Path <root>\j -Target C:\Windows`, then send `app.open {"path":"j/win.ini"}`. Expected: refused (cap-std should not follow a junction out of the root). If it opens, file it with A5-O3.

## Open questions

1. **D-S1:** hub token inline in `hub.json` with an owner-only DACL (recommended), or in Credential Manager with no secret on disk? (Paul)
2. **Attach tiers (A8-R4):** is tier C (hand-started, owner-verified) the right default, and should MCP connections be one step tighter on tier C instances, as proposed? (Paul)
3. **Human approvals (P8):** is a console `craftctl approve <id>` acceptable as the only approval surface until the status panel exists, or should v1 show a Windows notification? (Paul, with A7)
4. **MCP defaults:** should `write-app-state` from MCP (for example Open In) need the MCP client's own approval (proposed), or run freely since it doesn't touch files? This sets how chatty AC4 is. (Paul)
5. **Audit text:** keep script code in the audit log (proposed) while hashing document text: right trade-off for privacy vs forensics? (Paul)
6. **UNC allowlist (T12):** which NAS hosts should be allowed by default? (Paul; links to A5 open question 5)

## Inputs for downstream agents

- **A3 (§4–5), integrator:** replace `attach_unauthenticated` with `[attach] min_tier` and report `trust` per instance in `hub.apps` (A8-R4). Attach gets one more step between handshake and identity: the listener-owner check (A8-R3). The run dir is read only if it passes DG3 (M2). Hub-written manifests use `token_file`; inline tokens are accepted only under M3. SoundCraft config gets `control_policy = "headless-preferred"` (B4).
- **A4 (§6):** new error codes in A8-F3. Human-kind confirmations (`data.approval`) per P8. A dry run may issue a caller-kind token for its exact plan, 10 minutes, single use (P9; answers your open question 5). TCP loopback in v1 is confirmed with B1; AF_UNIX in v1.x (answers your open question 4). Curated `external-network` rows (A8-F1) and the meta-command rule P5 (A8-F2). Clients must run the pid/listener check before sending `hub.hello` (§11.2.2). `hub.token.rotate` is proposed as a hub command.
- **A5 (§7):** the hub never sets an automation root at a drive root or the user profile; write-root checks walk components for links/junctions; UNC hosts need an allowlist for non-interactive profiles (T12); hub-root writes are decided as `write-app-state` (P4).
- **A6 (§8–9):** audit records exist for every non-read recipe step; a recipe run from a script needs a `[[grant]]` or a human approval for user-file writes; the approval wait fits `hub.job.wait`-style polling.
- **A7 (§10, §12):** the shim copies `clientInfo.name` into `hello.client.mcpClient`; `craftctl` reports `tty`. New `craftctl` verbs: `approve <id>`, `policy reload|check`, `audit [--since] [--json]`, `audit export --redact-paths`, `attach <app> --port N`, `hub rotate-token [--drop]`, and later `token issue`. Mark app-returned text in MCP tool results as untrusted data (T4). MCP `ui.*` input methods and `ui.clipboard` are denied by default.
- **A9 (§13, §14, §16):** fuzz targets: the pre-hello line handler, `policy.toml` loader, audit redaction, A5's path validator. Contract tests: the deny rules in §11.4.2 hold for every profile; a fake app that ignores the first line (SoundCraft-style) must still be reachable by the hub but flagged `browser-reachable`. Upstream list S1–S12, smallest first: S1, S4, S5, S7, S8, S11, then S2, S3, S9, S10, S12. Stage mapping is the threat table's last column; DACL, listener-owner and token work are Stage 0.
- **A10 (§15):** new decision D-S1 (ephemeral tokens in owner-only files vs keyring). A8-R4 (attach tiers) and P8 (human approvals) are candidate decisions.
