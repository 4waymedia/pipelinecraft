# AGENTS.md: rules for coding agents

PipelineCraft is the local hub for the Crafting apps: discover, call, chain and observe them, and
expose the suite as one MCP server. Read this file, then `STATUS.md`, then `docs/spec/README.md`.

## Current stage: spec only

There is no code yet. Work in this repo is spec work in `docs/spec/` until Paul tags spec v0.1 and
decides D2 (where the hub code lives).

## Non-negotiables (suite house style, same as AssetCraft)

1. **Rust only.** No Tauri, Electron, webview or JS UI.
2. **Everything is a command.** Stable id plus JSON params, through one `execute(id, params)`.
   CLI, MCP and the control channel use the same commands.
3. **Never crash.** No `unsafe`; no `unwrap`, `expect` or `panic` outside tests. All input from
   disk, sockets and other apps is hostile.
4. **Local first.** Loopback only; nothing listens on a public interface. Secrets live in the OS
   keyring, never in files.
5. **Clean room.** No Adobe or Microsoft code or assets, no GPL/LGPL/AGPL dependencies. Code
   adapted from a sibling app gets a provenance line in `NOTICE` (repo, path, revision).
6. **Two APIs, never mixed.** Local app control is each app's command registry (`<app>-cli`, MCP
   over stdio, the opt-in loopback `--control` channel). The Storyteller web API is ArtCraft's
   account backend only and does not control local apps.
7. **Elo is out of scope.**

## Docs style

- Each doc opens with `_date · stage · status_` and three bullets: Scope, Project goal it serves,
  Immediate area of improvement.
- Claims about sibling apps cite repo, `file:line` and commit or tag.
- Findings have an ID, a fix, an owner and a status. Gaps are written as opportunities.
- Markdown tables are space-padded so columns line up. Superseded docs move to `docs/archive/`.

## Before you hand work back

- Update `STATUS.md`.
- **Never run git write commands.** Paul commits and pushes himself, so hand back a PowerShell
  block that starts with `Set-Location F:\Script-Projects\ArtCrafting\pipelinecraft`.
