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
| Extensions system that *is* an MCP client wrapper; session/recipe YAML | [goose](./goose) |
| Git-native, repo-map + diff edits; plain-file state (chat history, git commits as trace) | [aider](./aider) |
| Explicit event-stream/trajectory logging; multi-agent delegation (CodeActAgent + micro-agents) | [OpenHands](./OpenHands) |
| The reference MCP client/server implementation — tool discovery, capability negotiation at the protocol level | [mcp-python-sdk](./mcp-python-sdk) |
| IDE-first, not CLI-first; `.continue/` YAML config-loading per project | [continue](./continue) |
| Multi-agent orchestration exemplar; conversation-state persistence, group-chat manager | [autogen](./autogen) |
| VS Code extension harness; `.clinerules/` + documented "Memory Bank" markdown convention persisted across sessions, first-class MCP config UI | [cline](./cline) |
| First-party frontier-lab terminal harness (Rust rewrite); OS-level sandbox policies instead of prompt-level permission gates, own AGENTS.md/skills convention | [codex](./codex) |
| TypeScript monorepo (`pi-ai`/`pi-agent-core`/`pi-coding-agent`/`pi-tui`); unified multi-provider LLM API, session rewind/branching, subagents, MCP adapter as an extension package, own AGENTS.md | [pi](./pi) |

## Sub-inquiries driving this reading pass

Concrete questions from the parent plan (Phase 2), not a general survey:

1. How does `opencode`/`goose` decide which MCP servers + tools are in
   scope for a given session — is there a live enable/disable model we
   can copy for "swap context"?
2. How does `OpenHands`'s event-stream/trajectory log map onto a
   cross-product trace/audit log?
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
