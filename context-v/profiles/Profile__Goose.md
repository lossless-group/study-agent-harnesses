---
name: Goose Profile
title: "Goose — Profile"
slug: goose
upstream: https://github.com/aaif-goose/goose
package: "n/a (Rust crates: goose, goose-cli, goose-mcp; distributed as CLI/desktop binaries)"
license: Apache-2.0
maintainer: Agentic AI Foundation (AAIF) at the Linux Foundation — formerly Block, Inc.
study: studies/agent-harnesses
profile_path: studies/agent-harnesses/goose
profile_kind: Rust CLI + desktop (Electron) + API harness, MCP-native
date_created: 2026-07-13
date_modified: 2026-11-23
authors:
  - Michael Staton
site_uuid: 870167a3-fe82-422b-8b64-b826216d14d7
hex_code: 9kvq72
date_authored_initial_draft: 2026-07-13
date_authored_current_draft: 2026-11-23
lede: >-
  In Goose, "extension" just means "a live MCP client" — and two variants skip
  the subprocess entirely to run in-process.
summary: >-
  Source-cited profile of the Goose submodule in the agent-harnesses study. Documents the
  `ExtensionConfig` enum that unifies four shapes (Builtin and Platform in-process, Stdio and
  StreamableHttp via the oi-kit crates the agent loop was extracted into) behind
  one `McpClientTrait`, and the two independent scoping layers on top of it: an `available_tools`
  allowlist checked at config time and an `AlwaysAllow`/`AskBefore`/`NeverAllow` permission gate
  checked at call time. Also covers the SQLite session store (schema v18, WAL, per-call
  `usage_ledger` cost tracking) and its documented migration path from JSONL, YAML recipes that
  declare their own extension list inline, `tracing`-crate spans exported to Langfuse or OTLP, and
  the split between `SKILL.md` durable capabilities and ambient `.goosehints`/`AGENTS.md` hints
  loaded lazily per subdirectory. Use it as the reference for permission-gated MCP ecosystems.
status: Reference
publish: true
type: Profiles
---

# Goose — Profile

A profile of Goose as it lives in this study (`studies/agent-harnesses/goose/`). Cites pinned paths so you can jump to source rather than trust paraphrase. Read alongside [`Profile__Aider.md`](./Profile__Aider.md) and [`Profile__Opencode.md`](./Profile__Opencode.md) — the three form a spectrum from plain-file/git state (Aider) to structured session storage with an owned MCP client (opencode) to Goose, where the **extensions system is a first-class MCP client wrapper** with four config variants and a permission layer sitting in front of every tool call. `LICENSE:1-4` confirms Apache-2.0; `Cargo.toml:8-11` confirms Rust (edition 2021, workspace version 1.60.0 at this pin, 1.42.0 at the July pin), authored under `AAIF <ai-oss-tools@block.xyz>` — the repo is hosted under `aaif-goose/goose`, having moved from `block/goose`. Since the July pin the repo also ships **oi-kit**, a set of `oi-kit-*` crates that hold the reusable agent loop, and several mechanisms below now live there; see the changes section near the end.

## TL;DR

The README states the scope plainly (`README.md:23-25,45-47`):

> [oi-kit] provides the reusable Rust components for building agents. [goose] is a complete agent built with oi-kit: use its desktop app or CLI, or make it the agent backend for your own application over ACP.

> goose is a general-purpose agent for coding, research, writing, automation, and data analysis... Connect tools and data through the Model Context Protocol (MCP)... connect to over 85 providers.

(At the July pin the README led with "A native desktop app... An API to embed it anywhere... 15+ providers... 70+ extensions.")

Mechanically: **extensions are Goose's MCP client wrapper.** At this pin `ExtensionConfig` (`crates/goose/src/agents/extension_config.rs:55-88`) is a tagged enum with three arms: `Builtin`, `Platform`, and `External(TransportConfig)`, where `TransportConfig` is oi-kit's own `ExtensionConfig` with `Stdio` and `StreamableHttp` variants (`crates/oi-kit-agent/src/extension.rs:172-242`). That is four live shapes. The July pin had six (`Frontend` and `InlinePython` were removed in upstream #11645), plus an `Sse` variant kept only for deserialization; `Sse` is now not a variant at all, and an old `type: sse` entry produces a load-time warning, `"SSE is unsupported, migrate to streamable_http"` (`crates/goose/src/config/extensions.rs:323-327`). The single file `extension_manager.rs` became a module (`crates/goose/src/agents/extension_manager/`): oi-kit's generic `ExtensionManager` (`crates/oi-kit-agent/src/mcp/mod.rs:395`) connects `External` stdio/HTTP servers, and Goose's `connect()` (`extension_manager/mod.rs:188-206`) handles `Builtin`/`Platform` by running the builtin MCP server **in-process** over a pair of `tokio::io::duplex` pipes (`extension_manager/builtin.rs:37-43`), or inside a Docker container when one is attached (`builtin.rs:19-35`). Either way the result is a `Box<dyn McpClientTrait>`.

Sessions are **not** flat files. `SessionStorage` (`crates/goose/src/session/session_manager.rs:845-849`, implementation through roughly `:1420`) is a SQLite database (`sessions.db`, WAL journal mode, `CURRENT_SCHEMA_VERSION = 18`, up from 15 at the July pin) with a documented migration path *from* the older one-file-per-session `.jsonl` format (`crates/goose/src/session/legacy.rs:13-27`, `list_sessions()` still globs `*.jsonl`) — on first run, if no `schema_version` table exists yet, it calls `create_schema()` and then `Self::import_legacy(...)` (`session_manager.rs:1120-1133`). This is a harness that started life with Aider-shaped flat session files and deliberately migrated to a real embedded database as scale demanded it.

Recipes are YAML (or JSON) manifests — `Recipe` (`crates/goose/src/recipe/mod.rs:44-88`) — with `version`, `title`, `description`, `instructions`/`prompt`, an `extensions: Vec<ExtensionConfig>` list (so a recipe can declare its own MCP servers inline), `parameters` for Jinja-style `{{ var }}` templating, `sub_recipes`, and a `response.json_schema` for structured output.

Tool-call tracing runs on Rust's own `tracing` crate, not a bespoke log format: `ObservationLayer`/`SpanTracker` (`crates/goose/src/tracing/observation_layer.rs`) implement `tracing_subscriber::Layer`, and `LangfuseBatchManager` (`crates/goose/src/tracing/langfuse_layer.rs`) batches spans to Langfuse (an LLM-observability SaaS) over HTTP; `crates/goose/src/otel/` wires an OTLP exporter as an alternative sink. Every tool call additionally passes through a three-level `PermissionLevel` gate (`AlwaysAllow` / `AskBefore` / `NeverAllow`, `crates/goose/src/config/permission.rs:19-22`) before it's allowed to run at all.

Skills are discovered per the **agentskills.io open spec** (`crates/goose/src/skills/mod.rs:1-3,30-41`): `SKILL.md` files with YAML frontmatter (`name`, `description`, free-form `metadata`) under a canonical global dir `~/.agents/skills` or project dir `<project>/.agents/skills` — the same shape Claude Code and other agents are converging on, discovered by `discover_skills()` (`skills/mod.rs:445-476`). Ambient project instructions load from `.goosehints` and `AGENTS.md` (`crates/goose/src/hints/load_hints.rs:9-22`) — this very checkout's `AGENTS.md` is exactly that file, loaded lazily per-subdirectory as tool calls touch new paths (`SubdirectoryHintTracker`, `load_hints.rs:27-103`).

If you want one sentence: **Goose is a Rust-native harness where "extension" means "a live MCP client" across four config variants (including two — Builtin and Platform — that skip the subprocess and run in-process), tool calls are gated by a three-tier AlwaysAllow/AskBefore/NeverAllow permission layer, sessions live in a WAL-mode SQLite database that Goose itself migrated to from flat per-session JSONL files, recipes are YAML manifests that can declare their own extension list inline, and traces are real `tracing`-crate spans shipped to Langfuse or OTLP rather than a bespoke log — with skills discovered per the agentskills.io `SKILL.md` spec and project hints loaded from `.goosehints`/`AGENTS.md`.**

## Why this exists — extensions as the MCP client, not a bolt-on

Three load-bearing design choices separate Goose's extension system from a thinner MCP integration:

1. **`ExtensionConfig` is a closed, tagged enum that unifies four tool sources under one interface.** `Stdio` (subprocess with `cmd`/`args`/`envs`), `StreamableHttp` (remote MCP over HTTP, optionally tunneled through a Unix domain socket via `socket: Option<String>` — `crates/oi-kit-agent/src/extension.rs:210-215`), `Builtin` (bundled MCP servers shipped in the `goose-mcp` crate), and `Platform` (in-process, direct-access-to-the-agent extensions — the developer tools live here). The July pin also had `Frontend` (tools the Electron UI provided back to the agent) and `InlinePython` (ad hoc Python via `uvx`); both are gone. Every one of these is exposed to the model as ordinary MCP tools; the variant only changes *how* the process/client is stood up.
2. **`available_tools` is a per-extension allowlist, checked at the config layer, not the prompt layer.** `ExtensionConfig::is_tool_available()` (`crates/goose/src/agents/extension_config.rs:143`, delegating to `crates/oi-kit-agent/src/extension.rs:333-344` for external servers): if the list is empty, every tool the server advertises is available; if non-empty, only the named tools pass. This is the mechanism recipes and users use to trim a large MCP server (say, a GitHub server with 40 tools) down to the 3 actually wanted for a given task — done once, structurally, rather than by prompting the model not to call the rest.
3. **Secrets never touch the wire in plaintext config.** `Envs`, now in oi-kit (`crates/oi-kit-agent/src/extension.rs:88-168`), hard-codes a 31-entry `DISALLOWED_KEYS` denylist (`PATH`, `LD_PRELOAD`, `DYLD_INSERT_LIBRARIES`, `PYTHONPATH`, `NODE_OPTIONS`, `TEMP`/`TMP`, etc. — `extension.rs:105-142`) that is enforced at deserialization (`Envs::new`, silently dropping with a `warn!`) *and* at `validate()` (hard error). `ExtensionConfig::resolve()` (`crates/oi-kit-agent/src/extension.rs:346-421`) then does env-var substitution (`$VAR` / `${VAR}`) against the keyring-backed `Config`, so an HTTP extension's `Authorization: Bearer $AUTH_TOKEN` header resolves the secret only at connect time, and `env_keys` (references to keyring entries) get cleared to `vec![]` after resolution so the resolved secret value isn't re-serialized back into the on-disk config. (The test the July pin cited, `env_key_resolved` in the old `extension.rs`, was not re-located at this pin.)

## The extension manager — one chokepoint, four live transports

> **Stale at this pin.** This section describes `crates/goose/src/agents/extension_manager.rs` as of the July pin (`1e03bbb56110`). That file no longer exists: the manager is now `crates/goose/src/agents/extension_manager/` on top of oi-kit's generic `ExtensionManager` (`crates/oi-kit-agent/src/mcp/mod.rs:395`), with per-session `enable`/`disable`/`select` and `McpLease` handles (`extension_manager/mod.rs:269-335`). The behaviors below are kept as a record of the July design; the line numbers refer to the old file.

`ExtensionManager::add_extension()` (July pin: `extension_manager.rs:929-1010+`) is where every extension variant becomes a running `Box<dyn McpClientTrait>`:

- **Idempotent restart-avoidance.** Before doing anything, it clones and resolves the incoming config and compares against the *existing* extension's stored `config` and `resolved_config` (`extension_manager.rs:939-951`) — only restarts the subprocess/client if either the raw config changed (e.g., migrating plaintext envs to `env_keys`) or the resolved secrets rotated. This is why toggling an extension off/on in a long-running session doesn't necessarily tear down and re-spawn a process each time.
- **`Sse` is explicitly dead.** Matching on it returns `ExtensionError::ConfigError("SSE is unsupported, migrate to streamable_http")` (`extension_manager.rs:960-963`) — SSE transport support was removed, but the enum variant is kept solely so old config files still deserialize instead of hard-failing.
- **`StreamableHttp` resolves headers, URI, and an optional Unix-socket path all through the same env-substitution pipeline** (`extension_manager.rs:964-994`) before calling `create_streamable_http_client(...)` with a `GooseCredentialStore` for OAuth-backed servers.
- **`Builtin`/`Platform` skip the network/subprocess entirely.** They resolve `name_to_key(name)` against a static `PLATFORM_EXTENSIONS` registry (`extension_manager.rs:998-1004`, backed by `crate::agents::platform_extensions`) and construct the extension's Rust struct directly in the agent's process, still behind the `McpClientTrait` interface — this is how `developer` (write/edit/shell/tree/read_image) ships with zero subprocess overhead.

## Tool scoping and permissions — allowlist at config, gate at call time

Two independent layers decide what the model can actually do:

1. **Structural scoping** — `ExtensionConfig::is_tool_available()` (`extension_config.rs:143`) filters which of an extension's advertised tools are even presented to the model, per-extension, via the `available_tools` field. Empty means "all tools from this server."
2. **Runtime permission gating** — `crates/goose/src/config/permission.rs:21-25` defines `PermissionLevel::{AlwaysAllow, AskBefore, NeverAllow}`. `PermissionManager` (same file) resolves a tool's effective level by checking user-set permissions first, falling back to a "smart approve" tier (`update_smart_approve_permission`, tested at `permission.rs:270-273`), consulted before every `call_tool` — this is the human-in-the-loop confirmation layer that Aider has no equivalent of (Aider's closest analog is the coarse `--yes`/interactive-diff-approval flow) and that opencode implements as live tool enable/disable rather than a three-state permission gate per tool.

`crates/oi-kit-agent/src/mcp/malware_check.rs` (`deny_if_malicious`, `:34`; at the July pin this was `crates/goose/src/agents/extension_malware_check.rs`; not read in depth here) indicates Goose additionally screens extension configs for malware signatures before install — a supply-chain concern specific to a 70+-extension open ecosystem that neither Aider nor a git-native harness needs to think about.

## Session persistence — SQLite, migrated off flat JSONL

`crates/goose/src/session/session_manager.rs` (5,883 lines at this pin) is the load-bearing file:

- `SessionStorage { pool: Pool<Sqlite>, initialized: OnceCell<()>, session_dir: PathBuf }` (`session_manager.rs:845-849`), backed by `SqliteConnectOptions` with `.journal_mode(SqliteJournalMode::Wal)` and a 30-second busy timeout (`session_manager.rs:1092-1098`) — WAL mode specifically so concurrent readers (CLI, desktop UI, ACP server) don't block each other.
- `CURRENT_SCHEMA_VERSION: i32 = 18` (`session_manager.rs:32`; 15 at the July pin) with a documented `schema_version` table and an explicit comment about a `BEGIN IMMEDIATE` transaction fix for a real race condition seen in production: *"the previous flow... raced when two processes both saw 'doesn't exist' and the second one's CREATE TABLE failed"* (`session_manager.rs:1138-1152`).
- **Legacy import on first run.** If no `schema_version` table exists, `create_schema()` runs *and then* `Self::import_legacy(&self.pool, &self.session_dir)` is called (`session_manager.rs:1128-1132`), reading `crates/goose/src/session/legacy.rs` — which still knows how to glob `*.jsonl` (`legacy.rs:13-27`) and parse each one via `load_session()` (with a 50 MB file-size cap, `legacy.rs:11,39-43`). This confirms Goose's own history: it began with Aider/opencode-style one-file-per-session JSONL and migrated to SQLite as a deliberate schema-versioned upgrade, keeping a one-time import path rather than deleting the old format's readers.
- **Rich per-session metadata beyond messages**: `working_dir`, `session_type` (`User`/`Scheduled`/`SubAgent`/`Hidden`/`Terminal`/`Gateway`/`Acp` — `session_manager.rs:62-71`), `recipe` (the `Recipe` that spawned this session, stored as `recipe_json`), `usage`/`accumulated_usage` token counts, `accumulated_cost`, `model_config`, `parent_session_id` (sub-agent lineage), and a `usage_ledger` table logging every individual model call's token/cost delta (`insert_usage_ledger_row`, `session_manager.rs:1044-1079`) — cost tracking is a first-class table, not a derived aggregate.

## Recipes — YAML manifests that can carry their own extension list

`Recipe` (`crates/goose/src/recipe/mod.rs:44-88`): required `version`/`title`/`description`; at least one of `instructions` or `prompt`; optional `extensions: Vec<ExtensionConfig>`, `settings` (`goose_provider`, `goose_model`, `temperature`, `max_turns` — `mod.rs:100-112`), `activities` (UI-facing suggestion pills), `parameters` (typed, templated inputs), `response.json_schema` (structured-output contract), `sub_recipes` (composition — `SubRecipe { name, path, values, sequential_when_repeated, description }`, `mod.rs:126-135`), and `retry`.

At the July pin, `.github/recipes/code-review.yaml` showed the pattern end to end: a `builtin` `developer` extension plus a `stdio` extension pointed at a project-local script, two `parameters`, and a `prompt:` block that Jinja-interpolates both. That file is gone at this pin; `workflow_recipes/release_risk_check/recipe.yaml` is a current in-tree example of the same format. `recipe/manifest.rs:69-77` resolves sub-recipe paths relative to the recipe file's own directory, and `recipe/local_recipes.rs` + `recipe/build_recipe/` handle discovery/building of recipes from the local filesystem. Because `extensions` lives inside the recipe file itself, a recipe is a fully self-contained "which MCP servers are in scope for this task" declaration — closer to opencode's per-session tool registry than to Aider's file-membership model, but declared statically in a shareable YAML file rather than negotiated live.

## Tracing — real `tracing` spans, shipped to Langfuse or OTLP

`crates/goose/src/tracing/observation_layer.rs` implements `ObservationLayer`/`SpanTracker`/`BatchManager` as a `tracing_subscriber::Layer<S>` (`Layer` impl referencing `tracing::{span, Event, Id, Level, Metadata, Subscriber}`), meaning every `tracing::info!`/span in the Goose agent loop is interceptable at the subscriber layer rather than requiring bespoke instrumentation calls. `crates/goose/src/tracing/langfuse_layer.rs` wraps this in `LangfuseBatchManager`, batching spans over `reqwest` HTTP to a Langfuse endpoint (Langfuse is an open-source LLM-observability product — traces, not just logs, with tool-call-level spans). `crates/goose/src/otel/{mod.rs,otlp.rs}` provides the alternative, standards-based OTLP exporter path. At the July pin, `crates/goose/src/tracing/rate_limiter.rs` rate-limited telemetry events; that file is gone at this pin. This is a materially different answer to the study's "how does this harness trace tool calls" question than Aider's git-commit-as-trace or a flat JSONL transcript: Goose's trace is a structured span tree with dedicated observability-platform sinks, queryable outside the harness entirely.

## Skills and hints — agentskills.io spec + ambient project files

`crates/goose/src/skills/mod.rs:1-3` states the split directly in its module doc: *"filesystem discovery (`SKILL.md` walking + built-ins) and the runtime MCP client... User-facing CRUD lives in `crate::sources`."* Concretely:

- `SkillFrontmatter { name: Option<String>, description: String, metadata: HashMap<String, Value> }` (`skills/mod.rs:30-41`) — the comment at `skills/mod.rs:35-37` cites the spec by name: *"Per the agentskills.io specification... arbitrary metadata lives in this nested mapping so it doesn't collide with reserved frontmatter fields."*
- Two canonical locations: `global_skills_dir()` → `~/.agents/skills` (`skills/mod.rs:43-46`) and `project_skills_dir(project_dir)` → `<project>/.agents/skills` (`skills/mod.rs:49-52`) — the same `.agents/skills` convention other harnesses are converging toward, distinct from Claude Code's `~/.claude/skills`.
- Discovery looks for a `SKILL.md` marker file in each candidate directory (`skills/mod.rs:227,250`) and also merges in compiled-in `builtin` skills (`skills/builtin.rs`) and any skills shipped by installed plugins (`crate::plugins::installed_plugin_skill_dirs`, `skills/mod.rs:16,181`). Since the July pin, skills reach the model through oi-kit's `SkillOperation` and a load-skill tool (`crates/oi-kit-agent/src/skills/operation.rs`) rather than through a Skills platform extension, which was dropped (upstream #13574).
- **Ambient hints are separate from skills.** `crates/goose/src/hints/load_hints.rs:9-22` defines `GOOSE_HINTS_FILENAME = ".goosehints"` and `AGENTS_MD_FILENAME = "AGENTS.md"` as the two default `CONTEXT_FILE_NAMES` (configurable via the `CONTEXT_FILE_NAMES` config param). `SubdirectoryHintTracker` (`load_hints.rs:27-103`) lazily loads hints for a new directory the moment a tool call's `path` argument or a shell command token resolves into that directory — hints aren't all loaded up front; they accrue as the agent's working set expands. This checkout's own `AGENTS.md` (loaded into this very session via `CLAUDE.md`'s `@AGENTS.md` import) is a live instance of exactly this file.

## What's inside this submodule

| Path | What's there |
|---|---|
| `crates/goose/src/agents/extension_config.rs` | Goose's `ExtensionConfig` (`Builtin`, `Platform`, `External`) |
| `crates/goose/src/agents/extension_manager/` | Goose's side of the extension manager — in-process builtin/platform connect, per-session enable/disable |
| `crates/oi-kit-agent/src/extension.rs` | oi-kit's transport `ExtensionConfig` (`Stdio`, `StreamableHttp`), `Envs` secret-denylist, `is_tool_available`, `resolve()` env substitution |
| `crates/oi-kit-agent/src/mcp/` | Generic `ExtensionManager`, MCP client, stdio/HTTP transports, `malware_check.rs` |
| `crates/oi-kit-agent/` | The reusable agent loop as a state machine of `Operation`s (`README.md` in that crate) |
| `crates/goose/src/agents/platform_extensions/developer/` | In-process `write`/`edit`/`shell`/`tree`/`read_image` tools — no subprocess |
| `crates/goose/src/config/permission.rs` | `PermissionLevel::{AlwaysAllow, AskBefore, NeverAllow}`, `PermissionManager` |
| `crates/goose/src/session/session_manager.rs` | `SessionStorage` — SQLite/WAL, schema migrations (`CURRENT_SCHEMA_VERSION=18`), `usage_ledger` |
| `crates/goose/src/session/legacy.rs` | One-time `.jsonl` → SQLite import path for pre-migration sessions |
| `crates/goose/src/recipe/mod.rs` | `Recipe` YAML/JSON schema — `extensions`, `parameters`, `sub_recipes`, `response.json_schema` |
| `crates/goose/src/skills/mod.rs` | `SKILL.md` discovery per agentskills.io spec, `~/.agents/skills` + `<project>/.agents/skills` |
| `crates/goose/src/hints/load_hints.rs` | `.goosehints`/`AGENTS.md` ambient context, lazy per-subdirectory loading |
| `crates/goose/src/tracing/` | `ObservationLayer`, `LangfuseBatchManager` — `tracing`-crate-native span export |
| `crates/goose/src/otel/` | OTLP exporter, alternative to Langfuse |
| `crates/goose-mcp/src/` | Bundled/builtin MCP servers: `computercontroller`, `memory`, `autovisualiser`, `peekaboo`, `tutorial` |
| `crates/goose-cli/` | CLI entry point (`AGENTS.md` cites `crates/goose-cli/src/main.rs`) |
| `crates/oi-kit-*` | The embeddable SDK surface at this pin (`goose-sdk`/`goose-sdk-types` from the July pin are gone) |
| `crates/goose-acp-macros/`, `crates/goose-acp-types/`, `crates/goose/src/acp/` | Agent Client Protocol server implementation |
| `ui/desktop/` | Electron desktop app (`src/main.ts` entry point per `AGENTS.md`) |
| `ui/text/` | Ink-based terminal UI (React-to-fixed-grid — see this repo's own `AGENTS.md` for its overflow gotchas) |
| `workflow_recipes/` | In-tree recipe examples (the July pin's `.github/recipes/code-review.yaml` is gone) |
| `AGENTS.md` | Goose's own agent-facing dev instructions — build/test/lint commands, Ink UI rules, "never" list |

If you read three files: `crates/goose/src/agents/extension_config.rs` with `crates/oi-kit-agent/src/extension.rs` (the `ExtensionConfig` enums — the MCP-wrapper shape), `crates/goose/src/agents/extension_manager/mod.rs` (how each variant becomes a live client), and `crates/goose/src/session/session_manager.rs` (`SessionStorage` — the SQLite schema and the legacy-JSONL migration story).

## Mental model for using it well

- **Think of an "extension" as "one MCP client instance," regardless of transport.** Whether it's a spawned subprocess, a remote HTTP server, or an in-process Rust struct, it's the same `McpClientTrait` and the same `available_tools` scoping and permission gating apply uniformly.
- **Use `available_tools` to trim noisy MCP servers per-recipe or per-config**, not prompt instructions telling the model to ignore tools it can see.
- **Treat recipes as portable, versioned task definitions** — a recipe YAML with its own `extensions:` list is a complete, shareable "this task needs exactly these tools" bundle, checkable into a repo (as `workflow_recipes/` demonstrates).
- **Reach for Langfuse/OTLP, not raw session SQLite queries, for trace analysis.** The `usage_ledger` table gives you cost/token data directly, but tool-call-level trace trees live in the tracing layer's sinks.
- **Drop project conventions in `.goosehints` or `AGENTS.md`, and durable cross-session capabilities in `SKILL.md` under `.agents/skills/`** — the two are deliberately separate mechanisms (ambient always-loaded context vs. explicitly-invoked skill).
- **Respect the `PermissionLevel` gate as the trust boundary**, not just the extension's own allowlist — a tool can be "available" and still require `AskBefore` confirmation at call time.

## When NOT to reach for this

- **You want a single-binary, no-database CLI tool.** Goose's session layer is a real SQLite database with WAL mode and schema migrations — heavier than Aider's flat markdown file, by design, but not "drop a script in and go."
- **You need git-commit-as-trace simplicity.** Goose's trace is a `tracing`-crate span tree shipped to an external sink (Langfuse/OTLP); there's no equivalent of "just read `git log`" — you need a running exporter to see the tool-call history structurally.
- **You're not ready to operate an MCP-server ecosystem.** The extension system's entire value (a large MCP ecosystem, malware screening, four config variants) is wasted if the actual need is "let the model shell out to `rg` and `sed`" — Aider's much thinner model is the better fit there.
- **You want the model to manage its own long-term memory via self-editing tools** (Letta's `core_memory_append`/`replace` pattern). Goose's session store persists conversation + usage + recipe state, but memory-as-agent-editable-blocks isn't part of this crate's model — the `goose-mcp` `memory` extension is a bundled MCP server, not core-crate infrastructure, and wasn't read in depth for this profile.

## Changes since the previous pin (2026-07-13 → 2026-11-23)

- Old pin `1e03bbb56110`, new pin `2da00c6b02c3`, 1,062 commits on `main`. Goose releases cut from release branches in this window run from `v1.43.0` (2026-07-14) to **`v1.60.2` (2026-11-20)**; the workspace version at this pin is 1.60.0 (`Cargo.toml:9`). The nearest tag on `main` is `oi-kit-v0.1.0-alpha.15`, an oi-kit release.
- **The agent loop was extracted into oi-kit** (renamed from "GDK", upstream #13447). The `oi-kit-agent` crate assembles an agent as an ordered list of `Operation` steps over the persisted conversation, not a fixed call-model/run-tools loop (`crates/oi-kit-agent/README.md`). Goose's own steps are in `crates/goose/src/agents/state_machine/` (`ops_toolcalling.rs`, `ops_skills.rs`, `ops_subdirectory_hints.rs`, `tool_approval.rs`, and others). The README now presents the repo as two products: oi-kit for building agents and goose as a complete agent (`README.md:23-25`).
- **Extensions: fewer shapes, same thesis.** `Frontend` and `InlinePython` are gone and `Sse` is no longer a variant. `Builtin` and `Platform` still run in-process, now as an MCP server task wired to the client over in-memory duplex pipes (`crates/goose/src/agents/extension_manager/builtin.rs:37-43`), or inside a Docker container when the session has one. The transport configs, env denylist, and malware check moved into `crates/oi-kit-agent/`.
- **Skills are a state-machine operation.** The Skills platform extension was dropped in favor of oi-kit's `SkillOperation` with a load-skill tool (`crates/oi-kit-agent/src/skills/operation.rs`; upstream #13574, 2026-11-17). Discovery is still `SKILL.md` under `.agents/skills`.
- Session schema went from v15 to v18 (`crates/goose/src/session/session_manager.rs:32`). Removed paths cited at the July pin: `crates/goose/src/agents/extension_manager.rs`, `extension_malware_check.rs`, `crates/goose/src/tracing/rate_limiter.rs`, `crates/goose-sdk/`, `crates/goose-sdk-types/`, `.github/recipes/code-review.yaml`.

## How this compares to the rest of the study

| Axis | Aider | opencode | Goose |
|---|---|---|---|
| **Shape** | CLI, git-native pair-programmer | CLI-first harness, own MCP client, TS/Go | Rust CLI + Electron desktop + ACP backend, built on the oi-kit crates, MCP-native |
| **Tool-calling model** | **None** — LLM emits diff-shaped text, parsed and applied directly | Structured tool-call loop against a provider/tool registry | Structured tool-call loop (a state machine of operations since the July pin); extensions **are** the MCP client wrapper across 4 config variants |
| **MCP support** | **None** — no `mcp` string anywhere in source or docs | Native — opencode owns its MCP client | Native — `ExtensionConfig::{Builtin, Platform, External(Stdio \| StreamableHttp)}` (`extension_config.rs:55-88`), `Sse` rejected with a warning |
| **In-process tools (no subprocess)** | N/A | Not noted | Yes — `Platform`/`Builtin` extensions run in-agent-process (`developer` write/edit/shell/tree) |
| **Session/memory storage** | Flat markdown file (`.aider.chat.history.md`), append-only | Structured, readable session storage (JSON/DB-backed) | **SQLite** (`sessions.db`, WAL, schema v18) — migrated *from* one-file-per-session `.jsonl` (`session/legacy.rs`) |
| **Trace of actions** | Git commit history — each AI edit is a commit | Session storage doubles as trace (tool calls recorded per turn) | `tracing`-crate span layer → Langfuse batch export or OTLP (`tracing/`, `otel/`) |
| **Tool/context scoping** | File-membership only (`/add`, `/read-only`, `.aiderignore`) | Live enable/disable of tools per session | Two layers: `available_tools` allowlist per extension (config-time) + `PermissionLevel` gate (call-time: AlwaysAllow/AskBefore/NeverAllow) |
| **Recipe/task definition format** | N/A | Not the primary unit | YAML/JSON `Recipe` manifest with inline `extensions:`, `parameters`, `sub_recipes`, `response.json_schema` |
| **Skills/durable capability discovery** | N/A | Not noted | `SKILL.md` per agentskills.io spec, `~/.agents/skills` + `<project>/.agents/skills` |
| **Ambient project context** | N/A (repo map instead) | Not noted | `.goosehints` / `AGENTS.md`, lazily loaded per-subdirectory as tools touch new paths |
| **Best fit** | Git-repo-centric solo/pair coding where commit history as audit trail is valuable | Terminal-first agent work needing a real, swappable MCP toolset | Cross-surface (CLI/desktop/API) agent work needing a real MCP extension ecosystem, human-in-the-loop permissioning, and observability-platform-grade tracing |

The crucial axis Goose owns in this study is **extensions-as-unified-MCP-client across transport shapes, gated by an explicit permission layer** — no other entry in this study collapses subprocess MCP servers, remote HTTP MCP servers, and in-process native tools into one config enum with one allowlist mechanism and one runtime AlwaysAllow/AskBefore/NeverAllow gate sitting in front of all three.

## One-line summary

> Goose is a Rust-native, MCP-first harness where "extension" means "a live MCP client" spanning four config variants — including two, `Builtin` and `Platform`, that skip the subprocess entirely and run in-process — scoped by a per-extension `available_tools` allowlist and gated at call time by a three-tier `AlwaysAllow`/`AskBefore`/`NeverAllow` permission layer; sessions persist in a WAL-mode SQLite database (schema v18) that Goose itself migrated to from flat per-session `.jsonl` files, recipes are YAML manifests that can declare their own extension list and sub-recipes inline, tool-call tracing runs on real `tracing`-crate spans shipped to Langfuse or OTLP rather than a bespoke log, and durable capabilities load as `SKILL.md` files per the agentskills.io open spec while ambient project instructions come from `.goosehints`/`AGENTS.md`, loaded lazily as the agent's working set expands.
