# PipelineCraft (working name): spec draft v0

_2026-10-15 · spec draft v0 · status: drafted by section agents, integration pass and peer review not yet done_

- **Scope:** the local hub that discovers, calls, chains and observes the Crafting apps on Paul's machine, and exposes the suite as one MCP server. Separate from the Storyteller web API.
- **Project goal it serves:** local-first Crafting apps that hand work to each other (AssetCraft → StoryCraft → FilmCraft, later PrepressCraft) without the cloud.
- **Immediate area of improvement:** the apps can each be driven locally but only one at a time, with different auth and no discovery. This spec turns that into one contract.

## Reading order

| File                                        | Sections | Agent      | What's in it                                                                       |
|---------------------------------------------|----------|------------|------------------------------------------------------------------------------------|
| `00-outline.md`                             | all      | Claude     | Outline, agent assignments, review plan                                            |
| `15-decisions.md`                           | §15      | A10        | **Start here.** 31 decisions with recommendations, plus 29 cross-section conflicts |
| `01-goals-glossary.md`                      | §1–2     | A1         | Problem, goals, non-goals, AC1–AC4, P/R targets, name shortlist, glossary          |
| `03a-inventory.md`                          | §3       | A2a        | PhotoCraft, LightCraft, FilmCraft, PdfCraft, WordCraft control surfaces (O1–O14)   |
| `03b-inventory.md`                          | §3       | A2b        | DesignCraft, EffectCraft, VectorCraft, SoundCraft, ArtCraft, craft-libs, AssetCraft |
| `04-05-architecture-discovery.md`           | §4–5     | A3         | Components, crates and layer rows, adapters (TOML for all apps), discovery, lifecycle |
| `06-protocol.md`                            | §6       | A4         | Envelope, addressing, schemas, command effects, 35 error codes, 8 examples         |
| `07-asset-refs.md`                          | §7       | A5         | AssetRef (path + size/mtime + BLAKE3), handoff payloads, XMP, Windows paths, temp files |
| `08-09-events-jobs-workflows.md`            | §8–9     | A6         | Events, pollers, jobs, recipes, AC1–AC4 sequence diagrams, AssetCraft 1.5 commands |
| `10-12-mcp-craftctl.md`                     | §10, §12 | A7         | 14-tool MCP surface, MCP 2026-07-28, confirmation, resources, `craftctl`, `doctor` |
| `11-security.md`                            | §11      | A8         | Binding, tokens, browser attacks, trust tiers A–D, policy, audit, threat table, S1–S12 |
| `13-14-16-engineering-upstream-stages.md`   | §13–14, §16 | A9      | Quality gate, testing, 31-step Windows checklist, upstream PRs U1–U28, stages, AssetCraft 1.5 text |

## Evidence basis

Every source fact is a static read of `4waymedia/assetcraft` @ `3a41b2b` and the storytold repos at HEAD on 2026-10-15. Nothing was built or run. The consolidated live checks are in `13-14-16` §13.5.

## Next steps

1. **Paul:** decide the five "★ decide first" items in `15-decisions.md` (D9, D2, D8, D3, D10).
2. **Claude:** an integration pass that merges the sections into one `pipelinecraft-spec.md`, applies the conflict resolutions A10-X1 to X29, and marks undecided items as "default, not decided".
3. **Claude:** a peer review by three fresh agents, each with one focus (protocol/security, workflows/UX, upstream fit); findings go in a table.
4. **Paul:** review the H-severity findings, then tag v0.1.
