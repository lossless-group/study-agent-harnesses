# Profiles

* [Aider — Profile](Profile__Aider.md) - Aider has no tool-call loop and no MCP client — not a thin one, none at all. It reads diffs straight out of plain chat text.
* [AutoGen — Profile](Profile__AutoGen.md) - Every AutoGen team type overrides exactly one method — `select_speaker()` — and that single seam is the whole difference between them.
* [Cline — Profile](Profile__Cline.md) - "Memory Bank", the cross-session memory pattern Cline is best known for, has zero lines of code behind it.
* [Codex — Profile](Profile__Codex.md) - Codex compiles a real Apple Seatbelt profile and Landlock ruleset: a fully autonomous session that never prompts is still jailed by the OS.
* [Continue — Profile](Profile__Continue.md) - Five differently-scoped assistants can ship side by side in one project and swap mid-session — hot-swapping a skillset per project.
* [Goose — Profile](Profile__Goose.md) - In Goose, "extension" just means "a live MCP client" — and two variants skip the subprocess entirely to run in-process.
* [MCP Python SDK — Profile](Profile__MCP-Python-SDK.md) - The pinned checkout is the unstable half of a rework: v2 swaps `FastMCP`/`ClientSession` out for `MCPServer`/`Client`.
* [OpenCode — Profile](Profile__Opencode.md) - A terminal coding agent with 38 SQLite migrations and real OTLP tracing — built like a production SaaS backend, and no Go anywhere.
* [OpenHands — Profile](Profile__OpenHands.md) - Pin `OpenHands/OpenHands` today and you do not get the agent — the CodeAct loop moved out to a versioned pip dependency.
* [Pi — Profile](Profile__Pi.md) - Pi's docs refuse MCP, sub-agents, permission popups, and plan mode by name — then ship every one as a first-party example extension.
