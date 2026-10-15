# STATUS

_Last updated: 2026-10-15_

_2026-10-15 · spec draft v0 · status: imported into the repo; integration pass and peer review not done_

- **Scope:** the PipelineCraft repo and its spec in `docs/spec/`.
- **Project goal it serves:** one local contract so the Crafting apps can hand work to each other.
- **Immediate area of improvement:** Paul's five "★ decide first" decisions.

## Next

| #   | Step                                                                                       | Owner  | Status |
|-----|--------------------------------------------------------------------------------------------|--------|--------|
| 1   | Decide D9, D2, D8, D3, D10 in `docs/spec/15-decisions.md`                                  | Paul   | open   |
| 2   | Integration pass: merge sections into one `pipelinecraft-spec.md`, apply X1–X29            | Claude | open   |
| 3   | Peer review by three fresh agents (protocol/security, workflows/UX, upstream fit)          | Claude | open   |
| 4   | Review H-severity findings, tag spec v0.1                                                  | Paul   | open   |

## Decisions

- Decision (Paul, 2026-10-15): the spec gets its own repo, `4waymedia/pipelinecraft`, so it has
  history before any code exists. **This does not settle D2.** D2 (hub code incubates in AssetCraft
  vs lives here) stays open; the spec's recommendation is to incubate in AssetCraft and move the
  hub here at Stage 2.
- Decision (Paul, 2026-10-15): licence MIT OR Apache-2.0, matching AssetCraft and the sibling apps.

## Verified

- Spec files imported unchanged from `ArtCrafting\drafts\pipelinecraft\` (13 files).
- Evidence basis of the spec: static read of `4waymedia/assetcraft` @ `3a41b2b` and the storytold
  repos at HEAD on 2026-10-15. Nothing was built or run.
