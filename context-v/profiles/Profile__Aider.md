---
name: Aider Profile
slug: aider
upstream: https://github.com/Aider-AI/aider
package: aider-chat (PyPI)
license: Apache-2.0
maintainer: Aider-AI (Paul Gauthier, original author)
study: studies/agent-harnesses
profile_path: studies/agent-harnesses/aider
profile_kind: CLI coding agent (terminal, git-native)
date_created: 2026-07-13
---

# Aider — Profile

A profile of Aider as it lives in this study (`studies/agent-harnesses/aider/`). Cites pinned paths so you can jump to source rather than trust paraphrase. Read alongside [`Profile__Opencode.md`](./Profile__Opencode.md) and [`Profile__Goose.md`](./Profile__Goose.md) — the three form a spectrum from plain-file/git state (Aider) to structured session storage with an owned MCP client (opencode) to an extensions-system that *is* an MCP client wrapper (goose).

## TL;DR

Aider's own docs put it plainly (`aider/website/docs/git.md:9-10`):

> Aider works best with code that is part of a git repo. Aider is tightly integrated with git...

That single sentence is the whole architectural bet. Aider has **no tool-call loop, no MCP client, no structured trajectory log**. It talks to the LLM with plain chat messages, parses the LLM's text response for search/replace blocks or diffs, writes the resulting file contents straight to disk with the OS filesystem API (`aider/coders/editblock_coder.py:66` — `self.io.write_text(full_path, new_content)`), and then **commits the change to git as the unit of state, memory, and trace all at once** (`aider/coders/base_coder.py:2375-2394`, `aider/repo.py:131`).

There is no `mcp`, no `Model Context Protocol`, and no `modelcontextprotocol` string anywhere in the Python source or the docs site — verified by grep across `aider/` and `aider/website/docs/`. Aider predates the MCP-native harness generation and never adopted it; "tool scoping" in Aider means **which files are in the chat context** (`/add`, `/read-only`, `.aiderignore`), not which functions/servers the model may invoke.

Memory across turns is a **plain markdown transcript file**, `.aider.chat.history.md`, appended to on every message (`aider/io.py:271-287`, `aider/io.py:1117-1136`). Memory across *sessions* (i.e., what the codebase currently looks like and why) is **git history itself** — every AI edit is its own commit with a generated Conventional-Commits message and author/committer attribution marking it as `(aider)`-authored (`aider/repo.py:131-320`). Repo-scale context comes from the **repo map**: a tree-sitter-tagged, PageRank-ranked, token-budgeted summary of the whole codebase, cached in a per-project SQLite/diskcache directory (`aider/repomap.py:35-43`, `aider/repomap.py:365-420`).

If you want one sentence: **Aider is a terminal pair-programmer that has no tool-call loop and no MCP client at all — it parses LLM text for diffs, writes files directly, and uses git commits as its combined state/memory/trace mechanism, backed by a cached, PageRank-ranked repo map and a flat markdown chat-history file, with "tool scoping" reduced to which files are `/add`ed to the conversation.**

## Why this exists — diffs and commits instead of tool calls

Three load-bearing design choices separate Aider from every MCP-native harness in this study:

1. **The LLM never calls a function; it writes prose that looks like a diff.** `EditBlockCoder.get_edits()` (`aider/coders/editblock_coder.py:20-32`) parses `self.partial_response_content` — the raw assistant message — for ORIGINAL/UPDATED search-replace blocks. There are a dozen sibling `Coder` subclasses for different edit formats (`aider/coders/`: `editblock_coder.py`, `udiff_coder.py`, `wholefile_coder.py`, `patch_coder.py`, `architect_coder.py`, ...), but all of them end the same way: parse text, then `self.io.write_text()`. The `partial_response_function_call` / `tool_calls` machinery that does exist (`aider/coders/base_coder.py:1549,1705,1850-1852`) is a legacy OpenAI-function-calling *edit format* experiment — it is used to receive a structured edit payload, not to expose a general tool registry to the model.
2. **Git commits are the memory, the undo mechanism, and the audit trail, simultaneously.** `GitRepo.commit()` (`aider/repo.py:131`) is called from `Coder.auto_commit()` after every successful file edit (`aider/coders/base_coder.py:2375-2394`) and from `Coder.dirty_commit()` before an edit touches a file with pre-existing uncommitted changes (`aider/coders/base_coder.py:2411-2423`). `/undo` (`aider/commands.py:553`) is just `git reset` on Aider's own commit. There is no separate "trace log" file — `git log` *is* the trace log.
3. **Context is a cached, ranked map, not a tool the model queries.** The repo map is computed and pushed into the prompt proactively (`aider/website/docs/repomap.md:24-25` — *"Aider sends a repo map to the LLM along with each change request"*), not fetched on demand via a `list_resources`/`read_resource` round-trip the way an MCP resource would be. The model never asks for the map; Aider decides how much of it fits the token budget and includes it unconditionally.

## The chat-history file — flat markdown, not a session store

`.aider.chat.history.md` is computed once per git root (`aider/args.py:274-275`):

```python
default_chat_history_file = (
    os.path.join(git_root, ".aider.chat.history.md") if git_root else ".aider.chat.history.md"
)
```

`InputOutput.__init__` opens it and immediately timestamps a session boundary (`aider/io.py:335-336`):

```python
current_time = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
self.append_chat_history(f"\n# aider chat started at {current_time}\n\n")
```

Every subsequent write goes through `append_chat_history()` (`aider/io.py:1117-1136`), which does formatting (blockquote user input, line-break assistant output) then opens the file in **append mode** and writes:

```python
if self.chat_history_file is not None:
    try:
        self.chat_history_file.parent.mkdir(parents=True, exist_ok=True)
        with self.chat_history_file.open("a", encoding=self.encoding, errors="ignore") as f:
            f.write(text)
    except (PermissionError, OSError) as err:
        print(f"Warning: Unable to write to chat history file {self.chat_history_file}.")
        self.chat_history_file = None  # Disable further attempts to write
```

This is deliberately dumb: one file, append-only, human-readable markdown, no schema, no rotation, no per-turn IDs. `--restore-chat-history` (`aider/main.py:988`, `aider/args.py:290-293`) is the *only* recall mechanism — it re-reads this same file back into the LLM's context on the next launch. There is no separate structured "session" object; the transcript file *is* the session.

A companion `--input-history-file` (`aider/args.py:271-281`, default `.aider.input.history`) stores only the user's raw prompt-toolkit input line-history (for readline-style recall while typing), separate from the LLM-facing transcript.

Context gets pruned, not just appended, when it grows too large: `ChatSummary` (`aider/history.py:7-123`) recursively splits the message list, summarizes the head with the `--weak-model`, and keeps the tail verbatim (`aider/history.py:27-96`), governed by `--max-chat-history-tokens` (`aider/args.py:221-226`, default computed as `min(max(max_input_tokens/16, 1024), 8192)` in `aider/models.py:356-358`). Summarization is a token-budget mechanism, not a "memory" feature — it exists to keep the *live* context window under budget, not to produce durable cross-session memory.

## Git as state, memory, and trace — the `GitRepo.commit()` contract

`GitRepo.commit()` (`aider/repo.py:131-320`) is the single most load-bearing function in the codebase for this study's question. Its docstring spells out an attribution state machine that is unusually explicit for a coding-agent codebase (`aider/repo.py:149-199`):

> Author: The person who originally wrote the code changes. Committer: The person who last applied the commit... `aider_edits=True`: Changes were generated by Aider (LLM). `aider_edits=False`: Commit is user-driven (e.g., `/commit` manually staged changes).

Concretely:

- **Every AI edit is its own commit.** `Coder.auto_commit()` (`aider/coders/base_coder.py:2375-2394`) fires right after `apply_edits()` succeeds, passing `aider_edits=True`. The returned `(commit_hash, commit_message)` is fed back into the *next* prompt via `gpt_prompts.files_content_gpt_edits` — so the model is told its own commit hash as confirmation the edit landed.
- **Pre-existing dirty state is committed first, separately.** `Coder.dirty_commit()` (`aider/coders/base_coder.py:2411-2423`) calls `self.repo.commit(fnames=self.need_commit_before_edits, coder=self)` with `aider_edits` defaulted to `False` — this is the mechanism described in the docs as *"Aider takes special care before editing files that already have uncommitted changes... This keeps your edits separate from aider's edits"* (`aider/website/docs/git.md:20-21`).
- **Attribution is metadata on the commit, not a separate log.** By default, AI-authored commits get `(aider)` appended to git author/committer name fields (`aider/website/docs/git.md:61-64`); `--attribute-co-authored-by` instead appends a `Co-authored-by: aider (<model>) <aider@aider.chat>` trailer (`aider/repo.py:167-168`).
- **Commit messages are LLM-generated**, sent to the `--weak-model` with the diff + chat context (`aider/repo.py:326` `get_commit_message`, `aider/website/docs/git.md:47-50`), defaulting to Conventional Commits style.
- **Pre-commit hooks are skipped by default.** Aider passes `--no-verify` unless `--git-commit-verify` is set (`aider/website/docs/git.md:43`) — a deliberate choice so hook failures don't block the agent's edit loop.
- **`/undo` is `git reset` against Aider's own commit chain**, guarded by `last_aider_commit_hash` tracking (`aider/coders/base_coder.py:2397-2409`, `aider/commands.py:553`) — it refuses to undo a commit it didn't make.

The upshot for this study's question ("is there an explicit trace log?"): **no.** There is no `.jsonl` of tool calls, no OpenTelemetry, no event-stream. `git log --oneline` against Aider's own commits, each carrying a generated message and `(aider)` attribution, *is* the trace — reviewable with ordinary git tooling, diffable, revertible, bisectable.

## The repo map — cached, ranked, token-budgeted

`RepoMap` (`aider/repomap.py:42-88`) builds a **tree-sitter-tagged, PageRank-ranked map of the whole repository** and sends it with every request (`aider/website/docs/repomap.md:24-25`). Mechanically:

- **Tags cache** — a per-project directory `.aider.tags.cache.v{CACHE_VERSION}` (`aider/repomap.py:43`, `CACHE_VERSION = 3` or `4` depending on tree-sitter-language-pack availability, `aider/repomap.py:35-37`), backed by `diskcache.Cache` (SQLite-based) with mtime-keyed entries (`aider/repomap.py:217-262` `load_tags_cache`/`get_tags`). Cache errors (`SQLITE_ERRORS = (sqlite3.OperationalError, sqlite3.DatabaseError, OSError)`, `aider/repomap.py:32`) are caught and the cache is rebuilt in-memory rather than crashing the session (`aider/repomap.py:177-215` `tags_cache_error`).
- **Ranking** — `get_ranked_tags()` (`aider/repomap.py:365-420+`) builds a `networkx` graph where files are nodes and shared-identifier references are edges, then runs personalized PageRank, biasing `personalization` toward files already in the chat or explicitly mentioned (`aider/repomap.py:374-383`). This is how Aider decides which of the "most important classes and functions" (`aider/website/docs/repomap.md:13-15`) make the token-budget cut.
- **Token budgeting is dynamic.** `get_repo_map()` (`aider/repomap.py:103-168`) shrinks or grows the map depending on whether any files are already in the chat — with *no* files in chat, the target budget is `max_map_tokens * map_mul_no_files` (default multiplier 8), because the model needs a wider view of an unfamiliar repo (`aider/repomap.py:120-132`, docs at `aider/website/docs/repomap.md:93-96`). `--map-tokens` (default ~1024) is the base budget (`aider/website/docs/repomap.md:94`).
- **The map is proactive, not on-demand.** It's assembled and injected into every prompt by the coder, not exposed as something the model calls a tool to fetch — the inversion of how an MCP `resources/read` round-trip would work.

This is the mechanism that makes "no tool-call loop" viable at all: instead of letting the model grep/read files as tool calls (the ReAct-loop pattern used by MCP-native and OpenHands-style harnesses), Aider front-loads a compressed whole-repo view so the model can usually reason about unfamiliar code without any exploratory round-trip.

## File / tool scoping — chat membership, not permissions

Aider has no permission or capability-negotiation layer. "Scoping" is entirely about **which files are visible to the model in this turn**:

- `/add <file>` and `/drop <file>` (`aider/commands.py:799,912`) add/remove files from `abs_fnames` — the editable set.
- `/read-only <file>` (`aider/commands.py:1328`, `--read` flag at `aider/args.py:736-739`) adds a file as **context but not editable** — visible to the model, immune to the edit-apply path.
- `.aiderignore` (`aider/args.py:22,422-431`) is a gitignore-syntax file, resolved relative to the git root, that keeps whole paths out of `/add`, out of the repo map, and out of `--subtree-only` scans (`aider/commands.py:812` warns *"Skipping {fname} due to aiderignore or --subtree-only"*).
- The `FileWatcher` (`aider/watch.py:65-90`) layers a second, orthogonal ignore list on top — an internal always-ignore pattern list (`.aider*`, `.git`, editor swap files, `node_modules/`, `.env`, etc., `aider/watch.py:20-56`) plus any `.gitignore` files, used specifically for the `--watch-files` "AI comment" trigger mode.
- **There is no tool registry to scope at all.** No MCP servers, no function allowlist, no per-session capability negotiation. The closest analog to "which tools are available" is `--map-tokens 0` (disables the repo map) or `--no-auto-commits`/`--no-git` (disables the git integration) — coarse global flags, not a live enable/disable surface.

## The `AI!` / `AI?` comment trigger — the closest thing to an ambient interface

`FileWatcher.ai_comment_pattern` (`aider/watch.py:69-71`):

```python
ai_comment_pattern = re.compile(
    r"(?:#|//|--|;+) *(ai\b.*|ai\b.*|.*\bai[?!]?) *$", re.IGNORECASE
)
```

With `--watch-files`, Aider watches the working tree (via `watchfiles`) for source comments ending in `AI!` (request an edit) or `AI?` (ask a question), scoped by the same gitignore/`.aiderignore` rules (`aider/watch.py:15-62`). This is not a tool-call or an MCP resource subscription — it's a filesystem-poll loop that greps changed files for a magic-comment regex and turns a matched comment into a chat message. It is the one place Aider's design brushes up against "ambient triggering," and it still routes through the same parse-diffs-then-commit pipeline as everything else.

## What's inside this submodule

| Path | What's there |
|---|---|
| `aider/coders/base_coder.py` | The core `Coder` loop — send messages, get response, `apply_updates()`, `auto_commit()`/`dirty_commit()`, `/undo` bookkeeping |
| `aider/coders/editblock_coder.py`, `udiff_coder.py`, `wholefile_coder.py`, `patch_coder.py`, ... | Per-edit-format parsers; each turns LLM text into `(path, original, updated)` tuples then writes files directly |
| `aider/repo.py` | `GitRepo` — the `commit()` attribution state machine, dirty-file detection, diffs, ignore rules |
| `aider/repomap.py` | `RepoMap` — tree-sitter tagging, diskcache-backed tags cache, networkx PageRank, token-budgeted map rendering |
| `aider/io.py` | `InputOutput` — terminal I/O, `.aider.chat.history.md` writer (`append_chat_history`), input history, colors/prompts |
| `aider/history.py` | `ChatSummary` — recursive token-budget summarization of in-context messages via the weak model |
| `aider/watch.py`, `aider/watch_prompts.py` | `FileWatcher` — `AI!`/`AI?` comment-trigger mode |
| `aider/commands.py` | All `/`-commands (`/add`, `/drop`, `/commit`, `/undo`, `/diff`, `/lint`, `/test`, `/run`, `/git`, `/map`, `/read-only`, ...) |
| `aider/args.py` | CLI flag definitions — chat-history file, input-history file, `.aiderignore`, `.aider.conf.yml`, `.aider.model.settings.yml`, git-integration toggles |
| `aider/models.py` | Model registry, token counting, `max_chat_history_tokens` default computation |
| `aider/linter.py`, `aider/run_cmd.py` | `/lint` and `/run`/`/test` shell-out integration — the closest Aider gets to "tools," and they're plain subprocess calls, not model-invocable functions |
| `aider/website/docs/` | Jekyll docs source — `git.md`, `repomap.md`, `config.md`, `usage.md`, etc.; no `mcp*` file exists anywhere in this tree |
| `benchmark/` | Aider's own coding-benchmark harness (SWE-bench-style) |
| `tests/basic`, `tests/browser`, `tests/help`, `tests/scrape` | Pytest suite |
| `docker/`, `requirements/` | Container build and pinned dependency sets |

If you read three files: `aider/coders/base_coder.py` (the loop, `apply_updates`/`auto_commit`), `aider/repo.py` (the commit-attribution contract — read the docstring on `commit()` in full), and `aider/repomap.py` (`get_repo_map` → `get_ranked_tags` — the caching + PageRank pipeline).

## Mental model for using it well

- **Think in commits, not sessions.** Every accepted edit is a commit; `git log` and `git diff` are first-class debugging tools for what Aider actually did, not an afterthought.
- **Curate the chat-file set deliberately.** Since there's no tool-call loop for the model to go fetch files itself, what you `/add` (and what the repo map surfaces) *is* the model's whole world for that turn. Under-adding starves it; over-adding blows the token budget the repo map is fighting to stay under.
- **Use `/read-only` for context you don't want mutated.** It's the only scoping primitive between "invisible" and "editable."
- **Let dirty-commit separate your work from Aider's.** Committing your own changes before you start a session, or accepting Aider's automatic pre-edit dirty-commit, keeps `/undo` precise — it only ever reverts Aider's own commit chain.
- **Treat `--restore-chat-history` as the only session-resume lever.** There's no session picker, no session IDs — it's "replay the flat markdown file back into context."
- **The repo map is a budget you tune, not a tool you call.** `--map-tokens` and `--map-mul-no-files` are the levers; there's no equivalent of an MCP `resources/read` the model can invoke mid-turn to expand it.

## When NOT to reach for this

- **You need a general tool-calling / MCP surface.** Aider has zero MCP support and no generalized function-calling loop for external tools — if the shell you're building needs to swap MCP servers per context (this study's driving question), Aider is not a model to copy for that layer at all. Look to opencode's own MCP client or goose's extensions-as-MCP instead.
- **You need structured, machine-parseable trace data.** There's no event-stream JSON, no OTel, nothing queryable beyond `git log`/`git show`. If a downstream system needs to programmatically replay "what tools ran in what order with what args," Aider has nothing to hand it — the trace *is* prose commit messages plus diffs.
- **You need per-session or per-user isolated memory that isn't the shared codebase state.** Aider's "memory" is the repo itself (via git) and one flat chat-history file per git root. There's no per-user or per-conversation memory store separate from the code.
- **You need live tool/permission negotiation mid-session.** Everything is CLI-flag or `/command` driven at session start or via explicit slash-commands; there's no capability-negotiation handshake analogous to MCP's `initialize`.
- **Your repo isn't (and can't be) a git repo.** Nearly everything — undo, dirty-commit separation, attribution, `/diff` — degrades or disables without git (`--no-git` exists but the docs call it "not recommended," `aider/website/docs/git.md:36-42`).

## How this compares to the rest of the study

| Axis | Aider | opencode | goose |
|---|---|---|---|
| **Shape** | CLI, git-native pair-programmer | CLI-first harness, own MCP client | Rust CLI/desktop, extensions system |
| **Tool-calling model** | **None** — LLM emits diff-shaped text, parsed and applied directly | Structured tool-call loop against a provider/tool registry | Extensions *are* MCP client wrappers around tool calls |
| **MCP support** | **None** — no `mcp` string anywhere in source or docs | Native — opencode owns its MCP client | Native — extensions system built on MCP |
| **Session/memory storage** | Flat markdown file (`.aider.chat.history.md`), append-only | Structured, readable session storage (JSON/DB-backed) | Session/recipe YAML |
| **Trace of actions** | **Git commit history** — each AI edit is a commit with generated message + author/committer attribution | Session storage doubles as trace (tool calls recorded per turn) | Extension/tool-call log tied to session state |
| **Tool/context scoping** | File-membership only (`/add`, `/read-only`, `.aiderignore`) — no tool registry to scope | Live enable/disable of tools per session | Per-extension enable/disable, each extension = one MCP surface |
| **Repo/context awareness** | Proactive repo map — tree-sitter tags + PageRank, cached, pushed into every prompt | Provider/tool registry drives context; no repo-map equivalent noted | Extension-driven; context shaped by which extensions are active |
| **Undo mechanism** | `git reset` against Aider's own commit chain | N/A (no commit-per-edit model) | N/A |
| **Best fit** | Git-repo-centric solo/pair coding where commit history as audit trail is valuable | Terminal-first agent work needing a real, swappable MCP toolset | Desktop/CLI agent work built around composable MCP extensions |

The crucial axis Aider owns in this study is **git-commit-as-trace**: no other entry collapses "did the edit happen," "what did it change," "why," and "who/what authored it" into a single git object the way Aider does. That is a genuinely different answer to the study's tracing question than event-stream logging (OpenHands) or structured session storage (opencode, goose) — and it comes at the explicit cost of having no tool-call loop and no MCP client to scope in the first place.

## One-line summary

> Aider has no tool-call loop and no MCP client at all — it parses LLM-generated diffs from plain chat text, writes files straight to disk, and folds state, memory, and trace into one mechanism: every accepted edit becomes its own git commit, with generated Conventional-Commits messages and `(aider)` author/committer attribution, while session memory is a flat append-only `.aider.chat.history.md` and repo-scale context comes from a tree-sitter-tagged, PageRank-ranked, disk-cached repo map pushed proactively into every prompt — making "tool scoping" mean only "which files are `/add`ed to the chat," not which functions or MCP servers the model may invoke.
