---
name: OpenCode Profile
slug: opencode
upstream: https://github.com/anomalyco/opencode
package: opencode-ai (npm); binary distributed via `curl -fsSL https://opencode.ai/install | bash`, Homebrew (`anomalyco/tap/opencode`), and Nix
license: MIT
maintainer: anomalyco (org name as of this checkout; the repo, npm badges, Homebrew tap, and every `package.json` "repository" field consistently read `anomalyco/opencode` — see note below on the org-name question)
study: studies/agent-harnesses
profile_path: studies/agent-harnesses/opencode
profile_kind: coding-agent-harness (CLI + TUI + server + desktop + SDKs)
date_created: 2026-07-13
---

# OpenCode — Profile

A profile of OpenCode as it lives in this study (`studies/agent-harnesses/opencode/`). Cites pinned paths so you can jump to source rather than trust paraphrase. Read alongside the sibling profiles for `goose` (Rust, extensions-as-MCP-wrapper) and `aider` (git-native repo-map/diff-based, no MCP) — this is the "TypeScript, Effect-native, database-backed" corner of the study.

## TL;DR

OpenCode bills itself simply, in the README (`README.md:44`): **"The open source AI coding agent."** But the checkout reveals something considerably more architected than a CLI wrapper around an LLM: a 30+-package Bun/TypeScript monorepo built on the [Effect](https://effect.website) library, with a real **SQLite database** (Drizzle ORM, 38 migrations, WAL mode) for session/message persistence, a **JSON-file storage layer with its own migration chain** for legacy artifacts, a first-class **MCP client** (local stdio + remote HTTP/SSE, OAuth, tool-list-changed notifications), a **skill discovery system** that reads `.claude/skills`, `.agents/skills`, and `.opencode/skill(s)` directories plus remote skill registries, and an internal **"System Context" architecture** (documented in `CONTEXT.md`) that formalizes exactly how conversational memory is assembled and cached across turns.

The self-description of that system-context machinery is the most load-bearing pull-quote in the repo (`CONTEXT.md:16-20`):

> **Context Epoch**: The span during which one initially rendered **System Context** remains the immutable provider-cache baseline, ending at completed compaction, Session movement, or an incompatible context transition that requires a fresh baseline.

That's not marketing copy — it's an internal vocabulary document the team uses to keep its own contributors (and its own AGENTS.md-reading coding agents) aligned on how session memory should behave.

Root `package.json:2-8` confirms the shape directly: `"name": "opencode"`, `"packageManager": "bun@1.3.14"`, workspaces across `packages/*`. There is **no `go.mod` anywhere in the checkout** (`find . -iname go.mod` returns nothing) — this is a TypeScript/Bun project end to end, not the Go+TypeScript hybrid a naive guess might expect from a "coding agent CLI."

## Why this exists

OpenCode positions itself as the open, provider-agnostic terminal coding agent — a TUI (built on a custom `@opentui/*` rendering stack, `packages/tui/`), a server (`packages/opencode/src/server/`), a desktop app (`packages/desktop/`, Electron-based per `trustedDependencies` in root `package.json`), and a web app (`packages/app/`, `packages/web/`), all driven by one shared core (`packages/core/`, `packages/opencode/`). The repo ships 20+ localized READMEs (`README.ar.md` through `README.zht.md`), a public roadmap-shaped `specs/` directory, and a `CONTRIBUTING.md` — signals of a project optimizing for a broad open-source contributor base, not a single-vendor product.

The engineering culture is unusually explicit about its own conventions. `AGENTS.md` (the repo's own agent-facing style guide — there is no separate `CLAUDE.md`) lays out dependency direction rules (`AGENTS.md:3`: *"Keep runtime dependencies directed from Schema to Core and Protocol, then from Core and Protocol to Server"*), import discipline (`AGENTS.md:61-64`: never alias imports, never star-import), and a dense "V2 Session Core" section (`AGENTS.md:151-161`) that reads like architecture decision records compressed into agent instructions — durable prompt admission separated from model execution, Session-ID-scoped execution, explicit provider-turn boundaries. This is a project treating its own AGENTS.md as executable documentation for the agents (including OpenCode itself, dogfooded) that will edit its code.

## Deep dive 1 — file/session organization: XDG dirs, per-project instances, and a real database

### Where things live on disk

`packages/core/src/global.ts:10-29` derives every runtime path from the `xdg-basedir` package plus the app name `"opencode"`:

```ts
const app = "opencode"
const data = path.join(xdgData!, app)     // ~/.local/share/opencode
const cache = path.join(xdgCache!, app)   // ~/.cache/opencode
const config = path.join(xdgConfig!, app) // ~/.config/opencode
const state = path.join(xdgState!, app)   // ~/.local/state/opencode
```

`Global.Path` additionally exposes `bin` (cached binaries), `log`, `repos`, and `tmp` — all created eagerly at import time (`global.ts:35-43`, a top-level `await Promise.all([fs.mkdir(...), ...])`). Every other service in the codebase resolves paths through this one module; there is no scattered `~/.opencode` guessing.

### The database — SQLite via Drizzle, not "just JSON files"

The single most surprising finding in this study: **OpenCode's session/message persistence is a real relational database**, not flat JSON. `packages/core/src/database/database.ts:43-54` resolves the DB file:

```ts
export function path() {
  if (Flag.OPENCODE_DB) { ... }
  if (["latest", "beta", "prod"].includes(InstallationChannel) || ...)
    return join(Global.Path.data, "opencode.db")
  return join(Global.Path.data, `opencode-${InstallationChannel...}.db`)
}
```

So the canonical file is `~/.local/share/opencode/opencode.db` (channel-suffixed for dev/beta builds). On startup (`database.ts:22-37`) it runs `PRAGMA journal_mode = WAL`, `synchronous = NORMAL`, `busy_timeout = 5000`, `cache_size = -64000`, `foreign_keys = ON`, then applies `DatabaseMigration.apply(db)`. `packages/core/src/database/migration/` contains **38 numbered, timestamped migration files** (e.g. `20260622170816_reset_v2_session_state.ts`, `20260604172448_event_sourced_session_input.ts`) — this is a maintained schema with real evolution history, comparable in seriousness to a production SaaS backend.

The schema itself (`packages/core/src/session/sql.ts:22-176`) defines `SessionTable`, `MessageTable`, `PartTable`, `TodoTable`, `SessionMessageTable`, `SessionInputTable`, and `SessionContextEpochTable` — full Drizzle `sqliteTable()` definitions with foreign keys (`onDelete: "cascade"`), composite indexes (e.g. `session_message_session_seq_idx`, `message_session_time_created_id_idx`), and custom column types (`packages/core/src/database/path.ts:27-59`) that enforce absolute-path invariants at the ORM layer (`absoluteColumn`, `directoryColumn` reject non-absolute paths on write).

### The JSON storage layer — legacy, with its own migration chain

Coexisting with the database is `packages/opencode/src/storage/storage.ts` — a key-based JSON file store (`file(dir, key)` → `path.join(dir, ...key) + ".json"`, `storage.ts:63-65`) guarded by `TxReentrantLock` per resolved path (`storage.ts:257-264`). This module carries its **own** migration array (`MIGRATIONS: Migration[]`, `storage.ts:81-211`) that walks legacy `storage/session/{message,part}/*/*.json` layouts into a flatter `project/<id>.json`, `session/<projectID>/<id>.json`, `message/<id>/<id>.json` tree, tracked by a `migration` marker file (`storage.ts:225-241`). This is the artifact of a real production system's storage format evolving twice — first from a per-project directory tree to a flatter global one, then (per the database migrations above) from JSON files to SQLite. The JSON layer persists for whatever hasn't yet been fully moved into the DB (the migration comments reference `session_diff` extraction, `storage.ts:182-210`).

### Per-project instance scoping

`packages/opencode/src/project/instance-context.ts`, `instance-store.ts`, and `bootstrap.ts` establish an `InstanceContext` (directory + worktree) that every service (`Config`, `MCP`, `Skill`, `ToolRegistry`) reads through `InstanceState.make(...)` — so config discovery, skill discovery, and MCP connections are all resolved relative to *which project directory the session is rooted in*, not a single global process state. `Config.directories()` and `ConfigPaths.directories` (`packages/opencode/src/config/paths.ts:23-41`) union the project's `.opencode` dirs (walked upward from `directory` to `worktree`), the user's home `.opencode`, the global `Global.Path.config`, and an optional `OPENCODE_CONFIG_DIR` override.

## Deep dive 2 — memory/state persistence across sessions: the System Context algebra

`CONTEXT.md` (repo root) is a glossary the team maintains explicitly so contributors (and agents) use consistent vocabulary for session memory. Key terms, verbatim:

- **System Context** (`CONTEXT.md:8`): *"The structured collection of contextual facts presented to the model as initial instructions and chronological updates."* Explicitly *avoid* the term "system prompt."
- **Session History** (`CONTEXT.md:12`): the projected chronological conversation after compaction + Context Epoch cutoffs.
- **Context Source** (`CONTEXT.md:16`): one independently observed typed value in the System Context, with "a stable key, JSON codec, infallible loader, pure baseline/update renderers."
- **Context Epoch** (`CONTEXT.md:24`, quoted above): the caching boundary — one rendered System Context stays the provider-cache baseline until compaction, Session movement, or an incompatible transition forces a fresh one.

`AGENTS.md:161` operationalizes this: *"Keep the System Context algebra, registry, and built-ins in `src/system-context`; keep Context Source producers with their observed domains, and keep Session History selection plus Context Epoch persistence Session-owned."* The `SessionContextEpochTable` (`packages/core/src/session/sql.ts:168-176`) is the concrete persistence: `session_id` (PK), `baseline` (text), `snapshot` (JSON, typed as `SystemContext.Snapshot`), `baseline_seq`. This is the DB row that makes prompt-caching cache-hits durable across process restarts — the epoch's baseline text and its rendered snapshot survive a crash, so resuming a session doesn't force a fresh, expensive cache-cold system prompt.

`AGENTS.md:153-159` describes the durable-prompt-admission model: `SessionV2.prompt(...)` admits a `session_input` row before scheduling model execution (`SessionInputTable`, `sql.ts:140-166`, with `admitted_seq` / `promoted_seq` sequence columns), and `SessionRunCoordinator` (mentioned in `AGENTS.md:158`) coalesces wakeups per Session ID. Reusing a Session ID "adopts" the existing session; reusing a prompt message ID reconciles an exact retry only when Session, prompt, and delivery mode all match. This is a deliberately event-sourced design — the `20260604172448_event_sourced_session_input` migration name is the literal historical marker for when this shipped.

## Deep dive 3 — tool-call tracing: real OpenTelemetry, not ad hoc logs

`packages/core/src/observability/otlp.ts` wires actual OTLP export, gated on `Flag.OTEL_EXPORTER_OTLP_ENDPOINT` (`otlp.ts:7`, `56`):

```ts
export function loggers() {
  if (!endpoint) return []
  return [OtlpLogger.make({ url: `${endpoint}/v1/logs`, resource: resource(), headers })]
}

export async function tracingLayer() {
  if (!endpoint) return Layer.empty
  const NodeSdk = await import("@effect/opentelemetry/NodeSdk")
  ...
  return NodeSdk.layer(() => ({
    resource: resource(),
    spanProcessor: new SdkBase.BatchSpanProcessor(new OTLP.OTLPTraceExporter({ url: `${endpoint}/v1/traces`, headers })),
  }))
}
```

`resource()` (`otlp.ts:36-48`) tags every exported span/log with `serviceName: "opencode"`, `serviceVersion`, `opencode.client`, and a per-process `opencode.run` ID — so a single OTLP collector can distinguish TUI runs from server runs from CI runs. This is genuinely optional (empty `Layer` when no endpoint is configured) but not vaporware: it is wired through `@effect/opentelemetry`'s real `NodeSdk`, with an `AsyncLocalStorageContextManager` explicitly registered (`otlp.ts:63-66`) so that spans from the `ai` SDK's own instrumentation nest correctly under Effect's spans.

Underneath OTel, every service method in the codebase is wrapped in `Effect.fn("Namespace.method")(...)` (e.g. `MCP.tools`, `mcp/index.ts:666`; `Storage.remove`, `storage.ts:266`; `Skill.get`, `skill/index.ts:289`) — Effect's `Effect.fn` auto-instruments each call as a named span in the Effect runtime's own trace tree, which is what actually feeds the OTLP exporter above. Additionally, `tool/registry.ts:166-173` wraps every plugin-tool execution in an explicit `Effect.withSpan("Tool.execute", { attributes: { "tool.name": id, "session.id": ..., "message.id": ..., "tool.call_id": ... } })` — so a tool call's OTel span carries the session/message/call IDs needed to correlate it back to a specific row in `MessageTable`/`PartTable`. There is no separate ad hoc "trace log file" format; tracing is unified through Effect's span mechanism, exported via OTLP when configured, and additionally persisted as durable session/message/part rows in SQLite regardless of whether OTel is turned on.

## Deep dive 4 — tool/MCP scoping: registry composition + wildcard permission rules

### Tool registry composition

`packages/opencode/src/tool/registry.ts` is the single place all tools converge. `ToolRegistry.Service` (`registry.ts:84`) assembles, in its `InstanceState.make` initializer (`registry.ts:116-249`):

1. **Custom tools from disk** — glob `{tool,tools}/*.{js,ts}` across every `config.directories()` result (`registry.ts:178-192`), dynamically `import()`-ed and adapted via `fromPlugin()`.
2. **Custom tools from plugins** — `plugin.list()` entries' `.tool` maps (`registry.ts:194-199`).
3. **Built-ins** — `shell`, `read`, `glob`, `grep`, `edit`, `write`, `task`, `fetch`, `todo`, `search`, `skill`, `patch`, `question`, plus conditionally `lsp` (`flags.experimentalLspTool`) and `plan` (`flags.experimentalPlanMode && flags.client === "cli"`) (`registry.ts:224-244`).

`ToolRegistry.tools(input)` (`registry.ts:286-335`) then *filters per model/agent*: web search only if `webSearchEnabled(providerID, flags)` (`registry.ts:58-60`, true for the `opencode` provider or Exa/Parallel flags); `apply_patch` vs. `edit`/`write` chosen by model ID heuristics (`registry.ts:292-295`, e.g. GPT models get `apply_patch`); MCP-backed "code mode" tool description only rendered if a code-mode tool is actually present.

### MCP client — full local + remote support

`packages/opencode/src/mcp/index.ts` implements the MCP client end to end using `@modelcontextprotocol/sdk`. Local servers connect over `StdioClientTransport` (`connectLocal`, `mcp/index.ts:340-370`, spawning `mcp.command` with `cwd` resolved against the instance directory). Remote servers try `StreamableHTTPClientTransport` then fall back to `SSEClientTransport` (`connectRemote`, `mcp/index.ts:236-338`), with OAuth handled via `McpOAuthProvider`/`McpOAuthPendingProvider` (`mcp/oauth-provider.ts`) — including the "needs_client_registration" vs. "needs_auth" distinction (`mcp/index.ts:296-322`) and a full authorization-code exchange flow (`startAuth`/`authenticate`/`finishAuth`, `mcp/index.ts:806-942`). Live tool-list changes are handled via `ToolListChangedNotificationSchema` (`mcp/index.ts:462-471`), and server log messages are bridged into Effect's log levels (`serverLog`, `mcp/index.ts:474-490`). Client capabilities are declared narrowly and deliberately — `roots: {}` is enabled, while `sampling`, `elicitation`, and `tasks` are explicitly commented out with linked GitHub issue numbers (`mcp/index.ts:39-50`), a a clear record of "we know these MCP primitives exist, we haven't turned them on yet, here's why."

### Permission — wildcard rules, not a fixed allow-list

`packages/opencode/src/permission/index.ts` is the scoping mechanism that decides which *already-registered* tools/MCP servers/skills a given agent turn can actually invoke. `Permission.evaluate(permission, pattern, ...rulesets)` (`index.ts:28-38`) does a `findLast` wildcard match (`Wildcard.match`) over flattened rulesets, defaulting to `{ action: "ask", pattern: "*" }` if nothing matches. `Permission.fromConfig()` (`index.ts:186-198`) turns the user's `opencode.json` `permission` block into concrete rules, expanding `~/` and `$HOME` in patterns (`expand()`, `index.ts:178-184`). `Permission.disabled()` / `Permission.visibleTools()` (`index.ts:204-219`) compute the actually-hidden tool set for a given ruleset — this is what `ToolRegistry.describeCodeMode` (`registry.ts:275-284`) uses to filter MCP tools before describing them to the model. Default agent permissions (`agent/agent.ts:119-136`) allow-list `Truncate.GLOB`, the tmp dir, every discovered skill directory, and every discovered reference directory for `external_directory`, while gating `.env*` reads to `"ask"` (mirroring GitHub's `Node.gitignore` convention, per the inline comment). Live permission requests flow through a `Deferred`-based ask/reply protocol (`ask`/`reply`, `index.ts:67-167`) — a tool call blocks on a `Deferred.await` until a human (or an `"always"` rule) resolves it, and rejecting one pending request in a session cascades to reject all other pending requests in that same session (`index.ts:129-138`).

## Deep dive 5 — skills, agents, and commands: three loaders, one convergence point

### Skill discovery

`packages/opencode/src/skill/index.ts` scans, in priority order, per `discoverSkills()` (`skill/index.ts:173-233`):

1. `~/.claude/skills/**/SKILL.md` (`CLAUDE_EXTERNAL_DIR = ".claude"`, unless `disableClaudeCodeSkills`) and `~/.agents/skills/**/SKILL.md` (`AGENTS_EXTERNAL_DIR`) — global.
2. The same two external dirs walked upward from the project directory to the worktree root — project-scoped.
3. Every `config.directories()` result's `{skill,skills}/**/SKILL.md` (opencode's own convention, `OPENCODE_SKILL_PATTERN`).
4. Any `cfg.skills?.paths` explicitly configured in `opencode.json`.
5. Any `cfg.skills?.urls` — pulled and cached under `Global.Path.cache/skills/<name>/` by `Discovery.pull()` (`skill/discovery.ts:49-132`), which does a real index.json → per-file download → atomic staging-directory swap (`staging`/`backup` rename dance, `discovery.ts:93-121`) keyed on a `.opencode-version` marker file so re-pulls are idempotent unless the remote version string changes.

A skill is loaded via `ConfigMarkdown.parse(match)` (frontmatter `name`/`description` + markdown body, `skill/index.ts:105-140`) — the exact same "YAML frontmatter + prose body" shape as a Lossless `SKILL.md`. One built-in skill, `customize-opencode` (`skill/index.ts:32-35`, `278-283`), ships hard-coded in the binary and is registered *before* disk discovery specifically so a user's on-disk skill of the same name can override it — the stated rationale in the source comment is that models routinely guess wrong at opencode's own config schema and this skill "gives it the actual schemas instead of guesses" (`skill/index.ts:27-31`).

This checkout's own `.opencode/` directory (repo root) is a live example of the convention in action: `.opencode/skills/effect/SKILL.md`, `.opencode/agent/triage.md` + `duplicate-pr.md`, `.opencode/command/{learn,changelog,rmslop,ai-deps,issues,translate,commit,spellcheck}.md`, and `.opencode/tool/{github-triage,github-pr-search}.ts` — OpenCode dogfoods its own extensibility surface to maintain itself.

### Agent (mode) definitions

`packages/opencode/src/agent/agent.ts` defines `Agent.Info` (`agent.ts:35-56`) — `name`, `description`, `mode: "subagent" | "primary" | "all"`, `permission: PermissionV1.Ruleset`, optional `model`, `prompt`, `steps`. Built-in agents (`build`, seen starting at `agent.ts:140`) get their permission ruleset assembled via `Permission.merge(defaults, Permission.fromConfig(...))`. Custom agents load from markdown files (mirroring skills structurally) merged with `cfg.agent` JSON config, and `Agent.generate()` (`agent.ts:69-79`) can synthesize a brand-new agent definition (`identifier`/`whenToUse`/`systemPrompt`) from a natural-language description via `generateObject`/`streamObject` against the configured provider.

### Commands

`packages/opencode/src/command/index.ts` unifies three sources into one `Command.Info` map (`command/index.ts:65-157`): built-in `init`/`review` templates embedded as `.txt` files (`PROMPT_INITIALIZE`, `PROMPT_REVIEW`), user-defined `cfg.command` entries from `opencode.json`, **MCP server prompts** (fetched lazily via `mcp.getPrompt(...)`, with `$1`/`$2`-style positional argument substitution, `command/index.ts:105-132`), and — critically — **every discovered skill**, each exposed as an invocable command if no explicit command of that name already exists (`command/index.ts:134-152`). This is the concrete mechanism behind the observation above: skills and slash-commands converge into one namespace, so a `SKILL.md` and a `command/foo.md` are, from the model/user's perspective, the same kind of invocable unit.

## What's inside this submodule

```text
opencode/
├── AGENTS.md              # repo-level agent/style conventions (no separate CLAUDE.md; this IS the contract)
├── CONTEXT.md             # glossary defining System Context / Session History / Context Epoch vocabulary
├── README.md (+ 20 locales)
├── LICENSE                # MIT, "Copyright (c) 2025 opencode"
├── package.json           # root workspace manifest; bun@1.3.14; no go.mod anywhere in the tree
├── sst.config.ts          # SST (the deploy tool) config for the team's OWN cloud infra — unrelated to org naming
├── .opencode/             # the repo dogfooding its own extensibility surface
│   ├── agent/             # triage.md, duplicate-pr.md — custom agent definitions
│   ├── command/            # learn.md, changelog.md, commit.md, spellcheck.md, ...
│   ├── skills/effect/      # SKILL.md for the Effect library conventions
│   ├── tool/               # github-triage.ts, github-pr-search.ts — custom TS tools
│   └── opencode.jsonc      # project-level config
├── specs/                  # design specs
├── packages/
│   ├── core/                       # shared engine: Global paths, Database (Drizzle+SQLite), OTel, schema, providers
│   │   └── src/
│   │       ├── global.ts           # XDG path resolution — data/cache/config/state/tmp/log/bin/repos
│   │       ├── database/           # sqlite.ts, database.ts, migration/ (38 files), schema.sql.ts, path.ts (custom column types)
│   │       ├── observability/otlp.ts  # OTLP logger + tracer wiring, gated on OTEL_EXPORTER_OTLP_ENDPOINT
│   │       └── session/sql.ts      # SessionTable, MessageTable, PartTable, SessionInputTable, SessionContextEpochTable
│   ├── opencode/                   # the actual CLI/server runtime
│   │   └── src/
│   │       ├── mcp/                # index.ts (client), catalog.ts, oauth-provider.ts, oauth-callback.ts, browser.ts
│   │       ├── skill/               # index.ts (discovery + loading), discovery.ts (remote skill-pack pull/cache)
│   │       ├── permission/          # index.ts (wildcard rules), evaluate.ts, arity.ts
│   │       ├── tool/                # registry.ts + one file per built-in tool (read/write/edit/grep/glob/shell/task/...)
│   │       ├── agent/agent.ts       # Agent.Info schema, built-in "build" agent, agent generation
│   │       ├── command/index.ts    # unifies built-in / config / MCP-prompt / skill commands
│   │       ├── config/              # config.ts (load/merge), paths.ts (dir walk-up), markdown.ts, plugin.ts
│   │       ├── session/             # session.ts, message-v2.ts, compaction.ts, llm.ts, prompt/, system.ts
│   │       ├── storage/storage.ts   # legacy JSON key-value store + its own migration chain
│   │       └── plugin/              # plugin loader + built-in provider plugins (azure, cloudflare, openai, xai, ...)
│   ├── tui/                # terminal UI (OpenTUI-based)
│   ├── desktop/            # Electron desktop app
│   ├── app/, web/          # SolidJS web clients
│   ├── sdk/, sdk-next/     # generated + hand-composed SDKs
│   ├── protocol/, schema/  # wire schema + protocol definitions
│   ├── console/            # hosted console (SST-deployed)
│   └── plugin/             # public @opencode-ai/plugin package surface
└── script/, install/, infra/, nix/  # build, install, and deployment tooling
```

If you read four files, in this order: `CONTEXT.md` (the memory vocabulary), `AGENTS.md` (the engineering contract), `packages/opencode/src/mcp/index.ts` (MCP client + scoping), and `packages/core/src/session/sql.ts` (what "a session" actually is on disk).

## Mental model for using it well

- **Config and skills both walk up the directory tree.** A `.opencode/` at any ancestor between cwd and the git worktree root is discovered (`config/paths.ts:23-41`), and skills/commands/agents follow the same convention. Don't assume only the repo root is scanned.
- **Skills, commands, and MCP prompts are the same namespace.** A `SKILL.md` becomes a slash command automatically unless a same-named `command/*.md` already exists (`command/index.ts:134-152`). Naming collisions are a real design surface to think about, not a hypothetical.
- **Session memory has an explicit cache-invalidation model.** The "Context Epoch" concept means prompt-cache-friendly behavior is architected, not incidental — compaction, session movement, or an incompatible context change are the only three triggers that force a fresh baseline (`CONTEXT.md:24`).
- **The SQLite database is the source of truth for sessions; JSON files are legacy/transitional.** If you're inspecting a user's OpenCode data directory, look at `~/.local/share/opencode/opencode.db` first, not the `storage/` JSON tree — the latter has an active migration chain moving data *out* of it.
- **MCP servers are scoped per-agent via wildcard permission rules, not a static allow-list file.** `Permission.visibleTools()` filters the merged tool set (built-ins + MCP) per invocation based on the agent's + session's ruleset; there's no single "enabled MCP servers" list independent of permission evaluation.
- **OTel is opt-in and off by default.** Nothing is exported unless `OTEL_EXPORTER_OTLP_ENDPOINT` is set — but when it is, every `Effect.fn`-wrapped service call and every tool execution becomes a correlated span carrying session/message/tool-call IDs.
- **The repo's own `.opencode/` directory is the best worked example of every extensibility mechanism at once** — read it before designing a new agent/skill/command/tool for your own project.

## When NOT to reach for this

- **You want a minimal, dependency-light coding-agent CLI.** OpenCode is a 30+-package Bun monorepo with a real SQLite database, Effect-TS throughout, and a TUI rendering stack. If you want something closer to a single Python script, this is the wrong end of the spectrum.
- **You need a Go binary with no JS runtime.** There is no Go anywhere in this codebase (verified: no `go.mod`). Bun is a hard runtime dependency (`packageManager: "bun@1.3.14"`, `postinstall` scripts assume Bun-specific APIs like `Bun.file()` per `AGENTS.md:29`).
- **You want MCP sampling/elicitation today.** The client capabilities explicitly disable them (`mcp/index.ts:39-50`, commented out with tracked issue numbers) — if your MCP server depends on server-initiated sampling or elicitation, OpenCode won't drive that flow yet.
- **You need a stable, versioned public schema to build against long-term.** The dense migration history (38 DB migrations, an internal JSON-storage migration chain on top of that) signals an actively-evolving internal data model; `AGENTS.md`'s "V2 Session Core" section describes a system mid-refactor (durable prompt admission vs. legacy `SessionPrompt.loop`), not a frozen contract.
- **You're evaluating "skills" as OpenCode's own invention.** Its skill format is deliberately compatible with `.claude/skills` and `.agents/skills` conventions from other tools — if your interest is specifically in a from-scratch skills design, look elsewhere in the study.

## How this compares to the study's other harnesses

| Axis | OpenCode | goose | aider |
|---|---|---|---|
| **Language/runtime** | TypeScript, Bun-only (no Go anywhere in the tree) | Rust | Python |
| **Shape** | CLI + TUI + server + desktop (Electron) + web app, one shared core | CLI agent with extensions | CLI, git-native |
| **Tool/MCP model** | Full MCP client (local stdio + remote HTTP/SSE, OAuth); tools composed from built-ins + plugin files + MCP servers, scoped per-agent via wildcard permission rules | Extensions are themselves MCP-wrapper shaped — "extensions-as-MCP-wrapper" harness | No MCP; tools are the diff/edit protocol plus repo-map context, not a discoverable tool registry |
| **Session persistence** | SQLite (Drizzle ORM, WAL mode, 38 migrations) for sessions/messages/parts; legacy JSON-file store with its own migration chain for older data | (not verified in this profile — see goose's own profile) | Git commits are the persistence layer; no separate session database — history lives in the repo's own commit log and chat history files |
| **Tracing** | Real OTLP export (`@effect/opentelemetry`, gated on env var) + `Effect.withSpan` on every tool execution carrying session/message/call IDs | (see goose profile) | (see aider profile — no built-in OTel; diffs and commits are the audit trail) |
| **Skills/agents/commands** | Skills read from `.claude/skills`, `.agents/skills`, `.opencode/skill(s)`, config paths, and remote URLs; converge into one command namespace with MCP prompts and static commands | Extensions play the analogous role, wrapped as MCP | No formal skill/agent-definition system; behavior is driven by the repo-map + prompt, not loadable skill files |
| **Config layering** | Global (`~/.config/opencode/opencode.json`) + project (walked-up `.opencode/opencode.json`) + env-var overrides, deep-merged with array concatenation for `instructions` | (see goose profile) | `.aider.conf.yml` + CLI flags; no MCP-server config surface to layer |
| **Best fit** | Teams wanting a database-backed, observable, multi-surface (TUI/server/desktop) coding agent with first-class MCP and skill ecosystems | Teams wanting a Rust-native agent where every capability is naturally MCP-shaped | Teams wanting a minimal, git-native pair-programming loop without a tool-call/MCP layer at all |

The crucial axis OpenCode owns in this study is **persistence maturity paired with observability**: it is the only entry with a real migrated SQL schema for conversational state *and* wired OTLP tracing *and* a documented internal vocabulary (`CONTEXT.md`) for how session memory should behave across restarts and compactions.

## A note on the org-name question

The task brief asked to verify whether this checkout shows "sst/opencode" as a historical org name distinct from the current one. Across this entire pinned checkout — `git remote -v` (`https://github.com/anomalyco/opencode.git`), every `package.json`'s `"repository"` field, the README's install instructions (`brew install anomalyco/tap/opencode`, badge URLs pointing at `github.com/anomalyco/opencode/actions`), and a full-tree grep — **no reference to `sst/opencode` or `sst-dev/opencode` as a prior org name was found.** The only "sst" artifact in the repo is `sst.config.ts` at the root, which is the team's own use of the SST deployment tool to manage their cloud infrastructure (the console, stats app, etc.) — unrelated to the GitHub org name. If a rename from `sst-dev`/`sst` to `anomalyco` happened, it predates or is otherwise invisible in this pinned checkout; the only verifiable, cite-able fact is that **every current signal in the repo says `anomalyco`.**

## One-line summary

> OpenCode is a TypeScript/Bun (no Go), Effect-native coding-agent harness where sessions and messages persist in a real, migrated SQLite database (WAL mode, 38 migrations) rather than flat JSON files, memory-across-turns is governed by an explicitly documented "Context Epoch" caching algebra (`CONTEXT.md`), tool-call execution is traced through Effect's own span mechanism and optionally exported via real OTLP, MCP servers (local stdio and remote HTTP/SSE with OAuth) are connected and then scoped per-agent through wildcard permission rules rather than a static allow-list, and skills/commands/agents converge into one discoverable, `.claude/skills`-and-`.agents/skills`-compatible namespace that the project visibly dogfoods on itself via its own `.opencode/` directory.
