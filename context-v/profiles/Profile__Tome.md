---
name: Tome Profile
title: "Tome — Profile"
slug: tome
upstream: https://github.com/devrelaicom/tome
site: https://tome-mcp.com/
package: "tome-mcp (crates.io; the installed binary is `tome`); Homebrew tap aaronbassett/tap/tome"
license: MIT OR Apache-2.0
maintainer: Aaron Bassett / Midnight Foundation (devrelaicom)
study: studies/agent-harnesses
profile_path: studies/agent-harnesses/tome
profile_kind: cross-harness skill catalog manager (Rust CLI + stdio MCP server)
pinned_sha: e7974c17453f41314cb3e169d43490295cb6108d
pinned_version: "0.7.18 (Cargo.toml); latest release tag v0.7.18, 2026-07-10"
date_created: 2026-11-20
date_modified: 2026-11-23
date_authored_initial_draft: 2026-11-20
date_authored_current_draft: 2026-11-23
authors:
  - Michael Staton
augmented_with:
  - Claude Code on Claude Opus 5.5
site_uuid: 7b82da5c-40c0-4bd1-93b6-50d5b8366d14
hex_code: g9215o
lede: >-
  Tome is not a harness. It writes the config for seventeen of them, then serves
  every skill over MCP so the agent loads one at a time instead of all of them at once.
summary: >-
  Source-cited profile of Tome, the closest existing tool to context-v.dev's skill layer. One
  local library of skills, commands, agents and hooks (git-hosted "catalogs" of plugins) is indexed
  into SQLite with sqlite-vec, and every enabled harness gets its native MCP config plus a managed
  `<!-- tome:begin/end -->` block in its rules file. The block carries a three-tier routing
  directive: load now, load before matching work, search before related work. Search runs on
  local BGE embedders and rerankers with a Qwen summariser. Two drift findings: the homepage says
  "no telemetry" while the code ships opt-out telemetry, and the site badge says v0.6.0 while the
  crate is at 0.7.18.
status: Reference
publish: true
type: Profiles
tags:
  - Skills
  - MCP
  - Cross-Harness
  - Context-Layer
---

# Tome — Profile

A profile of Tome as pinned in this study (`studies/agent-harnesses/tome/`, shallow, at
`e7974c1`, 2026-07-16). Paths are relative to that directory. Read alongside
`Profile__Continue.md` (per-project skill swaps inside one harness) and `Profile__Pi.md`
(skills as `.md` files loaded by one harness). Tome is the first entry in this study that
isn't a harness: it sits *above* the harnesses.

## TL;DR

- **What it is.** A Rust CLI and stdio MCP server that keeps one library of agent knowledge
  and wires it into every harness you use. `README.md:1-5`: it "makes Claude Code's plugin
  ecosystem work across other agentic coding harnesses."
- **The unit is a catalog.** A git repo holds plugins (`tome-catalog.toml`,
  `src/catalog/manifest.rs:45`). Each plugin (`tome-plugin.toml`, `:187`) holds entries:
  skills, commands, agents and hooks. Manifests are parsed with unknown fields rejected
  (`CONSTITUTION.md`, principle IV).
- **It writes each harness's own files.** `tome harness use <name>` writes the MCP config
  entry and a rules-file block for each of 17 harnesses: Claude Code, Codex, Cursor, Gemini,
  OpenCode, Copilot (CLI and VS Code), Devin, Cline, Junie, JetBrains AI, Antigravity, Pi,
  Crush, Zed, Kiro and Goose (`README.md`, harness-support table; `src/harness/*.rs`, one
  file per harness).
- **The agent pulls skills on demand.** `tome mcp` exposes `search_skills`, `get_skill` (with
  a `metadata_only` middle tier, `src/mcp/tools/get_skill.rs:4-11`), `list_plugins`,
  `list_catalogs`, `status` and `meta` (`src/mcp/tools/`). User-invocable commands become MCP
  prompts (`src/mcp/prompts.rs`).
- **Search is local.** The index is SQLite with the vendored `sqlite-vec` extension, using
  quantized BGE embedders and rerankers and a `qwen2.5-0.5b-instruct` summariser. The default
  `medium` profile downloads about 1.16 GB on first enable (`README.md:399-413`).

## Why it is in this study

The study asks how a harness decides "which tools/MCP servers are in scope at a given moment,
and load[s] or evolve[s] skills." Every other entry answers that from inside one harness. Tome
answers it across all of them, which is the problem the Lossless tree solves today with
`context-v/agent-skills/` plus `sync-skills-symlinks.sh` (macOS) or GNU Stow (Linux). That
approach links every skill into `~/.claude/skills/` and lets the harness see them all. Tome
links nothing: the harness sees a search tool and a short routing directive.

It is also the nearest neighbour to **context-v.dev**. Both treat agent context as a
versioned, git-hosted library that outlives any one harness. Tome covers the skill and
plugin half; context-v covers the documents (specs, plans, explorations) that Tome doesn't
model at all.

## Deep dive 1 — file organization: `~/.tome/` central, `.tome/config.toml` per project

- **Central store.** Everything lives under `~/.tome/`: the index, the models
  (`~/.tome/models/`, each download checked against a pinned SHA-256, `README.md:401`),
  logs (`~/.tome/logs/mcp.log`, JSON lines) and workspaces (`~/.tome/workspaces/<name>/`).
- **Projects hold a pointer, not content.** A project binds to a named workspace through a
  thin `<project>/.tome/config.toml` carrying `workspace = "<name>"`
  (`src/workspace/mod.rs:5`, `src/workspace/binding.rs:127`). Different projects can see
  different plugin sets without copying anything.
- **Writes to harness files are fenced.** In a rules file the developer already owns, Tome
  writes only between `<!-- tome:begin -->` and `<!-- tome:end -->` and preserves everything
  outside verbatim (`src/harness/rules_file.rs:10-12`, markers at `:58-65`). Mismatched
  markers are an error, not a guess (`:98-99`).
- **Drift is reported, not assumed.** `tome doctor` reports drift between what Tome wrote and
  what's on disk, and `tome doctor --fix` repairs it (`src/doctor/`, about 6k lines).

## Deep dive 2 — memory and state: an index, not a memory

Tome keeps no conversation memory. Its persistent state is the index
(`src/index/schema.rs:65-137`):

| Table | Holds |
|---|---|
| `skills` | one row per entry: catalog, plugin, name, `kind` (`skill`/`command`), description, `content_hash`, `searchable`, `user_invocable`, `when_to_use` (`:81-95`) |
| `skill_embeddings` | a raw f32 BLOB per skill with **no dimension in the schema**, so swapping embedders needs a re-embed, never a migration (`:102-110`) |
| `workspaces`, `workspace_skills` | which entries each workspace enables, plus a `tier` (default 3) per entry (`:111-120`) |
| `workspace_catalogs` | catalog URL and `pinned_ref` per workspace (`:121-127`) |
| `workspace_projects` | which project path is bound to which workspace (`:128-132`) |

`src/index/migrations.rs` carries the schema from v1 to v6 in place. Changing the embedder
profile prints a notice and blocks partial re-embeds until `tome reindex` runs, so a
half-migrated index can't exist (`README.md:420-428`).

## Deep dive 3 — tracing: logs and telemetry, no tool-call trace

- **MCP diagnostics** go to `~/.tome/logs/mcp.log` as JSON lines (`src/mcp/log.rs`).
- **Telemetry is on unless you turn it off.** `TELEMETRY.md:1-13` describes anonymous
  telemetry: closed enum values, integers, a random per-install UUID
  (`~/.tome/telemetry/id`), and plugin names only for an allowlisted catalog (today,
  Midnight). The default endpoint is `https://gauge-telemetry.fly.dev` (`TELEMETRY.md:29`).
  Turn it off with `tome telemetry off` or `TOME_TELEMETRY=0`; CI is auto-disabled. Code:
  `src/telemetry/` (10 files).
- **Drift finding.** The homepage colophon says the index, embeddings and reranker are on your
  disk with "no telemetry, no cloud." That sentence traces to the site design specs
  (`site/specs/2026-06-10-site-overhaul-design.md:104`,
  `site/specs/2026-06-01-tome-site-design.md:29`), written before telemetry shipped. Search
  is local; usage events are not.
- **Exit codes are the contract.** Every named failure has its own exit code (`CONSTITUTION.md`
  principle II; `src/error.rs`, 2.4k lines), which matters more to Tome than tracing because
  it's driven from scripts and CI.

## Deep dive 4 — tool/MCP scoping: one MCP server per harness, five read-only tools

Tome registers itself, not your plugins' MCP servers, as the harness's MCP entry. The
harness then sees a small fixed toolset whatever the library size. Where a harness has no
writable MCP config (JetBrains AI is UI-only; Pi needs `pi-mcp-adapter`),
`tome harness info <name>` prints a snippet to paste (`README.md`, harness-support notes).
`src/harness/mcp_config.rs` (1.6k lines) handles the per-harness config formats.

Every rules block opens with a self-healing preamble
(`src/harness/routing.rs:29`): if the Tome tools are missing, the agent tells the user to run
`tome harness use <harness>` and restart, instead of failing silently.

## Deep dive 5 — skills: tiered routing instead of loading everything

This is the part worth borrowing. `build_directive()` (`src/harness/routing.rs:140-218`)
sorts each workspace's entries by `tier` and renders three sections:

1. **"Load now (Tier 1)."** Call `get_skill` for these at session start (`:171-180`).
2. **"Load before matching work (Tier 2)."** Load before a task that matches (`:182-194`).
3. **"Search before related work (Tier 3)."** A prose summary of what the workspace covers,
   written by the local summariser and capped at 1,500 characters
   (`TIER3_SUMMARY_MAX_CHARS`, `:40`), with a fallback to listing entries (`:196-217`).

The directive is delivered at session start through a session-start hook where the harness
has one (`session_start_hook`, `:243`; Codex variant at `:264`), a Tome-shipped TypeScript
plugin shim for OpenCode, Cline and Pi, or the rules file alone. Tiers are set per entry or
per plugin (`src/index/skills.rs:1240`, `:1282`).

**Authoring.** `tome {catalog,plugin,skill} create` scaffolds, `convert` imports a Claude Code
marketplace, plugin or loose `SKILL.md` (`src/authoring/import/claude_code.rs`, 2.2k lines),
and `lint` checks for CI (`src/authoring/lint/rules.rs`). Conversion rewrites
`${CLAUDE_SKILL_DIR}` to `${TOME_SKILL_DIR}` (`src/authoring/rewrite.rs`).

## What's inside this submodule

```text
tome/
├── README.md            # concepts, quick start, 17-harness support table, models
├── ARCHITECTURE.md      # module map
├── CONSTITUTION.md      # 12 binding principles: exit codes, strict schemas, KISS, CI gates
├── TELEMETRY.md         # the complete telemetry contract
├── CHANGELOG.md         # Keep-a-Changelog; 0.3.0 (2026-05-14) → 0.7.0 (2026-06-16) → Unreleased
├── build.rs             # compiles vendored sqlite-vec; llama.cpp via llama-cpp-2
├── vendor/sqlite-vec/   # vector search, compiled into the binary
├── site/                # Docusaurus site for tome-mcp.com, with its own specs/ and plans/
├── src/
│   ├── catalog/         # tome-catalog.toml parsing, git clone/update
│   ├── plugin/          # tome-plugin.toml, enable/disable lifecycle
│   ├── index/           # SQLite schema v6, migrations, KNN query, tiers
│   ├── embedding/       # BGE embedders + rerankers (ONNX via ort)
│   ├── summarise/       # qwen2.5-0.5b summariser for the Tier 3 prose
│   ├── harness/         # one file per harness; rules_file.rs, routing.rs, mcp_config.rs, hooks.rs, reconcile/
│   ├── mcp/             # stdio server; tools/ (search_skills, get_skill, list_*, status, meta); prompts.rs
│   ├── workspace/       # named workspaces + .tome/config.toml binding
│   ├── authoring/       # create / convert / lint
│   ├── doctor/          # status + drift detection and --fix
│   └── telemetry/       # opt-out usage events (gauge-telemetry)
└── tests/               # integration tests per command, real git fixtures
```

About 211k lines of Rust across `src/` and `tests/`.

## Mental model for using it well

- **Tome is a librarian, not a harness.** It never runs a model loop. It decides what the
  harness *can find*, and the three tiers decide what the agent *loads unprompted*.
- **For Lossless:** the skills half of context-v could ship as a Tome catalog (one
  `tome-catalog.toml` at the root of `lossless-agent-skills`). Every harness would then get
  the skills by search, without Stow or the symlink script. What Tome doesn't cover is the
  rest of context-v: specs, plans and explorations aren't entries in its model.

## Open questions

1. Could `lossless-agent-skills` publish as a Tome catalog without changing its `SKILL.md`
   files? Run `tome catalog convert` and `lint` against it to find out.
2. Does the Tier 3 summary hold up for a library of 60+ skills, or does search carry it?
3. Is a 1.16 GB model download acceptable for collaborators, versus the `small` profile?

## Changes since the previous pin (2026-07-16 → 2026-07-16)

Nothing changed: `main` is still at `e7974c17453f` (0 new commits as of 2026-11-23) and the latest release is still `v0.7.18`, so every citation above holds.

## Drift noted at this pin

- The homepage says "no telemetry"; `TELEMETRY.md` documents opt-out telemetry.
- The site's version badge reads v0.6.0 (`site/docusaurus.config.ts:51`); the crate and the
  latest release are 0.7.18 (`Cargo.toml`; GitHub release `v0.7.18`, 2026-07-10).
- `Cargo.toml` sets `homepage` to `https://tome-mcp.netlify.app/`, noting the custom domain
  "is not live yet"; tome-mcp.com is live now.
- The site's install line uses `aaronbassett/homebrew-tap/tome`; the README uses
  `aaronbassett/tap/tome`. Both resolve to the same tap in Homebrew's naming.
