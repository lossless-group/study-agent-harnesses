---
name: Codex Profile
title: "Codex — Profile"
slug: codex
upstream: https://github.com/openai/codex
package: n/a (Rust crates under codex-rs/, published as @openai/codex CLI + platform binaries)
license: Apache-2.0
maintainer: OpenAI
study: studies/agent-harnesses
profile_path: studies/agent-harnesses/codex
profile_kind: Rust terminal coding-agent CLI, OS-level sandbox enforcement, MCP client+server
date_created: 2026-07-13
date_modified: 2026-11-23
authors:
  - Michael Staton
site_uuid: b5d39c8d-feec-404d-8412-11d2ba000c1c
hex_code: jflus4
date_authored_initial_draft: 2026-07-13
date_authored_current_draft: 2026-11-23
lede: >-
  Codex compiles a real Apple Seatbelt profile and a bubblewrap-plus-seccomp jail:
  a fully autonomous session that never prompts is still jailed by the OS.
summary: >-
  Source-cited profile of the OpenAI Codex submodule in the agent-harnesses study. The load-bearing
  finding is the two-axis model: the sandbox (`PermissionProfile`, formerly `SandboxPolicy`: what the operating system physically permits) and
  `AskForApproval` (when a human is interrupted) are orthogonal enums, giving four independent
  quadrants rather than one linear trust dial. Also documents the inverse-of-Goose persistence
  relationship — append-only JSONL rollouts with zstd cold compression are the durable truth and
  SQLite is a rebuildable derived index — plus hierarchical `AGENTS.md` concatenation root-to-cwd
  with an `AGENTS.override.md` escape hatch, a separate `SKILL.md` loader, and a native MCP client
  (the MCP server mode was removed in September 2026). Reach for this profile when the design question
  is blast-radius containment rather than tool visibility.
status: Reference
publish: true
type: Profiles
---

# Codex — Profile

A profile of Codex as it lives in this study (`studies/agent-harnesses/codex/`). Cites pinned paths so you can jump to source rather than trust paraphrase. Read alongside [`Profile__Goose.md`](./Profile__Goose.md) and the `opencode` profile — where Goose gates tool calls with a three-tier `AlwaysAllow`/`AskBefore`/`NeverAllow` permission layer and opencode uses wildcard permission rules, **Codex enforces at the operating-system level**: real Seatbelt policies on macOS, bubblewrap namespaces + seccomp-bpf on Linux (Landlock kept only as a legacy option), restricted tokens on Windows — with a *separate*, orthogonal human-approval policy layered on top. `LICENSE:1-3` confirms Apache-2.0. The crate tree lives at `codex-rs/` (workspace of 200+ crates at this pin, Bazel + Cargo dual-built per `BUILD.bazel`/`MODULE.bazel`), with a legacy `codex-cli/` (the original TypeScript CLI, now superseded).

## TL;DR

The README states the identity plainly (`README.md:1`):

> **Codex CLI** is a coding agent from OpenAI that runs locally on your computer.

Mechanically: **sandboxing and approval are two independent axes, not one gate.** `SandboxPolicy` (`codex-rs/protocol/src/protocol.rs:1293-1341`) is a closed enum — `DangerFullAccess`, `ReadOnly { network_access }`, `ExternalSandbox { network_access }`, `WorkspaceWrite { writable_roots, network_access, .. }` — describing *what the OS will physically allow the child process to do*, regardless of whether a human was ever asked. `AskForApproval` (`protocol.rs:1206-1229`) is a completely separate enum — `UnlessTrusted`, `OnRequest`, `Granular(GranularApprovalConfig)`, `Never` — describing *when a human is interrupted for a yes/no*. A command can be sandboxed and never trigger a prompt, or prompted-for and still run inside a Seatbelt/bubblewrap jail if approved. Since the July pin, the runtime's canonical form of the sandbox axis is `PermissionProfile` (`Managed`/`Disabled`/`External`, `codex-rs/protocol/src/models.rs:428-442`), with named profiles such as `:read-only`, `:workspace`, `:danger-full-access` (`models.rs:416-422`); `SandboxPolicy` survives as the legacy shape that gets converted into it. The two-axis model is unchanged. This is a materially different shape than a single `AlwaysAllow`/`AskBefore`/`NeverAllow` tri-state per tool.

The OS enforcement is real, not simulated. On macOS, `build_seatbelt_access_policy()` and `create_seatbelt_command_args()` (`codex-rs/sandboxing/src/seatbelt.rs:428-533,761-770`) compile a live Apple Sandbox (`.sbpl`) profile — starting from `(deny default)` (`seatbelt_base_policy.sbpl:8`, explicitly modeled on Chromium's own sandbox policy per the file's header comment) — and shells out to `/usr/bin/sandbox-exec -p <policy>` (`MACOS_PATH_TO_SEATBELT_EXECUTABLE`, `seatbelt.rs:105`, hardcoded to `/usr/bin` specifically to defeat a PATH-injection attack, per the comment at `seatbelt.rs:101-104`). On Linux, the helper uses **bubblewrap** for filesystem and namespace isolation and a `seccompiler`-built BPF filter for syscalls, blocking network syscalls outright when network access is disabled (`codex-rs/linux-sandbox/README.md:10-14`; `apply_sandbox_policies_to_current_thread()`, `codex-rs/linux-sandbox/src/seccomp.rs:45`). The July version of this profile described a Landlock `Ruleset` as the Linux filesystem mechanism; bubblewrap was already preferred then, and Landlock is now a legacy option (`features.use_legacy_landlock`) that is rejected for filesystem-restricted policies (`linux-sandbox/README.md:85-87`). Either way this is kernel-enforced, not a userspace check the model could talk its way around. Windows gets its own `windows-sandbox-rs` crate using restricted tokens.

Sessions persist as append-only JSONL "rollouts" — `codex-rs/rollout/src/recorder.rs:1` states the purpose directly: *"Persist Codex session rollouts (.jsonl) so sessions can be replayed or inspected later."* Cold rollouts get transparently zstd-compressed to `.jsonl.zst` (`codex-rs/rollout/src/compression.rs`), and a separate SQLite-backed `codex_state::StateRuntime` (`codex-rs/rollout/src/state_db.rs:29-100`) indexes/backfills metadata from those files for fast listing — the JSONL is the durable source of truth; SQLite is a derived index, the inverse of Goose's SQLite-primary/JSONL-legacy relationship.

Project instructions load from hierarchical `AGENTS.md` files: `codex-rs/core/src/agents_md.rs:1-16` documents the algorithm — walk up from cwd to the project root (marked by `.git` by default), then concatenate every `AGENTS.md` found from root down to cwd, inclusive, joined by a literal `--- project-doc ---` separator (`AGENTS_MD_SEPARATOR`, `agents_md.rs:51`) — plus an `AGENTS.override.md` local-override filename that takes precedence (`LOCAL_AGENTS_MD_FILENAME`, `agents_md.rs:47`). A separate `skills` crate (`codex-rs/skills/src/parser.rs`, `loading.rs`; the July pin's `core-skills/src/loader.rs` was removed in upstream #37457) discovers durable, explicitly-invoked capabilities via `SKILL.md` frontmatter (`name`, `description`, `metadata.short-description`), distinct from the always-on `AGENTS.md` context.

MCP is bidirectional and fully native: Codex is both an MCP **client** (`codex-rs/rmcp-client/` — stdio, streamable-HTTP, and OAuth-backed transports, `StdioServerLauncher`/`ExecutorStdioServerLauncher`, `rmcp-client/src/lib.rs:13-47`; `codex-rs/codex-mcp/` layers connection management, tool-call catalogs, and elicitation on top). At the July pin it was also an MCP **server** (`codex-rs/mcp-server/`, exposing Codex itself as a tool); the `codex mcp-server` command and crate were removed on 2026-09-05 (upstream #42993), so at this pin MCP runs client-side only and outside integrations go through `codex-rs/app-server/`.

If you want one sentence: **Codex is a Rust harness where OS-level sandboxing (`SandboxPolicy`: real Seatbelt `.sbpl` compilation on macOS via `sandbox-exec`, bubblewrap+seccomp-bpf kernel enforcement on Linux, restricted tokens on Windows) is a completely separate, orthogonal axis from human-approval policy (`AskForApproval`: UnlessTrusted/OnRequest/Granular/Never), sessions persist as zstd-cold-compressed append-only JSONL "rollouts" backed by a derived SQLite index, project context loads from hierarchical `AGENTS.md` files concatenated root-to-cwd with an override filename, durable capabilities load separately as `SKILL.md`, and Codex is a native MCP client (stdio/HTTP/OAuth); the MCP server mode it had at the July pin has been removed.**

## Why this exists — sandbox as OS enforcement, not a prompt-layer gate

Three load-bearing design choices separate Codex's sandbox from a permission-gate model like Goose's or opencode's:

1. **The policy is compiled to a real OS access-control mechanism per platform, not interpreted by application code at call time.** `build_seatbelt_access_policy()` (`seatbelt.rs:428-533`) builds an actual Seatbelt profile string — `(allow file-read* (subpath (param "READABLE_ROOT_0")))`-style S-expressions — and passes it to the kernel via `sandbox-exec -p`. On Linux, bubblewrap builds the filesystem view in a new mount namespace; the legacy path, `install_filesystem_landlock_rules_on_current_thread()` (now `codex-rs/linux-sandbox/src/seccomp.rs:154-173`, renamed from `landlock.rs` in upstream #52610), still calls `landlock::Ruleset::default()...restrict_self()` when that option is enabled. Once installed, no amount of clever tool-calling by the model can escape it; the enforcement point is the kernel, not the harness's own dispatch logic.
2. **Filesystem grants are hierarchical roots with carve-outs, not a flat allow/deny per path.** `SeatbeltAccessRoot { root, excluded_subpaths, protected_metadata_names }` (`seatbelt.rs:332-336`) lets a writable root (say, the project cwd) exclude specific subpaths (e.g., a `.git` metadata directory) even though it's nested inside an otherwise-writable tree — `protected_metadata_names_for_writable_root()` (moved to `codex-rs/protocol/src/permissions.rs:2607`) walks `PROTECTED_METADATA_PATH_NAMES` (`permissions.rs:57`) to auto-derive these exclusions. `SandboxPolicy::WorkspaceWrite { writable_roots, network_access, .. }` (`protocol.rs:1319-1341`) is the config-level shape this compiles from.
3. **Network is denied by default and only reopened through an explicit, narrow policy — including a managed-proxy mode.** `dynamic_network_policy_for_network()` (`seatbelt.rs:273-322`) emits a proxy-restricted policy when a managed proxy is configured, full network only when network is enabled with no proxy, and an empty policy otherwise (`seatbelt.rs:310-321`). (The "Fail closed" comment the July profile quoted is no longer in this function.) On Linux, disabling network installs an actual seccomp-bpf filter blocking network syscalls (`network_seccomp_mode`, `codex-rs/linux-sandbox/src/seccomp.rs:122`) — not an application-level check the model's shell command could bypass by calling a syscall directly.

## The approval layer — orthogonal to the sandbox, not a substitute for it

`AskForApproval` (`codex-rs/protocol/src/protocol.rs:1206-1229`) governs *whether a human is interrupted*, independent of what the OS sandbox already prevents:

- `UnlessTrusted` — only commands `is_safe_command()` classifies as read-only are auto-approved; everything else prompts.
- `OnRequest` (the default, `#[default]`, `protocol.rs:1215`) — the model itself decides when to ask.
- `Granular(GranularApprovalConfig)` (`protocol.rs:1224`, fields at `1232-1247`) — per-category booleans: `sandbox_approval` (escalation requests), `rules` (execpolicy `prompt` rule hits), `skill_approval` (skill script execution), `request_permissions` (an explicit `request_permissions` tool call). Each can be independently allowed or auto-rejected.
- `Never` — failures go straight back to the model, no human escalation at all.

Because this is separate from `SandboxPolicy`, the two compose: a session can run `WorkspaceWrite` sandboxing with `Never` approval (fully autonomous but still OS-jailed), or `DangerFullAccess` with `OnRequest` (unsandboxed but human-gated per command) — four independent quadrants rather than one linear trust dial. `codex-rs/execpolicy/` adds a further layer of declarative per-command policy (`rules` in `GranularApprovalConfig`) evaluated before a command reaches either the sandbox or the approval prompt.

## Session persistence — JSONL rollouts, SQLite as a derived index

`codex-rs/rollout/src/recorder.rs:1` is explicit about the design: *"Persist Codex session rollouts (.jsonl) so sessions can be replayed or inspected later."*

- `SESSIONS_SUBDIR = "sessions"` / `ARCHIVED_SESSIONS_SUBDIR = "archived_sessions"` (`rollout/src/lib.rs:88-89`) under the Codex home directory (`codex-home` crate).
- Files are plain `.jsonl` while "hot," transparently compressed to `.jsonl.zst` once cold (`compress_rollout_if_cold_blocking`, `compression.rs:887`), and readers (`open_rollout_line_reader`, `compression.rs:67`) handle both transparently — `existing_rollout_path()` (`compression.rs:1329`) prefers the plain file over its compressed sibling if both exist.
- `codex_state::StateRuntime` (`state_db.rs:29`, `StateDbHandle = Arc<StateRuntime>`) is a **SQLite-backed index built from the JSONL files**, not the primary store — `init()`/`try_init_with_roots()` (`state_db.rs:45-100`) opens the runtime and runs `backfill_sessions()` (`metadata.rs:139-153`) to catch up the index against `sessions/` and `archived_sessions/` on disk. This is the inverse of Goose's relationship (SQLite primary, JSONL legacy-import-only): here the flat file is durable truth and the database is a rebuildable cache.
- `session_index.rs`, `list.rs` (`ThreadListConfig`, `ThreadSortKey`, cursor-based pagination) support listing/searching threads without loading full rollouts.

## AGENTS.md and skills — hierarchical concatenation vs. explicit invocation

`codex-rs/core/src/agents_md.rs:1-16` documents the discovery algorithm inline:

1. Walk upward from cwd until a `project_root_markers` entry is found (default: `.git`); an empty marker list disables the walk entirely.
2. Collect every `AGENTS.md` from the project root down to cwd, **inclusive**, and concatenate in that order.
3. Never walk past the project root.

`DEFAULT_AGENTS_MD_FILENAME = "AGENTS.md"` and `LOCAL_AGENTS_MD_FILENAME = "AGENTS.override.md"` (`agents_md.rs:45,47`) — the override file, when present, supersedes the discovered chain; multiple contributing docs (user-level + project-level) join via the literal `AGENTS_MD_SEPARATOR = "\n\n--- project-doc ---\n\n"` (`agents_md.rs:51`). At the July pin the repo's own root `AGENTS.md` was a live instance, instructing agents never to touch `CODEX_SANDBOX_NETWORK_DISABLED_ENV_VAR`/`CODEX_SANDBOX_ENV_VAR`; that file, along with `.codex/skills/`, was deleted on 2026-09-30 (upstream #49713, "Remove repository-local Codex guidance, skills, and environment config").

Skills are a separate mechanism, now in the `skills` crate (`codex-rs/skills/src/`): `SkillFrontmatter { name, description, metadata }` (`parser.rs:7-20`) parsed from discovered `SKILL.md` files, with loading, selection, mentions, and invocation in sibling modules (`loading.rs`, `selection.rs`, `mentions.rs`, `invocation.rs`) — explicitly invoked capabilities, not ambient context like `AGENTS.md`. (At the July pin this was `codex-rs/core-skills/src/loader.rs`.)

## MCP — client and (formerly) server

At the July pin Codex owned both directions of the MCP relationship; the server half has since been removed:

- **As a client**: `codex-rs/rmcp-client/` provides transports — `StdioServerLauncher`/`ExecutorStdioServerLauncher`/`LocalStdioServerLauncher` (`lib.rs:45-47`) for subprocess MCP servers, `http_client_adapter.rs`/`oauth_http_client.rs`/`oauth.rs` for remote HTTP MCP servers with OAuth, `in_process_transport.rs` for in-process wiring, `executor_process_transport.rs` for sandboxed-executor-mediated process transport. `codex-rs/codex-mcp/` layers a `connection_manager.rs` (mutation of live tool sets and tool calls — at the July pin the repo's own `AGENTS.md` told contributors to route MCP tool-call changes through this file specifically), a `catalog.rs` of available tools, and `elicitation.rs`/`auth_elicitation.rs` for interactive auth flows.
- **As a server (removed 2026-09-05, upstream #42993)**: `codex-rs/mcp-server/` (`main.rs`, `message_processor.rs`, `codex_tool_runner.rs`) exposed Codex itself as an MCP tool — `exec_approval.rs` and `patch_approval.rs` are the approval-flow hooks surfaced to whatever external MCP host is driving Codex this way.

## What's inside this submodule

| Path | What's there |
|---|---|
| `codex-rs/sandboxing/src/seatbelt.rs` | macOS Seatbelt `.sbpl` policy compiler; `create_seatbelt_command_args`, `sandbox-exec` invocation |
| `codex-rs/sandboxing/src/seatbelt_base_policy.sbpl` | The actual Chrome-inspired base policy, `(deny default)` |
| `codex-rs/sandboxing/src/linux.rs`, `bwrap.rs`, `manager.rs` | Linux/Windows CLI-arg construction, sandbox-type dispatch (the July pin's `sandboxing/src/landlock.rs` is gone) |
| `codex-rs/linux-sandbox/src/bwrap.rs` | Bubblewrap filesystem/namespace sandbox, the primary Linux path |
| `codex-rs/linux-sandbox/src/seccomp.rs` | seccomp-bpf filter installation plus the legacy Landlock path (renamed from `landlock.rs`) |
| `codex-rs/windows-sandbox-rs/` | Windows restricted-token sandbox implementation |
| `codex-rs/protocol/src/protocol.rs`, `models.rs`, `permissions.rs` | `AskForApproval`, legacy `SandboxPolicy`, and the canonical `PermissionProfile` — the two orthogonal axes |
| `codex-rs/execpolicy/` | Declarative per-command policy layer (rules that can trigger `Granular` prompts) |
| `codex-rs/rollout/src/recorder.rs`, `compression.rs` | JSONL rollout writer, zstd cold-compression |
| `codex-rs/rollout/src/state_db.rs`, `metadata.rs` | SQLite-backed derived index, backfill-from-JSONL |
| `codex-rs/core/src/agents_md.rs` | `AGENTS.md` hierarchical discovery + concatenation, `AGENTS.override.md` |
| `codex-rs/skills/src/` | `SKILL.md` skill parsing, loading, and invocation, separate from `AGENTS.md` |
| `codex-rs/rmcp-client/`, `codex-rs/codex-mcp/` | MCP client: stdio/HTTP/OAuth transports, connection manager, catalog |
| `codex-rs/app-server/` and related `app-server-*` crates | The integration surface for IDEs and other hosts |
| `codex-rs/tui/` | Terminal UI |
| `codex-rs/exec/`, `codex-rs/exec-server/` | Non-interactive `codex exec` mode; sandboxed executor process/filesystem |
| `codex-cli/` | Legacy TypeScript CLI, superseded by `codex-rs` |
| `sdk/` | Language SDKs (e.g. `sdk/python/src/openai_codex/_sandbox.py`) |
| `docs/sandbox.md` | Points to hosted docs (`developers.openai.com/codex/security`) rather than in-repo detail |

If you read three files: `codex-rs/sandboxing/src/seatbelt.rs` (the whole macOS policy-compilation shape), `codex-rs/linux-sandbox/README.md` with `src/bwrap.rs` (the bubblewrap+seccomp Linux path), and `codex-rs/protocol/src/protocol.rs` around `SandboxPolicy`/`AskForApproval` (the two-axis model).

## Mental model for using it well

- **Think in two independent dials, not one trust slider**: `SandboxPolicy` (what the OS physically permits) and `AskForApproval` (when a human is asked). Configure them separately; a fully-sandboxed session can still run with zero prompts, and an unsandboxed session can still prompt for everything.
- **Trust the sandbox as a hard boundary, not a suggestion** — Seatbelt/bubblewrap/seccomp are kernel/OS mechanisms; a model cannot argue its way past a `(deny default)` policy or a mount namespace the way it might talk past a prompt-layer check.
- **Use `WorkspaceWrite`'s `writable_roots` + exclusions for surgical scoping** rather than falling back to `DangerFullAccess` — the seatbelt builder auto-protects metadata directories like `.git` even inside writable roots.
- **Read JSONL rollouts directly for ground truth**; treat the SQLite state index as a rebuildable cache for fast listing, not the record of note.
- **Keep `AGENTS.md` files small and hierarchical** — the loader concatenates every one from project root to cwd, so redundant restatement at multiple levels bloats every prompt.
- **Reach for `SKILL.md` only for durable, explicitly-invoked capabilities** — it's a distinct mechanism from the always-loaded `AGENTS.md` context.

## When NOT to reach for this

- **You need a lightweight single-file CLI with no OS-level dependencies.** The sandbox stack pulls in bubblewrap, `seccompiler`, macOS's `sandbox-exec`, and a Windows restricted-token implementation — real platform-specific security infrastructure, not a config toggle you can ignore.
- **You want prompt-level, tool-by-tool allow/deny with no kernel involvement** (Goose's `AlwaysAllow`/`AskBefore`/`NeverAllow`, opencode's wildcard rules). Codex's sandbox operates a layer below the model entirely; if your threat model is "don't let the model see this tool," a permission gate is simpler and sufficient — Codex's mechanism answers "don't let this *process* touch this file/socket even if it tries," a stronger but heavier guarantee.
- **You're on a platform without a working sandbox backend** and need graceful degradation — `SandboxPolicy::ExternalSandbox` exists precisely for "already sandboxed by something else, don't double-enforce," but that still assumes an external sandbox exists.
- **You want one unified session store.** The JSONL-plus-derived-SQLite split is deliberate but is two files to reason about (a durable log and a rebuildable index) rather than a single database of record.

## Changes since the previous pin (2026-07-13 → 2026-11-23)

- Old pin `bbfa08fe3b90`, new pin `5f645d732cc2`, 5,896 commits. Stable Rust releases in range run from `rust-v0.145.0` (2026-07-21) to **`rust-v0.171.0` (2026-11-19)**. The manifest's nearest tag, `libcap-v2.75`, is a vendored-dependency tag, not a Codex release.
- **The thesis holds, with new names.** Sandboxing and approval are still separate axes. The sandbox axis is now carried at runtime as `PermissionProfile` (`codex-rs/protocol/src/models.rs:428-442`) with named, extendable profiles (`:read-only`, `:workspace`, `:danger-full-access`, or user-defined `[permissions.<id>]`), and `SandboxPolicy` is converted from as a legacy shape (upstream #36183, #37031).
- **Linux is bubblewrap plus seccomp.** The `landlock.rs` files are gone (`codex-rs/linux-sandbox/src/landlock.rs` became `seccomp.rs` in #52610; `codex-rs/sandboxing/src/landlock.rs` was removed in #62215). Landlock survives only behind `features.use_legacy_landlock`, which is rejected for filesystem-restricted policies (`codex-rs/linux-sandbox/README.md:85-87`). The lede and body were corrected to match.
- **MCP server mode removed.** `codex mcp-server` and `codex-rs/mcp-server/` were deleted on 2026-09-05 (#42993). Codex remains an MCP client.
- **Skills moved.** The legacy `core-skills` loader was removed (#37457); `SKILL.md` handling lives in `codex-rs/skills/`.
- **Repo-local agent guidance removed.** The root `AGENTS.md` and `.codex/skills/` were deleted on 2026-09-30 (#49713). The `AGENTS.md` loader in `codex-rs/core/src/agents_md.rs` is unchanged in behavior.

## How this compares to the rest of the study

| Axis | opencode | Goose | Codex |
|---|---|---|---|
| **Shape** | CLI-first, own MCP client, TS/Go | Rust CLI + Electron desktop + API/SDK, MCP-native | Rust CLI (+ TUI, exec-server, app-server), MCP client |
| **Permission/sandbox model** | Wildcard permission rules (prompt-layer) | Three-tier `AlwaysAllow`/`AskBefore`/`NeverAllow` per tool (prompt-layer) | **Two orthogonal axes**: `SandboxPolicy` (OS-kernel enforcement: Seatbelt/bubblewrap+seccomp/Windows tokens) × `AskForApproval` (human-prompt policy, independently configurable) |
| **Enforcement point** | Application-layer tool dispatch | Application-layer `PermissionManager` gate | **Kernel/OS** — `sandbox-exec` policy, bubblewrap namespaces, seccomp-bpf syscall filter |
| **Network control** | Not sandboxed at OS level | Not sandboxed at OS level | Fails closed by default; explicit proxy-only mode; Linux blocks network syscalls via seccomp-bpf |
| **Session/memory storage** | Structured, readable session storage (JSON/DB-backed) | SQLite (WAL, schema v18), migrated *from* per-session `.jsonl` | **JSONL rollouts** (zstd-cold-compressed) as durable truth; SQLite is a **derived, rebuildable index** backfilled from JSONL — inverse of Goose's relationship |
| **Ambient project context** | Not noted | `.goosehints` / `AGENTS.md`, lazy per-subdirectory loading | `AGENTS.md` (+ `AGENTS.override.md`), hierarchical root-to-cwd concatenation, loaded eagerly at session start |
| **Durable capability discovery** | Not noted | `SKILL.md` per agentskills.io spec | `SKILL.md` frontmatter via the dedicated `skills` crate, separate loader from `AGENTS.md` |
| **MCP support** | Native — opencode owns its MCP client | Native — extensions **are** the MCP client wrapper (4 config variants) | Native client — `rmcp-client`/`codex-mcp` (stdio/HTTP/OAuth); the `mcp-server` mode was removed after the July pin |
| **Best fit** | Terminal-first agent work needing a swappable MCP toolset | Cross-surface agent work needing an MCP extension ecosystem + observability-grade tracing | Autonomous or semi-autonomous coding work where the operator needs a genuine, kernel-enforced blast-radius limit independent of how much the model is trusted to ask permission |

The crucial axis Codex owns in this study is **OS-level sandbox enforcement decoupled from human-approval policy** — no other entry in this study compiles a real platform sandbox profile (Seatbelt `.sbpl`, bubblewrap + seccomp-bpf, Windows restricted tokens) and then treats "should a human be asked" as a completely separate, independently configurable concern.

## One-line summary

> Codex is a Rust-native coding-agent CLI where sandboxing is enforced by the operating system itself — a compiled Seatbelt `.sbpl` policy shelled through `/usr/bin/sandbox-exec` on macOS, bubblewrap namespaces plus a seccomp-bpf network filter on Linux, restricted tokens on Windows — completely decoupled from a separate `AskForApproval` policy (`UnlessTrusted`/`OnRequest`/`Granular`/`Never`) that governs only when a human is interrupted; sessions persist as zstd-cold-compressed append-only JSONL rollouts with a derived, rebuildable SQLite index; project context loads from hierarchical `AGENTS.md` files concatenated root-to-cwd (with an `AGENTS.override.md` escape hatch); durable capabilities load separately via `SKILL.md`; and Codex is a native MCP client (stdio/HTTP/OAuth transports); its MCP server mode was removed in September 2026.
