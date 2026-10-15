# PipelineCraft (working name): spec outline

_2026-10-14 · pre-Stage 0 · outline, ready for section agents_

- **Scope:** a local hub that discovers the Crafting apps running on Paul's machine, normalizes how they're called, and routes commands, assets and events between them. It also exposes the whole suite as one MCP server.
- **Project goal it serves:** a suite of local-first creative tools that work together, so that AssetCraft → StoryCraft → FilmCraft (and later PrepressCraft) can hand work to each other without the cloud.
- **Immediate area of improvement:** each app can be driven locally today, but only one at a time, each with different auth and no discovery. This spec turns that into one contract.

> Not in scope: the Storyteller web API (accounts, credits, generation). That stays a single AI provider inside AssetCraft and never routes through this hub.

---

## 1. Purpose, goals, non-goals

- 1.1 Problem statement: the two APIs (local app control vs Storyteller web API) and why neither is enough on its own
- 1.2 Goals: discover, call, chain, observe, automate. Usable by apps, by `craftctl`, and by MCP clients (Claude, local LLM stack)
- 1.3 Non-goals: replacing each app's own command registry; cloud sync; Elo integration; remote/LAN access (v1 is loopback only); 3D
- 1.4 Success criteria: three acceptance workflows that work end to end (see §9)
- 1.5 Naming: PipelineCraft is a working name; candidates go here and Paul decides

## 2. Glossary

- App, command, command id, params, effect, job, event, asset ref, manifest, adapter, hub, session, recipe
- The two APIs, defined once and referenced everywhere

## 3. Current landscape: what each app exposes today

- 3.1 Inventory table per app: control transports (`--control`, `-cli`, MCP stdio), auth model, command count, event support, tag/commit
- 3.2 Auth variance: PhotoCraft 256-bit token, PdfCraft token file, WordCraft key file, the rest none
- 3.3 Command naming conventions observed across apps, and where they agree or diverge
- 3.4 AssetCraft's own planned surface (Stage 1.5: serve, MCP, `--control`, command effects)
- 3.5 Every claim cites `repo` + `file:line` + tag, and is verified live with `<app>-cli commands --json`

## 4. Architecture

- 4.1 Components: hub daemon, discovery, per-app **adapters**, router, event bus, job manager, MCP front door, `craftctl` CLI, optional egui status panel (L6)
- 4.2 Deployment shape: separate binary vs library inside AssetCraft (open decision, see §15)
- 4.3 Crate layout and layer-table rows (`cargo xtask layers`)
- 4.4 Adapter model: one adapter per transport (control socket, headless CLI, MCP stdio), with per-app quirks in config rather than code
- 4.5 Diagram: apps ↔ adapters ↔ router ↔ {craftctl, MCP, apps}

## 5. Discovery and app lifecycle

- 5.1 How the hub finds apps: manifest folder, running-process probe, or a config list of installed apps
- 5.2 Manifest schema (app, version, pid, port, auth ref, capabilities) and where it lives
- 5.3 Attach vs launch: attach to a running GUI, or launch `-cli` headless for batch work
- 5.4 Health, heartbeat, crash/restart, stale-manifest cleanup
- 5.5 Version negotiation and what happens when an app is older than the hub expects

## 6. Hub protocol: the contract

- 6.1 Transport and framing (align with the suite's `engine.execute` / `engine.commands` shape rather than inventing a new one)
- 6.2 Addressing: `app/command.id`, plus broadcast and capability-based routing ("whoever can export OTIO")
- 6.3 Params and results: JSON Schema per command, sourced from each app's registry
- 6.4 **Effects** declaration on every command: read / write-user-file / write-app-state / external-network / destructive
- 6.5 Errors: code table, retryable vs fatal, and app-originated vs hub-originated
- 6.6 Versioning: `protocol` version, deprecation rules
- 6.7 Concrete examples: request/response JSON for 6 to 8 real calls

## 7. Asset reference model (the critical one)

- 7.1 How two apps agree they mean the same asset: path, content hash, XMP document id, or AssetCraft id
- 7.2 Handoff payloads: single asset, selection, collection, storyline block, OTIO timeline
- 7.3 Sidecars and metadata travel (XMP), and who owns writes
- 7.4 Path rules on Windows: long paths, UNC, drive letters, moves and renames after handoff
- 7.5 Temp/derived files: thumbnails, proxies, renders, and where they live and how long

## 8. Events, jobs and long-running work

- 8.1 Event subscription: topics, filtering, back-pressure
- 8.2 Core events: app up/down, selection changed, asset changed, job progress/done/failed
- 8.3 Jobs: id, progress, cancel, timeout, result location
- 8.4 Chaining: a step's output becomes the next step's input (the seed of recipes)
- 8.5 Batch rules: dry run first, then commit, with undo where the target app supports it

## 9. Cross-app workflows (acceptance cases)

- 9.1 **Open In:** AssetCraft selection → PhotoCraft / LightCraft / VectorCraft
- 9.2 **Thumbnails:** AssetCraft asks the owning app to render previews for formats it can't decode
- 9.3 **Plan → FilmCraft:** Storyline Builder blocks → OTIO → FilmCraft timeline
- 9.4 Recipes: a saved chain run headless across apps, with a dry run
- 9.5 Later: PrepressCraft preflight from DesignCraft/PdfCraft output; screen capture → AssetCraft ingest
- Each case: sequence diagram, commands used, failure paths, what "done" looks like

## 10. MCP front door

- 10.1 Tool generation from the merged command registry, with naming and grouping to keep the tool count usable
- 10.2 Effects → MCP annotations (read-only hint, destructive hint)
- 10.3 Confirmation policy for write/destructive tools
- 10.4 Resources: expose running apps, current selection and recent assets as MCP resources
- 10.5 Works with Claude and with the local llama.cpp stack (stdio first, then streamable HTTP on loopback)

## 11. Security and trust

- 11.1 Loopback-only binding, with no LAN access in v1
- 11.2 Auth normalization: the hub holds each app's token/key file, and clients authenticate to the hub once
- 11.3 Browser-origin rejection (DNS rebinding / CSRF), secrets in the OS keyring
- 11.4 Per-client permissions: which caller may run which effect class
- 11.5 Audit log of every write/destructive call (who, what, params, result)
- 11.6 Threat model table: malicious web page, rogue local process, prompt-injected MCP client

## 12. `craftctl` CLI

- 12.1 Commands: `apps`, `commands <app>`, `call <app> <id> --params`, `watch`, `run <recipe> --dry-run`
- 12.2 Output: human by default, `--json` for scripts
- 12.3 PowerShell examples, each starting with `Set-Location`

## 13. Engineering rules and quality gate

- 13.1 Inherit AssetCraft `AGENTS.md`: Rust only, no `unsafe`, no `unwrap`/`expect`/`panic` outside tests, clean room, no GPL/LGPL/AGPL
- 13.2 Testing: a **fake app** fixture that speaks each adapter's transport; contract tests run against every real app's `commands --json`
- 13.3 `cargo xtask ci` gate; Windows verification checklist
- 13.4 Performance targets: call overhead, discovery time, event latency

## 14. Upstream and fork strategy

- 14.1 What should become a suite standard and be PR'd to storytold (manifest file, common auth, effects field)
- 14.2 What stays hub-side as adapter config if upstream says no
- 14.3 Fork workflow per §6 of the project context, with personal notes kept out of PRs

## 15. Open decisions (each with a recommendation)

| ID | Decision                              | Recommendation (to be argued by the agent)                                  | Owner |
|----|---------------------------------------|-----------------------------------------------------------------------------|-------|
| D1 | Final name                            | Shortlist of 5                                                              | Paul  |
| D2 | Separate repo vs crate in AssetCraft  | Start as a workspace crate + `craftctl`, then promote to a repo at Stage 2  | Paul  |
| D3 | Hub protocol shape                    | Mirror `engine.execute`/`engine.commands`; no new dialect                   | Paul  |
| D4 | Canonical asset id                    | Content hash + path, with XMP id when present                               | Paul  |
| D5 | Daemon always-on vs on-demand         | On-demand, started by the first client                                      | Paul  |
| D6 | Upstream standardization ask          | One small PR first (manifest file) to test the waters                       | Paul  |

## 16. Stages

- **0 See:** discovery + health + `craftctl apps`, read-only
- **1 Ask:** call read-only commands on any app through the hub
- **2 Hand off:** asset refs + Open In + thumbnails (§9.1–9.2)
- **3 Chain:** jobs, events, recipes with dry run
- **4 MCP:** front door with effect annotations and confirmations
- **5 Produce:** Plan → FilmCraft end to end (§9.3)

---

## Agent assignments (for the drafting pass)

Each agent gets: this outline, `claude/artcrafting-context.md`, AssetCraft `AGENTS.md`, and the section brief below. Each one writes `docs/pipelinecraft/NN-<section>.md` in the house doc format (header, padded tables, file:line evidence, an explicit "not verified" list, and an open-questions list).

| Agent | Sections   | Must produce                                                         | Depends on |
|-------|------------|----------------------------------------------------------------------|------------|
| A1    | 1, 2       | Goals/non-goals, glossary, success criteria                          | none       |
| A2    | 3          | Verified inventory of every app's control surface, with file:line    | none       |
| A3    | 4, 5       | Architecture, crate/layer rows, discovery and lifecycle              | A2         |
| A4    | 6          | Protocol contract with JSON examples and error table                 | A2         |
| A5    | 7          | Asset reference model and handoff payloads                           | A2         |
| A6    | 8, 9       | Events, jobs, and the acceptance workflows as sequence diagrams      | A4, A5     |
| A7    | 10, 12     | MCP front door and `craftctl`                                        | A4         |
| A8    | 11         | Security model and threat table                                      | A3, A4     |
| A9    | 13, 14, 16 | Engineering gate, upstream strategy, stage plan                      | A3         |
| A10   | 15         | Decision memo, one page per decision, each with a recommendation     | all        |

**Order:** A1, A2 → A3, A4, A5 → A6–A9 → A10.

## Review plan

1. **Integration pass:** one agent merges the sections, removes contradictions, and logs conflicts as findings.
2. **Peer review:** two to three fresh agents that didn't draft anything, each with its own lens (protocol/security, workflows/UX, upstream-fit). Findings go in the house table format: ID, file, fix, owner, status.
3. **Paul review:** decisions in §15 plus any H-severity findings.
4. **Revise:** archive superseded drafts in `docs/archive/` and tag the spec v0.1.
