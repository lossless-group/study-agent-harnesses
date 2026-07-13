---
name: Cline Profile
slug: cline
upstream: https://github.com/cline/cline
package: "cline (npm, CLI), Cline (VS Code Marketplace / OpenVSX), @cline/core / @cline/sdk (workspace packages)"
license: Apache-2.0
maintainer: Cline (cline.bot)
study: studies/agent-harnesses
profile_path: studies/agent-harnesses/cline
profile_kind: VS Code extension + CLI + headless "Cline Core", gRPC/protobuf host-bridge
date_created: 2026-07-13
---

# Cline — Profile

A profile of Cline as it lives in this study (`studies/agent-harnesses/cline/`). Cites pinned paths so you can jump to source rather than trust paraphrase. Read alongside `Profile__Continue.md` (also IDE-first, now archived) and `Profile__Opencode.md` (CLI-first, Effect/SQLite-native) — Cline is the entry in this study mid-transition from a single VS Code extension into a **three-surface product (VS Code extension, CLI, Kanban) sharing one extracted core SDK**, with MCP configuration kept deliberately **global-machine-scoped** rather than per-project, and a "Memory Bank" convention that is **pure prompt engineering with zero code support** — the opposite design choice from `.clinerules` itself, which is fully code-driven.

## TL;DR

The README states the scope plainly (`README.md:8-9`):

> The open source coding agent in your IDE and terminal.

Verified: `LICENSE:1-3` is the standard Apache License 2.0 header. Root `package.json:2` names the workspace `@cline/packages`, a Bun-managed monorepo (`"packageManager": "bun@1.3.13"`, `package.json:44`) with workspaces spanning `sdk/packages/*`, `apps/*`, and the VS Code webview (`package.json:5-15`).

Mechanically, three things define this checkout:

1. **A real SDK has been extracted from the extension, not the other way around.** `sdk/packages/core` (`@cline/core`), `sdk/packages/llms` (`@cline/llms`), and `sdk/packages/shared` (`@cline/shared`) are now dependencies *of* `apps/vscode` (`apps/vscode/package.json:467-470`) and of `apps/cli` (`apps/cli/package.json:99-100`), not the reverse. `.clinerules/sdk-migration.md:1-4` states this directly: *"The VSCode extension runs on the Cline SDK (`@cline/core`, `@cline/llms`, `@cline/shared`) through an adapter layer in `apps/vscode/src/sdk/`. The webview still talks gRPC; the adapter translates between gRPC handlers and SDK calls."* `.clinerules/general.md` additionally refers to "the standalone cline-core" as a real build target alongside the VS Code extension host, and `dist-standalone/` is called out as generated build output to avoid grepping (`.clinerules/general.md:22-26`).
2. **`.clinerules` is fully code-driven** — a directory (or legacy single file) walked, toggled per-file, YAML-frontmatter-scoped, and reconciled by `synchronizeRuleToggles()` (`apps/vscode/src/core/context/instructions/user-instructions/rule-helpers.ts:40-104`) — while **"Memory Bank" is pure documentation**: a `.clinerules/memory-bank.md` custom-instructions blob (`docs/best-practices/memory-bank.mdx:77-149`) with no dedicated parser, slash command, or file-watcher anywhere in `sdk/` or `apps/`.
3. **MCP servers are configured once, globally, for the whole machine** — `cline_mcp_settings.json` (`GlobalFileNames.mcpSettings`, `apps/vscode/src/core/storage/disk.ts:28`) lives under VS Code's *global* storage directory (`ensureSettingsDirectoryExists()` → `getGlobalStorageDir("settings")`, `disk.ts:130-132`) on the extension side, and under `resolveClineDataDir()/settings/cline_mcp_settings.json` (`sdk/packages/shared/src/storage/paths.ts:295-301`) on the SDK/CLI side — there is no per-workspace MCP config file analogous to Claude Code's `.mcp.json` or Continue's per-profile `mcpServers:`.

If you want one sentence: **Cline is mid-extraction from "one VS Code extension" into "a shared `@cline/core`/`@cline/llms`/`@cline/shared` SDK consumed by both the VS Code extension (via a gRPC-to-SDK adapter layer) and a standalone CLI/cline-core," `.clinerules` is a fully code-driven, per-file-toggleable, conditionally-scoped rules directory, "Memory Bank" is a documented prompt convention with no code behind it at all, and MCP servers are configured in one machine-global JSON file guarded by a cross-process directory-lock rather than scoped per project.**

## Why this exists — extracting a core out from under a running extension

Three load-bearing design choices explain the shape of this checkout:

1. **The SDK boundary is enforced by dependency direction, not aspiration.** `apps/vscode/package.json:467-470` lists `@cline/agents`, `@cline/core`, `@cline/llms`, `@cline/shared` as `workspace:*` dependencies of the *extension*; `apps/cli/package.json:99-100` lists `@cline/core` and `@cline/shared` as dependencies of the *CLI*. Both surfaces consume the same core rather than the CLI reimplementing extension logic or the extension being the source of truth other surfaces reach into.
2. **The adapter is explicit and load-bearing, not incidental glue.** `apps/vscode/src/sdk/` (20+ files: `cline-session-factory.ts`, `message-translator.ts`, `hooks-adapter.ts`, `legacy-state-reader.ts`, `provider-failure-telemetry.ts`, …) is where gRPC-shaped webview messages get translated into SDK calls and back. `.clinerules/sdk-migration.md:9-12` tells contributors to reference the pre-SDK implementation via `git show origin/main:path` when replacing a module — an explicit acknowledgment that this is a live migration with an old implementation still worth consulting, not a green-field rewrite.
3. **`HostProvider` is the platform-abstraction seam that makes a shared core possible at all.** `apps/vscode/src/hosts/host-provider.ts:6-18` states the rationale in its own doc comment: *"This system runs on two different platforms (VSCode extension and cline-core), so all the host-specific classes and properties are contained in here."* `HostProvider.hostBridge` (`host-provider.ts:26`) is a `HostBridgeClientProvider` — the same interface backed by a real in-process implementation inside VS Code (`apps/vscode/src/hosts/vscode/hostbridge-grpc-service.ts`, `hostbridge-grpc-handler.ts`) or an out-of-process gRPC client when running headless (`apps/vscode/src/hosts/external/host-bridge-client-manager.ts`, `grpc-types.ts`).

## The proto/gRPC layer — 23 `.proto` files, generated code excluded from search

`apps/vscode/proto/` contains 23 `.proto` files across two namespaces: `cline/*` (18 files — `task.proto`, `mcp.proto`, `state.proto`, `checkpoints.proto`, `browser.proto`, `worktree.proto`, `marketplace.proto`, `oca_account.proto`, `hooks.proto`, `remote_config.proto`, `slash.proto`, `ui.proto`, `commands.proto`, `models.proto`, `file.proto`, `web.proto`, `account.proto`, `common.proto`) and `host/*` (`window.proto`, `testing.proto`, and others) — the latter defining the `HostBridge` surface the extension/cline-core split rides on. `apps/vscode/package.json` pulls in `@grpc/grpc-js`, `@grpc/proto-loader`, `@grpc/reflection`, `nice-grpc`, and `grpc-tools`/`grpc-health-check` as real runtime and build dependencies (`package.json:446,472-474,517,528-529`), not vestigial. Generated protobuf output lands in `src/generated/` and `src/shared/proto/`, both explicitly flagged in `.clinerules/general.md:19-26` as build artifacts to avoid searching — *"Auto-generated from `proto/`; not the source of truth."* This confirms the webview-to-core channel is a real protobuf/gRPC wire protocol, with the VS Code in-process transport and a genuine network-capable gRPC transport as two implementations of the same `HostBridge` contract.

## `.clinerules` — directory-first, per-file toggles, YAML-frontmatter conditionals

This very checkout dogfoods the convention at its own root: `.clinerules/{storage.md, network.md, protobuf-development.md, cline-overview.md, sdk-migration.md, general.md, bun-and-node.md, debug-harness.md}` plus `.clinerules/workflows/{release.md, pr-review.md, hotfix-release.md, address-pr-comments.md, find-pr-reviewers.md, git-branch-analysis.md, writing-documentation.md}` and `.clinerules/hooks/`.

- **Discovery and toggle state.** `refreshClineRulesToggles()` (`apps/vscode/src/core/context/instructions/user-instructions/cline-rules.ts:7-34`) resolves two independent toggle sets — global (`ensureRulesDirectoryExists()`, a `~/Documents/Cline/Rules` directory) and local (`.clinerules` resolved against the workspace root) — and calls `synchronizeRuleToggles()` for each, excluding `.clinerules/workflows`, `.clinerules/hooks`, and `.clinerules/skills` from the local rules walk (`cline-rules.ts:23-27`) since those three subdirectories are separately-loaded conventions.
- **File-vs-directory precedence.** `synchronizeRuleToggles()` (`rule-helpers.ts:40-104`) branches on whether `.clinerules` is a file or a directory: directory case recursively enumerates files and toggles each individually (default-enabled, `rule-helpers.ts:56-68`); file case treats the single file as one toggle and clears any stale per-file toggles (`rule-helpers.ts:77-91`). `ensureLocalClineDirExists()` (`rule-helpers.ts:305-334`) is the one-way migration: if `.clinerules` exists as a plain file when a directory is needed, it's backed up, converted into a directory, and the original content becomes `default-rules.md` inside it — with rollback on failure.
- **YAML frontmatter conditionals, fail-open on parse error.** `getRuleFilesTotalContentWithMetadata()` (`rule-helpers.ts:206-259`) parses each rule file's frontmatter (`parseYamlFrontmatter`) and evaluates conditionals (`evaluateRuleConditionals`) against a `RuleEvaluationContext`; a malformed-but-present frontmatter fence is **not** dropped — the raw content ships to the model anyway (`rule-helpers.ts:229-235`), on the stated rationale that the LLM can still reason about the author's intended scoping even if the harness itself can't evaluate it.
- **Rule sources beyond `.clinerules` itself.** `RuleFileController.stateManager` (`rule-helpers.ts:160-182`) tracks toggle state not just for Cline's own rules/workflows but for **imported** `.cursor/rules`, `.cursorrules`, `.windsurfrules`, and `AGENTS.md` (`localCursorRulesToggles`, `localWindsurfRulesToggles`, `localAgentsRulesToggles`) — confirmed by the `GlobalFileNames` map itself (`apps/vscode/src/core/storage/disk.ts:36-39`: `cursorRulesDir`, `cursorRulesFile`, `windsurfRules`, `agentsRulesFile`). Cline explicitly ingests competitor-tool rule formats as first-class, toggleable sources, not just its own.
- **Remote rules layer separately.** `synchronizeRemoteRuleToggles()`/`getRemoteRulesTotalContentWithMetadata()` (`rule-helpers.ts:110-134,261-297`) handle an organization-pushed `GlobalInstructionsFile[]` remote-config layer, each with its own `alwaysEnabled` flag and toggle set — a fourth rule source beyond local/global/imported-format.

## Memory Bank — a documented convention with zero code behind it

`docs/best-practices/memory-bank.mdx` is the entire implementation. It describes a `memory-bank/` directory of six markdown files — `projectbrief.md`, `productContext.md`, `activeContext.md`, `systemPatterns.md`, `techContext.md`, `progress.md` (`memory-bank.mdx:19-27`, table at `35-42`) — and instructs the user to paste a block of custom instructions into a Cline Rules file, canonically `.clinerules/memory-bank.md` (`memory-bank.mdx:11-13,79`). The pasted instructions (`memory-bank.mdx:82-149`) are themselves just prose that tells the model, in first person, to treat these six files as its only continuity across a memory reset and to update them on request ("update memory bank", `memory-bank.mdx:145`).

A full-tree grep for `memory.bank`/`memory_bank`/`memoryBank` across `sdk/` and `apps/` (excluding `docs/`) returns **zero matches** — no parser, no dedicated slash command, no file-watcher, no special-cased directory in `GlobalFileNames`. The mechanism that makes this work at all is the generic `.clinerules` loader described above: `memory-bank.md` is loaded and injected into context exactly like `general.md` or `network.md`, with no Memory-Bank-specific code path. The FAQ section is explicit that this is deliberate and tool-agnostic (`memory-bank.mdx:159-160`): *"Memory Bank is a documentation methodology that works with any AI that can read docs."* This is the clean converse of `.clinerules` itself: the *rule-loading substrate* is code; the *Memory Bank pattern built on top of it* is 100% prompt content.

## MCP configuration — one global settings file, two independent implementations, a cross-process lock

Two MCP stacks coexist in this checkout, both centered on the same on-disk file, `cline_mcp_settings.json`:

- **`apps/vscode/src/services/mcp/McpHub.ts`** (1,909 lines) is the mature, extension-side implementation: a `chokidar`-backed settings-file watcher (`McpHub.ts:58-83`) that recomputes a **content-based fingerprint** of the connection-relevant view of the file before reacting, specifically to avoid a "self-perpetuating watcher → reconnect → write loop" during OAuth token churn (doc comment, `McpHub.ts:62-82`). `readAndValidateMcpSettingsFile()` (`McpHub.ts:186`) validates against `McpSettingsSchema`/`ServerConfigSchema` (Zod, `schemas.ts`), and `getMcpSettingsFilePath()` (`apps/vscode/src/core/storage/disk.ts:143-158`) resolves the path via `ensureSettingsDirectoryExists()` → VS Code **global** storage (not workspace storage) — confirmed by the `settingsDirectoryPath` argument's only call site chain terminating at `getGlobalStorageDir("settings")` (`disk.ts:130-132`).
- **`sdk/packages/core/src/extensions/mcp/config-loader.ts`** is the newer SDK-native implementation, sharing the same file via `resolveMcpSettingsPath()` (`sdk/packages/shared/src/storage/paths.ts:295-301`: `join(resolveClineDataDir(), "settings", "cline_mcp_settings.json")`, overridable only by `CLINE_MCP_SETTINGS_PATH`). It adds a genuine **cross-process advisory lock** implemented as a populated-directory rename dance (`tryAcquireSettingsLock`, `config-loader.ts:312-334`): a staging directory is created and populated with a unique owner-token file, then renamed into place, so the visible lock directory is never observably empty; a lock older than `SETTINGS_LOCK_STALE_MS = 10_000`ms (`config-loader.ts:270`) is treated as abandoned by a crashed holder and forcibly reclaimed (`reclaimStaleLock`, `config-loader.ts:336-368`). `atomicWriteSettingsFile()` (`config-loader.ts:245-261`) writes via temp-file-plus-rename so a concurrent reader — CLI, multiple VS Code windows, a JetBrains client — never observes a torn write. `runPureSettingsMutator()` (`config-loader.ts:530-551`) even **re-runs the mutator against a shadow copy** to assert it's pure/deterministic before trusting its result, throwing `McpSettingsMutatorPurityError` if a caller's mutator has side effects.
- **Schema evolution, not scope evolution.** `mcpRegistrationBodySchema` (`config-loader.ts:171-175`) is a `z.union` of a new nested `{ transport: {...} }` shape and two legacy flat shapes (`legacyStdioRegistrationSchema`, `legacyUrlRegistrationSchema`, `config-loader.ts:96-169`), the latter transformed forward into the new shape on load. This is schema migration within the same **global** file — nothing in either implementation introduces a per-workspace or per-project MCP config surface. `docs/mcp/mcp-overview.mdx` is the only MCP doc page in the tree and does not describe project-level scoping either.

## What's inside this submodule

| Path | What's there |
|---|---|
| `apps/vscode/src/` | The VS Code extension host — Controller, Task, webview provider, now running atop the SDK via an adapter |
| `apps/vscode/src/sdk/` | The gRPC-handler-to-SDK adapter layer (`cline-session-factory.ts`, `message-translator.ts`, `hooks-adapter.ts`, `legacy-state-reader.ts`) |
| `apps/vscode/src/hosts/` | `HostProvider` singleton + two `HostBridge` backends: `hosts/vscode/` (in-process gRPC service) and `hosts/external/` (out-of-process gRPC client, for headless/cline-core) |
| `apps/vscode/proto/{cline,host}/*.proto` | 23 protobuf definitions — the webview/core wire contract (`mcp.proto`, `task.proto`, `hooks.proto`, `state.proto`, `window.proto`, …) |
| `apps/vscode/src/services/mcp/McpHub.ts` | Legacy/extension-side MCP client — chokidar watcher, fingerprint-gated reconnect, Zod schemas |
| `apps/vscode/src/core/context/instructions/user-instructions/rule-helpers.ts` | `.clinerules` directory/file toggle sync, YAML-frontmatter conditionals, Cursor/Windsurf/AGENTS.md ingestion |
| `apps/vscode/src/core/storage/disk.ts` | `GlobalFileNames` — every on-disk convention name in one map; global vs. workspace path resolution |
| `apps/vscode/src/core/storage/skill-directories.ts` | `getSkillsDirectoriesForScan()` — 4 project dirs + 2 global dirs, project-first order |
| `sdk/packages/core/` (`@cline/core`) | The extracted core engine, consumed by both `apps/vscode` and `apps/cli` |
| `sdk/packages/core/src/extensions/mcp/` | SDK-native MCP manager, config-loader with cross-process lock, OAuth, name-transform |
| `sdk/packages/llms/` (`@cline/llms`) | Provider/model catalog package |
| `sdk/packages/shared/` (`@cline/shared`) | `storage/paths.ts` — every global path resolver (`resolveClineDataDir`, `resolveMcpSettingsPath`, `resolveSkillsConfigSearchPaths`, …) |
| `sdk/packages/agents/` (`@cline/agents`) | Agent-definition package |
| `apps/cli/` (`@cline/cli`, published as `cline`) | Terminal surface — `acp/`, `commands/`, `connectors/`, `tui/`, `wizards/mcp/` — depends on `@cline/core` + `@cline/shared` directly |
| `apps/cline-hub/` | Companion app referenced from root scripts (`bun -F @cline/cline-hub`) |
| `.clinerules/` | This repo's own dogfood rules — `sdk-migration.md`, `cline-overview.md`, `general.md`, `bun-and-node.md`, `network.md`, `protobuf-development.md`, `debug-harness.md`, plus `workflows/` and `hooks/` |
| `.cline/skills/`, `.agents/skills/` | This repo's own skills, in both of Cline's supported skill-directory conventions |
| `docs/best-practices/memory-bank.mdx` | The entire Memory Bank specification — no corresponding code |
| `docs/mcp/mcp-overview.mdx` | The only MCP doc page; describes usage, not per-project scoping (there isn't any) |
| `evals/`, `.kanban/` | Benchmark harness (`cline-bench`) and Kanban multi-agent task-board integration |

If you read three files: `.clinerules/sdk-migration.md` (states the extraction directly, in prose, from the team itself), `apps/vscode/src/hosts/host-provider.ts` (the `HostProvider`/`HostBridge` seam that makes one core runnable on two platforms), and `sdk/packages/core/src/extensions/mcp/config-loader.ts` (the cross-process lock around the one global MCP settings file).

## Mental model for using it well

- **Don't look for a per-project MCP config file — there isn't one.** Every MCP server Cline knows about is global to the machine (`cline_mcp_settings.json` under global/`~/.cline` storage), reconciled by a directory-based lock so multiple windows/processes can write safely. If you want per-project MCP scoping, this is the wrong harness to reach for as-is.
- **Treat `.clinerules` as a directory of independently toggleable files, not one blob.** Each file gets its own enabled/disabled state, can carry YAML-frontmatter conditionals, and — critically — a malformed frontmatter fence still ships its content to the model (fail-open), so a broken conditional doesn't silently delete a rule.
- **Memory Bank is prompt content you author, not a feature you enable.** There's no toggle, no special file format beyond plain markdown, and no code will ever look for `activeContext.md` by name — the entire mechanism is "paste these instructions into a `.clinerules/memory-bank.md` file" and let the generic rules loader do the rest.
- **The SDK boundary (`@cline/core`/`@cline/llms`/`@cline/shared`) is where new engine logic belongs**, with `apps/vscode/src/sdk/` and `apps/cli` as thin(ner) adapters — check `.clinerules/sdk-migration.md`'s advice to consult `origin/main` before reimplementing something that already existed pre-SDK.
- **Search `src/`, never `out/`, `dist/`, `dist-standalone/`, or `src/generated/`** — the latter are build output or generated protobuf code, explicitly called out in `.clinerules/general.md` as noise traps for `grep`/`search_files`.
- **Skills load from four project-scoped directories before two global ones**, in this order: `.clinerules/skills`, `.cline/skills`, `.claude/skills`, `.agents/skills` (project), then `~/.cline/skills`, `~/.agents/skills` (global) — `getSkillsDirectoriesForScan()` is the single source of truth for this ordering.

## When NOT to reach for this

- **You want per-project MCP server configuration.** Cline's MCP settings are global-machine-scoped by design (one JSON file, lock-guarded); if isolating MCP servers per repository/workspace is a hard requirement, Continue's per-profile `mcpServers:` array or Claude Code's `.mcp.json` is the better fit.
- **You want a code-enforced "the model must re-read these six files" memory mechanism.** Memory Bank is 100% convention — if the model ignores the instruction, or a session starts without loading `.clinerules`, nothing forces `activeContext.md`/`progress.md` to be read. There is no equivalent of Goose's SQLite session store or OpenCode's `SessionContextEpochTable` here; persistence is "files the user asked the model to maintain," not a database.
- **You want a stable, finished architecture to build against.** The SDK extraction is visibly mid-flight — `.clinerules/sdk-migration.md` explicitly tells contributors to diff against `origin/main`'s pre-SDK implementation, and the adapter layer (`apps/vscode/src/sdk/`) exists specifically to bridge an old gRPC-handler shape onto a new SDK shape. Expect internal churn.
- **You need a minimal, dependency-light single binary.** This is a Bun monorepo with a real protobuf/gRPC layer (23 `.proto` files, generated code, `grpc-tools`), a VS Code extension host, a webview React app, and a CLI, all sharing one core — much heavier than a "wrap an LLM with tool-calling" script.
- **You're evaluating Cursor/Windsurf-format rule ingestion as Cline's own invention.** It's real (`.cursor/rules`, `.cursorrules`, `.windsurfrules`, `AGENTS.md` all get their own toggle sets) but it's explicitly an interop feature layered onto `.clinerules`, not the primary format.

## How this compares to the rest of the study

| Axis | Continue | opencode | Cline |
|---|---|---|---|
| **Shape** | IDE extension (VS Code, JetBrains) + CLI, shared core (archived) | CLI + TUI + server + desktop + web, one shared core | VS Code extension + CLI + Kanban, extracted `@cline/core` SDK consumed by both, gRPC/protobuf host-bridge between webview and core |
| **Core/host split** | One shared TS `core/` package, no proto layer | One shared core, in-process (no IDE-host split needed) | Explicit and mid-migration: `HostProvider`/`HostBridge` abstracts VS Code-in-process vs. headless cline-core, over real gRPC (23 `.proto` files) |
| **Project-level rules/instructions** | `.continue/{assistants,agents,configs}/*.yaml`, multi-profile, live-switchable | `AGENTS.md`, `.opencode/` conventions | `.clinerules/` directory, per-file toggles, YAML-frontmatter conditionals, fail-open on parse error; also ingests `.cursorrules`/`.windsurfrules`/`AGENTS.md` |
| **Cross-session memory convention** | Not a named pattern in this study | `CONTEXT.md`-documented "Context Epoch" algebra, code-enforced, DB-persisted | "Memory Bank" — documented in `docs/best-practices/memory-bank.mdx` only; **zero code support**, purely a `.clinerules/memory-bank.md` prompt convention |
| **MCP config scope** | Per-profile `mcpServers:` array, project **and** global, live-diffed | Per-project + global config layering via walked-up `.opencode/opencode.json` | **Global only** — one `cline_mcp_settings.json` under machine-global storage, no per-project file, guarded by a cross-process directory-lock with staleness reclaim |
| **MCP transport support** | `stdio`, `sse`, `streamable-http` | `stdio` (local), `streamable-http`→`sse` fallback (remote), OAuth | `stdio`, `sse`, `streamableHttp`, OAuth (both `McpHub.ts` and the SDK's `manager.ts`/`oauth.ts`) |
| **Skills/durable capability discovery** | Not the primary unit (Hub `uses`/`with`/`override` blocks instead) | `.claude/skills`, `.agents/skills`, `.opencode/skill(s)`, remote URLs | `.clinerules/skills`, `.cline/skills`, `.claude/skills`, `.agents/skills` (project, in that order) then `~/.cline/skills`, `~/.agents/skills` (global) |
| **Maintenance status** | Archived, read-only | Active | Active, visibly mid-SDK-extraction |
| **Best fit** | IDE-native coding assistance with hot-swappable per-project rule/MCP bundles | Terminal-first, database-backed, observable agent work | Teams wanting one agent logic core reachable from both an IDE and a terminal/headless surface, at the cost of global-only MCP scoping and a memory convention that lives entirely in prose |

The crucial axis Cline owns in this study is **a real, in-progress core/host extraction with a protobuf-defined seam** — no other profiled harness shows its work this explicitly: the adapter layer, the dual in-process/out-of-process `HostBridge` implementations, and a `.clinerules/sdk-migration.md` file telling contributors to diff against the pre-extraction implementation are a live case study in turning a single IDE extension into a multi-surface product without a rewrite.

## One-line summary

> Cline is a VS Code extension, CLI, and Kanban front end mid-extraction onto a shared `@cline/core`/`@cline/llms`/`@cline/shared` SDK, bridged to the VS Code webview over a real protobuf/gRPC `HostBridge` (23 `.proto` files, dual in-process/out-of-process implementations) with an explicit adapter layer (`apps/vscode/src/sdk/`) translating old gRPC handlers onto new SDK calls; `.clinerules` is a fully code-driven, per-file-toggleable, YAML-frontmatter-conditional rules directory that also ingests Cursor/Windsurf/AGENTS.md formats, while the "Memory Bank" cross-session-memory pattern is pure documentation (`docs/best-practices/memory-bank.mdx`) with zero dedicated code — it rides entirely on the generic `.clinerules` loader; and MCP servers are configured in exactly one machine-global JSON file, guarded by a directory-rename cross-process lock with stale-lock reclaim, rather than scoped per project.
