# PipelineCraft (working name): §7 Asset reference model

_2026-10-15 · spec draft v0 · status: draft (agent A5)_

- **Scope:** how two Craft apps agree they mean the same asset (§7.1), what a handoff carries (§7.2), who may write XMP and sidecars (§7.3), Windows path rules and per-app path translation (§7.4), and where derived and temp files live and die (§7.5). Rust type sketches are in §7.6.
- **Project goal it serves:** AssetCraft → StoryCraft → FilmCraft (and later PrepressCraft) handoffs that survive renames, copies, long paths and per-app file confinement, so acceptance cases AC1 (Open In), AC2 (Thumbnails) and AC3 (Plan → FilmCraft) from §1.4 can pass.
- **Immediate area of improvement:** today every app takes a bare path string, and four apps hash files four different ways that can't be compared. This section defines one asset ref that degrades to a plain path for every existing command, plus one hash the hub and AssetCraft share.

**Evidence basis.** Static read of the shallow clones in `/home/claude/src` at the commits below. Nothing was built or run. Wave-1 IDs (A1-*, O1–O14 from §3a, O-B1–O-B13 from §3b) are cited, not re-derived. Where HEAD differs from a wave-1 claim, that is said.

| Repo         | Commit    | Repo          | Commit    |
|--------------|-----------|---------------|-----------|
| `assetcraft` | `3a41b2b` | `designcraft` | `7ed5697` |
| `photocraft` | `fd28da5` | `effectcraft` | `2c08f67` |
| `lightcraft` | `38adc17` | `vectorcraft` | `0946bc5` |
| `filmcraft`  | `9ed9650` | `soundcraft`  | `3a28a65` |
| `pdfcraft`   | `ac89a11` | `wordcraft`   | `3292ba0` |

---

## 7.0 What exists today (research)

### 7.0.1 How AssetCraft identifies assets

| Fact                                                                                                                                                                                                                                  | Cite (`assetcraft` @ `3a41b2b`)                              |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------|
| Identity is the **path**. The selection is a `HashSet<PathBuf>` plus anchor and focus, reported as `{selected: [path], focus}`                                                                                                        | `crates/engine/src/select.rs:20-24`; `commands.rs:561`       |
| Every path-taking command takes strings: `select.set {paths}`, `item.probe {path}`, `file.openIn {app, paths?}`                                                                                                                       | `crates/engine/src/commands.rs:95, 123, 131`                 |
| `file.openIn` refuses non-Unicode paths and paths without a root; ≤ 100 files and ≤ 30,000 chars per launch                                                                                                                           | `commands.rs:450-456`; `open_in.rs:159, 162`                 |
| Path equality ignores case on Windows (`to_lowercase`), exact elsewhere; `C:` means `C:\`; quotes trimmed                                                                                                                             | `crates/engine/src/paths.rs:13-40`                           |
| The only hash is the **thumbnail cache key**: SipHash-1-3-128 (fixed keys) over renderer version, side bucket, size, mtime, first and last 64 KiB. It is a cache key, not an identity (it includes mtime and renderer version)        | `crates/thumbs/src/disk.rs:3-12, 49, 88-104`; `hash.rs:1-30` |
| Cache layout `<dir>/<2 hex>/<32 hex>.png`, temp + rename, 1 GiB budget pruned to 80 %, touches only its own name shapes                                                                                                               | `disk.rs:1-26, 52`                                           |
| Cache folder `%LOCALAPPDATA%\AssetCraft\thumbs`                                                                                                                                                                                       | `crates/ui-egui/src/prefs_file.rs:40-46, 70-79`              |
| **No catalog id exists.** The catalog (append-only op log) is Stage 3; XMP ratings/labels/keywords via `lightcraft-meta` are Stage 2; `crs:` sidecars Stage 3.5                                                                       | `docs/assetcraft-plan.md:59-61, 149-166, 174-189`            |
| "Nothing writes to the user's files before Stage 2"                                                                                                                                                                                   | `AGENTS.md` non-negotiable 5                                 |
| Near-duplicate finder planned on "content hash plus perceptual hash"                                                                                                                                                                  | `docs/assetcraft-plan.md:152-153`                            |
| Planned L3 `boards`, `interchange` (storyline model, OTIO) and L4 `bridge`                                                                                                                                                            | `AGENTS.md:35-36`                                            |
| Storyline Builder blocks: a field table (id, title, purpose, duration, description, requirements, assets, timing rules, entry/exit, continuity, source ref, status); no JSON shape yet. Exact rational time "like FilmCraft's `Tick`" | `docs/storyline-builder.md:50-70`                            |
| OTIO plan: one `Clip` per layer, `source_range` for trim, `Gap.1` for empty slots, `Marker.2` for notes; missing media imports offline                                                                                                | `docs/storyline-builder.md:206-211`                          |
| `lightcraft-meta` is already a dependency (thumbs, raw orientation)                                                                                                                                                                   | `Cargo.toml:42`; `crates/thumbs/src/decoders/raw.rs:52`      |
| Test fixtures that matter here: `environments/dupes/snow_clearing_plate_copy.jpg` (exact copy), `edge-cases/Réaper ünïcode name.png`                                                                                                  | `test_data/README.md:18`; `test_data/library/`               |

### 7.0.2 How the siblings identify, open and export files

**Fingerprints in the suite (four, mutually incompatible):**

| App         | What                                                                               | Algorithm and input                                                | Cite                                                                                                                                                                            |
|-------------|------------------------------------------------------------------------------------|--------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| AssetCraft  | Thumbnail cache key                                                                | SipHash-1-3-128, keys "AssetCra/ft thumb"; size, mtime, 2 × 64 KiB | `assetcraft` `crates/thumbs/src/hash.rs:24-30`, `disk.rs:88-104` @ `3a41b2b`                                                                                                    |
| LightCraft  | `Photo.content_hash` for duplicate detection, preview keys, and **move detection** | SipHash-1-3-128 of the **whole file**, different fixed keys        | `lightcraft` `crates/preview/src/hash.rs:1-31, 59-61`; `crates/engine/src/files.rs:135`; `crates/catalog/src/model.rs:268-270`; `crates/engine/src/sync.rs:272-305` @ `38adc17` |
| FilmCraft   | `MediaIdentity {size, fingerprint}` ("Clip ID") gating relink                      | 64-bit multiply–xorshift over size + first 1 MiB + last 1 MiB      | `filmcraft` `crates/engine/src/relink.rs:3-13, 28-31`; `crates/project/src/lib.rs:230-235` @ `9ed9650`                                                                          |
| PhotoCraft  | Content-addressed tiles and blobs inside `.pcraft`                                 | BLAKE3 (`pure` feature)                                            | `photocraft` `crates/format/src/store.rs:53-55`; `crates/engine/Cargo.toml:34` @ `fd28da5`                                                                                      |
| DesignCraft | Link status `ok/modified/missing`                                                  | No hash: compares file length, then bytes, with the embedded copy  | `designcraft` `crates/engine/src/cmd/links.rs:1-6, 92-102` @ `7ed5697`                                                                                                          |

**XMP document ids.** No suite app reads or writes `xmpMM:DocumentID` / `InstanceID` as identity. LightCraft and its merge code only *preserve* `xmpMM:History` (`lightcraft` `crates/meta/src/xmp_merge.rs:14`; `crates/engine/src/sidecar.rs:10`); VectorCraft's PDF/X writer emits only `xmpMM:VersionID` (`vectorcraft` `crates/pdf/src/pdfx.rs:141-145`). PhotoCraft's `documentId` is an internal session id, not XMP (`photocraft` `crates/ui-egui/src/prefs_ui.rs:293`). `lightcraft-meta` exposes a generic `parse_xmp` → `XmpData` that could read them (`lightcraft` `crates/meta/src/xmp.rs:67, 457`; `lib.rs:31`), but `extract()`'s `Metadata` has no field for them (`lib.rs:80-129`).

**XMP and sidecar writers today:**

| App        | Sidecar name                                                                          | Write policy                                                                                                                                                           | Cite                                                                                   |
|------------|---------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------|
| LightCraft | `<stem>.xmp` (default) or `<file>.xmp`; raw wins a shared stem                        | Merge: owns a fixed property list (`OWNED`, `OWNED_IF_STATED`), keeps everything else byte for byte, backs up an unreadable sidecar, atomic write; optional auto-write | `lightcraft` `crates/engine/src/sidecar.rs:1-24, 39-60, 361, 384, 599-603` @ `38adc17` |
| FilmCraft  | `<stem>.xmp` (`with_extension("xmp")`)                                                | `clip.updateMetadata`: writes a **whole new packet**; skips the file only if it lacks `xmp:CreatorTool="FilmCraft"`                                                    | `filmcraft` `crates/engine/src/project_tools.rs:183, 1066-1068, 1085-1106` @ `9ed9650` |
| AssetCraft | none until Stage 2 (planned: merge via `lightcraft-meta`, round-trip with LightCraft) | n/a                                                                                                                                                                    | `assetcraft` `docs/assetcraft-plan.md:149-157`                                         |
| Others     | none found (grep `xmp`, `sidecar`, static)                                            | n/a                                                                                                                                                                    | n/a                                                                                    |

**Open, import and export params (from §3a.8 and §3b.7b, with the path-relevant detail):**

| App         | Open / import (params)                                                                                                             | Takes bytes?                  | Cite                                                                                            |
|-------------|------------------------------------------------------------------------------------------------------------------------------------|-------------------------------|-------------------------------------------------------------------------------------------------|
| PhotoCraft  | control `app.open {path}` under `--automation-read-root`                                                                           | no                            | §3a PhotoCraft; `photocraft` `crates/ui-egui/src/control.rs:1217-1254`                          |
| LightCraft  | `library.import {paths:[abs], mode?: add\|copy\|move, album?}`; `library.browse {path}`                                            | no                            | §3a LightCraft; `lightcraft` `crates/engine/src/cmd/library.rs:54, 549`                         |
| FilmCraft   | `file.import {paths:[str], bin?, imageSequence?, replace?}` (also detects `.otio/.fcpxml/.xml/.edl/.aaf/.omf`); `file.open {path}` | no                            | `filmcraft` `crates/engine/src/commands.rs:1331-1380`; `crates/engine/src/interchange.rs:11-17` |
| PdfCraft    | control `ui.open {path}`; tool `doc_open {path}`                                                                                   | no                            | §3a PdfCraft                                                                                    |
| WordCraft   | `file.open {path, data?: base64, password?}`                                                                                       | **yes** (base64)              | §3a WordCraft; `wordcraft` `crates/engine/src/cmd/file.rs:68-71`                                |
| DesignCraft | `file.open {path}`; `file.place {path\|base64, …}`                                                                                 | **yes** (place)               | §3b DesignCraft; `designcraft` `crates/engine/src/cmd/file.rs:102`                              |
| EffectCraft | `file.open {path}`; `file.import {paths, …}`                                                                                       | no (render can return base64) | §3b EffectCraft; `effectcraft` `crates/engine/src/commands/file.rs:351, 738`                    |
| VectorCraft | `document.open {path}`; control `app.open {path}`; `file.place.info {path, thumbnail}`                                             | no (thumbnail returns base64) | §3b VectorCraft                                                                                 |
| SoundCraft  | `session.open {path}`; `file.import_audio {path\|paths, track?, at?}`                                                              | no                            | `soundcraft` `crates/engine/src/cmd/file.rs:12, 58` @ `3a28a65`                                 |

**Timeline interchange.** FilmCraft reads and writes OTIO JSON natively, plus FCP7 XML, FCPXML, EDL, AAF and OMF:

| Fact                                                                                                                                                                                                                              | Cite (`filmcraft` @ `9ed9650`)                                    |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------|
| OTIO schemas read and written: `Timeline.1`, `Stack.1`, `Track.1`, `Clip.1/2`, `Gap.1`, `Transition.1`, `Marker.1/2`, `ExternalReference.1`, `MissingReference.1`, `GeneratorReference.1`, time warps, `SerializableCollection.1` | `crates/interchange/src/otio.rs:1-12`                             |
| Import goes through `file.import {paths}`: content detection with the extension as a hint                                                                                                                                         | `crates/engine/src/commands.rs:1378-1379`; `interchange.rs:11-17` |
| Export: `file.exportInterchange {format: edl\|xml\|fcpxml\|otio\|aaf\|omf, path, sequence?}`                                                                                                                                      | `crates/engine/src/commands.rs:1445-1451`                         |
| Clip.2 uses `media_references.DEFAULT_MEDIA`; Clip.1's `media_reference` is read too                                                                                                                                              | `otio.rs:292-299, 622-627`                                        |
| `ExternalReference.target_url` → path via `file_url_to_path`; relative URLs resolve against the `.otio` folder                                                                                                                    | `otio.rs:692-694`; `crates/interchange/src/common.rs:60-79`       |
| Media items are de-duplicated by the **string** key `file:{path}`                                                                                                                                                                 | `otio.rs:696-698`                                                 |
| Only `metadata.filmcraft` is read on import; any other metadata namespace is dropped                                                                                                                                              | `otio.rs:74-76, 678`; `fill_media` `:751-770`                     |
| Exact time base: `TICKS_PER_SECOND = 254_016_000_000`                                                                                                                                                                             | `crates/time/src/lib.rs:20`                                       |
| EffectCraft exports timelines through the same formats (`file.exportTimeline`)                                                                                                                                                    | §3b EffectCraft (study line)                                      |

**File confinement (relevant to §7.4):**

| App         | Confinement                                                    | Applies to                                   | Comparison                                                                                                                                                                                                                           | Cite                                                                                                                                |
|-------------|----------------------------------------------------------------|----------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------|
| PhotoCraft  | `--automation-read-root` / `--automation-write-root` (cap-std) | GUI control **and** CLI/MCP                  | **Textual.** Absolute paths accepted only if they start with the root's text; drive prefix compared without case, **other components with case**; `\`, `:`, `.`, `..`, trailing dot/space, device names refused in the relative part | `photocraft` `apps/photocraft/src/main.rs:211-212`; `crates/automation/src/workspace.rs:24-31, 138-172, 174-200`                    |
| VectorCraft | same flags (or env)                                            | GUI with `--control`, and `mcp`              | Canonicalized (links followed), whole components, case-insensitive on Windows                                                                                                                                                        | `vectorcraft` `crates/engine/src/file_access.rs:1-23, 60, 212, 258`; `apps/vectorcraft/src/main.rs:12-13, 689`                      |
| DesignCraft | same flags                                                     | `designcraft-cli mcp` headless only          | Same as VectorCraft; strips `\\?\` for display                                                                                                                                                                                       | `designcraft` `crates/engine/src/file_access.rs:1-21, 44-60, 168-180, 238-260`; `apps/designcraft-cli/src/main.rs:41, 189`          |
| WordCraft   | same flags                                                     | `wordcraft-cli mcp` headless only            | Same file-access design                                                                                                                                                                                                              | `wordcraft` `crates/engine/src/file_access.rs:1-20, 57, 168, 254`; `apps/wordcraft-cli/src/main.rs:34-36, 443`                      |
| SoundCraft  | same flags                                                     | `soundcraft-cli mcp` headless only           | Same file-access design                                                                                                                                                                                                              | `soundcraft` `apps/soundcraft-cli/src/main.rs:11, 416`; `crates/automation/src/lib.rs:72`                                           |
| EffectCraft | same flags                                                     | `effectcraft-cli mcp` only; GUI rejects them | some tools refused while confined                                                                                                                                                                                                    | `effectcraft` `apps/effectcraft-cli/src/main.rs:120, 609`; `crates/automation/src/tools.rs:117`; `apps/effectcraft/src/main.rs:571` |
| PdfCraft    | `--root`                                                       | `pdfcraft-cli mcp` and batch                 | not read in detail                                                                                                                                                                                                                   | `pdfcraft` `apps/pdfcraft-cli/src/main.rs:251, 258, 734`                                                                            |
| FilmCraft   | none (`mcp --automation-read-root` is an unknown option)       | n/a                                          | n/a                                                                                                                                                                                                                                  | `filmcraft` `apps/filmcraft-cli/src/args.rs:163`                                                                                    |
| LightCraft  | none found (grep)                                              | n/a                                          | n/a                                                                                                                                                                                                                                  | n/a                                                                                                                                 |

> **Refinement of O13 (§3a).** O13 says PhotoCraft "needs relative paths". At HEAD it also accepts an absolute path whose *text* lies under the root (`workspace.rs:151-172`). The catch is that the comparison is case-sensitive past the drive letter and doesn't understand `\\?\`, so the hub still has to translate (§7.4.4).

**Long paths.** No app embeds a `longPathAware` manifest: the only Windows manifest found sets DPI awareness (`pdfcraft` `apps/pdfcraft/src/windows_manifest.rs:1-9`); the other `build.rs` files embed only icon and VERSIONINFO (e.g. `filmcraft` `apps/filmcraft/build.rs:1-11`). Rust's standard library adds the `\\?\` prefix itself for most file-system calls on long paths (general knowledge, not checked against the toolchain here), so plain Rust I/O probably copes. Process launch, shell calls and textual root checks may not (Not verified, step 3).

**Proxies and caches.** FilmCraft proxies go to the chosen `destination`, else **`<media folder>\Proxies`**, a write into the user's folders (`filmcraft` `crates/engine/src/proxies.rs:336-342`; `media.createProxies {…, destination?, attach, wait}` `:493-497`). FilmCraft's media cache is a configured location or `<data dir>\Media Cache` (`crates/engine/src/settings.rs:1603-1614`).

---

## 7.1 The canonical AssetRef

### 7.1.1 Candidates compared

| Candidate                       | Survives rename/move        | Survives copy as "same"                      | Detects in-place edit             | Cost to get                       | Who has it today                                           | Verdict                                     |
|---------------------------------|-----------------------------|----------------------------------------------|-----------------------------------|-----------------------------------|------------------------------------------------------------|---------------------------------------------|
| Absolute path                   | no                          | no (a copy is another path)                  | no                                | free                              | every app, every command                                   | **Required anchor.** Every command needs it |
| Size + mtime                    | yes, as a hint              | partly (copy keeps size)                     | usually (mtime changes on save)   | one `stat`                        | AssetCraft listing, FilmCraft, LightCraft                  | Cheap freshness check, never identity       |
| Quick hash (size + head + tail) | yes                         | yes                                          | mostly (misses middle-only edits) | ≤ 2 MiB read                      | FilmCraft (64-bit), AssetCraft (as cache key)              | **Default content check**                   |
| Full content hash               | yes                         | yes                                          | yes                               | whole file (minutes on big video) | LightCraft (SipHash), PhotoCraft (BLAKE3 inside `.pcraft`) | **On demand only**                          |
| XMP `DocumentID`                | yes                         | yes, and *also* across "Save As" derivatives | no                                | XMP parse                         | none in suite; Adobe-written files                         | **Lineage hint**, tie-breaker only          |
| XMP `InstanceID`                | yes                         | yes until next save                          | yes, if the writer updates it     | XMP parse                         | none in suite                                              | Hint only; suite writers don't maintain it  |
| AssetCraft catalog id           | yes (catalog follows moves) | no (a copy gets a new id)                    | n/a                               | catalog lookup                    | nobody until Stage 3                                       | **Authoritative when present**              |

Three relations follow from this, and the spec keeps them apart:

| Relation      | Meaning                             | Decided by                                                  | Used for                                  |
|---------------|-------------------------------------|-------------------------------------------------------------|-------------------------------------------|
| same location | the same file on disk now           | equal path keys (§7.4.2)                                    | Open In, every command call               |
| same asset    | what the user means by "this asset" | equal `catalogId`; else same location; else a resolved move | handoffs, the ledger (§7.4.6), relink     |
| same content  | byte-identical                      | equal full hash (quick hash is a strong hint)               | dedupe, verify-after-copy, move detection |

`snow_clearing_plate_copy.jpg` and `snow_clearing_plate.jpg` are **same content, different assets**. A content hash alone would wrongly merge them, which is why the path stays the anchor.

### 7.1.2 Recommendation (D4)

**A5-R1 (D4).** The canonical AssetRef is a **composite**: a required absolute `path`, plus `size` and `mtime` as a freshness stamp, plus a BLAKE3 **quick hash**, with a BLAKE3 **full hash**, XMP ids and the AssetCraft `catalogId` as optional fields. This refines the outline's D4 ("content hash + path, with XMP id when present") by splitting the hash into quick and full, and by ranking the catalog id first once it exists. Owner: **Paul** (decides D4); Claude writes the `assetref` crate.

Why BLAKE3:

- It is already in the suite (PhotoCraft, `pure` feature, so no C toolchain).
- Its licence (CC0-1.0 / Apache-2.0) is on AssetCraft's allow list (`assetcraft` `deny.toml:20-32`).
- It is fast enough that a full hash is limited by the disk, not the CPU.
- It is cryptographic, so a malicious collision can't be planted: the hub treats every file as hostile (`AGENTS.md` rule 4).

SipHash (AssetCraft, LightCraft) is fine for cache keys but isn't collision-resistant against a chosen input. The two existing SipHash users also use **different keys**, so their values can't be compared anyway (`assetcraft` `hash.rs:28`, `lightcraft` `hash.rs:31`).

Why not XMP ids as identity: no suite app writes them (§7.0.2), a "Save As" copy keeps the `DocumentID` by design, and they live in a file the user can edit. They are a lineage hint and a tie-breaker, never an override.

### 7.1.3 Hashing cost strategy

| Hash   | Definition (v1)                                                                                                                          | Reads              | When computed                                                                                                                        |
|--------|------------------------------------------------------------------------------------------------------------------------------------------|--------------------|--------------------------------------------------------------------------------------------------------------------------------------|
| `q1`   | BLAKE3 over `"pipelinecraft-q1"`, `size` (u64 LE), the first 1 MiB, and the last 1 MiB (non-overlapping; a file ≤ 2 MiB is hashed whole) | ≤ 2 MiB, two reads | When a ref is minted for a handoff (AssetCraft or the hub), and whenever a size/mtime check fails                                    |
| `full` | BLAKE3 of all bytes, the plain `blake3::hash` of the file (so `b3sum` gives the same value)                                              | whole file         | On demand only: ambiguous move candidates, `verify: "full"` from the caller, Stage 2 dedupe, before a destructive batch on the asset |

The 1 MiB span matches FilmCraft's `FINGERPRINT_SPAN` (`relink.rs:28`), so both see the same bytes. The `q1` name versions the recipe: a sampled `q2` (adding middle samples) can follow if middle-only edits turn out to matter, without breaking stored refs.

Draft cost model (estimates to baseline in Not verified step 5, not measurements):

| File             | `q1`                              | `full` at NVMe ~3 GB/s | `full` at HDD ~150 MB/s | Policy                                                              |
|------------------|-----------------------------------|------------------------|-------------------------|---------------------------------------------------------------------|
| 25 MB raw / JPEG | ~1–5 ms                           | ~10 ms                 | ~0.2 s                  | `full` allowed inline                                               |
| 2 GB TIFF / PSD  | ~1–5 ms (SSD), ~20 ms (HDD seeks) | ~0.7 s                 | ~14 s                   | `full` as a hub job                                                 |
| 50 GB ProRes     | same                              | ~17 s                  | ~6 min                  | `full` only as a cancellable background job; never blocks a handoff |

Rules:

1. **Never block a handoff on `full`.** A ref may carry `"full": null`; a later `asset.hashed` event fills it (A6).
2. **Cache hashes** by path key + size + mtime in the hub's hash index (§7.5) and, from Stage 3, in AssetCraft's catalog, which becomes the authoritative index. A size or mtime change invalidates the cached hash.
3. **Full hash is a job** (§8.3) with progress in bytes, cancel, and a hard per-call timeout. Single-threaded by default so it doesn't starve the apps; multi-threaded only when the caller asks.
4. **The cache key is not the identity.** AssetCraft keeps its SipHash thumbnail key (it rightly includes mtime and renderer version); the hub never reuses it as content identity.

### 7.1.4 Resolution rules (D4)

`resolve(ref) → Resolution` turns an AssetRef into a path that is valid on this machine now. It runs in this order and stops at the first answer:

| Step | Condition                                                                                   | Action                                                                                                                                                                                                             | Status returned                                     |
|------|---------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------|
| R0   | Ref fails validation (§7.4.2), or `version` unknown                                         | Refuse                                                                                                                                                                                                             | error `asset_ref_invalid`                           |
| R1   | `catalogId` present and AssetCraft is reachable with the (proposed) `asset.resolve` command | Ask AssetCraft; its answer wins                                                                                                                                                                                    | whatever AssetCraft says (`ok`, `moved`, `missing`) |
| R2   | `path` exists, and size and mtime equal the ref's                                           | Accept with no read                                                                                                                                                                                                | `ok`                                                |
| R3   | `path` exists, stamp differs                                                                | Compute `q1`. Equal → accept and refresh the stamp. Different → the content changed in place                                                                                                                       | `ok` (refreshed) or `modified`                      |
| R4   | `path` is missing, and the ref has a hash                                                   | Look the hash up: hub hash index first, then a **bounded** search (the old folder, its siblings and parent down to depth 2, AssetCraft's watched roots; ≤ 10,000 entries or 5 s), filtering by size before hashing | `moved` (one match) or step R5                      |
| R5   | Several candidates                                                                          | Tie-break in order: same file name, nearest folder to the old path, equal XMP `DocumentID`, equal `full` hash (computed now as a job). Still tied → return all of them                                             | `moved` or `ambiguous {candidates}`                 |
| R6   | Nothing found                                                                               | n/a                                                                                                                                                                                                                | `missing`                                           |

Who acts on each status:

- **`modified`:** strict callers (dry runs, recipes) stop. Interactive callers get the path plus a warning. The hub emits `asset.changed`.
- **`moved` and `ambiguous`:** the hub never relinks silently. The caller confirms, then the hub updates the ref and the handoff ledger (§7.4.6).
- **The search is read-only and bounded.** It never crawls whole drives, and never follows links or junctions (the same rule as AssetCraft's `fs` crate, `crates/fs/src/real.rs:150`).

### 7.1.5 JSON shape

```json
{
  "path": "F:\\Art\\Reapers\\environments\\snow_clearing_plate.jpg",
  "name": "snow_clearing_plate.jpg",
  "size": 1843200,
  "mtime": "2026-10-14T21:03:11.1234567Z",
  "mediaType": "image/jpeg",
  "hash": {
    "q1": "9a1f…64 hex…",
    "full": null
  },
  "xmp": { "documentId": "xmp.did:…", "instanceId": "xmp.iid:…" },
  "catalogId": null,
  "origin": { "app": "assetcraft", "version": "0.2.0" }
}
```

| Field       | Type                              | Required | Rules                                                                                                                                               |
|-------------|-----------------------------------|----------|-----------------------------------------------------------------------------------------------------------------------------------------------------|
| `path`      | string                            | **yes**  | Absolute, plain form (no `\\?\`), normalized per §7.4.2, casing as the file system reports it. Satisfies A1's "an asset ref must degrade to a path" |
| `name`      | string                            | no       | Display only; derived from `path` when absent                                                                                                       |
| `size`      | u64                               | no       | Bytes at mint time                                                                                                                                  |
| `mtime`     | RFC 3339 UTC string, up to 100 ns | no       | Windows FILETIME resolution; compared exactly (a mismatch only costs a `q1`)                                                                        |
| `mediaType` | string                            | no       | From AssetCraft's magic-byte sniffing (`crates/probe`), never from the extension alone                                                              |
| `hash.q1`   | 64 lower-case hex, or null        | no       | §7.1.3. Required in a payload that will be **stored** (recipes, ledger, storyline)                                                                  |
| `hash.full` | 64 lower-case hex, or null        | no       | Plain BLAKE3 of the bytes                                                                                                                           |
| `xmp`       | object or absent                  | no       | Read-only hints (§7.3)                                                                                                                              |
| `catalogId` | string `ac:<ULID>`, or null       | no       | Minted only by AssetCraft's Stage 3 catalog                                                                                                         |
| `origin`    | `{app, version}`                  | no       | Who minted the ref; for diagnostics                                                                                                                 |

Unknown fields are kept and passed through (forward compatibility). A ref is at most 4 KiB serialized; anything bigger is refused (`asset_ref_invalid`).

**Degrade rule.** For any app command that takes `path` or `paths`, the adapter sends `resolve(ref).path`, translated per app (§7.4.4). No upstream change is needed for v1.

---

## 7.2 Handoff payloads

Every payload is a JSON object with a `type` and an integer `version`. The hub refuses a `version` it doesn't know (`payload_version_unsupported`, a §6.5 code) and keeps unknown fields. The version is per payload type; v1 for all of them at launch.

**Size rule.** The control channels cap a line at 4 MiB (§1.4 R2; SoundCraft ~86 MiB). A payload over 1 MiB serialized, or a selection over 10,000 items, is written by the hub to its handoff folder (§7.5), and the call carries a `handoff.file` envelope instead. Inline bytes (`blob`) are capped at 1 MiB decoded; larger content goes to a file.

### 7.2.1 Single asset

```json
{ "type": "handoff.asset", "version": 1,
  "asset": { "path": "F:\\Art\\Reapers\\characters\\reaper\\reaper_v03.psd", "size": 48211337,
             "mtime": "2026-10-13T08:12:40Z", "hash": { "q1": "…", "full": null } },
  "intent": "open",
  "options": { "page": null, "frame": null } }
```

`intent` is one of the capability aliases from O-B8 (`asset.open`, `asset.place`, `asset.import`, `asset.thumbnail`, `asset.export`), written without the `asset.` prefix. The adapter maps it to the app's own command, e.g. FilmCraft `file.import`, VectorCraft `document.open`, SoundCraft `session.open`.

### 7.2.2 Selection

```json
{ "type": "handoff.selection", "version": 1,
  "source": { "app": "assetcraft", "location": "F:\\Art\\Reapers\\environments" },
  "items": [ { "path": "F:\\Art\\Reapers\\environments\\snow_clearing_plate.jpg", "hash": { "q1": "…" } },
             { "path": "F:\\Art\\Reapers\\environments\\ruins_vertical.png",      "hash": { "q1": "…" } } ],
  "focus": 0,
  "intent": "open" }
```

Order is AssetCraft's item order and `focus` is an index, matching `{selected, focus}` (`assetcraft` `commands.rs:561`). Targets that take one path per call (PhotoCraft `app.open`, PdfCraft `ui.open`) get one call per item, in order, as one hub job.

### 7.2.3 Collection

```json
{ "type": "handoff.collection", "version": 1,
  "collection": { "id": "ac:01JA…", "name": "Snow plates", "kind": "smart",
                  "rule": "kw:snow rating:>3", "snapshotAt": "2026-10-15T09:00:00Z" },
  "items": [ { "path": "…", "catalogId": "ac:01JB…", "hash": { "q1": "…" } } ] }
```

A smart collection travels as a **snapshot**: the rule is informational, and the receiver uses `items`. Collections exist only from AssetCraft Stage 3 (`assetcraft-plan.md:159-166`); until then this type is reserved.

### 7.2.4 Storyline block

The fields follow `docs/storyline-builder.md` §3 one to one (`:50-62`). Time is integer ticks at FilmCraft's 254,016,000,000 per second (`filmcraft` `crates/time/src/lib.rs:20`), as the Builder's review asks (`storyline-builder.md:68-69`).

```json
{ "type": "handoff.storyline", "version": 1,
  "storyline": { "id": "sl_01JC…", "title": "Reaper teaser", "rate": { "num": 24000, "den": 1001 },
                 "aspect": "16:9", "music": { "asset": { "path": "…\\score.wav" }, "cueMap": null } },
  "blocks": [
    { "id": "blk_010", "title": "Intro", "purpose": "hook",
      "durationTicks": 1270080000000,
      "description": "The Reaper walks toward a faint blue soul light.",
      "requirements": ["Preserve cloak, staff, eye colour"],
      "assets": [
        { "role": "video", "asset": { "path": "…\\reaper_walk_v02.mov", "hash": { "q1": "…" } },
          "sourceInTicks": 254016000000, "durationTicks": 1270080000000, "layer": 0 },
        { "role": "text", "text": "THEY COME AT DUSK", "layer": 1,
          "startTicks": 254016000000, "durationTicks": 508032000000 }
      ],
      "timing": { "mode": "fixed", "cue": null },
      "entry": { "transition": "cut" }, "exit": { "transition": "dissolve", "durationTicks": 254016000000 },
      "continuity": { "entities": ["ac:01JD…"], "palette": null },
      "sourceRef": { "kind": "transcript", "asset": { "path": "…\\interview.srt" }, "startTicks": 0, "endTicks": 2540160000000 },
      "status": "approved",
      "locks": ["wording", "timing"] } ] }
```

| Block field     | From `storyline-builder.md`               | Notes                                                                                                                     |
|-----------------|-------------------------------------------|---------------------------------------------------------------------------------------------------------------------------|
| `id`, `title`   | "Block ID and title" (:52)                | Stable id, never reused                                                                                                   |
| `purpose`       | "Story purpose" (:53)                     | `hook \| introduction \| explanation \| conflict \| reveal \| closing`                                                    |
| `durationTicks` | "Duration" (:54); block vs asset (:64-65) | Block length on the storyline, separate from each asset's `durationTicks`                                                 |
| `assets[]`      | "Assets" (:57), asset types §4 (:73-90)   | `role`: image, video, broll, text, character, effect, voiceover, music; file roles carry an AssetRef, text carries `text` |
| `timing`        | "Timing rules" (:58)                      | `fixed \| flexible \| cue`                                                                                                |
| `entry`, `exit` | "Entry / exit" (:59)                      | Small vocabulary FilmCraft can build; anything else becomes a marker (:92-94)                                             |
| `continuity`    | "Continuity references" (:60)             | Entities are catalog ids (Stage 3)                                                                                        |
| `sourceRef`     | "Source reference" (:61)                  | Transcript passage or timestamp                                                                                           |
| `status`        | "Status" (:62)                            | `draft \| readyToGenerate \| generating \| needsReview \| approved`                                                       |
| `locks`         | per-field locks (:162)                    | `wording \| assets \| order \| timing`                                                                                    |

This is the hub's *transport* shape. AssetCraft's L3 `boards` crate owns the model and its `schemars` schema (`storyline-builder.md:163-166`), and the payload must be generated from that schema, not hand-kept. Recorded as A5-O7.

### 7.2.5 Timeline

FilmCraft imports OTIO through `file.import` (§7.0.2), so the timeline payload is **an OTIO file plus a media manifest**, not inline OTIO JSON:

```json
{ "type": "handoff.timeline", "version": 1,
  "format": "otio",
  "file": { "path": "C:\\Users\\paul\\AppData\\Local\\PipelineCraft\\handoff\\job_01JE…\\reaper_teaser.otio",
            "hash": { "q1": "…" } },
  "sequenceName": "Reaper teaser",
  "rate": { "num": 24000, "den": 1001 },
  "expect": { "clips": 14, "durationTicks": 7620480000000 },
  "media": [ { "path": "…\\reaper_walk_v02.mov", "hash": { "q1": "…" } } ],
  "storyline": { "id": "sl_01JC…" } }
```

The hub runs `filmcraft/file.import {"paths": [file.path]}` and records the returned sequences in the ledger. `expect` lets the hub check AC3(c) (clip count and duration, ±1 frame) after import.

OTIO writing rules for AssetCraft's `interchange` crate (L3):

1. **Absolute `file://` URLs** in `ExternalReference.target_url`, in FilmCraft's form (`file:///F:/Art/x.mov`, UNC as `file://server/share/…`; `common.rs:81-100`). Relative URLs resolve against the `.otio` folder (`otio.rs:694`), which is the hub's handoff folder here, so they would point at nothing.
2. **One spelling per file** (§7.4.2), because FilmCraft de-duplicates media by the string `file:{path}` (`otio.rs:696`). Two casings of the same clip would become two media items.
3. `Clip.2` per layer with `source_range` for the trim, `Gap.1` for empty slots, `Marker.2` for block notes and anything FilmCraft can't build (`storyline-builder.md:92-94, 207-209`).
4. `available_range` set when the media duration is known; a missing file is written as `MissingReference.1` and imports offline (`otio.rs:226, 732-735`).
5. The AssetRef goes in `ExternalReference.metadata.pipelinecraft.assetRef`. FilmCraft drops it today (`otio.rs:678`), so the hub's ledger is the source of truth for clip → asset, and A5-O5 asks FilmCraft to keep foreign metadata.

Other formats (`fcpxml`, `edl`, `xml`) use the same envelope with `format` changed. They are out of scope for v1.

### 7.2.6 Blob and file envelopes

```json
{ "type": "handoff.blob", "version": 1, "mediaType": "image/png", "name": "poster.png", "base64": "iVBOR…" }
{ "type": "handoff.file", "version": 1, "payloadType": "handoff.selection",
  "file": { "path": "…\\PipelineCraft\\handoff\\job_01JE…\\payload.json", "hash": { "q1": "…" } } }
```

`blob` covers the three byte-level surfaces that exist (WordCraft `file.open {data}`, DesignCraft `file.place {base64}`; VectorCraft and EffectCraft *return* base64 thumbnails and frames). A blob is not an asset: it has no path until someone saves it.

---

## 7.3 Sidecars and XMP: who owns writes

### 7.3.1 Rules

| ID  | Rule                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           |
|-----|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| X1  | **The hub never writes XMP, sidecars or embedded metadata itself.** It only routes app commands, and a sidecar write is the command effect `write-user-file` (§6.4) with confirmation (§10.3). This follows NG8 and A1's goal note: no user-file writes except through an app command whose command effects say so                                                                                                                                                                                                             |
| X2  | **AssetCraft writes nothing before Stage 2** (`AGENTS.md` rule 5). Until then AssetCraft may *read* XMP (via `lightcraft-meta`) to fill `ref.xmp`, and the hub routes sidecar writes only to apps that already do them (LightCraft, FilmCraft)                                                                                                                                                                                                                                                                                 |
| X3  | **Namespace ownership.** Each writer changes only the properties it owns and keeps everything else byte for byte: LightCraft's `merge_xmp` contract (`xmp_merge.rs:1-20`). Owners: LightCraft its `OWNED` list plus `lc:*` (`sidecar.rs:361, 384`); AssetCraft (Stage 2) `xmp:Rating`, `xmp:Label`, `dc:subject`, `lr:hierarchicalSubject`, using the same merge code; AssetCraft (Stage 3.5) `crs:*` (`assetcraft-plan.md:174-189`); FilmCraft `dc:title/description` and `xmpDM:*` log fields (`project_tools.rs:1066-1080`) |
| X4  | **Sidecar naming** follows LightCraft: `<stem>.xmp` by default, `<file>.xmp` where two originals would share a stem, raw first (`sidecar.rs:15-18, 39-60`). AssetCraft reads both names. The hub's ref may list the sidecar it found as a hint, never as identity                                                                                                                                                                                                                                                              |
| X5  | **Embedded XMP is read-only in v1.** No hub-routed command may rewrite metadata inside an original. That is a pixel-file write, and every app with a "save" already declares it                                                                                                                                                                                                                                                                                                                                                |
| X6  | **One writer per sidecar per job.** Inside one hub job or recipe, the hub serializes calls whose target sidecar has the same path key (§7.4.2). Across independent apps the last writer wins, and X3 limits the damage to the properties that writer owns                                                                                                                                                                                                                                                                      |
| X7  | **After a routed sidecar write**, the hub emits `asset.changed {ref, what: "sidecar"}` (§8.2) so AssetCraft refreshes its badges and metadata panel                                                                                                                                                                                                                                                                                                                                                                            |
| X8  | **XMP ids never decide identity** (§7.1.2). AssetCraft never mints a `DocumentID`, so it doesn't pretend to be the document's creator                                                                                                                                                                                                                                                                                                                                                                                          |

### 7.3.2 Conflict found: FilmCraft can erase a merged sidecar

1. FilmCraft's `clip.updateMetadata` refuses a sidecar only when it lacks `xmp:CreatorTool="FilmCraft"`, and otherwise **replaces the whole file** (`project_tools.rs:1089-1106`).
2. LightCraft's merge deliberately keeps the existing `xmp:CreatorTool` (`xmp_merge.rs:12-13`). AssetCraft's Stage 2 merge will too, since it uses the same code.
3. The failure, step by step: FilmCraft writes `clip.xmp`. LightCraft or AssetCraft then merges a rating into it, and `CreatorTool` still says FilmCraft. The next Update Metadata in FilmCraft overwrites the file, and the rating is gone.

This is static reasoning; Not verified step 4 reproduces it. Fix: A5-O4.

---

## 7.4 Windows path rules

### 7.4.1 Forms the hub meets

| Form                      | Example                                        | Source                    | Hub rule                                                                                                                          |
|---------------------------|------------------------------------------------|---------------------------|-----------------------------------------------------------------------------------------------------------------------------------|
| Drive absolute            | `F:\Art\a.png`                                 | listings, Explorer        | Canonical wire form                                                                                                               |
| Drive-relative / rooted   | `\Art\a.png`, `F:Art\a.png`                    | typed by users            | Refused (`path_not_absolute`); AssetCraft fixes typed `C:` already (`paths.rs:5-26`)                                              |
| Verbatim                  | `\\?\F:\Art\a.png`                             | `std::fs::canonicalize`   | Strip to `F:\Art\a.png`, as DesignCraft (`file_access.rs:168-180`) and FilmCraft (`common.rs:84-92`) do                           |
| Verbatim UNC              | `\\?\UNC\nas\media\a.mov`                      | `canonicalize` on a share | Strip to `\\nas\media\a.mov`                                                                                                      |
| UNC                       | `\\nas\media\a.mov`                            | network shares            | Allowed; kept as is                                                                                                               |
| Mapped drive / `subst`    | `Z:\a.mov` for `\\nas\media\a.mov`             | user setup                | Kept as given, **not** mapped to UNC (that needs a Win32 call). Two spellings → two path keys; R4 hash resolution reconciles them |
| Device / volume GUID      | `\\.\COM1`, `\\?\Volume{…}\`, `\\?\GLOBALROOT` | hostile input             | Refused (`path_unsupported`)                                                                                                      |
| Long (> 259 UTF-16 units) | deep project trees                             | real projects             | Allowed in plain form, flagged `longPath: true` in resolution results (§7.4.3)                                                    |
| URL                       | `file:///F:/Art/a.mov`                         | OTIO                      | Converted by the OTIO writer and reader only (§7.2.5)                                                                             |

### 7.4.2 Normalization (path key)

The hub keeps two strings per path: the **wire path** (sent to apps, shown to people) and the **path key** (compared, indexed).

| Step | Wire path                                                                                                                                                                                                                                                  | Path key                                                                                                                                                                        |
|------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| N1   | Must have a root and, on Windows, a drive or UNC prefix                                                                                                                                                                                                    | same                                                                                                                                                                            |
| N2   | Strip `\\?\` and `\\?\UNC\` (§7.4.1)                                                                                                                                                                                                                       | same                                                                                                                                                                            |
| N3   | `/` → `\`; collapse repeated separators (except the UNC leading `\\`); drop `.` components; **refuse `..`** (no textual resolution, because a link below it could make that wrong)                                                                         | same                                                                                                                                                                            |
| N4   | Upper-case the drive letter                                                                                                                                                                                                                                | same                                                                                                                                                                            |
| N5   | Refuse: NUL characters, unpaired surrogates (not valid in JSON; AssetCraft refuses non-Unicode too, `commands.rs:450-452`), reserved device names (`CON`, `NUL`, `COM1`…), components ending in `.` or space, `:` after the drive (alternate data streams) | same                                                                                                                                                                            |
| N6   | **Keep component case and Unicode form exactly** as the file system reports them. NTFS doesn't normalize Unicode, so NFC and NFD names are different files (`Réaper ünïcode name.png` is the test)                                                         | n/a                                                                                                                                                                             |
| N7   | n/a                                                                                                                                                                                                                                                        | Lower-case each component with Unicode `to_lowercase` on Windows (as AssetCraft `paths.rs:30-40` and DesignCraft `same_component` `file_access.rs:248-260` do); exact elsewhere |
| N8   | No link resolution                                                                                                                                                                                                                                         | No link resolution                                                                                                                                                              |

Caveats:

- **N7 approximates NTFS case folding,** which uses the volume's upcase table. Windows can also turn on per-folder case sensitivity (WSL).
- **Collision rule:** if two files that both exist map to the same path key, the hub treats them as distinct and flags `case_ambiguous`. It never merges them.
- **Hostile input:** the validator is pure, takes `&str`, returns `Result`, and is fuzzed (§13.2). It never panics.

### 7.4.3 Long paths

- Emit the plain form to every app. No app declares `accepts_verbatim` today, and PhotoCraft's textual check would refuse it (the `:` in `\\?\F:` fails `relative_path`, `workspace.rs:184-186`).
- Rust std probably handles long paths for file I/O by itself (general knowledge). Spawning a GUI with long path arguments (AssetCraft Open In) is capped at 30,000 chars (`open_in.rs:162`) and stays under CreateProcess's 32,767-char command line.
- Whether each app actually opens a > 260-char path is **not verified** (step 3). Until it is, the adapter config has `long_paths: unknown | ok | refuse` per app, and the hub returns `path_too_long_for_app` rather than letting an app fail obscurely.

### 7.4.4 Per-app path translation

The adapter turns the resolved wire path into what each app accepts. All rows come from §7.0.2.

| App                                             | Accepts                                                                    | Translation by the adapter                                                                                                                                                                                                                                                       | Refusal mapped to                       |
|-------------------------------------------------|----------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------|
| PhotoCraft                                      | Root-relative `a/b.png` (forward slashes), or absolute text under the root | The hub launches PhotoCraft with `--automation-read-root` = the wire path of the folder (casing as listed) and sends the **root-relative forward-slash form**. Paths outside the root, or with components PhotoCraft refuses (trailing dot/space, `:`) → refused before the call | `path_outside_root`, `path_unsupported` |
| VectorCraft                                     | Absolute; with roots: inside the canonicalized root, case-insensitive      | Plain wire path. When the hub launches it for automation it sets roots to the job's input folder and the hub output folder (§7.5)                                                                                                                                                | `path_outside_root`                     |
| DesignCraft, WordCraft, SoundCraft, EffectCraft | Absolute; roots only on headless `mcp`                                     | Plain wire path. Headless batch: the hub sets the read root = smallest common folder of the inputs, and the write root = the job's hub output folder                                                                                                                             | `path_outside_root`                     |
| PdfCraft                                        | Absolute (`ui.open`, `doc_open`); `--root` on headless                     | Plain wire path; `--root` as above for headless                                                                                                                                                                                                                                  | `path_outside_root`                     |
| FilmCraft                                       | Absolute; no confinement                                                   | Plain wire path. OTIO via the `file://` rule (§7.2.5)                                                                                                                                                                                                                            | n/a                                     |
| LightCraft                                      | Absolute `paths[]`; `mode: add` imports in place                           | Plain wire path; the hub always sends `mode: "add"` unless the caller asks for copy/move (those are `write-user-file`)                                                                                                                                                           | n/a                                     |
| AssetCraft                                      | Absolute, rooted, Unicode                                                  | Plain wire path                                                                                                                                                                                                                                                                  | n/a                                     |

Attaching to an app that is **already running** with unknown roots: the hub sends the call and maps the app's refusal text to `path_outside_root`, with a hint to relaunch through the hub. §3a O13's adapter behaviour stays, with the refinement in §7.0.2.

### 7.4.5 Moves and renames after a handoff

The receiving app stores a raw path (FilmCraft `MediaRef::File {path}`, `crates/project/src/lib.rs:160-165`; DesignCraft `Asset.link: Option<String>`, `crates/doc/src/lib.rs:438`). When the file later moves:

1. AssetCraft's watcher sees renames inside watched folders (`STATUS.md` step 3, `crates/watch`). AssetCraft reports `asset.moved {from: ref, to: path}` to the hub (proposed, §8.2).
2. The hub looks the asset up in the **handoff ledger** (§7.4.6) and finds which apps hold it.
3. The hub **offers** each holder's own relink command; it never runs one without confirmation:
   - FilmCraft `media.relink {item, path}` or `media.autoRelink {from, to}` (`relink.rs:647-651, 682-686`). FilmCraft's own fingerprint check still guards the swap.
   - DesignCraft `links.relink {asset, path}` (`links.rs:21-28`).
   - LightCraft relinks moved files by content hash during folder sync on its own (`sync.rs:272-305`), so the hub only nudges a sync.
4. If the move happened outside any watched folder, the next `resolve()` finds it by hash (R4) and the same offer is made.

### 7.4.6 Handoff ledger

The hub records every handoff that leaves state in another app:

`{ledgerId, jobId, at, app, appLocal: {item | asset | photo ids returned by the call}, ref}`

It lives in the hub's state folder (§7.5) as an append-only JSON-lines file, capped and compacted like LightCraft's journal pattern (`assetcraft` `docs/crafting-apps-notes.md:80`). The ledger is what makes "moves after handoff" and AC3's clip → asset mapping possible without upstream changes.

---

## 7.5 Derived and temp files

### 7.5.1 Location

All hub-owned files live under one root, `%LOCALAPPDATA%\PipelineCraft\`. It is created with a user-only ACL (O12), never roams, and is the same family as AssetCraft's `%LOCALAPPDATA%\AssetCraft\thumbs` (`prefs_file.rs:40-46`). The final folder name follows D1.

```
%LOCALAPPDATA%\PipelineCraft\
├── state\
│   ├── ledger.jsonl            handoff ledger (§7.4.6)          kept, compacted
│   └── hashes.jsonl            hash index: key, size, mtime, q1, full    rebuilt on demand
├── handoff\<jobId>\            payload.json, .otio, decoded blobs        job end + 24 h
├── derived\
│   └── thumbs\<2hex>\<32hex>.png   owning-app thumbnails (AC2)          LRU, 512 MiB budget
├── renders\<jobId>\            outputs when the caller names no destination   7 days, or until claimed
└── tmp\                        partial writes (same volume, renamed into place)   deleted on start
```

### 7.5.2 Naming by hash

- **Thumbnails:** `<32hex>` = the first 128 bits of BLAKE3 over `q1 ‖ app id ‖ app version ‖ max side`. A new app version or size re-renders, and identical content shares a file. The shape mirrors AssetCraft's `<2 hex>/<32 hex>.png` (`disk.rs:1`), so AssetCraft can copy a hub thumbnail straight into its own cache, keyed by its own key. AssetCraft stays the owner of what it shows.
- **Renders:** named by the app (its own export naming), inside the job folder, so two jobs never collide.
- **Handoff files:** fixed names inside the job folder (`payload.json`, `<sequence>.otio`).

### 7.5.3 Lifetime, GC and who cleans up

| ID  | Rule                                                                                                                                                                                                                                                                                                                         |
|-----|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| G1  | **The hub cleans only its own root**, and only names of its own shapes (job ids, `<2hex>/<32hex>.png`). It never follows links. This is AssetCraft's "own files only" rule (`disk.rs:23-25`)                                                                                                                                 |
| G2  | GC runs at hub start and hourly while the hub runs, on a background thread, as a hub job visible in `craftctl` (§12)                                                                                                                                                                                                         |
| G3  | `handoff\<jobId>` is deleted 24 h after the job ends, unless the ledger still points at a file in it. FilmCraft has imported the OTIO by then; the project keeps its own copy of the timeline                                                                                                                                |
| G4  | `renders\<jobId>` is kept 7 days. A caller "claims" a render by moving it out (a `write-user-file` call through the hub); claimed or not, it is never deleted while a running job references it                                                                                                                              |
| G5  | `derived\thumbs` is LRU within 512 MiB, pruned to 80 % (AssetCraft's numbers, `disk.rs:19-21`)                                                                                                                                                                                                                               |
| G6  | **App caches belong to the apps.** AssetCraft thumbs and FilmCraft Media Cache are never touched by hub GC. The hub may call an app's own `cache.clear` only on an explicit user request                                                                                                                                     |
| G7  | **Proxies are not temp files.** FilmCraft attaches them to the project, so deleting them breaks playback. The hub passes `destination` only when the recipe or user sets one; otherwise FilmCraft's default `<media folder>\Proxies` applies, which is `write-user-file` and is declared as such (`proxies.rs:336-342, 497`) |
| G8  | Disk-full or permission errors in GC are logged and skipped. GC never fails a job and never crashes the hub                                                                                                                                                                                                                  |
| G9  | **Dry run** (§8.5) writes into `handoff\` and `tmp\` only, never into `renders\` or user folders                                                                                                                                                                                                                             |

---

## 7.6 Rust type sketches

These are sketches for the shared `assetref` crate (proposed L0, no UI, no I/O beyond `std::fs` reads behind a trait). They follow the house rules: no `unsafe`, no `unwrap`/`expect`/`panic`, and every parser returns `Result`.

```rust
// crates/assetref/src/lib.rs  (sketch; serde + serde_json + blake3 { default-features = false, features = ["pure"] })

use serde::{Deserialize, Serialize};
use serde_json::Map;

/// Plain absolute wire path (§7.4.2 N1–N6). Constructed only through `WirePath::parse`.
#[derive(Clone, Debug, PartialEq, Eq, Serialize, Deserialize)]
#[serde(try_from = "String", into = "String")]
pub struct WirePath(String);

/// Comparison key (§7.4.2 N7). Never sent to apps.
#[derive(Clone, Debug, PartialEq, Eq, Hash, PartialOrd, Ord)]
pub struct PathKey(String);

#[derive(Clone, Copy, Debug, PartialEq, Eq, Hash, Serialize, Deserialize)]
#[serde(try_from = "String", into = "String")] // 64 lower-case hex
pub struct Blake3Hex([u8; 32]);

#[derive(Clone, Debug, Default, PartialEq, Eq, Serialize, Deserialize)]
pub struct Hashes {
    #[serde(default)]
    pub q1: Option<Blake3Hex>,
    #[serde(default)]
    pub full: Option<Blake3Hex>,
}

#[derive(Clone, Debug, Default, PartialEq, Eq, Serialize, Deserialize)]
#[serde(rename_all = "camelCase")]
pub struct XmpIds {
    pub document_id: Option<String>, // read-only hint (§7.3 X8)
    pub instance_id: Option<String>,
}

#[derive(Clone, Debug, PartialEq, Eq, Hash, Serialize, Deserialize)]
#[serde(try_from = "String", into = "String")] // "ac:<ULID>"
pub struct CatalogId(String);

#[derive(Clone, Debug, PartialEq, Eq, Serialize, Deserialize)]
pub struct Origin {
    pub app: String,
    pub version: String,
}

#[derive(Clone, Debug, PartialEq, Serialize, Deserialize)]
#[serde(rename_all = "camelCase")]
pub struct AssetRef {
    pub path: WirePath,
    #[serde(default)] pub name: Option<String>,
    #[serde(default)] pub size: Option<u64>,
    #[serde(default)] pub mtime: Option<String>, // RFC 3339 UTC, validated on parse
    #[serde(default)] pub media_type: Option<String>,
    #[serde(default)] pub hash: Hashes,
    #[serde(default)] pub xmp: Option<XmpIds>,
    #[serde(default)] pub catalog_id: Option<CatalogId>,
    #[serde(default)] pub origin: Option<Origin>,
    #[serde(flatten)] pub extra: Map<String, serde_json::Value>, // unknown fields kept
}

#[derive(Clone, Debug, PartialEq, Serialize, Deserialize)]
#[serde(tag = "status", rename_all = "camelCase")]
pub enum Resolution {
    Ok { path: WirePath, refreshed: bool, long_path: bool },
    Modified { path: WirePath },
    Moved { from: WirePath, to: WirePath },
    Ambiguous { candidates: Vec<WirePath> },
    Missing,
}

#[derive(Clone, Debug, PartialEq, Eq, thiserror::Error)] // or a hand-written Display; licence-checked
pub enum RefError {
    #[error("path is not absolute: {0}")] NotAbsolute(String),
    #[error("path form not supported: {0}")] Unsupported(String),     // device, GUID, ADS, `..`
    #[error("path is not valid Unicode")] NotUnicode,
    #[error("asset ref too large or malformed: {0}")] Invalid(String),
    #[error("payload version {found} of {kind} is not supported")] Version { kind: String, found: u32 },
}

impl WirePath {
    /// §7.4.2 N1–N6. Pure; `windows` passed in so both behaviours are tested everywhere
    /// (the pattern of `assetcraft` `paths.rs:13`).
    pub fn parse(raw: &str, windows: bool) -> Result<Self, RefError> { /* … */ }
    pub fn key(&self, windows: bool) -> PathKey { /* N7 */ }
}

/// Hashing behind a read trait, so tests use a fake file system (AssetCraft `Services` pattern).
pub trait ReadAt {
    fn len(&self) -> std::io::Result<u64>;
    fn read_at(&self, offset: u64, buf: &mut [u8]) -> std::io::Result<usize>;
}
pub const Q1_SPAN: u64 = 1 << 20; // = filmcraft relink.rs:28
pub fn quick_hash(file: &dyn ReadAt) -> std::io::Result<Blake3Hex> { /* "pipelinecraft-q1" ‖ size ‖ head ‖ tail */ }
/// Full hash with progress and cancel; returns Ok(None) when cancelled.
pub fn full_hash(file: &dyn ReadAt, progress: &mut dyn FnMut(u64) -> bool) -> std::io::Result<Option<Blake3Hex>> { /* … */ }
```

```rust
// Handoff payloads (§7.2): one enum, tagged by `type`, each variant versioned.

#[derive(Clone, Debug, PartialEq, Serialize, Deserialize)]
#[serde(tag = "type")]
pub enum Handoff {
    #[serde(rename = "handoff.asset")]      Asset(Versioned<AssetPayload>),
    #[serde(rename = "handoff.selection")]  Selection(Versioned<SelectionPayload>),
    #[serde(rename = "handoff.collection")] Collection(Versioned<CollectionPayload>),
    #[serde(rename = "handoff.storyline")]  Storyline(Versioned<StorylinePayload>),
    #[serde(rename = "handoff.timeline")]   Timeline(Versioned<TimelinePayload>),
    #[serde(rename = "handoff.blob")]       Blob(Versioned<BlobPayload>),
    #[serde(rename = "handoff.file")]       File(Versioned<FilePayload>),
}

#[derive(Clone, Debug, PartialEq, Serialize, Deserialize)]
pub struct Versioned<T> {
    pub version: u32, // refuse unknown → RefError::Version
    #[serde(flatten)]
    pub body: T,
}

#[derive(Clone, Copy, Debug, PartialEq, Eq, Serialize, Deserialize)]
#[serde(rename_all = "camelCase")]
pub enum Intent { Open, Place, Import, Thumbnail, Export } // O-B8 capability aliases

#[derive(Clone, Debug, PartialEq, Serialize, Deserialize)]
pub struct SelectionPayload {
    pub source: Option<SourceLocation>,
    pub items: Vec<AssetRef>, // ≤ 10,000, else handoff.file
    pub focus: Option<usize>,
    pub intent: Intent,
}

#[derive(Clone, Copy, Debug, PartialEq, Eq, Serialize, Deserialize)]
pub struct Rate { pub num: u32, pub den: u32 }

/// Ticks at 254,016,000,000 per second (filmcraft crates/time/src/lib.rs:20).
pub type Ticks = i64;

#[derive(Clone, Debug, PartialEq, Serialize, Deserialize)]
#[serde(rename_all = "camelCase")]
pub struct TimelinePayload {
    pub format: InterchangeFormat, // Otio in v1
    pub file: AssetRef,
    pub sequence_name: String,
    pub rate: Rate,
    pub expect: Option<TimelineExpect>, // AC3(c) check
    pub media: Vec<AssetRef>,
    pub storyline: Option<StorylineLink>,
}
// AssetPayload, CollectionPayload, StorylinePayload (block fields per §7.2.4),
// BlobPayload {media_type, name, base64}, FilePayload {payload_type, file} follow the JSON above.
```

Two crate notes:

- **`thiserror`** is MIT/Apache, but if the workspace avoids derive crates, use a hand-written `Display`.
- **Layer placement:** the crate's home follows D2 (A3). If PipelineCraft starts inside AssetCraft, `assetref` is a new L0 row in `xtask/src/layers.rs` `TABLE` (`AGENTS.md` rule 2), used by the planned L3 `interchange` and L4 `bridge`.

---

## Findings and opportunities from this section

| ID     | Scope                           | Opportunity                                                                                                                                                  | Suggested fix                                                                                                                                                                                                 | Owner                                          | Status |
|--------|---------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------|--------|
| A5-R1  | §7.1, D4                        | No shared asset identity; four incompatible fingerprints (§7.0.2)                                                                                            | Adopt the composite AssetRef: path anchor, size/mtime stamp, BLAKE3 `q1` + on-demand `full`, optional XMP ids and catalog id, resolution rules R0–R6                                                          | Paul (D4); Claude (`assetref` crate)           | open   |
| A5-O1  | AssetCraft Stage 2–3            | AssetCraft's planned "content hash" for dupes (`assetcraft-plan.md:152`) isn't specified yet                                                                 | Use BLAKE3 `full` (and `q1` as the pre-filter) as AssetCraft's content hash, so its catalog *is* the hub's hash index. Keep the SipHash thumbnail key as is                                                   | Claude (AssetCraft plan); Paul approves        | open   |
| A5-O2  | AssetCraft Stage 1.5 (O-B9)     | No way for the hub to ask AssetCraft "where is this asset now?"                                                                                              | Add `asset.ref {path}` → AssetRef (Stage 1.5, path + stamp + `q1`) and `asset.resolve {ref}` → Resolution (Stage 3, catalog-backed) to AssetCraft's registry                                                  | Claude                                         | open   |
| A5-O3  | §7.4.4 PhotoCraft (refines O13) | PhotoCraft's absolute-under-root check is case-sensitive past the drive and doesn't strip `\\?\` (`workspace.rs:157-172`)                                    | Hub: always send the root-relative forward-slash form. Upstream: compare components case-insensitively on Windows, as the shared `file_access.rs` copies do                                                   | Claude (adapter); PhotoCraft (PR)              | open   |
| A5-O4  | §7.3.2 XMP                      | FilmCraft `clip.updateMetadata` replaces a sidecar wholesale whenever `CreatorTool` is FilmCraft, which can erase ratings merged in by LightCraft/AssetCraft | Upstream: write through `lightcraft-meta::merge_xmp` (owned = its `dc:`/`xmpDM:` fields), or refuse when foreign namespaces are present. Hub meanwhile: mark it `write-user-file` + `destructive` and confirm | FilmCraft (PR, Paul files); Claude (hub table) | open   |
| A5-O5  | §7.2.5 OTIO                     | FilmCraft's OTIO import drops every metadata namespace except `filmcraft` (`otio.rs:678`)                                                                    | Upstream: keep unknown `metadata.*` on media items and write it back on export. Hub meanwhile: the ledger holds clip → AssetRef                                                                               | FilmCraft (PR); Claude (ledger)                | open   |
| A5-O6  | §7.2.5 OTIO                     | FilmCraft de-duplicates media by the raw path string (`otio.rs:696`)                                                                                         | AssetCraft's OTIO writer emits one wire path per file (§7.4.2). Upstream (optional): key by a case-folded path on Windows                                                                                     | Claude (writer); FilmCraft (optional)          | open   |
| A5-O7  | §7.2.4 Storyline                | Block JSON is defined here by hand from a field table                                                                                                        | Generate the storyline payload schema from the L3 `boards` crate's `schemars` output; this section's shape becomes the first test fixture                                                                     | Claude (AssetCraft Stage 4)                    | open   |
| A5-O8  | §7.4.3 long paths               | No app declares `longPathAware`; real behaviour unknown                                                                                                      | Run Not verified step 3; record `long_paths` per app in adapter config; if an app fails, a one-line manifest PR (PdfCraft's `windows_manifest.rs` is the template)                                            | Paul (run); Claude (config); apps (PR)         | open   |
| A5-O9  | §7.5 G7 proxies                 | FilmCraft writes proxies into the user's media folder by default (`proxies.rs:341`)                                                                          | Hub recipes pass an explicit `destination` (a user-chosen proxies root in hub settings); the default path is declared `write-user-file`                                                                       | Claude (hub); Paul (pick proxies root)         | open   |
| A5-O10 | §7.1 file identity              | Same-volume moves could be detected without hashing via the NTFS file id                                                                                     | Later: add an optional `fileId {volume, index}` hint through a small, licence-checked crate (e.g. `same-file`, MIT/Unlicense), never `unsafe` in the workspace. Not in v1                                     | Claude (later)                                 | open   |
| A5-O11 | Suite standard (§14)            | Each app hashes differently                                                                                                                                  | Offer `assetref` (or its hash recipe) to craft-libs next to the O-B1 `craft-control` family, so FilmCraft's Clip ID and LightCraft's `content_hash` can converge later. No ask in v1                          | Claude (incubate); craft-libs (later)          | open   |

---

## Not verified

Everything above is a static read; nothing was built or run. These need Paul's Windows machine.

**Next action (Paul):**

1. **Long paths enabled?**
   ```powershell
   Set-Location F:\Script-Projects\ArtCrafting
   Get-ItemProperty HKLM:\SYSTEM\CurrentControlSet\Control\FileSystem -Name LongPathsEnabled
   ```
   Expected: `LongPathsEnabled : 1` or `0`. Note which; step 3's results depend on it.

2. **Make a long-path fixture** (about 330 characters):
   ```powershell
   Set-Location F:\Script-Projects\ArtCrafting\assetcraft
   $deep = "\\?\F:\Script-Projects\ArtCrafting\lp\" + (("a" * 50 + "\") * 5) + "Réaper ünïcode name.png"
   New-Item -ItemType Directory -Force -Path (Split-Path $deep) | Out-Null
   Copy-Item -LiteralPath "test_data\library\edge-cases\Réaper ünïcode name.png" -Destination $deep
   $plain = $deep.Substring(4); $plain.Length
   ```
   Expected: a length over 300, and the file exists.

3. **Does each app open it?** (Run each line; note OK or the error text.)
   ```powershell
   Set-Location F:\Script-Projects\ArtCrafting\assetcraft
   cargo run -q -p assetcraft-cli -- probe "$plain" --json
   Set-Location F:\Script-Projects\ArtCrafting\lightcraft
   cargo run -q -p lightcraft-cli -- render "$plain" -o $env:TEMP\lp-lightcraft.png --size 256
   Set-Location F:\Script-Projects\ArtCrafting\filmcraft
   cargo run -q -p filmcraft-cli -- probe "$plain"
   ```
   Expected (static guess): all three succeed, because Rust std extends long paths. Any failure → `long_paths: refuse` for that app (A5-O8). Repeat with PhotoCraft `app.open` in step 6.

4. **Reproduce the sidecar conflict (§7.3.2)** on a copy, never on real media:
   ```powershell
   Set-Location F:\Script-Projects\ArtCrafting\assetcraft
   New-Item -ItemType Directory -Force $env:TEMP\xmp-test | Out-Null
   Copy-Item test_data\library\environments\snow_clearing_flythrough.mp4 $env:TEMP\xmp-test\
   ```
   Then: (a) in FilmCraft import it, set a Description, run Clip ▸ Update Metadata. (b) In LightCraft add `$env:TEMP\xmp-test` in place and give the clip 4 stars, which writes the sidecar. (c) Back in FilmCraft run Update Metadata again. (d) Run `Select-String -Path $env:TEMP\xmp-test\snow_clearing_flythrough.xmp -Pattern 'Rating'`.
   Expected per static read: after (b) the file has `xmp:Rating` and still says `CreatorTool="FilmCraft"`; after (c) the rating is **gone**. If the rating survives, close A5-O4 as not reproduced.

5. **BLAKE3 cost baseline** (the `b3sum` tool gives the same value as `full`):
   ```powershell
   Set-Location F:\Script-Projects\ArtCrafting
   cargo install b3sum --locked
   $big = "<path to your largest local video>"
   (Get-Item $big).Length / 1GB
   Measure-Command { b3sum --num-threads 1 $big } | Select-Object TotalSeconds
   Measure-Command { b3sum $big } | Select-Object TotalSeconds
   ```
   Expected: single-thread time ≈ size ÷ disk speed (around 17 s for 50 GB on NVMe). Record both; they set the "inline full hash" threshold in §7.1.3.

6. **PhotoCraft root case-sensitivity (A5-O3):**
   ```powershell
   Set-Location F:\Script-Projects\ArtCrafting\photocraft
   $tok = "$env:TEMP\pc-token.txt"
   Start-Process cargo -ArgumentList 'run','-q','-p','photocraft','--','--control','7990','--control-token-file',$tok,'--automation-read-root','F:\Script-Projects\ArtCrafting\assetcraft\test_data\library'
   # when the window is up:
   $c = New-Object System.Net.Sockets.TcpClient('127.0.0.1', 7990); $s = $c.GetStream()
   $w = New-Object System.IO.StreamWriter($s); $r = New-Object System.IO.StreamReader($s)
   $w.WriteLine('{"id":0,"method":"auth","params":{"token":"' + (Get-Content $tok -Raw).Trim() + '"}}'); $w.Flush(); $r.ReadLine()
   $w.WriteLine('{"id":1,"method":"app.open","params":{"path":"environments/snow_clearing_plate.jpg"}}'); $w.Flush(); $r.ReadLine()
   $w.WriteLine('{"id":2,"method":"app.open","params":{"path":"F:\\Script-Projects\\ArtCrafting\\assetcraft\\test_data\\library\\environments\\ruins_vertical.png"}}'); $w.Flush(); $r.ReadLine()
   $w.WriteLine('{"id":3,"method":"app.open","params":{"path":"F:\\script-projects\\artcrafting\\assetcraft\\test_data\\library\\environments\\ruins_vertical.png"}}'); $w.Flush(); $r.ReadLine()
   $c.Close()
   ```
   Expected per static read: id 1 and 2 return `ok`; id 3 returns "absolute paths must be inside the automation root". The `auth` frame shape is `photocraft` `docs/control-protocol.md:20-23` @ `fd28da5`; the token file is created if missing (`:18`).

7. **Two spellings of one file (`subst`):**
   ```powershell
   Set-Location F:\Script-Projects\ArtCrafting\assetcraft
   subst X: F:\Script-Projects\ArtCrafting\assetcraft\test_data
   cargo run -q -p assetcraft-cli -- probe X:\library\environments\snow_clearing_plate.jpg --json
   subst X: /D
   ```
   Expected: the probe succeeds. It confirms that the hub will see `X:\…` and `F:\…` as different path keys for the same file, which R4 (hash) reconciles.

8. **FilmCraft OTIO round trip with absolute `file://` URLs from a temp folder** (AC3 precondition, extends A1 step 4): export any sequence with `file.exportInterchange {"format":"otio","path":"$env:LOCALAPPDATA\\PipelineCraft\\handoff\\t1\\a.otio"}`, then `file.import {"paths":[that]}` in a new project. Expected: same clip count, and media online (the URLs are absolute; `relative_to` is unset by default, `otio.rs:239-242`).

## Open questions

1. **D4 hash choice:** BLAKE3 with a 1 MiB head/tail `q1` and an on-demand `full`, as recommended? And should AssetCraft adopt the same BLAKE3 value as its Stage 2/3 content hash (A5-O1), so the catalog doubles as the hub's index? (Paul)
2. **Lifetimes and locations:** are 24 h for handoff folders, 7 days for unclaimed renders and 512 MiB for hub thumbnails right for how you work? And where should proxies live by default: next to the media (FilmCraft today) or one proxies root you pick (A5-O9)? (Paul)
3. **Upstream asks:** file the FilmCraft sidecar-overwrite issue (A5-O4) and the "keep foreign OTIO metadata" request (A5-O5) now, or wait until the hub reproduces them (Not verified steps 4 and 8)? (Paul)
4. **Catalog id format:** `ac:<ULID>` minted by AssetCraft Stage 3. Is there a reason to prefer a UUID, or a content-derived id? (Paul, with A5-O2)
5. **Mapped drives:** is any of your media on a NAS reached both by drive letter and by UNC? If so, it may be worth resolving mapped drives to UNC in the path key (a Win32 call through a small crate) rather than relying on hash resolution. (Paul)

## Inputs for downstream agents

- **A3 (§4–5):** a new pure crate `assetref` (proposed L0: `WirePath`, `PathKey`, `AssetRef`, `Handoff`, `quick_hash`, `full_hash` behind a `ReadAt` trait). It is shared by the hub and AssetCraft (L3 `interchange`, L4 `bridge`) and sits with D2. Adapter config needs per-app path fields: `path_form: absolute | root-relative-slash`, `roots: none | gui | headless | both`, `long_paths: unknown | ok | refuse`. The hub owns `%LOCALAPPDATA%\PipelineCraft\{state,handoff,derived,renders,tmp}` with a user-only ACL. When the hub launches PhotoCraft or VectorCraft for automation it must pass read/write roots: inputs' common folder and the job's output folder.
- **A4 (§6):** the error codes this section needs are `asset_ref_invalid`, `path_not_absolute`, `path_unsupported`, `path_not_unicode`, `path_outside_root`, `path_too_long_for_app`, `case_ambiguous`, `asset_missing`, `asset_modified`, `asset_ambiguous` and `payload_version_unsupported`. Commands taking `path`/`paths` get an AssetRef-aware variant at the hub level, with the degrade rule (§7.1.5). Payloads are tagged `type` + `version`, and keep unknown fields. Sidecar-writing commands are `write-user-file`; FilmCraft `clip.updateMetadata` is also `destructive` until A5-O4 is fixed.
- **A6 (§8–9):** new events `asset.moved {from, to}`, `asset.changed {ref, what}` and `asset.hashed {ref, full}`. `full_hash` is a cancellable job reporting bytes. The handoff ledger (§7.4.6) records app-local ids returned by `file.import` etc., which AC3 uses for its clip → asset check (`expect` in `handoff.timeline`). Relink is an *offer* needing confirmation (§7.4.5). Dry runs write only to `handoff\` and `tmp\` (G9). GC is a hub job (G2).
- **A7 (§10, §12):** MCP resources for selection and recent assets should return `handoff.selection` / AssetRef JSON. Blobs ≤ 1 MiB inline, larger as `handoff.file`. `craftctl` needs `resolve <path|ref.json>` and `hash <path> [--full]` for debugging (suggestion).
- **A8 (§11):** path validation N1–N5 is hostile-input handling and must be fuzzed. Refused forms are device paths, volume GUIDs, ADS, `..`, unpaired surrogates and NULs. The hub's folder needs a user-only ACL (O12). Hash search R4 is bounded and never follows links. BLAKE3 was chosen partly so a planted collision can't redirect a relink.
- **A9 (§13):** test fixtures are `test_data/library/environments/dupes/snow_clearing_plate_copy.jpg` (same content, different asset) and `edge-cases/Réaper ünïcode name.png` (NFC name), plus a generated > 260-char path and a fake `ReadAt` for hash tests. Pure `WirePath::parse(raw, windows)` lets Windows rules be tested on Linux CI.
- **A10 (§15):** D4 recommendation = A5-R1 (composite; BLAKE3 `q1` + on-demand `full`; catalog id authoritative once it exists; XMP ids as hints only).
