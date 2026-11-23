---
name: MCP Python SDK Profile
title: "MCP Python SDK — Profile"
slug: mcp-python-sdk
upstream: https://github.com/modelcontextprotocol/python-sdk
package: mcp (PyPI)
license: MIT
maintainer: modelcontextprotocol org (Linux Foundation); Anthropic, PBC copyright holder per LICENSE
study: studies/agent-harnesses
profile_path: studies/agent-harnesses/mcp-python-sdk
profile_kind: library (client + server SDK)
date_created: 2026-07-13
date_modified: 2026-11-23
authors:
  - Michael Staton
site_uuid: 01e448b7-4400-429f-82d9-0aeb5e08290d
hex_code: 9a9xuz
date_authored_initial_draft: 2026-07-13
date_authored_current_draft: 2026-11-23
lede: >-
  v2 is now the stable line: `MCPServer`/`Client` replace `FastMCP`/`ClientSession`,
  and a new Skills extension serves agent skills over MCP.
summary: >-
  Source-cited profile of the MCP Python SDK submodule, pinned on the stable v2 line (past v2.3.0). Distinct
  from the spec repo profiled under open-specs-and-standards: that one is schema and docs, this one
  is the Python client/server a harness author actually imports. Documents the mechanism this study
  cares about most — dynamic tool discovery as a cached `list_tools()` round-trip, with an automatic
  probe-then-fall-back negotiation between the modern `server/discover` path and the legacy
  `initialize()` handshake — plus automatic JSON Schema generation from type hints, the three
  transports behind one API (including in-process with no framing at all, which makes the best test
  harness in the study), the shielded stdio shutdown escalation worth trusting rather than
  reimplementing, and `ClientSessionGroup` for merging N servers into one namespace. These API
  shapes are v2; code on `mcp<2` uses the older `FastMCP`/`ClientSession` names.
status: Reference
publish: true
type: Profiles
---

# MCP Python SDK — Profile

A profile of the MCP Python SDK as it lives in this study (`studies/agent-harnesses/mcp-python-sdk/`, pinned at commit `77001c96` / 2026-11-05, after the `v2.3.0` release; first profiled at `1216c536` / 2026-07-10). Cites pinned paths so you can jump to source rather than trust paraphrase. This is a different object from `studies/open-specs-and-standards/modelcontextprotocol/` (the wire-protocol spec repo, profiled in [`Profile__Model-Context-Protocol.md`](../../open-specs-and-standards/context-v/profiles/Profile__Model-Context-Protocol.md)) — that repo is the schema and docs; this one is the actual Python client/server implementation that a harness author imports and runs.

## TL;DR

The pinned checkout is **stable v2** (`v2.0.0` shipped 2026-07-28 alongside the 2026-07-28 MCP spec; this pin is past `v2.3.0`, 2026-10-02). At the July pin this was a beta line and the README warned that v1.x was the stable one; that notice is gone, and the README now points at a "What's new in v2" page and a migration guide (`README.md:16-20`). v2 changes the public API shape from the well-known v1 `FastMCP`/`ClientSession` pattern to a new `MCPServer` + `Client` pair. v1.x continues as a maintenance line (`v1.30.0`, 2026-09-05).

> "Notice what you did **not** write: no JSON Schema (`a: int, b: int` _is_ the schema), no request parsing, no validation code, no protocol handling. Two type-hinted Python functions and a docstring." (`README.md:76`)

The full server pitch, in the README's own 15 lines (`README.md:47-63`):

```python
from mcp.server import MCPServer

mcp = MCPServer("Demo")

@mcp.tool()
def add(a: int, b: int) -> int:
    """Add two numbers."""
    return a + b

@mcp.resource("greeting://{name}")
def greeting(name: str) -> str:
    """Greet someone by name."""
    return f"Hello, {name}!"
```

And the client side, in 10 (`README.md:90-103`). At the July pin this example passed the in-process server object; it now leads with a URL (upstream #3315, #3443):

```python
from mcp import Client

async def main() -> None:
    async with Client("http://localhost:8000/mcp") as client:
        result = await client.call_tool("add", {"a": 1, "b": 2})
        print(result.structured_content)  # {'result': 3}
```

The one line worth memorizing: **`Client` connects to a URL, a stdio subprocess (now by passing `StdioServerParameters` directly), a custom `Transport`, or (for tests) directly to a `Server`/`MCPServer` instance in memory with no transport at all** (`README.md:105`, `src/mcp/client/client.py:289-297`) — the exact same `call_tool`/`list_tools`/`list_resources` code path works whether the peer is in-process or a remote HTTP server. This is what makes the SDK's `Client` class the thing a harness author (opencode, goose, or any custom agent loop) actually imports to gain **dynamic tool discovery**: connect, call `list_tools()`, and the tool schemas come back as `Tool` objects with a JSON-Schema `input_schema` derived automatically from the server's type-hinted Python function — no schema is hand-written on either side.

## Why this exists — the SDK's job in the protocol stack

The spec (profiled separately) defines the wire format; this package is **the reference implementation of both halves of that wire format for Python** — build a server that speaks it, or build a client/host that speaks it. `main` is now the stable v2 line and `v1.x` is a maintenance line for critical fixes (upstream #3178, #3212). A harness author choosing "which MCP client do I import in Python" is choosing between staying on `mcp<2` (familiar `ClientSession`, fixes only) or moving to v2 (`Client`, `MCPServer`, a new `server/discover` negotiation path, response caching, extensions).

Three things this package does that a harness author would otherwise have to hand-roll:

1. **Turns a Python function signature into a JSON Schema tool definition.** `func_metadata.py` (`src/mcp/server/mcpserver/utilities/func_metadata.py:75-83`, `:287-370`) builds a Pydantic `arg_model` from the function's type hints via `create_model`, and a `StrictJsonSchema` generator that **raises instead of warning** on non-serializable types — so a tool author gets a loud error at registration time, not a silent bad schema shipped to the model.
2. **Owns the JSON-RPC 2.0 session lifecycle** — handshake, capability negotiation, request/response correlation, cancellation, notifications — so neither client nor server code has to hand-write JSON-RPC framing (`src/mcp/shared/jsonrpc_dispatcher.py`, `src/mcp/client/session.py`).
3. **Speaks every standard transport** — stdio (subprocess), Streamable HTTP, SSE (`README.md:28`) — behind the same `Client`/`ClientSession` API, so switching from a local dev server to a remote deployed one is a one-line change (`server: Server | MCPServer | Transport | StdioServerParameters | str` — a URL string routes straight to `streamable_http_client`, `src/mcp/client/client.py:401-404`).

## Dynamic tool discovery — the mechanism a harness author actually calls

This is the crux of what this study cares about. Discovery in this SDK is not a bespoke registry lookup — it's a **wire round-trip that returns typed, cacheable schema objects**, and the v2 line adds a second, faster negotiation path on top of the classic one.

### The classic path: `initialize()` then `list_tools()`

`ClientSession.initialize()` (`src/mcp/client/session.py:651-674`) sends an `InitializeRequest` carrying the client's capabilities, gets back `InitializeResult` (server name/version + `ServerCapabilities`), sends `InitializedNotification`, and only then is the session usable. `list_tools()` (`:1278-1289`) sends `ListToolsRequest` and returns `ListToolsResult`, whose `tools: list[Tool]` each carry a `name`, `description`, and `input_schema` (JSON Schema) plus an optional `output_schema` — this is literally what a harness feeds to an LLM's tool-calling API.

### The v2 addition: `server/discover` — negotiation without a stateful handshake

`ClientSession.discover()` (`:757-794`) sends a single `server/discover` request proposing the newest "modern" protocol version (`LATEST_MODERN_VERSION`, still `"2026-07-28"` — `src/mcp-types/mcp_types/version.py:41,56`). If the server answers `UNSUPPORTED_PROTOCOL_VERSION` (-32022), the client intersects the server's advertised `supported` list with its own `MODERN_PROTOCOL_VERSIONS` and retries once at the highest mutual version (`:782-790`). `Client`'s default `mode="auto"` (`src/mcp/client/client.py:336-343`) runs this probe-then-fallback policy automatically via `negotiate_auto()` (`src/mcp/client/_probe.py:52-110`): try `server/discover`; any error that isn't "the server is modern-only and shares nothing with me" falls back to the legacy `initialize()` handshake. The docstring in `_probe.py:3-6` frames this precisely as **a denylist, not an allowlist** — only the disjoint-modern-version case is fatal; everything else (timeouts, method-not-found, network errors from a truly legacy server) is treated as "fall back and try the old way."

### Discovery is response-cacheable

`Client.list_tools()` (`:967-984`) routes through `_cached_fetch` (`:574-611`), which is wired to a per-client `ClientResponseCache` (`src/mcp/client/caching.py`) honoring server-supplied `ttlMs`/`cacheScope` hints (SEP-2549, 2026-07-28). A negative inbound `ttlMs` is floored to zero rather than failing validation (`_clamp_inbound_ttl`, `session.py:74`) — a defensive normalization for buggy servers. This means repeated discovery calls in a long-lived agent loop don't necessarily round-trip the network every time; the SDK owns that caching decision, not the harness.

### Schema generation is automatic, and fails loudly

`Tool.from_function` → `func_metadata()` builds the `arg_model` (`src/mcp/server/mcpserver/utilities/func_metadata.py:287-370`) from the function's live type hints via `get_type_hints` + `create_model`, using `StrictJsonSchema` (`:75-83`) which **raises a `ValueError`** instead of emitting a warning on a JSON-Schema-incompatible type. A tool author who returns or accepts something Pydantic can't serialize finds out at server-startup/registration time, not when a client tries to call it. `validate_arguments` (`:151`) re-validates raw wire arguments into the Pydantic model before the function is invoked, so a malformed `tools/call` never reaches user code un-typed.

### Header-mirroring for tool-call routing (2026-07-28 only)

`ClientSession._absorb_tool_listing` (`:1291-1332`) inspects each listed tool's `input_schema` for `x-mcp-header` annotations and caches an argument→header map (`_x_mcp_header_maps`); on a subsequent `tools/call`, matching arguments are mirrored into `Mcp-Param-*` HTTP headers (`_resolve_param_headers`, `:1111-1116`). Tools whose header annotations are invalid are **dropped from the listing entirely** with a logged warning (`:1298-1302`) — the client enforces a spec MUST by silently hiding a malformed tool rather than surfacing a broken one to the model.

## The `Client` class — what a harness actually imports

`src/mcp/client/client.py:267-1001`. A `@dataclass` wrapping `ClientSession` plus transport resolution, done once at `__post_init__` based purely on the shape of the `server=` argument (`:382-406`):

| `server=` value | Connector | Use case |
|---|---|---|
| `Server` or `MCPServer` instance | in-process, `DirectDispatcher` (no JSON-RPC framing at all in `mode="auto"`/pinned-version modes) or `InMemoryTransport` in `mode="legacy"` | tests, same-process tool orchestration |
| `str` (URL) | `streamable_http_client(url)` | talking to a remote deployed MCP server |
| `StdioServerParameters` | `stdio_client(params)` (added after the July pin, upstream #3321) | launching a local server as a subprocess |
| a custom `Transport` instance | used directly | SSE, or a hand-rolled transport |

`Client.call_tool` / `list_tools` / `list_resources` / `list_prompts` / `read_resource` / `get_prompt` / `complete` (`:613-984`) are the full surface a harness needs. Every call-shaped method (`call_tool`, `read_resource`, `get_prompt`) transparently drives an **`InputRequiredResult` multi-round-trip loop** (SEP-2322) — if the server needs the client's sampling/elicitation/roots callback to supply more information mid-call, `_drive_input_required` (`:926-947`) dispatches the embedded request through the same callback table used for legacy server-initiated RPCs, and retries, up to `input_required_max_rounds` (default `DEFAULT_INPUT_REQUIRED_MAX_ROUNDS`, `src/mcp/client/_input_required.py`). A harness author never sees this loop unless they opt into manual control via `client.session.call_tool(..., allow_input_required=True)`.

## `MCPServer` — how a harness-facing tool provider declares capabilities

`src/mcp/server/mcpserver/server.py:157-1378`. The ergonomic server wraps a `Server` (the lowlevel JSON-RPC handler, `src/mcp/server/lowlevel/server.py`) and three managers: `ToolManager`, `ResourceManager`, `PromptManager` (`:200-204`). Three decorators cover the whole declarative surface:

- **`@mcp.tool()`** (`:669-738`) — registers a plain or async Python function as a callable tool. Optionally injects a `Context` object (parameter typed `Context`) giving the tool access to logging, progress reporting, and resource reads mid-call (`:681-713`).
- **`@mcp.resource(uri)`** (`:788-948`) — registers a function as a URI-addressable resource. A URI with `{param}` placeholders (RFC 6570) is parsed via `UriTemplate.parse` (`:869`) at decoration time — malformed templates fail immediately, and mismatches between URI variables and function parameters raise `ValueError` before the server ever starts serving requests (`:881-930`).
- **`@mcp.prompt()`** (`:968-1030`) — registers a function returning prompt messages (or an `InputRequiredResult` for the multi-round flow).

Server transports are one call away: `mcp.run("stdio")`, `mcp.run("sse")`, or `mcp.run("streamable-http")` (`:371-430`), or grab the raw Starlette app (`sse_app()` / `streamable_http_app()`, `:1153-1378`) to mount inside an existing ASGI app. `Settings` (`:111`) is now a plain Pydantic `BaseModel` filled from constructor arguments. At the July pin it was a `pydantic_settings.BaseSettings` that read `MCP_*` env vars; that was removed along with the `pydantic-settings` dependency (upstream #3170).

Auth is wired at the `MCPServer` constructor: pass either an `auth_server_provider` (full OAuth AS) or a `token_verifier` (bearer-token RS), never both (`:250-254`); `sse_app`/`streamable_http_app` install `BearerAuthBackend` + `RequireAuthMiddleware` around the transport routes when configured (`:1189-1251`).

## Transports — the three wire carriers

- **stdio** (`src/mcp/client/stdio.py`, 355 lines) — spawns the server as a subprocess (`_create_platform_compatible_process`, `:329-348`, using `start_new_session=True` on POSIX for tree-kill isolation) and bridges stdin/stdout with two `anyio` tasks. Shutdown is a carefully sequenced escalation shielded from cancellation: close stdin → wait `PROCESS_TERMINATION_TIMEOUT` (2s) → SIGTERM the process group → wait `FORCE_KILL_TIMEOUT` (2s) → SIGKILL (`:249-268`, `_stop_server_process`). The module docstring (`:1-9`) is explicit that this whole sequence runs "inside a cancellation shield with every wait bounded, so a cancelled caller can neither leak a live server process nor hang on one" — a load-bearing correctness property for any harness that spawns local MCP servers and must survive its own timeouts.
- **Streamable HTTP** (`src/mcp/client/streamable_http.py`, 793 lines) — the modern remote-server transport; a `Client(url_string)` resolves straight to `streamable_http_client(srv)` (`client.py:401-402`).
- **SSE** (`src/mcp/client/sse.py`, 171 lines) — the older HTTP streaming transport, still present for backward compatibility (Streamable HTTP replaces it per the protocol's own changelog, but this SDK still ships an `SseServerTransport` and `sse_client`).

`ClientSessionGroup` (`src/mcp/client/session_group.py:87-452`) is the multi-server aggregator: it holds `ServerParameters` unions across `StdioServerParameters | SseServerParameters | StreamableHttpParameters` (`:69`) and merges tools/resources/prompts from N connected servers into one namespace, with a caller-supplied collision hook — this is the piece a harness reaches for when it needs "one tool list across every MCP server the user configured," which is the common case for IDE-style hosts.

## `mcp_types` — schema types now live in a sibling package

A structural detail worth flagging for anyone extending this SDK: as of this v2 checkout, the Pydantic models for the wire types (`Tool`, `CallToolResult`, `InitializeResult`, etc.) are **not defined inside `mcp/`** — they live in a separate workspace member, `src/mcp-types/mcp_types/`, imported everywhere as `import mcp_types as types` (e.g. `session.py:15`, `client.py:15`). Since the July pin, `mcp.types` is a permanent alias that re-exports the same objects, so v1-style `import mcp.types as types` keeps working (`src/mcp/types/__init__.py:1-17`, upstream #3190). The workspace splits the types by protocol era into **private** subpackages, `src/mcp-types/mcp_types/_v2025_11_25/` and `src/mcp-types/mcp_types/_v2026_07_28/` (renamed with a leading underscore after the July pin, upstream #3191; `pyproject.toml:166` lists `src/mcp-types/mcp_types` as a package source; `pyproject.toml:136` pins the published `mcp` wheel to an exact `mcp-types=={{ version }}`). This is the SDK's own concrete answer to "how do you support multiple protocol versions in one library without either forking the whole SDK per version or fighting Python's type system" — version the *types package*, not the client/server logic, and let `KNOWN_PROTOCOL_VERSIONS` / `HANDSHAKE_PROTOCOL_VERSIONS` / `MODERN_PROTOCOL_VERSIONS` (`src/mcp-types/mcp_types/version.py:24-44`) gate which era's schema applies at runtime.

## What's inside this submodule

| Path | What's there |
|---|---|
| `src/mcp/client/` | `client.py` (`Client`), `session.py` (`ClientSession`), `stdio.py`, `streamable_http.py`, `sse.py`, `session_group.py` (multi-server aggregator), `auth/` (OAuth2 client flows), `caching.py` (response cache), `extensions/skills.py` (client side of the Skills extension) |
| `src/mcp/server/mcpserver/` | The ergonomic `MCPServer` (`server.py`), `tools/`, `resources/`, `prompts/` managers, `utilities/func_metadata.py` (type-hint → JSON Schema) |
| `src/mcp/server/lowlevel/` | `Server` — the raw JSON-RPC method-handler registry `MCPServer` wraps |
| `src/mcp/server/auth/` | OAuth 2.0 authorization-server + resource-server middleware, handlers, routes |
| `src/mcp/server/skills.py`, `src/mcp/shared/skills.py` | The Skills extension (`io.modelcontextprotocol/skills`, SEP-2640), added after `v2.3.0` |
| `src/mcp/shared/` | `jsonrpc_dispatcher.py`, `direct_dispatcher.py` (in-process, no wire framing), `exceptions.py`, `extension.py` (SEP-2133 extension mechanism) |
| `src/mcp/cli/` | `mcp dev` (Inspector), `mcp install` (Claude Desktop config) — Typer-based |
| `src/mcp-types/mcp_types/` | Sibling workspace package: Pydantic wire types, versioned by protocol era (private `_v2025_11_25/`, `_v2026_07_28/`); mirrored as `mcp.types` |
| `docs_src/`, `docs/` | mkdocs-driven documentation source, organized by `nav:` sections not directory names (per `AGENTS.md`) |
| `tests/` | Mirrors `src/mcp/` 1:1 (`tests/client/test_stdio.py` ↔ `src/mcp/client/stdio.py`); 100% branch coverage required in CI |
| `AGENTS.md` (project instructions, symlinked as `CLAUDE.md`) | Branching model, `uv`-only package management, exception-handling rules, coverage discipline |

If you read three files: `README.md` (the whole pitch in under 130 lines), `src/mcp/client/client.py` (the class every harness imports), `src/mcp/server/mcpserver/server.py` (the class every tool-provider imports).

## Mental model for using it well

- **Pin the major version deliberately.** v2 is the stable line and an unpinned `pip install mcp` now installs it; code written against v1's `FastMCP`/`ClientSession` needs the migration guide linked from `README.md:20`, or a `mcp<2` pin. v2 also requires Python 3.11+ (`README.md:32`).
- **`Client(server)` accepting a bare `Server`/`MCPServer` instance is the fastest test harness there is.** No transport, no subprocess, no network — `AGENTS.md`'s own testing guidance calls this "the cleanest approach" for end-to-end behavior tests (`tests/client/test_client.py` is the canonical pattern per `AGENTS.md`).
- **Discovery is a `Client.list_tools()` call, not a config file.** A harness that wants dynamic capability discovery connects, calls `list_tools()` (cached, auto-refreshed on `notifications/tools/list_changed` in older protocol eras or the `listen()` subscription stream in 2026-07-28), and feeds the returned `Tool.input_schema` straight to the model's tool-calling API.
- **Let type hints be the schema.** Don't hand-write JSON Schema for tool parameters — annotate the Python function and let `func_metadata` build it; `StrictJsonSchema` will tell you immediately if a type can't serialize.
- **Use `mode="auto"` unless you have a reason not to.** It gets you the faster 2026-07-28 negotiation path when available and degrades to the classic handshake otherwise, with the fallback policy already worked out (`_probe.py`).
- **Treat the stdio shutdown sequence as a reason to trust, not reimplement, this transport.** Spawning subprocess-backed MCP servers has real process-tree-leak risk under cancellation; the SDK's shielded, bounded-timeout escalation (`stdio.py:248-267`) is exactly the kind of code not worth re-deriving.

## When NOT to reach for this

- **You have a large v1 codebase and no time to migrate.** v2 is stable but breaks the v1 API shapes; `v1.x` still gets maintenance releases, so pinning `mcp<2` is a reasonable holding position.
- **Your tools live in the same process and never need to cross a language or trust boundary.** MCP (any SDK) is for cross-process/cross-vendor tool-calling; if a harness's "tools" are just Python functions it already owns, calling them directly is faster and simpler — the `Profile__Model-Context-Protocol.md` "when not to reach for this" section applies unchanged.
- **You're building the client/host logic in another language.** This SDK is Python-only; TypeScript, Java, C#, Go, Kotlin, Rust, and Swift SDKs live in sibling repos under the same GitHub org.
- **You want zero dependency on Pydantic/anyio/Starlette.** The ergonomic `MCPServer` path pulls in Pydantic (schema + settings), anyio (structured concurrency), and Starlette/uvicorn (HTTP transports) — a fair amount of surface for a minimal embedded tool server. The `lowlevel.Server` path avoids some of this if only raw JSON-RPC method dispatch is needed.

## Changes since the previous pin (2026-07-10 → 2026-11-05)

- Old pin `1216c5369378`, new pin `77001c96f950`, 132 commits. Releases in range: `v2.0.0b2` (2026-07-14), `v2.0.0rc1` (2026-07-27), **`v2.0.0` (2026-07-28)**, `v2.0.1`, `v2.1.0`, `v2.1.1`, `v2.2.0` (2026-09-07), `v2.3.0` (2026-10-02). On the maintenance line: `v1.29.0` through `v1.30.0` (2026-09-05).
- **v2 is stable.** The beta warning is gone from `README.md`, docs are served at the site root with per-major paths, and the old `mcp.server.fastmcp` import now points users to the migration guide (`src/mcp/server/fastmcp.py`). This profile's lede, summary, and "when not to reach for this" advice were rewritten to match.
- **A Skills extension now ships in the SDK** (`src/mcp/server/skills.py`, `src/mcp/shared/skills.py`, `src/mcp/client/extensions/skills.py`; upstream #3485, after `v2.3.0`). It implements SEP-2640: skills are directories of files under a `skill://` scheme, enumerated with `skills/list` and fetched with `skills/get`, plus an optional `resources/directory/read` (`src/mcp/server/skills.py:1-26`). The SDK supplies the protocol plumbing and validation only; the server author supplies the catalog. For this study, this is the protocol-level version of what Tome does with its own MCP tools (see `Profile__Tome.md`).
- **Wire types reorganized.** The per-era packages became private (`mcp_types._v2025_11_25`, `mcp_types._v2026_07_28`) and `mcp.types` is a permanent alias of `mcp_types` (`src/mcp/types/__init__.py`).
- **Client and server surface changes a harness author will hit:** `Client` accepts `StdioServerParameters` directly (`src/mcp/client/client.py:403-404`); `CacheConfig()` is the client cache default and `None` turns it off (#3164); `MCPServer` settings no longer read `MCP_*` env vars and `pydantic-settings` is gone (`src/mcp/server/mcpserver/server.py:111`); `MCPServer` exposes a middleware chain (`server.py:242-248`); Streamable HTTP sessions now expire when idle and are capped by default (#3395); `httpx`/`httpx-sse` were replaced by `httpx2` (#2972).
- Python 3.10 support was dropped and 3.15 added (`pyproject.toml:6`, `requires-python = ">=3.11"`).

## How this compares to opencode and goose (sibling MCP-client implementers)

| Axis | MCP Python SDK (`Client`) | opencode | goose |
|---|---|---|---|
| **Language / runtime** | Python (this package) | TypeScript/Bun | Rust |
| **Relationship to the protocol** | The reference implementation — literally what the spec's own maintainers ship | An independent client implementation on top of the same wire protocol | An independent client implementation on top of the same wire protocol |
| **Discovery mechanism** | `Client.list_tools()` / `discover()` round-trip, cached per SEP-2549 hints | Own MCP client wired into its provider-agnostic tool-calling loop | Own MCP client wired into its Rust agent-loop tool registry |
| **Transport coverage** | stdio, Streamable HTTP, SSE, plus an in-process `DirectDispatcher`/`InMemoryTransport` for zero-transport testing | Implements a subset needed for its own harness (stdio-first, typically) | Implements a subset needed for its own harness |
| **Server authoring** | Also ships `MCPServer` — the same package can be the tool *provider*, not just the consumer | N/A — consumer only | N/A — consumer only |
| **Protocol-version negotiation** | Explicit dual path: legacy `initialize()` handshake vs. 2026-07-28 `server/discover`, with a documented fallback policy (`_probe.py`) | Whatever protocol version its MCP client library targets — typically the latest stable spec version, not the pre-release `server/discover` era | Same — targets stable spec versions, not the v2-SDK's in-progress modern era |
| **Auth** | Built-in OAuth 2.0 AS/RS middleware for HTTP transports (`server/auth/`) | Delegates to whatever the connected MCP servers require, via its own config surface | Same — config-driven, no first-party OAuth server implementation |
| **Best fit** | Any Python process — agent harness, IDE plugin, CLI tool — that wants to either consume or expose MCP tools | A TypeScript-first coding agent that needs MCP as one of several tool-integration surfaces | A Rust-first, extensibility-focused coding agent with MCP as one of several extension mechanisms |

The structural point: opencode and goose are proof that the protocol's "N hosts × M servers → N+M" thesis (see the sibling MCP spec profile) actually works — they implement their own clients from the same public spec this SDK also implements, and (protocol-version quirks aside) any of the three can talk to any spec-compliant server, including one built with this very package's `MCPServer`.

## One-line summary

> The MCP Python SDK is the reference Python implementation of both halves of the Model Context Protocol — a `Client` that discovers and calls tools/resources/prompts on any MCP server (in-process, stdio subprocess, or remote HTTP, with dynamic discovery reducing to a cached `list_tools()` round-trip and an automatic legacy-handshake-or-modern-`server/discover` negotiation) and an `MCPServer` that turns type-hinted Python functions into wire-schema tools/resources/prompts with zero hand-written JSON Schema — now stable as v2 (since 2026-07-28, at `v2.3.0` plus a new Skills extension at this pin), with the actual wire types split into a separately-versioned sibling package (`mcp_types`) so multiple protocol eras coexist in one codebase.
