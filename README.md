# Agent Harnesses — Study

## The question

> How do coding-agent / conversational-agent runtimes organize and write
> to files on disk, persist memory across sessions, log traces of tool
> calls and reasoning, decide which tools/MCP servers are in scope at a
> given moment, and load or evolve skills?

This study exists in service of a concrete decision: whether to extend
`memopop-ai/apps/memopop-native/` (a working Tauri 2 + SvelteKit shell)
into a multi-context native chat app that swaps between augment-it,
dididecks-ai, and memopop-ai backends at runtime, dynamically loading and
unloading skills and MCP servers per context. See
`ai-labs/context-v/plans/Study-Agent-Harnesses-and-Conversational-UI-Before-Cross-Product-Shell.md`
for the full plan this study serves.

This is not memory-store internals (that's
[[../memory-layers-for-agents]]) and not file-format specs like
`AGENTS.md`/`SKILL.md`/the MCP spec itself (that's
[[../open-specs-and-standards]]). This study is about **running harness
behavior** — what a harness process actually does with the filesystem,
the session store, and the tool registry while it's alive.

## Pins last refreshed 2026-11-23

Every submodule was moved to the head of its upstream default branch on
2026-11-23. Each profile in `context-v/profiles/` has a "Changes since the
previous pin" section with the details.

| Repo | Old → new commit date | Commits | Latest release at refresh |
|---|---|---|---|
| aider | 2026-05-22 → 2026-05-22 | 0 | `v0.86.0` (2025-08-09) |
| autogen | 2026-04-06 → 2026-11-12 | 3 | `python-v0.7.5` (2025-09-29); repo archived |
| cline | 2026-07-13 → 2026-11-20 | 1,278 | `v4.1.27` (2026-11-20) |
| codex | 2026-07-13 → 2026-11-23 | 5,896 | `rust-v0.171.0` (2026-11-19) |
| continue | 2026-06-18 → 2026-07-20 | 3 | none in range; `main` last tagged `v1.3.40-vscode` (2026-06-15) |
| goose | 2026-07-13 → 2026-11-23 | 1,062 | `v1.60.2` (2026-11-20) |
| mcp-python-sdk | 2026-07-10 → 2026-11-05 | 132 | `v2.3.0` (2026-10-02) |
| opencode | 2026-07-13 → 2026-11-23 | 8,240 | `v2.3.2` (2026-11-18) |
| OpenHands | 2026-07-13 → 2026-11-22 | 2,145 | `v1.43.0` (2026-11-20) |
| pi | 2026-07-13 → 2026-11-22 | 3,236 | `v1.6.0` (2026-11-20) |
| tome | 2026-07-16 → 2026-07-16 | 0 | `v0.7.18` (2026-07-10) |

What changed for this study: AutoGen is archived; OpenHands is now a
frontend only; Pi built MCP into its core; Codex dropped its MCP server
mode and moved Linux sandboxing to bubblewrap + seccomp; Goose extracted
its agent loop into `oi-kit`; OpenCode deleted `packages/opencode/` and
`CONTEXT.md`; the MCP Python SDK shipped stable v2 and added a Skills
extension.

## What we are looking at, repo by repo

Working checklist per entry:

- **File organization.** What does the harness write to disk, and where —
  session transcripts, working-directory scratch files, config?
- **Memory across sessions.** Does it persist any state between runs, and
  in what shape?
- **Tracing.** Is there an explicit event/trajectory log of tool calls and
  reasoning steps? Plain files, structured JSON, a DB?
- **Tool/MCP scoping.** How does the harness decide which tools or MCP
  servers are available for a given session or project — static config,
  live enable/disable, per-directory discovery?
- **Skill/extension loading.** Is there a "skills" or "extensions" or
  "rules" concept, and how does it get loaded, scoped, or evolved?

## The design space at a glance

| Bet | Entry |
|---|---|
| CLI-first harness; readable session storage, provider/tool registry, own MCP client | [opencode](./opencode) |
| Extensions system that *is* an MCP client wrapper; session/recipe YAML. Since mid-2026 the agent loop is a state machine of operations in the repo's own `oi-kit` crates | [goose](./goose) |
| Git-native, repo-map + diff edits; plain-file state (chat history, git commits as trace) | [aider](./aider) |
| Agent Canvas: a control-plane frontend (React + Electron) that runs OpenHands, Claude Code, Codex, Pi, OpenCode, or any ACP agent against separately run Agent Servers. The event stream, MCP, and agent loop now live in `OpenHands/software-agent-sdk`, not here (the Python app-server was deleted 2026-07-27) | [OpenHands](./OpenHands) |
| The reference MCP client/server implementation — tool discovery, capability negotiation at the protocol level. v2 is stable since 2026-07-28, and `main` adds a Skills extension (SEP-2640) for serving skills over MCP | [mcp-python-sdk](./mcp-python-sdk) |
| IDE-first, not CLI-first; `.continue/` YAML config-loading per project. Upstream README declares the repo read-only; no commits since 2026-07-20 | [continue](./continue) |
| Multi-agent orchestration exemplar; conversation-state persistence, group-chat manager. **Archived** by Microsoft (November 2026); reference only, successor is Microsoft Agent Framework | [autogen](./autogen) |
| VS Code extension + CLI + desktop harness on a shared `@cline/core` SDK; `.clinerules/` (moving to `.cline/rules`) + documented "Memory Bank" markdown convention persisted across sessions, first-class MCP config UI, MCP deliberately machine-global | [cline](./cline) |
| First-party frontier-lab terminal harness (Rust rewrite); OS-level sandbox policies (Seatbelt; bubblewrap + seccomp) instead of prompt-level permission gates, own AGENTS.md/skills convention; MCP client only since the `mcp-server` mode was removed in September 2026 | [codex](./codex) |
| TypeScript npm monorepo (`pi-ai`/`pi-agent-core`/`pi-coding-agent`/`pi-tui`, plus `pi-mcp`, `pi-durable`, `pi-telemetry` and others at v1.x); unified multi-provider LLM API, session rewind/branching, subagents and plan mode as example extensions, MCP built in since v0.99 on its own `pi-mcp` client, own AGENTS.md | [pi](./pi) |
| Not a harness but a layer above 17 of them: one local catalog of skills, commands, agents and hooks, wired into each harness's native MCP config and rules file, then served over MCP so the agent searches and loads one skill at a time, by tier. Closest existing tool to context-v.dev's skill layer. Shallow pin; fetch with `git submodule update --init --depth 1 tome` | [tome](./tome) |

## Candidates — surfaced, not yet pinned

Tools worth a submodule pin but not yet walked in. Listed here so the
consideration is on record before the pin decision is made.

- **[youtu-agent](https://github.com/TencentCloudADP/youtu-agent)**
  (`TencentCloudADP/youtu-agent`, Apache-2.0 *expected — LICENSE not yet
  confirmed against the in-repo file*, 4.6k★) — Python 3.12+ agent
  framework/runtime built on the OpenAI Agents SDK; open-weight-model-first
  (DeepSeek, GPT-OSS), Docker-deployable, YAML-based agent configuration.
  Fits the study's checklist on **tool/MCP scoping** (YAML agent config,
  MCP support, plus meta-agents that *auto-generate* tools) and especially
  **skill/extension loading — the *evolve* half**: "Training-Free GRPO"
  experience-based learning + meta-agent tool generation are the closest
  thing on this list to a **self-improving harness**, which is the
  thinnest-covered clause ("load or evolve skills") in the design-space
  table today — every pinned entry does static/loaded skills, none does
  self-improvement. Within the table its nearest kin is the **autogen**
  row: a framework for *building* multi-agent systems, not a coding-CLI
  harness (opencode/aider/codex). Also ships benchmark eval (WebWalkerQA,
  GAIA) and an RL training pipeline — signals it's oriented toward agent
  *performance research* as much as day-to-day harness use. Caveat before
  pinning: reconcile the license against the in-repo LICENSE (same
  discipline already applied to onyx/zed elsewhere in these studies).

## Sub-inquiries driving this reading pass

Concrete questions from the parent plan (Phase 2), not a general survey:

1. How does `opencode`/`goose` decide which MCP servers + tools are in
   scope for a given session — is there a live enable/disable model we
   can copy for "swap context"?
2. How does `OpenHands`'s event-stream/trajectory log map onto a
   cross-product trace/audit log? (As of the 2026-11-23 pin that code is
   no longer in the pinned repo; it lives in `OpenHands/software-agent-sdk`,
   which is not pinned here.)
3. How does `continue`'s `.continue/` config loading handle per-project
   skill/rule swaps — the closest existing analog to swapping skillsets
   when switching between augment-it/dididecks/memopop?

Notes go in `context-v/inquiry/`, cited by path
(`studies/agent-harnesses/<repo>/<file>:<line>`), not paraphrased as prose.

## Excluded (verified, not just assumed)

- **Claude Code**, **Cursor / Cursor CLI** — closed source, not
  inspectable.
- **smol-developer** — effectively unmaintained.
- **Roo Code** (`RooCodeInc/Roo-Code`) — confirmed archived via the GitHub
  API (`archived: true`); also a Cline fork, redundant with `cline`.
- **Zed** (`zed-industries/zed`) — agent panel is real and inspectable, but
  the license classifier returns `NOASSERTION` (AGPL-3.0 core mixed with
  proprietary crates/branding assets) — flagged for a licensing check
  before pinning, not included here.

## Related

- `ai-labs/context-v/plans/Study-Agent-Harnesses-and-Conversational-UI-Before-Cross-Product-Shell.md`
  — the plan this study serves
- [[../conversational-ui-and-native-shells]] — the sibling study for
  frontend/native-shell architecture
- [[../memory-layers-for-agents]] — memory-store internals (not
  duplicated here)
- [[../open-specs-and-standards]] — file-format specs incl. the MCP spec
  itself (not duplicated here)
