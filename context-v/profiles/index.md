# Profiles

* [Aider — Profile](Profile__Aider.md) - Aider has no tool-call loop and no MCP client — not a thin one, none at all. It reads diffs straight out of plain chat text.
* [AutoGen — Profile](Profile__AutoGen.md) - Every AutoGen team type overrides exactly one method — `select_speaker()` — and that single seam is the whole difference between them.
* [Cline — Profile](Profile__Cline.md) - "Memory Bank", the cross-session memory pattern Cline is best known for, has zero lines of code behind it.
* [Codex — Profile](Profile__Codex.md) - Codex compiles a real Apple Seatbelt profile and a bubblewrap-plus-seccomp jail: a fully autonomous session that never prompts is still jailed by the OS.
* [Continue — Profile](Profile__Continue.md) - Five differently-scoped assistants can ship side by side in one project and swap mid-session — hot-swapping a skillset per project.
* [Goose — Profile](Profile__Goose.md) - In Goose, "extension" just means "a live MCP client" — and two variants skip the subprocess entirely to run in-process.
* [MCP Python SDK — Profile](Profile__MCP-Python-SDK.md) - v2 is now the stable line: `MCPServer`/`Client` replace `FastMCP`/`ClientSession`, and a new Skills extension serves agent skills over MCP.
* [OpenCode — Profile](Profile__Opencode.md) - A terminal coding agent with 64 SQLite migrations and real OTLP tracing — built like a production SaaS backend, and still no Go anywhere.
* [OpenHands — Profile](Profile__OpenHands.md) - Pin `OpenHands/OpenHands` today and you get a React frontend only: the agent and its server now live in other repos.
* [Pi — Profile](Profile__Pi.md) - Pi still refuses sub-agents and plan mode by name and ships them as examples, but MCP went from refused to built in at v0.99.
* [Tome — Profile](Profile__Tome.md) - Tome is not a harness. It writes the config for seventeen of them, then serves every skill over MCP so the agent loads one at a time.
