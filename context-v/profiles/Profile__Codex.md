---
name: Codex Profile
slug: codex
upstream: https://github.com/openai/codex
package: n/a (Rust crates under codex-rs/, published as @openai/codex CLI + platform binaries)
license: Apache-2.0
maintainer: OpenAI
study: studies/agent-harnesses
profile_path: studies/agent-harnesses/codex
profile_kind: Rust terminal coding-agent CLI, OS-level sandbox enforcement, MCP client+server
date_created: 2026-07-13
---

# Codex — Profile

A profile of Codex as it lives in this study (`studies/agent-harnesses/codex/`). Cites pinned paths so you can jump to source rather than trust paraphrase. Read alongside [`Profile__Goose.md`](./Profile__Goose.md) and the `opencode` profile — where Goose gates tool calls with a three-tier `AlwaysAllow`/`AskBefore`/`NeverAllow` permission layer and opencode uses wildcard permission rules, **Codex enforces at the operating-system level**: real Seatbelt policies on macOS, Landlock LSM + seccomp-bpf on Linux, restricted tokens on Windows — with a *separate*, orthogonal human-approval policy layered on top. `LICENSE:1-3` confirms Apache-2.0. The crate tree lives at `codex-rs/` (workspace of 90+ crates, Bazel + Cargo dual-built per `BUILD.bazel`/`MODULE.bazel`), with a legacy `codex-cli/` (the original TypeScript CLI, now superseded).

## TL;DR

The README states the identity plainly (`README.md:1`):

> **Codex CLI** is a coding agent from OpenAI that runs locally on your computer.

Mechanically: **sandboxing and approval are two independent axes, not one gate.** `SandboxPolicy` (`codex-rs/protocol/src/protocol.rs:1001-1040+`) is a closed enum — `DangerFullAccess`, `ReadOnly { network_access }`, `ExternalSandbox { network_access }`, `WorkspaceWrite { writable_roots, network_access, .. }` — describing *what the OS will physically allow the child process to do*, regardless of whether a human was ever asked. `AskForApproval` (`protocol.rs:914-938`) is a completely separate enum — `UnlessTrusted`, `OnRequest`, `Granular(GranularApprovalConfig)`, `Never` — describing *when a human is interrupted for a yes/no*. A command can be sandboxed and never trigger a prompt, or prompted-for and still run inside a Seatbelt/Landlock jail if approved. This is a materially different shape than a single `AlwaysAllow`/`AskBefore`/`NeverAllow` tri-state per tool.

The OS enforcement is real, not simulated. On macOS, `create_seatbelt_command_args()` (`codex-rs/sandboxing/src/seatbelt.rs:623-771`) compiles a live Apple Sandbox (`.sbpl`) profile — starting from `(deny default)` (`seatbelt_base_policy.sbpl:8`, explicitly modeled on Chromium's own sandbox policy per the file's header comment) — and shells out to `/usr/bin/sandbox-exec -p <policy>` (`MACOS_PATH_TO_SEATBELT_EXECUTABLE`, `seatbelt.rs:30`, hardcoded to `/usr/bin` specifically to defeat a PATH-injection attack, per the comment at `seatbelt.rs:26-30`). On Linux, `install_sandbox_on_current_thread()` (`codex-rs/linux-sandbox/src/landlock.rs`) calls the real `landlock` crate's `Ruleset` API (`Ruleset::default()...restrict_self()`, `landlock.rs:144-156`) to install kernel Landlock filesystem rules, plus a `seccompiler`-built BPF filter to block network syscalls outright when network access is disabled — this is kernel-enforced, not a userspace check the model could talk its way around. Windows gets its own `windows-sandbox-rs` crate using restricted tokens.

Sessions persist as append-only JSONL "rollouts" — `codex-rs/rollout/src/recorder.rs:1` states the purpose directly: *"Persist Codex session rollouts (.jsonl) so sessions can be replayed or inspected later."* Cold rollouts get transparently zstd-compressed to `.jsonl.zst` (`compression.rs:43-75`), and a separate SQLite-backed `codex_state::StateRuntime` (`codex-rs/rollout/src/state_db.rs:26-58`) indexes/backfills metadata from those files for fast listing — the JSONL is the durable source of truth; SQLite is a derived index, the inverse of Goose's SQLite-primary/JSONL-legacy relationship.

Project instructions load from hierarchical `AGENTS.md` files: `codex-rs/core/src/agents_md.rs:1-16` documents the algorithm — walk up from cwd to the project root (marked by `.git` by default), then concatenate every `AGENTS.md` found from root down to cwd, inclusive, joined by a literal `--- project-doc ---` separator (`AGENTS_MD_SEPARATOR`, `agents_md.rs:44`) — plus an `AGENTS.override.md` local-override filename that takes precedence (`LOCAL_AGENTS_MD_FILENAME`, `agents_md.rs:40`). A separate `core-skills` crate (`codex-rs/core-skills/src/loader.rs`) discovers durable, explicitly-invoked capabilities via `SKILL.md`-shaped frontmatter (`name`, `description`), distinct from the always-on `AGENTS.md` context.

MCP is bidirectional and fully native: Codex is both an MCP **client** (`codex-rs/rmcp-client/` — stdio, streamable-HTTP, and OAuth-backed transports, `StdioServerLauncher`/`ExecutorStdioServerLauncher`, `rmcp-client/src/lib.rs:13-47`; `codex-rs/codex-mcp/` layers connection management, tool-call catalogs, and elicitation on top) and an MCP **server** (`codex-rs/mcp-server/` — exposes Codex itself as a tool other MCP hosts can call, `codex_tool_runner.rs`, `exec_approval.rs`, `patch_approval.rs`).

If you want one sentence: **Codex is a Rust harness where OS-level sandboxing (`SandboxPolicy`: real Seatbelt `.sbpl` compilation on macOS via `sandbox-exec`, Landlock+seccomp-bpf kernel enforcement on Linux, restricted tokens on Windows) is a completely separate, orthogonal axis from human-approval policy (`AskForApproval`: UnlessTrusted/OnRequest/Granular/Never), sessions persist as zstd-cold-compressed append-only JSONL "rollouts" backed by a derived SQLite index, project context loads from hierarchical `AGENTS.md` files concatenated root-to-cwd with an override filename, durable capabilities load separately as `SKILL.md`, and Codex is simultaneously a native MCP client (stdio/HTTP/OAuth) and an MCP server exposing itself as a callable tool.**

## Why this exists — sandbox as OS enforcement, not a prompt-layer gate

Three load-bearing design choices separate Codex's sandbox from a permission-gate model like Goose's or opencode's:

1. **The policy is compiled to a real OS access-control mechanism per platform, not interpreted by application code at call time.** `create_seatbelt_command_args()` (`seatbelt.rs:623-771`) builds an actual Seatbelt profile string — `(allow file-read* (subpath (param "READABLE_ROOT_0")))`-style S-expressions — and passes it to the kernel via `sandbox-exec -p`. `install_filesystem_landlock_rules_on_current_thread()` (`codex-rs/linux-sandbox/src/landlock.rs:137-156`) calls `landlock::Ruleset::default().add_rules(landlock::path_beneath_rules(&["/"], access_ro))...restrict_self()` — this is a Linux Security Module rule installed in-kernel via `landlock_add_rule`/`landlock_restrict_self` syscalls. Once installed, no amount of clever tool-calling by the model can escape it; the enforcement point is the kernel, not the harness's own dispatch logic.
2. **Filesystem grants are hierarchical roots with carve-outs, not a flat allow/deny per path.** `SeatbeltAccessRoot { root, excluded_subpaths, protected_metadata_names }` (`seatbelt.rs:345-350`) lets a writable root (say, the project cwd) exclude specific subpaths (e.g., a `.git` metadata directory) even though it's nested inside an otherwise-writable tree — `protected_metadata_names_for_writable_root()` (`seatbelt.rs:423-439`) walks `PROTECTED_METADATA_PATH_NAMES` to auto-derive these exclusions. `SandboxPolicy::WorkspaceWrite { writable_roots, network_access, .. }` (`protocol.rs:1026-1040`) is the config-level shape this compiles from.
3. **Network is denied by default and only reopened through an explicit, narrow policy — including a managed-proxy mode.** `dynamic_network_policy_for_network()` (`seatbelt.rs:274-336`) fails closed: if a managed network proxy is configured but no valid loopback endpoint can be resolved, it returns an empty policy rather than silently widening access (`seatbelt.rs:312-322`, comment: *"Fail closed to avoid silently widening network access in proxy-enforced sessions"*). On Linux, disabling network installs an actual seccomp-bpf filter blocking network syscalls (`network_seccomp_mode`, `landlock.rs:96-119`) — not an application-level check the model's shell command could bypass by calling a syscall directly.

## The approval layer — orthogonal to the sandbox, not a substitute for it

`AskForApproval` (`codex-rs/protocol/src/protocol.rs:914-938`) governs *whether a human is interrupted*, independent of what the OS sandbox already prevents:

- `UnlessTrusted` — only commands `is_safe_command()` classifies as read-only are auto-approved; everything else prompts.
- `OnRequest` (the default, `#[default]`, `protocol.rs:924`) — the model itself decides when to ask.
- `Granular(GranularApprovalConfig)` (`protocol.rs:927-933`, fields at `940-952`) — per-category booleans: `sandbox_approval` (escalation requests), `rules` (execpolicy `prompt` rule hits), `skill_approval` (skill script execution), `request_permissions` (an explicit `request_permissions` tool call). Each can be independently allowed or auto-rejected.
- `Never` — failures go straight back to the model, no human escalation at all.

Because this is separate from `SandboxPolicy`, the two compose: a session can run `WorkspaceWrite` sandboxing with `Never` approval (fully autonomous but still OS-jailed), or `DangerFullAccess` with `OnRequest` (unsandboxed but human-gated per command) — four independent quadrants rather than one linear trust dial. `codex-rs/execpolicy/` adds a further layer of declarative per-command policy (`rules` in `GranularApprovalConfig`) evaluated before a command reaches either the sandbox or the approval prompt.

## Session persistence — JSONL rollouts, SQLite as a derived index

`codex-rs/rollout/src/recorder.rs:1` is explicit about the design: *"Persist Codex session rollouts (.jsonl) so sessions can be replayed or inspected later."*

- `SESSIONS_SUBDIR = "sessions"` / `ARCHIVED_SESSIONS_SUBDIR = "archived_sessions"` (`rollout/src/lib.rs:23-24`) under the Codex home directory (`codex-home` crate).
- Files are plain `.jsonl` while "hot," transparently compressed to `.jsonl.zst` once cold (`compression.rs:43,60-75`, `compress_rollout_if_cold_blocking`), and readers (`open_rollout_line_reader`, `compression.rs:47`) handle both transparently — `existing_rollout_path()` (`compression.rs:900-913`) prefers the plain file over its compressed sibling if both exist.
- `codex_state::StateRuntime` (`state_db.rs:26-27`, `StateDbHandle = Arc<StateRuntime>`) is a **SQLite-backed index built from the JSONL files**, not the primary store — `init()`/`try_init_with_roots()` (`state_db.rs:43-70`) opens the runtime and runs `backfill_sessions()` (`metadata.rs:139-153`) to catch up the index against `sessions/` and `archived_sessions/` on disk. This is the inverse of Goose's relationship (SQLite primary, JSONL legacy-import-only): here the flat file is durable truth and the database is a rebuildable cache.
- `session_index.rs`, `list.rs` (`ThreadListConfig`, `ThreadSortKey`, cursor-based pagination) support listing/searching threads without loading full rollouts.

## AGENTS.md and skills — hierarchical concatenation vs. explicit invocation

`codex-rs/core/src/agents_md.rs:1-16` documents the discovery algorithm inline:

1. Walk upward from cwd until a `project_root_markers` entry is found (default: `.git`); an empty marker list disables the walk entirely.
2. Collect every `AGENTS.md` from the project root down to cwd, **inclusive**, and concatenate in that order.
3. Never walk past the project root.

`DEFAULT_AGENTS_MD_FILENAME = "AGENTS.md"` and `LOCAL_AGENTS_MD_FILENAME = "AGENTS.override.md"` (`agents_md.rs:38,40`) — the override file, when present, supersedes the discovered chain; multiple contributing docs (user-level + project-level) join via the literal `AGENTS_MD_SEPARATOR = "\n\n--- project-doc ---\n\n"` (`agents_md.rs:44`). This very checkout's own `AGENTS.md` at the repo root is a live instance, and it explicitly instructs agents never to touch `CODEX_SANDBOX_NETWORK_DISABLED_ENV_VAR`/`CODEX_SANDBOX_ENV_VAR` because *"you operate in a sandbox where `CODEX_SANDBOX_NETWORK_DISABLED=1` will be set whenever you use the shell tool"* and *"`CODEX_SANDBOX=seatbelt` will be set on the child process"* (`AGENTS.md:8-10`) — the sandbox's existence is baked into the repo's own dev instructions to agents working on Codex itself.

Skills are a separate mechanism (`codex-rs/core-skills/src/loader.rs`): `SkillFrontmatter { name: Option<String>, description: Option<String> }` (`loader.rs:55-60`) parsed from discovered skill files, walked via `discover_skills`/`SkillDiscoveryOptions` (`discovery` submodule) with a `MAX_CONCURRENT_SKILL_LOADS` bound and symlink/hidden-directory policies — explicitly invoked capabilities, not ambient context like `AGENTS.md`.

## MCP — client and server, in the same binary

Codex owns both directions of the MCP relationship:

- **As a client**: `codex-rs/rmcp-client/` provides transports — `StdioServerLauncher`/`ExecutorStdioServerLauncher`/`LocalStdioServerLauncher` (`lib.rs:45-47`) for subprocess MCP servers, `http_client_adapter.rs`/`oauth_http_client.rs`/`oauth.rs` for remote HTTP MCP servers with OAuth, `in_process_transport.rs` for in-process wiring, `executor_process_transport.rs` for sandboxed-executor-mediated process transport. `codex-rs/codex-mcp/` layers a `connection_manager.rs` (mutation of live tool sets and tool calls — the repo's own `AGENTS.md:35` tells contributors to route MCP tool-call changes through this file specifically), a `catalog.rs` of available tools, and `elicitation.rs`/`auth_elicitation.rs` for interactive auth flows.
- **As a server**: `codex-rs/mcp-server/` (`main.rs`, `message_processor.rs`, `codex_tool_runner.rs`) exposes Codex itself as an MCP tool — `exec_approval.rs` and `patch_approval.rs` are the approval-flow hooks surfaced to whatever external MCP host is driving Codex this way.

## What's inside this submodule

| Path | What's there |
|---|---|
| `codex-rs/sandboxing/src/seatbelt.rs` | macOS Seatbelt `.sbpl` policy compiler; `create_seatbelt_command_args`, `sandbox-exec` invocation |
| `codex-rs/sandboxing/src/seatbelt_base_policy.sbpl` | The actual Chrome-inspired base policy, `(deny default)` |
| `codex-rs/sandboxing/src/landlock.rs`, `manager.rs` | Linux/Windows CLI-arg construction, `SandboxType` dispatch, `get_platform_sandbox` |
| `codex-rs/linux-sandbox/src/landlock.rs` | Real kernel Landlock `Ruleset` + seccomp-bpf filter installation |
| `codex-rs/linux-sandbox/src/bwrap.rs` | Bubblewrap-based alternative/legacy Linux sandbox path |
| `codex-rs/windows-sandbox-rs/` | Windows restricted-token sandbox implementation |
| `codex-rs/protocol/src/protocol.rs` | `SandboxPolicy` and `AskForApproval` enums — the two orthogonal axes |
| `codex-rs/execpolicy/` | Declarative per-command policy layer (rules that can trigger `Granular` prompts) |
| `codex-rs/rollout/src/recorder.rs`, `compression.rs` | JSONL rollout writer, zstd cold-compression |
| `codex-rs/rollout/src/state_db.rs`, `metadata.rs` | SQLite-backed derived index, backfill-from-JSONL |
| `codex-rs/core/src/agents_md.rs` | `AGENTS.md` hierarchical discovery + concatenation, `AGENTS.override.md` |
| `codex-rs/core-skills/src/loader.rs` | `SKILL.md`-shaped skill discovery, separate from `AGENTS.md` |
| `codex-rs/rmcp-client/`, `codex-rs/codex-mcp/` | MCP client: stdio/HTTP/OAuth transports, connection manager, catalog |
| `codex-rs/mcp-server/` | MCP server: exposes Codex itself as a tool, approval hooks |
| `codex-rs/tui/` | Terminal UI |
| `codex-rs/exec/`, `codex-rs/exec-server/` | Non-interactive `codex exec` mode; sandboxed executor process/filesystem |
| `codex-cli/` | Legacy TypeScript CLI, superseded by `codex-rs` |
| `sdk/` | Language SDKs (e.g. `sdk/python/src/openai_codex/_sandbox.py`) |
| `AGENTS.md` | Codex's own contributor instructions, references its own sandbox env vars |
| `docs/sandbox.md` | Points to hosted docs (`developers.openai.com/codex/security`) rather than in-repo detail |

If you read three files: `codex-rs/sandboxing/src/seatbelt.rs` (the whole macOS policy-compilation shape), `codex-rs/linux-sandbox/src/landlock.rs` (real kernel Landlock+seccomp installation), and `codex-rs/protocol/src/protocol.rs` around `SandboxPolicy`/`AskForApproval` (the two-axis model).

## Mental model for using it well

- **Think in two independent dials, not one trust slider**: `SandboxPolicy` (what the OS physically permits) and `AskForApproval` (when a human is asked). Configure them separately; a fully-sandboxed session can still run with zero prompts, and an unsandboxed session can still prompt for everything.
- **Trust the sandbox as a hard boundary, not a suggestion** — Seatbelt/Landlock/seccomp are kernel/OS mechanisms; a model cannot argue its way past a `(deny default)` policy or a Landlock ruleset the way it might talk past a prompt-layer check.
- **Use `WorkspaceWrite`'s `writable_roots` + exclusions for surgical scoping** rather than falling back to `DangerFullAccess` — the seatbelt builder auto-protects metadata directories like `.git` even inside writable roots.
- **Read JSONL rollouts directly for ground truth**; treat the SQLite state index as a rebuildable cache for fast listing, not the record of note.
- **Keep `AGENTS.md` files small and hierarchical** — the loader concatenates every one from project root to cwd, so redundant restatement at multiple levels bloats every prompt.
- **Reach for `SKILL.md` only for durable, explicitly-invoked capabilities** — it's a distinct mechanism from the always-loaded `AGENTS.md` context.

## When NOT to reach for this

- **You need a lightweight single-file CLI with no OS-level dependencies.** The sandbox stack pulls in `landlock`, `seccompiler`, macOS's `sandbox-exec`, and a Windows restricted-token implementation — real platform-specific security infrastructure, not a config toggle you can ignore.
- **You want prompt-level, tool-by-tool allow/deny with no kernel involvement** (Goose's `AlwaysAllow`/`AskBefore`/`NeverAllow`, opencode's wildcard rules). Codex's sandbox operates a layer below the model entirely; if your threat model is "don't let the model see this tool," a permission gate is simpler and sufficient — Codex's mechanism answers "don't let this *process* touch this file/socket even if it tries," a stronger but heavier guarantee.
- **You're on a platform without a working sandbox backend** and need graceful degradation — `SandboxPolicy::ExternalSandbox` exists precisely for "already sandboxed by something else, don't double-enforce," but that still assumes an external sandbox exists.
- **You want one unified session store.** The JSONL-plus-derived-SQLite split is deliberate but is two files to reason about (a durable log and a rebuildable index) rather than a single database of record.

## How this compares to the rest of the study

| Axis | opencode | Goose | Codex |
|---|---|---|---|
| **Shape** | CLI-first, own MCP client, TS/Go | Rust CLI + Electron desktop + API/SDK, MCP-native | Rust CLI (+ TUI, exec-server, app-server), MCP-native both directions |
| **Permission/sandbox model** | Wildcard permission rules (prompt-layer) | Three-tier `AlwaysAllow`/`AskBefore`/`NeverAllow` per tool (prompt-layer) | **Two orthogonal axes**: `SandboxPolicy` (OS-kernel enforcement: Seatbelt/Landlock+seccomp/Windows tokens) × `AskForApproval` (human-prompt policy, independently configurable) |
| **Enforcement point** | Application-layer tool dispatch | Application-layer `PermissionManager` gate | **Kernel/OS** — `sandbox-exec` policy, Landlock LSM rules, seccomp-bpf syscall filter |
| **Network control** | Not sandboxed at OS level | Not sandboxed at OS level | Fails closed by default; explicit proxy-only mode; Linux blocks network syscalls via seccomp-bpf |
| **Session/memory storage** | Structured, readable session storage (JSON/DB-backed) | SQLite (WAL, schema v15), migrated *from* per-session `.jsonl` | **JSONL rollouts** (zstd-cold-compressed) as durable truth; SQLite is a **derived, rebuildable index** backfilled from JSONL — inverse of Goose's relationship |
| **Ambient project context** | Not noted | `.goosehints` / `AGENTS.md`, lazy per-subdirectory loading | `AGENTS.md` (+ `AGENTS.override.md`), hierarchical root-to-cwd concatenation, loaded eagerly at session start |
| **Durable capability discovery** | Not noted | `SKILL.md` per agentskills.io spec | `SKILL.md`-shaped frontmatter via dedicated `core-skills` crate, separate loader from `AGENTS.md` |
| **MCP support** | Native — opencode owns its MCP client | Native — extensions **are** the MCP client wrapper (6 transport variants) | Native **both ways** — `rmcp-client`/`codex-mcp` as client (stdio/HTTP/OAuth) and `mcp-server` exposing Codex itself as a tool |
| **Best fit** | Terminal-first agent work needing a swappable MCP toolset | Cross-surface agent work needing an MCP extension ecosystem + observability-grade tracing | Autonomous or semi-autonomous coding work where the operator needs a genuine, kernel-enforced blast-radius limit independent of how much the model is trusted to ask permission |

The crucial axis Codex owns in this study is **OS-level sandbox enforcement decoupled from human-approval policy** — no other entry in this study compiles a real platform sandbox profile (Seatbelt `.sbpl`, Landlock ruleset + seccomp-bpf, Windows restricted tokens) and then treats "should a human be asked" as a completely separate, independently configurable concern.

## One-line summary

> Codex is a Rust-native coding-agent CLI where sandboxing is enforced by the operating system itself — a compiled Seatbelt `.sbpl` policy shelled through `/usr/bin/sandbox-exec` on macOS, a kernel Landlock ruleset plus a seccomp-bpf network filter on Linux, restricted tokens on Windows — completely decoupled from a separate `AskForApproval` policy (`UnlessTrusted`/`OnRequest`/`Granular`/`Never`) that governs only when a human is interrupted; sessions persist as zstd-cold-compressed append-only JSONL rollouts with a derived, rebuildable SQLite index; project context loads from hierarchical `AGENTS.md` files concatenated root-to-cwd (with an `AGENTS.override.md` escape hatch); durable capabilities load separately via `SKILL.md`; and Codex is simultaneously a native MCP client (stdio/HTTP/OAuth transports) and an MCP server exposing itself as a callable tool to other hosts.
