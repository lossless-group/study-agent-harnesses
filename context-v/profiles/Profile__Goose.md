---
name: Goose Profile
slug: goose
upstream: https://github.com/aaif-goose/goose
package: n/a (Rust crates: goose, goose-cli, goose-mcp; distributed as CLI/desktop binaries)
license: Apache-2.0
maintainer: Agentic AI Foundation (AAIF) at the Linux Foundation — formerly Block, Inc.
study: studies/agent-harnesses
profile_path: studies/agent-harnesses/goose
profile_kind: Rust CLI + desktop (Electron) + API harness, MCP-native
date_created: 2026-07-13
---

# Goose — Profile

A profile of Goose as it lives in this study (`studies/agent-harnesses/goose/`). Cites pinned paths so you can jump to source rather than trust paraphrase. Read alongside [`Profile__Aider.md`](./Profile__Aider.md) and [`Profile__Opencode.md`](./Profile__Opencode.md) — the three form a spectrum from plain-file/git state (Aider) to structured session storage with an owned MCP client (opencode) to Goose, where the **extensions system is a first-class MCP client wrapper** with four transport shapes and a permission layer sitting in front of every tool call. `LICENSE:1-4` confirms Apache-2.0; `Cargo.toml:9-15` confirms Rust (edition 2021, workspace version 1.42.0), authored under `AAIF <ai-oss-tools@block.xyz>` — the repo is now hosted under `aaif-goose/goose` per the README, having moved from `block/goose`.

## TL;DR

The README states the scope plainly (`README.md:26-29`):

> A native desktop app for macOS, Linux, and Windows. A full CLI for terminal workflows. An API to embed it anywhere. Built in Rust for performance and portability. goose works with 15+ providers... Connect to 70+ extensions via the Model Context Protocol open standard.

Mechanically: **extensions are Goose's MCP client wrapper.** `ExtensionConfig` (`crates/goose/src/agents/extension.rs:161-304`) is a tagged enum with six variants — `Stdio`, `StreamableHttp`, `Builtin`, `Platform`, `Frontend`, `InlinePython` (a seventh, `Sse`, is kept only for config-file backward-compatibility and explicitly rejected at connect time: `"SSE is unsupported, migrate to streamable_http"`, `extension_manager.rs:960-963`). `ExtensionManager::add_extension()` (`extension_manager.rs:929` onward) is the single chokepoint that turns any of these into a live `Box<dyn McpClientTrait>` — spawning a subprocess for `Stdio`, opening a `StreamableHttpClientTransport` for HTTP MCP servers, or, for `Platform`/`Builtin`, instantiating an **in-process** Rust struct that still speaks the MCP `McpClientTrait` interface without a subprocess at all (`extension_manager.rs:996-1004`, dispatching into `PLATFORM_EXTENSIONS`).

Sessions are **not** flat files. `SessionStorage` (`crates/goose/src/session/session_manager.rs:652-847`) is a SQLite database (`sessions.db`, WAL journal mode, `CURRENT_SCHEMA_VERSION = 15`) with a documented migration path *from* the older one-file-per-session `.jsonl` format (`crates/goose/src/session/legacy.rs:13-27`, `list_sessions()` still globs `*.jsonl`) — `SessionStorage::pool()` calls `Self::import_legacy(&self.pool, &self.session_dir)` on first run if no `schema_version` table exists yet (`session_manager.rs:874-884`). This is a harness that started life with Aider-shaped flat session files and deliberately migrated to a real embedded database as scale demanded it.

Recipes are YAML (or JSON) manifests — `Recipe` (`crates/goose/src/recipe/mod.rs:43-87`) — with `version`, `title`, `description`, `instructions`/`prompt`, an `extensions: Vec<ExtensionConfig>` list (so a recipe can declare its own MCP servers inline, e.g. `.github/recipes/code-review.yaml:15-23`), `parameters` for Jinja-style `{{ var }}` templating, `sub_recipes`, and a `response.json_schema` for structured output.

Tool-call tracing runs on Rust's own `tracing` crate, not a bespoke log format: `ObservationLayer`/`SpanTracker` (`crates/goose/src/tracing/observation_layer.rs`) implement `tracing_subscriber::Layer`, and `LangfuseBatchManager` (`crates/goose/src/tracing/langfuse_layer.rs`) batches spans to Langfuse (an LLM-observability SaaS) over HTTP; `crates/goose/src/otel/` wires an OTLP exporter as an alternative sink. Every tool call additionally passes through a three-level `PermissionLevel` gate (`AlwaysAllow` / `AskBefore` / `NeverAllow`, `crates/goose/src/config/permission.rs:19-22`) before it's allowed to run at all.

Skills are discovered per the **agentskills.io open spec** (`crates/goose/src/skills/mod.rs:1-3,27-33`): `SKILL.md` files with YAML frontmatter (`name`, `description`, free-form `metadata`) under a canonical global dir `~/.agents/skills` or project dir `<project>/.agents/skills` — the same shape Claude Code and other agents are converging on, walked at `skills/mod.rs:404-445`. Ambient project instructions load from `.goosehints` and `AGENTS.md` (`crates/goose/src/hints/load_hints.rs:9-22`) — this very checkout's `AGENTS.md` is exactly that file, loaded lazily per-subdirectory as tool calls touch new paths (`SubdirectoryHintTracker`, `load_hints.rs:26-56`).

If you want one sentence: **Goose is a Rust-native harness where "extension" means "a live MCP client" across six transport variants (including two — Builtin and Platform — that skip the subprocess and run in-process), tool calls are gated by a three-tier AlwaysAllow/AskBefore/NeverAllow permission layer, sessions live in a WAL-mode SQLite database that Goose itself migrated to from flat per-session JSONL files, recipes are YAML manifests that can declare their own extension list inline, and traces are real `tracing`-crate spans shipped to Langfuse or OTLP rather than a bespoke log — with skills discovered per the agentskills.io `SKILL.md` spec and project hints loaded from `.goosehints`/`AGENTS.md`.**

## Why this exists — extensions as the MCP client, not a bolt-on

Three load-bearing design choices separate Goose's extension system from a thinner MCP integration:

1. **`ExtensionConfig` is a closed, tagged enum that unifies six different tool sources under one interface.** `Stdio` (subprocess with `cmd`/`args`/`envs`), `StreamableHttp` (remote MCP over HTTP, optionally tunneled through a Unix domain socket via `socket: Option<String>` — `extension.rs:253-256`), `Builtin` (bundled MCP servers shipped in the `goose-mcp` crate), `Platform` (in-process, direct-access-to-the-agent extensions — the developer tools live here), `Frontend` (tools the Electron UI itself provides back to the agent, `extension.rs:265-283`), and `InlinePython` (ad hoc Python executed via `uvx`, `extension.rs:284-303`). Every one of these is exposed to the model as ordinary MCP tools; the variant only changes *how* the process/client is stood up.
2. **`available_tools` is a per-extension allowlist, checked at the config layer, not the prompt layer.** `ExtensionConfig::is_tool_available()` (`extension.rs:427-453`): if the list is empty, every tool the server advertises is available; if non-empty, only the named tools pass. This is the mechanism recipes and users use to trim a large MCP server (say, a GitHub server with 40 tools) down to the 3 actually wanted for a given task — done once, structurally, rather than by prompting the model not to call the rest.
3. **Secrets never touch the wire in plaintext config.** `Envs` (`extension.rs:61-158`) hard-codes a 31-entry `DISALLOWED_KEYS` denylist (`PATH`, `LD_PRELOAD`, `DYLD_INSERT_LIBRARIES`, `PYTHONPATH`, `NODE_OPTIONS`, `TEMP`/`TMP`, etc. — `extension.rs:81-118`) that is enforced at deserialization (`Envs::new`, silently dropping with a `warn!`) *and* at `validate()` (hard error). `ExtensionConfig::resolve()` (`extension.rs:455-521`) then does env-var substitution (`$VAR` / `${VAR}` via `RE_ENV_BRACES`/`RE_ENV_SIMPLE`, `extension_manager.rs:104-107`) against the keyring-backed `Config`, so an HTTP extension's `Authorization: Bearer $AUTH_TOKEN` header resolves the secret only at connect time, and `env_keys` (references to keyring entries) get cleared to `vec![]` after resolution so the resolved secret value isn't re-serialized back into the on-disk config (test coverage at `extension.rs:952-964`, `env_key_resolved`).

## The extension manager — one chokepoint, four live transports

`ExtensionManager::add_extension()` (`crates/goose/src/agents/extension_manager.rs:929-1010+`) is where every extension variant becomes a running `Box<dyn McpClientTrait>`:

- **Idempotent restart-avoidance.** Before doing anything, it clones and resolves the incoming config and compares against the *existing* extension's stored `config` and `resolved_config` (`extension_manager.rs:939-951`) — only restarts the subprocess/client if either the raw config changed (e.g., migrating plaintext envs to `env_keys`) or the resolved secrets rotated. This is why toggling an extension off/on in a long-running session doesn't necessarily tear down and re-spawn a process each time.
- **`Sse` is explicitly dead.** Matching on it returns `ExtensionError::ConfigError("SSE is unsupported, migrate to streamable_http")` (`extension_manager.rs:960-963`) — SSE transport support was removed, but the enum variant is kept solely so old config files still deserialize instead of hard-failing.
- **`StreamableHttp` resolves headers, URI, and an optional Unix-socket path all through the same env-substitution pipeline** (`extension_manager.rs:964-994`) before calling `create_streamable_http_client(...)` with a `GooseCredentialStore` for OAuth-backed servers.
- **`Builtin`/`Platform` skip the network/subprocess entirely.** They resolve `name_to_key(name)` against a static `PLATFORM_EXTENSIONS` registry (`extension_manager.rs:998-1004`, backed by `crate::agents::platform_extensions`) and construct the extension's Rust struct directly in the agent's process, still behind the `McpClientTrait` interface — this is how `developer` (write/edit/shell/tree/read_image) ships with zero subprocess overhead.

## Tool scoping and permissions — allowlist at config, gate at call time

Two independent layers decide what the model can actually do:

1. **Structural scoping** — `ExtensionConfig::is_tool_available()` (`extension.rs:427-453`) filters which of an extension's advertised tools are even presented to the model, per-extension, via the `available_tools` field. Empty means "all tools from this server."
2. **Runtime permission gating** — `crates/goose/src/config/permission.rs:19-22` defines `PermissionLevel::{AlwaysAllow, AskBefore, NeverAllow}`. `PermissionManager` (same file) resolves a tool's effective level by checking user-set permissions first, falling back to a "smart approve" tier (`update_smart_approve_permission`, tested at `permission.rs:270-273`), consulted before every `call_tool` — this is the human-in-the-loop confirmation layer that Aider has no equivalent of (Aider's closest analog is the coarse `--yes`/interactive-diff-approval flow) and that opencode implements as live tool enable/disable rather than a three-state permission gate per tool.

`crates/goose/src/agents/extension_malware_check.rs` (present in the tree; not read in depth here) indicates Goose additionally screens extension configs for malware signatures before install — a supply-chain concern specific to a 70+-extension open ecosystem that neither Aider nor a git-native harness needs to think about.

## Session persistence — SQLite, migrated off flat JSONL

`crates/goose/src/session/session_manager.rs:652-847` is the load-bearing file:

- `SessionStorage { pool: Pool<Sqlite>, initialized: OnceCell<()>, session_dir: PathBuf }` (`session_manager.rs:652-656`), backed by `SqliteConnectOptions` with `.journal_mode(SqliteJournalMode::Wal)` and a 30-second busy timeout (`session_manager.rs:848-859`) — WAL mode specifically so concurrent readers (CLI, desktop UI, ACP server) don't block each other.
- `CURRENT_SCHEMA_VERSION: i32 = 15` (`session_manager.rs:27`) with a documented `schema_version` table and an explicit comment about a `BEGIN IMMEDIATE` transaction fix for a real race condition seen in production: *"the previous flow... raced when two processes both saw 'doesn't exist' and the second one's CREATE TABLE failed"* (`session_manager.rs:895-908`).
- **Legacy import on first run.** If no `schema_version` table exists, `create_schema()` runs *and then* `Self::import_legacy(&self.pool, &self.session_dir)` is called (`session_manager.rs:876-884`), reading `crates/goose/src/session/legacy.rs` — which still knows how to glob `*.jsonl` (`legacy.rs:13-27`) and parse each one via `load_session()` (with a 50 MB file-size cap, `legacy.rs:11,39-43`). This confirms Goose's own history: it began with Aider/opencode-style one-file-per-session JSONL and migrated to SQLite as a deliberate schema-versioned upgrade, keeping a one-time import path rather than deleting the old format's readers.
- **Rich per-session metadata beyond messages**: `working_dir`, `session_type` (`User`/`Scheduled`/`SubAgent`/`Hidden`/`Terminal`/`Gateway`/`Acp` — `session_manager.rs:44-53`), `recipe` (the `Recipe` that spawned this session, stored as `recipe_json`), `usage`/`accumulated_usage` token counts, `accumulated_cost`, `model_config`, `parent_session_id` (sub-agent lineage), and a `usage_ledger` table logging every individual model call's token/cost delta (`insert_usage_ledger_row`, `session_manager.rs:817-846`) — cost tracking is a first-class table, not a derived aggregate.

## Recipes — YAML manifests that can carry their own extension list

`Recipe` (`crates/goose/src/recipe/mod.rs:43-87`): required `version`/`title`/`description`; at least one of `instructions` or `prompt`; optional `extensions: Vec<ExtensionConfig>`, `settings` (`goose_provider`, `goose_model`, `temperature`, `max_turns` — `mod.rs:99-111`), `activities` (UI-facing suggestion pills), `parameters` (typed, templated inputs), `response.json_schema` (structured-output contract), `sub_recipes` (composition — `SubRecipe { name, path, values, sequential_when_repeated, description }`, `mod.rs:120-129`), and `retry`.

The real recipe at `.github/recipes/code-review.yaml:1-44` shows the pattern end to end: a `builtin` `developer` extension plus a `stdio` extension pointed at a project-local script (`cmd: uv`, `args: [run, '{{ recipe_dir }}/../scripts/pr-review-mcp.py']`), two `parameters` (`pr_directory`, `instructions`), and a `prompt:` block that Jinja-interpolates both. `recipe/manifest.rs:69-77` resolves sub-recipe paths relative to the recipe file's own directory, and `recipe/local_recipes.rs` + `recipe/build_recipe/` handle discovery/building of recipes from the local filesystem. Because `extensions` lives inside the recipe file itself, a recipe is a fully self-contained "which MCP servers are in scope for this task" declaration — closer to opencode's per-session tool registry than to Aider's file-membership model, but declared statically in a shareable YAML file rather than negotiated live.

## Tracing — real `tracing` spans, shipped to Langfuse or OTLP

`crates/goose/src/tracing/observation_layer.rs` implements `ObservationLayer`/`SpanTracker`/`BatchManager` as a `tracing_subscriber::Layer<S>` (`Layer` impl referencing `tracing::{span, Event, Id, Level, Metadata, Subscriber}`), meaning every `tracing::info!`/span in the Goose agent loop is interceptable at the subscriber layer rather than requiring bespoke instrumentation calls. `crates/goose/src/tracing/langfuse_layer.rs` wraps this in `LangfuseBatchManager`, batching spans over `reqwest` HTTP to a Langfuse endpoint (Langfuse is an open-source LLM-observability product — traces, not just logs, with tool-call-level spans). `crates/goose/src/otel/{mod.rs,otlp.rs}` provides the alternative, standards-based OTLP exporter path. `crates/goose/src/tracing/rate_limiter.rs` batches/rate-limits the telemetry events (`RateLimitedTelemetrySender`, `TelemetryEvent`) so tracing overhead doesn't become the bottleneck in a busy agent loop. This is a materially different answer to the study's "how does this harness trace tool calls" question than Aider's git-commit-as-trace or a flat JSONL transcript: Goose's trace is a structured span tree with dedicated observability-platform sinks, queryable outside the harness entirely.

## Skills and hints — agentskills.io spec + ambient project files

`crates/goose/src/skills/mod.rs:1-3` states the split directly in its module doc: *"filesystem discovery (`SKILL.md` walking + built-ins) and the runtime MCP client... User-facing CRUD lives in `crate::sources`."* Concretely:

- `SkillFrontmatter { name: Option<String>, description: String, metadata: HashMap<String, Value> }` (`skills/mod.rs:23-34`) — the comment at `skills/mod.rs:29-32` cites the spec by name: *"Per the agentskills.io specification... arbitrary metadata lives in this nested mapping so it doesn't collide with reserved frontmatter fields."*
- Two canonical locations: `global_skills_dir()` → `~/.agents/skills` (`skills/mod.rs:38-40`) and `project_skills_dir(project_dir)` → `<project>/.agents/skills` (`skills/mod.rs:44-46`) — the same `.agents/skills` convention other harnesses are converging toward, distinct from Claude Code's `~/.claude/skills`.
- Discovery walks directories looking for a `SKILL.md` marker file (`skills/mod.rs:236,404-445`), skipping directories per a `should_skip_dir` predicate, and also merges in compiled-in `builtin` skills (`skills/builtin.rs`) and any skills shipped by installed plugins (`crate::plugins::installed_plugin_skill_dirs`, referenced at `skills/mod.rs:15`).
- **Ambient hints are separate from skills.** `crates/goose/src/hints/load_hints.rs:9-22` defines `GOOSE_HINTS_FILENAME = ".goosehints"` and `AGENTS_MD_FILENAME = "AGENTS.md"` as the two default `CONTEXT_FILE_NAMES` (configurable via the `CONTEXT_FILE_NAMES` config param). `SubdirectoryHintTracker` (`load_hints.rs:26-56`) lazily loads hints for a new directory the moment a tool call's `path` argument or a shell command token resolves into that directory — hints aren't all loaded up front; they accrue as the agent's working set expands. This checkout's own `AGENTS.md` (loaded into this very session via `CLAUDE.md`'s `@AGENTS.md` import) is a live instance of exactly this file.

## What's inside this submodule

| Path | What's there |
|---|---|
| `crates/goose/src/agents/extension.rs` | `ExtensionConfig` (6+1 variants), `Envs` secret-denylist, `is_tool_available`, `resolve()` env substitution |
| `crates/goose/src/agents/extension_manager.rs` | `ExtensionManager::add_extension` — the chokepoint that spawns/reuses stdio/HTTP/builtin/platform clients |
| `crates/goose/src/agents/extension_malware_check.rs` | Pre-install malware screening for extension configs |
| `crates/goose/src/agents/platform_extensions/developer/` | In-process `write`/`edit`/`shell`/`tree`/`read_image` tools — no subprocess |
| `crates/goose/src/config/permission.rs` | `PermissionLevel::{AlwaysAllow, AskBefore, NeverAllow}`, `PermissionManager` |
| `crates/goose/src/session/session_manager.rs` | `SessionStorage` — SQLite/WAL, schema migrations (`CURRENT_SCHEMA_VERSION=15`), `usage_ledger` |
| `crates/goose/src/session/legacy.rs` | One-time `.jsonl` → SQLite import path for pre-migration sessions |
| `crates/goose/src/recipe/mod.rs` | `Recipe` YAML/JSON schema — `extensions`, `parameters`, `sub_recipes`, `response.json_schema` |
| `crates/goose/src/skills/mod.rs` | `SKILL.md` discovery per agentskills.io spec, `~/.agents/skills` + `<project>/.agents/skills` |
| `crates/goose/src/hints/load_hints.rs` | `.goosehints`/`AGENTS.md` ambient context, lazy per-subdirectory loading |
| `crates/goose/src/tracing/` | `ObservationLayer`, `LangfuseBatchManager`, `rate_limiter.rs` — `tracing`-crate-native span export |
| `crates/goose/src/otel/` | OTLP exporter, alternative to Langfuse |
| `crates/goose-mcp/src/` | Bundled/builtin MCP servers: `computercontroller`, `memory`, `autovisualiser`, `peekaboo`, `tutorial` |
| `crates/goose-cli/` | CLI entry point (`AGENTS.md` cites `crates/goose-cli/src/main.rs`) |
| `crates/goose-sdk/`, `crates/goose-sdk-types/` | Embeddable API/SDK surface |
| `crates/goose-acp-macros/`, `crates/goose/src/acp/` | Agent Client Protocol server implementation |
| `ui/desktop/` | Electron desktop app (`src/main.ts` entry point per `AGENTS.md`) |
| `ui/text/` | Ink-based terminal UI (React-to-fixed-grid — see this repo's own `AGENTS.md` for its overflow gotchas) |
| `.github/recipes/code-review.yaml` | Real, working recipe example — inline `stdio` extension + `developer` builtin |
| `AGENTS.md` | Goose's own agent-facing dev instructions — build/test/lint commands, Ink UI rules, "never" list |

If you read three files: `crates/goose/src/agents/extension.rs` (the `ExtensionConfig` enum — the whole MCP-wrapper shape lives here), `crates/goose/src/agents/extension_manager.rs` (`add_extension` — how each variant becomes a live client), and `crates/goose/src/session/session_manager.rs` (`SessionStorage` — the SQLite schema and the legacy-JSONL migration story).

## Mental model for using it well

- **Think of an "extension" as "one MCP client instance," regardless of transport.** Whether it's a spawned subprocess, a remote HTTP server, or an in-process Rust struct, it's the same `McpClientTrait` and the same `available_tools` scoping and permission gating apply uniformly.
- **Use `available_tools` to trim noisy MCP servers per-recipe or per-config**, not prompt instructions telling the model to ignore tools it can see.
- **Treat recipes as portable, versioned task definitions** — a recipe YAML with its own `extensions:` list is a complete, shareable "this task needs exactly these tools" bundle, checkable into a repo (as `.github/recipes/code-review.yaml` demonstrates).
- **Reach for Langfuse/OTLP, not raw session SQLite queries, for trace analysis.** The `usage_ledger` table gives you cost/token data directly, but tool-call-level trace trees live in the tracing layer's sinks.
- **Drop project conventions in `.goosehints` or `AGENTS.md`, and durable cross-session capabilities in `SKILL.md` under `.agents/skills/`** — the two are deliberately separate mechanisms (ambient always-loaded context vs. explicitly-invoked skill).
- **Respect the `PermissionLevel` gate as the trust boundary**, not just the extension's own allowlist — a tool can be "available" and still require `AskBefore` confirmation at call time.

## When NOT to reach for this

- **You want a single-binary, no-database CLI tool.** Goose's session layer is a real SQLite database with WAL mode and schema migrations — heavier than Aider's flat markdown file, by design, but not "drop a script in and go."
- **You need git-commit-as-trace simplicity.** Goose's trace is a `tracing`-crate span tree shipped to an external sink (Langfuse/OTLP); there's no equivalent of "just read `git log`" — you need a running exporter to see the tool-call history structurally.
- **You're not ready to operate an MCP-server ecosystem.** The extension system's entire value (70+ extensions, `extension_malware_check.rs`, six transport variants) is wasted if the actual need is "let the model shell out to `rg` and `sed`" — Aider's much thinner model is the better fit there.
- **You want the model to manage its own long-term memory via self-editing tools** (Letta's `core_memory_append`/`replace` pattern). Goose's session store persists conversation + usage + recipe state, but memory-as-agent-editable-blocks isn't part of this crate's model — the `goose-mcp` `memory` extension is a bundled MCP server, not core-crate infrastructure, and wasn't read in depth for this profile.

## How this compares to the rest of the study

| Axis | Aider | opencode | Goose |
|---|---|---|---|
| **Shape** | CLI, git-native pair-programmer | CLI-first harness, own MCP client, TS/Go | Rust CLI + Electron desktop + API/SDK, MCP-native |
| **Tool-calling model** | **None** — LLM emits diff-shaped text, parsed and applied directly | Structured tool-call loop against a provider/tool registry | Structured tool-call loop; extensions **are** the MCP client wrapper across 6 config variants |
| **MCP support** | **None** — no `mcp` string anywhere in source or docs | Native — opencode owns its MCP client | Native — `ExtensionConfig::{Stdio, StreamableHttp, Builtin, Platform, Frontend, InlinePython}` (`extension.rs:161-304`), `Sse` explicitly deprecated |
| **In-process tools (no subprocess)** | N/A | Not noted | Yes — `Platform`/`Builtin` extensions run in-agent-process (`developer` write/edit/shell/tree) |
| **Session/memory storage** | Flat markdown file (`.aider.chat.history.md`), append-only | Structured, readable session storage (JSON/DB-backed) | **SQLite** (`sessions.db`, WAL, schema v15) — migrated *from* one-file-per-session `.jsonl` (`session/legacy.rs`) |
| **Trace of actions** | Git commit history — each AI edit is a commit | Session storage doubles as trace (tool calls recorded per turn) | `tracing`-crate span layer → Langfuse batch export or OTLP (`tracing/`, `otel/`) |
| **Tool/context scoping** | File-membership only (`/add`, `/read-only`, `.aiderignore`) | Live enable/disable of tools per session | Two layers: `available_tools` allowlist per extension (config-time) + `PermissionLevel` gate (call-time: AlwaysAllow/AskBefore/NeverAllow) |
| **Recipe/task definition format** | N/A | Not the primary unit | YAML/JSON `Recipe` manifest with inline `extensions:`, `parameters`, `sub_recipes`, `response.json_schema` |
| **Skills/durable capability discovery** | N/A | Not noted | `SKILL.md` per agentskills.io spec, `~/.agents/skills` + `<project>/.agents/skills` |
| **Ambient project context** | N/A (repo map instead) | Not noted | `.goosehints` / `AGENTS.md`, lazily loaded per-subdirectory as tools touch new paths |
| **Best fit** | Git-repo-centric solo/pair coding where commit history as audit trail is valuable | Terminal-first agent work needing a real, swappable MCP toolset | Cross-surface (CLI/desktop/API) agent work needing a real MCP extension ecosystem, human-in-the-loop permissioning, and observability-platform-grade tracing |

The crucial axis Goose owns in this study is **extensions-as-unified-MCP-client across transport shapes, gated by an explicit permission layer** — no other entry in this study collapses subprocess MCP servers, remote HTTP MCP servers, and in-process native tools into one config enum with one allowlist mechanism and one runtime AlwaysAllow/AskBefore/NeverAllow gate sitting in front of all three.

## One-line summary

> Goose is a Rust-native, MCP-first harness where "extension" means "a live MCP client" spanning six transport variants — including two, `Builtin` and `Platform`, that skip the subprocess entirely and run in-process — scoped by a per-extension `available_tools` allowlist and gated at call time by a three-tier `AlwaysAllow`/`AskBefore`/`NeverAllow` permission layer; sessions persist in a WAL-mode SQLite database (schema v15) that Goose itself migrated to from flat per-session `.jsonl` files, recipes are YAML manifests that can declare their own extension list and sub-recipes inline, tool-call tracing runs on real `tracing`-crate spans shipped to Langfuse or OTLP rather than a bespoke log, and durable capabilities load as `SKILL.md` files per the agentskills.io open spec while ambient project instructions come from `.goosehints`/`AGENTS.md`, loaded lazily as the agent's working set expands.
