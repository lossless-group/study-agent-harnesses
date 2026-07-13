---
name: Continue Profile
slug: continue
upstream: https://github.com/continuedev/continue
package: "@continuedev/cli (npm), Continue (VS Code / OpenVSX), Continue (JetBrains)"
license: Apache-2.0
maintainer: continuedev (archived — read-only as of the pinned commit)
study: studies/agent-harnesses
profile_path: studies/agent-harnesses/continue
profile_kind: IDE extension (VS Code, JetBrains) + CLI, shared YAML config core
date_created: 2026-07-13
---

# Continue — Profile

A profile of Continue as it lives in this study (`studies/agent-harnesses/continue/`). Cites pinned paths so you can jump to source rather than trust paraphrase. Read alongside [`Profile__Aider.md`](./Profile__Aider.md) (no MCP, git-commit-as-trace) and `Profile__Goose.md` (extensions-as-MCP) — Continue is the entry in this study built around **declarative, file-based, per-project YAML configuration that composes models, rules, prompts, and MCP servers into swappable "assistants."**

## TL;DR

The README states plainly, and with a note worth taking seriously (`README.md:19`):

> _Note: The `continuedev/continue` repository is no longer actively maintained and is read-only for all users._

Continue ships as three surfaces — VS Code extension, JetBrains plugin, and a standalone CLI (`extensions/cli/`) — all built on one shared TypeScript core (`core/`) and one shared YAML config package (`packages/config-yaml/`). Verified: `LICENSE` at the repo root is the standard Apache License 2.0 header (`LICENSE:1-3`).

The load-bearing mechanism for this study's question is `.continue/` **as a per-project directory of YAML/Markdown definition files**, not a single config blob. `ConfigHandler` (`core/config/ConfigHandler.ts:161-190`) builds a list of `ProfileLifecycleManager`s by scanning three subdirectories per workspace — `.continue/assistants`, `.continue/agents`, `.continue/configs` — for `.yaml`/`.yml` files, one profile per file, **plus** a global `~/.continue` profile (`getAllDotContinueDefinitionFiles`, `core/config/loadLocalAssistants.ts:131-156`). Each YAML file is a self-contained `ConfigYaml` (`packages/config-yaml/src/schemas/index.ts:113-153`) that can declare its own `models`, `context`, `rules`, `prompts`, `docs`, and — the piece this study cares most about — `mcpServers`. Users pick which profile ("assistant") is active per workspace via `setSelectedProfileId` (`core/config/ConfigHandler.ts:207-231`), and the choice is persisted per-workspace-id in a `GlobalContext` (`lastSelectedProfileForWorkspace`), so **switching skillsets is a live, in-session dropdown action, not a restart.**

MCP servers declared in a profile's `mcpServers:` array (schema at `packages/config-yaml/src/schemas/mcp/index.ts:4-35`, supporting `stdio`, `sse`, and `streamable-http` transports) are diffed and reconciled by `MCPManagerSingleton.setConnections()` (`core/context/mcp/MCPManagerSingleton.ts:68-109`) — servers no longer present are disconnected, new ones connected, unchanged ones left alone — and there's an explicit `setEnabled(serverId, enabled)` toggle (`MCPManagerSingleton.ts:23-37`) that disconnects/reconnects a single server without touching the others or reloading the whole config.

Rules compose from three independent sources simultaneously, not just the active profile's `rules:` array: (1) profile YAML `rules:` entries, (2) a single legacy `.continuerules` dotfile per workspace root (`core/config/getWorkspaceContinueRuleDotFiles.ts:4-32`), and (3) colocated `rules.md` files discovered anywhere in the tree via directory walk, cached in a `CodebaseRulesCache` singleton (`core/config/markdown/loadCodebaseRules.ts:10-26`) that supports incremental per-file `update()` — so editing a nested `rules.md` doesn't force a full repo rescan.

If you want one sentence: **Continue's `.continue/` directory is a live-reloadable, multi-profile YAML+Markdown configuration surface where every `.continue/{assistants,agents,configs}/*.yaml` file is an independently-selectable "assistant" bundling its own models/rules/prompts/MCP-servers, MCP connections are diffed and hot-swapped without a restart, and project rules layer from three sources (profile YAML, one legacy `.continuerules` file, and directory-scoped `rules.md` colocated files) at once.**

## Why this exists — profiles as swappable bundles, not one config

Three load-bearing design choices separate Continue's config system from a single monolithic config file:

1. **Every YAML file under `.continue/{assistants,agents,configs}/` is its own profile.** `getLocalProfiles()` (`core/config/ConfigHandler.ts:161-190`) fans out over all three subdirectory names and wraps each discovered file in its own `LocalProfileLoader` + `ProfileLifecycleManager`. There is no single canonical `config.yaml` requirement at the profile level — a project can ship five differently-scoped assistants side by side, and the global `~/.continue` profile (`globalLocalProfileManager`) is always present as a baseline (`ConfigHandler.ts:56-60,167-169`).
2. **MCP servers are declared as data, reconciled as a diff, not restarted as a process.** `setConnections()` (`MCPManagerSingleton.ts:68-109`) compares each incoming `InternalMcpOptions` against existing `MCPConnection`s by transport identity (`compareTransportOptions`, `MCPManagerSingleton.ts:111-128` — command+args+env for stdio, url for sse/http) and only tears down/rebuilds what actually changed. `setEnabled()` is a narrower per-server on/off switch used by the UI to flip one server without a config reload at all.
3. **Rules are additive across scopes, not a single override chain.** A rule can be global (`~/.continue`), profile-scoped (YAML `rules:` array with `globs`/`regex`/`alwaysApply` fields — `packages/config-yaml/src/schemas/index.ts:39-47`), workspace-wide-but-legacy (`.continuerules`), or directory-scoped (`rules.md` colocated next to the code it governs, discovered via `walkDirs` and cached per-file). All three feed the same `RuleWithSource[]` shape and are unioned into the final system prompt.

## The `.continue/` layout, verified from this checkout's own dogfood config

Continue dogfoods its own config system — the repo root's `.continue/` (not a fixture, the live one used to develop Continue itself) contains:

```
.continue/environment.json      — { "install": "npm i" }  (env bootstrap hint)
.continue/agents/*.md           — 4 files: breaking-change-detector, dependency-security-review,
                                   test-coverage, input-validation, error-message-quality
.continue/checks/*.md           — 6 files: anti-slop, stale-comments, update-continue-docs,
                                   react-best-practices, security-audit, setup-scripts, update-agents-md
.continue/rules/*.md            — 18 files: css-units, colors, overeager, personality,
                                   typescript-enum-usage, no-any-types, ...
.continue/prompts/*.prompt      — update-llm-info.prompt, core-unit-test.prompt
```

Each `rules/*.md` file carries YAML frontmatter that maps directly onto `ruleObjectSchema` (`packages/config-yaml/src/schemas/index.ts:39-47`): `.continue/rules/css-units.md` uses only `globs: "gui/**/*.tsx"` (glob-scoped, applies only when matching files are in context); `.continue/rules/intellij-plugin-test-execution.md` uses the fuller shape — `name`, `description`, `alwaysApply: false`, `globs: extensions/intellij/**/*Test.kt`. This is the same per-directory-scoping mechanism a monorepo needs to keep rules from bleeding across unrelated subprojects, expressed as plain frontmatter rather than a custom DSL.

`.continue/agents/test-coverage.md` (read in full) shows the sub-agent shape: `name` + `description` frontmatter, then a prose spec ("When Tests Are Expected" / "When Tests Are NOT Expected" / "What to Do") — these are prompt-defined checker agents invoked against PRs, not code.

## MCP server declaration and enablement — the schema and the runtime

`packages/config-yaml/src/schemas/mcp/index.ts:4-35` defines the on-disk shape:

```ts
const stdioMcpServerSchema = baseMcpServerSchema.extend({
  command: z.string(),
  type: z.literal("stdio").optional(),
  args: z.array(z.string()).optional(),
  env: z.record(z.string()).optional(),
  cwd: z.string().optional(),
});
const sseOrHttpMcpServerSchema = baseMcpServerSchema.extend({
  url: z.string(),
  type: z.union([z.literal("sse"), z.literal("streamable-http")]).optional(),
  apiKey: z.string().optional(),
  requestOptions: requestOptionsSchema.optional(),
});
```

A minimal real example from the test fixtures (`packages/config-yaml/src/__tests__/local-files/mcpServer.yaml`):

```yaml
name: Local MCP Server
version: 0.0.1
schema: v1
mcpServers:
  - name: Browser search
    command: npx
    args:
      - "@playwright/mcp@latest"
```

At the top level, `configYamlSchema.mcpServers` (`packages/config-yaml/src/schemas/index.ts:128-139`) allows either an inline `MCPServer` object or a `{ uses, with, override }` reference to a hub-hosted shared block — the same reuse pattern applied to `models`, `context`, `rules`, `prompts`, and `docs`. This means an MCP server definition, like a rule, can be authored once (in a Continue Hub package) and referenced with parameter overrides from many project configs — the closest thing here to a "skill package" registry, though it depends on the (now-archived) Continue Hub service rather than pure local files.

At runtime, `MCPManagerSingleton` is a process-wide singleton (`getInstance()`, `MCPManagerSingleton.ts:16-21`) holding a `Map<string, MCPConnection>`. Loading a new config calls `setConnections(servers, forceRefresh, extras)`, which:
- disconnects+removes any connection whose id is no longer present **or** whose transport options changed (`MCPManagerSingleton.ts:76-89`);
- creates new `MCPConnection`s for servers not yet tracked (`:91-103`);
- otherwise just patches metadata (name, favicon) on the existing live connection (`:93-98`);
- only triggers `refreshConnections()` if something actually needs reconnecting (`:105-108`).

`setEnabled(serverId, enabled)` (`:23-37`) is a separate, narrower lever exposed to the UI/CLI for toggling one server off (`disconnect(true)`) or back on (`status = "not-connected"` then `refreshConnection`) without touching sibling servers or reloading YAML at all — this is Continue's answer to "gate a single tool surface" without a full config reload.

## Live per-project profile switching — confirmed, not restart-gated

`ConfigHandler.setSelectedProfileId()` (`core/config/ConfigHandler.ts:207-231`) is the mechanism: given a `profileId`, it looks it up in the already-loaded `this.profiles` array (populated once per `cascadeInit`, not re-scanned per switch), persists the choice keyed by `workspaceId` (`ide.getWorkspaceDirs().join("&")`, `ConfigHandler.ts:76-81`) into `GlobalContext`, and calls `reloadConfig()` — which reloads *only* the newly-current profile (`ConfigHandler.ts:237-283`), explicitly clearing the others (`profile.clearConfig()`, `:251-258`) rather than reloading everything. The comment at `:233-236` is candid about the cost this design accepted: *"IMPORTANT - must always refresh when switching profiles / Because of e.g. MCP singleton and docs service using things from config / Could improve this"* — switching profiles is a full MCP-manager reconciliation, not a zero-cost pointer swap, but it is confirmed **live** (in-session) rather than requiring an extension/IDE restart.

File-system triggers for reload: `isContinueConfigRelatedUri()` (`core/config/loadLocalAssistants.ts:16-30`) recognizes `.continuerc.json`, `.prompt` files, any `SUPPORTED_AGENT_FILES` name, the `.continuerules` dotfile, and — broadly — any `.yaml`/`.yml`/`.json` path containing `.continue`, or any path under `.continue/{blockType}` for every `BLOCK_TYPES` entry plus `agents`/`assistants`/`configs`. This function is what a file-watcher integration (VS Code/JetBrains) consults to decide "does this file change warrant a config reload" — confirming the reload trigger is filesystem-driven, keyed on path shape, not a manual "restart the extension" step.

## What's inside this submodule

| Path | What's there |
|---|---|
| `core/config/ConfigHandler.ts` | The profile-lifecycle orchestrator — `cascadeInit`, `loadProfiles`, `setSelectedProfileId`, `reloadConfig` |
| `core/config/loadLocalAssistants.ts` | Discovers `.continue/{assistants,agents,configs}` YAML/MD files, both workspace and global (`~/.continue`) |
| `core/config/getWorkspaceContinueRuleDotFiles.ts` | Legacy single-file `.continuerules` loader |
| `core/config/markdown/loadCodebaseRules.ts` | `CodebaseRulesCache` — directory-walked, incrementally-updatable colocated `rules.md` discovery |
| `core/config/profile/LocalProfileLoader.ts`, `ProfileLifecycleManager.ts` | Per-file profile wrapper; lazy load/reload/clear per profile |
| `core/config/load.ts` | Legacy JSON `config.json`/`config.ts` resolution path (pre-YAML, still supported) |
| `core/context/mcp/MCPManagerSingleton.ts`, `MCPConnection.ts` | The MCP connection pool — diffing, enable/disable, status, prompt passthrough |
| `core/context/mcp/json/loadJsonMcpConfigs.ts` | Legacy JSON-based MCP config loading (parallel to YAML) |
| `packages/config-yaml/src/schemas/index.ts` | `configYamlSchema`, `assistantUnrolledSchema`, `ruleObjectSchema`, `blockSchema` — the whole on-disk contract |
| `packages/config-yaml/src/schemas/mcp/index.ts` | `stdioMcpServerSchema`, `sseOrHttpMcpServerSchema` — the MCP server on-disk shape |
| `core/llm/rules/constants.ts` | `RULES_MARKDOWN_FILENAME = "rules.md"` — the one magic filename for colocated rules |
| `.continue/` (this repo's own) | Live dogfood config — `agents/`, `checks/`, `rules/`, `prompts/`; read this before any fixture |
| `extensions/cli/src/configLoader.ts`, `config.ts`, `services/ConfigService.ts` | CLI-specific config plumbing on top of the same `core/config` machinery |
| `extensions/vscode/`, `extensions/intellij/` | The two IDE integrations; both consume `core/` |
| `packages/continue-sdk/` | Generated OpenAPI-style SDK (Python + TS) for hub/assistants API |

If you read three files: `core/config/ConfigHandler.ts` (the whole profile-selection and reload lifecycle), `core/context/mcp/MCPManagerSingleton.ts` (the live-diff/enable-disable MCP mechanism), and `packages/config-yaml/src/schemas/index.ts` (the on-disk contract every `.continue/*.yaml` file must satisfy).

## Mental model for using it well

- **Think in profiles, not "the config."** Every file in `.continue/assistants|agents|configs/` is a fully independent, swappable bundle. Don't assume one workspace has one config — it's designed for several, switched live.
- **MCP server identity is transport-shape, not name.** `compareTransportOptions` keys on `command+args+env` (stdio) or `url` (sse/http) — renaming a server's `name` field alone won't force a reconnect; changing its `command`/`args`/`url` will.
- **Rules stack; they don't override.** A glob-scoped `rules/*.md`, a workspace `.continuerules`, and a colocated `rules.md` next to the code can all be active simultaneously. Scope with `globs`/`alwaysApply` deliberately rather than assuming "last one wins."
- **Colocated `rules.md` is the low-ceremony per-directory lever.** No frontmatter required at all — `loadCodebaseRules.ts` will pick up any file literally named `rules.md` anywhere in the walked tree, and updates to a single file are incremental (`CodebaseRulesCache.update()`), not a full rescan.
- **Switching the active profile is not free but is live.** It reconciles the whole MCP manager and clears every other profile's cached config — treat profile switches as a real (if fast) event, not a no-op UI toggle.
- **Hub `uses`/`with`/`override` blocks are the shared-package mechanism** — an MCP server, rule, or prompt authored once can be referenced with parameter overrides from many project configs, but this depends on the (now-archived) Continue Hub remaining reachable.

## When NOT to reach for this

- **The repo is archived and read-only.** The README says so explicitly (`README.md:19`) — there is no path to file a PR upstream; treat this checkout as a frozen reference, not a moving target.
- **You want a CLI-only harness with no IDE surface.** Continue's core is genuinely shared across VS Code/JetBrains/CLI, which means real IDE-extension concerns (webviews, editor decorations, language-server-adjacent plumbing) are baked into the architecture even if you only ever touch `extensions/cli/`.
- **You want a single-file config with no profile/hub indirection.** The `uses`/`with`/`override` block-reference system is powerful but adds a resolution layer (`assistantUnrolledSchema` exists specifically to represent the *unrolled*, fully-resolved form) — if you want "one YAML file, no indirection, no hub," the legacy `config.json`/`config.ts` path (`core/config/load.ts`) is closer, but it's explicitly the older, parallel system, not the recommended one.
- **You need a trace log of tool calls for audit.** Nothing found in this reading plays the role Aider's git-commit-per-edit or an OTel event-stream would — MCP calls flow through `MCPConnection`/`Client` from `@modelcontextprotocol/sdk`, and dev-data logging exists (`core/data/log.ts`, referenced from `.continue/rules/dev-data-guide.md`) but is telemetry-shaped, not a reviewable audit trail.
- **You need per-server secrets fully out-of-repo.** `env` on a stdio MCP server is a plain `Record<string,string>` in the YAML (`packages/config-yaml/src/schemas/mcp/index.ts:16`); there's a top-level `env:` substitution mechanism for `config.json` (`resolveSerializedConfig`, `core/config/load.ts:74-96`) that swaps in `process.env` values by name, but it's a text-replace against the raw JSON string, not a secrets-manager integration.

## How this compares to the rest of the study

| Axis | Continue | opencode | goose |
|---|---|---|---|
| **Shape** | IDE extension (VS Code, JetBrains) + CLI, shared core | CLI-first harness, own MCP client | Rust CLI/desktop, extensions system |
| **Config surface** | Multiple YAML/MD files under `.continue/{assistants,agents,configs}/`, each an independent profile | Session/config storage, single active config per session | Extension/recipe YAML |
| **Per-project skill/rule swap** | **Yes, live** — `setSelectedProfileId` switches the active profile mid-session without restart; rules additionally stack from 3 independent sources (profile YAML, `.continuerules`, colocated `rules.md`) | Live enable/disable of tools per session | Per-extension enable/disable, each extension = one MCP surface |
| **MCP support** | Native — `mcpServers:` array per profile, `stdio`/`sse`/`streamable-http`, reconciled by diff | Native — opencode owns its MCP client | Native — extensions system built on MCP |
| **MCP enable/disable granularity** | **Per-server, live** — `MCPManagerSingleton.setEnabled(id, bool)` disconnects/reconnects one server without touching others | Live enable/disable of tools per session | Per-extension enable/disable |
| **Rule/prompt scoping mechanism** | Frontmatter `globs`/`regex`/`alwaysApply` per rule file + directory-colocated `rules.md`, no ceremony required | N/A noted in this study's opencode profile | Recipe-level, not per-directory |
| **Shared/reusable blocks** | `uses`/`with`/`override` referencing Continue-Hub-hosted blocks (models, rules, MCP servers, prompts, docs) | N/A noted | N/A noted |
| **Trace of actions** | Dev-data JSONL telemetry (`~/.continue/dev_data/`), not an audit-grade trace | Session storage doubles as trace | Extension/tool-call log tied to session state |
| **Maintenance status** | **Archived, read-only** (per README) | — | — |
| **Best fit** | IDE-native coding assistance with per-project, hot-swappable rule/MCP bundles | Terminal-first agent work needing a real, swappable MCP toolset | Desktop/CLI agent work built around composable MCP extensions |

The crucial axis Continue owns in this study is **live, file-driven, multi-profile configuration**: no other entry lets a project ship several fully independent named "assistants" (each its own models/rules/prompts/MCP-servers bundle) side by side in plain files, switchable at runtime by the user, with MCP connections reconciled by transport-identity diff rather than blanket restart. The cost is real — the repo is archived, the config-resolution layer (`uses`/`with`/`override` against a hub) adds indirection, and there's no audit-grade trace of what a tool call actually did.

## One-line summary

> Continue's `.continue/` directory is a multi-profile, file-driven configuration surface — every `.continue/{assistants,agents,configs}/*.yaml` file is an independently loadable, live-switchable "assistant" bundling its own models, rules, prompts, and `mcpServers:` declarations (stdio/sse/streamable-http, schema-validated), with MCP connections reconciled by transport-identity diff and a per-server `setEnabled` toggle that survives without a config reload, and project rules stacking from three simultaneous sources — profile YAML, one legacy `.continuerules` dotfile, and directory-colocated `rules.md` files cached with incremental per-file updates — making it the closest analog in this study to "swap the active skillset per project, live," at the cost of being an archived, read-only repository with an IDE-extension's worth of architecture baked into what looks, from `.continue/`, like a simple config directory.
