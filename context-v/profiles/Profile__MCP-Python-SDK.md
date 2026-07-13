---
name: MCP Python SDK Profile
slug: mcp-python-sdk
upstream: https://github.com/modelcontextprotocol/python-sdk
package: mcp (PyPI)
license: MIT
maintainer: modelcontextprotocol org (Linux Foundation); Anthropic, PBC copyright holder per LICENSE
study: studies/agent-harnesses
profile_path: studies/agent-harnesses/mcp-python-sdk
profile_kind: library (client + server SDK)
date_created: 2026-07-13
---

# MCP Python SDK — Profile

A profile of the MCP Python SDK as it lives in this study (`studies/agent-harnesses/mcp-python-sdk/`, pinned at commit `1216c536` / 2026-07-10). Cites pinned paths so you can jump to source rather than trust paraphrase. This is a different object from `studies/open-specs-and-standards/modelcontextprotocol/` (the wire-protocol spec repo, profiled in [`Profile__Model-Context-Protocol.md`](../../open-specs-and-standards/context-v/profiles/Profile__Model-Context-Protocol.md)) — that repo is the schema and docs; this one is the actual Python client/server implementation that a harness author imports and runs.

## TL;DR

The pinned checkout is the **v2 pre-release line** (alpha/beta, published as `2.0.0aN`/`2.0.0bN` on PyPI) — the README is explicit that v1.x, not v2, is what's stable for production right now (`README.md:16-21`). v2 is a rework targeting the **2026-07-28 MCP spec release**, and it changes the public API shape from the well-known v1 `FastMCP`/`ClientSession` pattern to a new `MCPServer` + `Client` pair.

> "Notice what you did **not** write: no JSON Schema (`a: int, b: int` _is_ the schema), no request parsing, no validation code, no protocol handling. Two type-hinted Python functions and a docstring." (`README.md:83`)

The full server pitch, in the README's own 15 lines (`README.md:53-70`):

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

And the client side, in 10 (`README.md:91-106`):

```python
from mcp import Client
from server import mcp

async def main() -> None:
    async with Client(mcp) as client:
        result = await client.call_tool("add", {"a": 1, "b": 2})
        print(result.structured_content)  # {'result': 3}
```

The one line worth memorizing: **`Client` connects to a URL, a stdio subprocess, a custom `Transport`, or (for tests) directly to a `Server`/`MCPServer` instance in memory with no transport at all** (`README.md:89`, `src/mcp/client/client.py:289-295`) — the exact same `call_tool`/`list_tools`/`list_resources` code path works whether the peer is in-process or a remote HTTP server. This is what makes the SDK's `Client` class the thing a harness author (opencode, goose, or any custom agent loop) actually imports to gain **dynamic tool discovery**: connect, call `list_tools()`, and the tool schemas come back as `Tool` objects with a JSON-Schema `input_schema` derived automatically from the server's type-hinted Python function — no schema is hand-written on either side.

## Why this exists — the SDK's job in the protocol stack

The spec (profiled separately) defines the wire format; this package is **the reference implementation of both halves of that wire format for Python** — build a server that speaks it, or build a client/host that speaks it. Per `AGENTS.md`, the branching model makes the split explicit: `main` is the v2 rework (breaking changes expected, spec-alignment driven), `v1.x` is the stable release branch that still ships critical fixes. A harness author choosing "which MCP client do I import in Python" is choosing between pinning `mcp<2` (stable, familiar `ClientSession`) or trying the v2 pre-release (`Client`, `MCPServer`, a new `server/discover` negotiation path, response caching, extensions).

Three things this package does that a harness author would otherwise have to hand-roll:

1. **Turns a Python function signature into a JSON Schema tool definition.** `func_metadata.py` (`src/mcp/server/mcpserver/utilities/func_metadata.py:36-44`) builds a Pydantic `arg_model` from the function's type hints via `create_model`, and a `StrictJsonSchema` generator that **raises instead of warning** on non-serializable types — so a tool author gets a loud error at registration time, not a silent bad schema shipped to the model.
2. **Owns the JSON-RPC 2.0 session lifecycle** — handshake, capability negotiation, request/response correlation, cancellation, notifications — so neither client nor server code has to hand-write JSON-RPC framing (`src/mcp/shared/jsonrpc_dispatcher.py`, `src/mcp/client/session.py`).
3. **Speaks every standard transport** — stdio (subprocess), Streamable HTTP, SSE (`README.md:35`) — behind the same `Client`/`ClientSession` API, so switching from a local dev server to a remote deployed one is a one-line change (`server: Server | MCPServer | Transport | str` — a URL string routes straight to `streamable_http_client`, `src/mcp/client/client.py:387-390`).

## Dynamic tool discovery — the mechanism a harness author actually calls

This is the crux of what this study cares about. Discovery in this SDK is not a bespoke registry lookup — it's a **wire round-trip that returns typed, cacheable schema objects**, and the v2 line adds a second, faster negotiation path on top of the classic one.

### The classic path: `initialize()` then `list_tools()`

`ClientSession.initialize()` (`src/mcp/client/session.py:561-583`) sends an `InitializeRequest` carrying the client's capabilities, gets back `InitializeResult` (server name/version + `ServerCapabilities`), sends `InitializedNotification`, and only then is the session usable. `list_tools()` (`:1139-1150`) sends `ListToolsRequest` and returns `ListToolsResult`, whose `tools: list[Tool]` each carry a `name`, `description`, and `input_schema` (JSON Schema) plus an optional `output_schema` — this is literally what a harness feeds to an LLM's tool-calling API.

### The v2 addition: `server/discover` — negotiation without a stateful handshake

`ClientSession.discover()` (`:663-699`) sends a single `server/discover` request proposing the newest "modern" protocol version (`LATEST_MODERN_VERSION`, currently `"2026-07-28"` — `src/mcp-types/mcp_types/version.py`). If the server answers `UNSUPPORTED_PROTOCOL_VERSION` (-32022), the client intersects the server's advertised `supported` list with its own `MODERN_PROTOCOL_VERSIONS` and retries once at the highest mutual version (`:667-671`). `Client`'s default `mode="auto"` (`src/mcp/client/client.py:325-332`) runs this probe-then-fallback policy automatically via `negotiate_auto()` (`src/mcp/client/_probe.py:47-101`): try `server/discover`; any error that isn't "the server is modern-only and shares nothing with me" falls back to the legacy `initialize()` handshake. The docstring in `_probe.py:5-6` frames this precisely as **a denylist, not an allowlist** — only the disjoint-modern-version case is fatal; everything else (timeouts, method-not-found, network errors from a truly legacy server) is treated as "fall back and try the old way."

### Discovery is response-cacheable

`Client.list_tools()` (`:908-928`) routes through `_cached_fetch` (`:537-575`), which is wired to a per-client `ClientResponseCache` (`src/mcp/client/caching.py`) honoring server-supplied `ttlMs`/`cacheScope` hints (SEP-2549, 2026-07-28). A negative inbound `ttlMs` is floored to zero rather than failing validation (`_clamp_inbound_ttl`, `session.py:66-70`) — a defensive normalization for buggy servers. This means repeated discovery calls in a long-lived agent loop don't necessarily round-trip the network every time; the SDK owns that caching decision, not the harness.

### Schema generation is automatic, and fails loudly

`Tool.from_function` → `func_metadata()` builds the `arg_model` (`src/mcp/server/mcpserver/utilities/func_metadata.py:66-80`) from the function's live type hints via `get_type_hints` + `create_model`, using `StrictJsonSchema` (`:36-44`) which **raises a `ValueError`** instead of emitting a warning on a JSON-Schema-incompatible type. A tool author who returns or accepts something Pydantic can't serialize finds out at server-startup/registration time, not when a client tries to call it. `validate_arguments` (`:72-80`) re-validates raw wire arguments into the Pydantic model before the function is invoked, so a malformed `tools/call` never reaches user code un-typed.

### Header-mirroring for tool-call routing (2026-07-28 only)

`ClientSession._absorb_tool_listing` (`:1152-1184`) inspects each listed tool's `input_schema` for `x-mcp-header` annotations and caches an argument→header map (`_x_mcp_header_maps`); on a subsequent `tools/call`, matching arguments are mirrored into `Mcp-Param-*` HTTP headers (`_resolve_param_headers`, `:1011-1016`). Tools whose header annotations are invalid are **dropped from the listing entirely** with a logged warning (`:1162-1167`) — the client enforces a spec MUST by silently hiding a malformed tool rather than surfacing a broken one to the model.

## The `Client` class — what a harness actually imports

`src/mcp/client/client.py:263-935`. A `@dataclass` wrapping `ClientSession` plus transport resolution, done once at `__post_init__` based purely on the shape of the `server=` argument (`:369-390`):

| `server=` value | Connector | Use case |
|---|---|---|
| `Server` or `MCPServer` instance | in-process, `DirectDispatcher` (no JSON-RPC framing at all in `mode="auto"`/pinned-version modes) or `InMemoryTransport` in `mode="legacy"` | tests, same-process tool orchestration |
| `str` (URL) | `streamable_http_client(url)` | talking to a remote deployed MCP server |
| a custom `Transport` instance | used directly | stdio subprocess (`stdio_client`), SSE, or a hand-rolled transport |

`Client.call_tool` / `list_tools` / `list_resources` / `list_prompts` / `read_resource` / `get_prompt` / `complete` (`:735-928`) are the full surface a harness needs. Every call-shaped method (`call_tool`, `read_resource`, `get_prompt`) transparently drives an **`InputRequiredResult` multi-round-trip loop** (SEP-2322) — if the server needs the client's sampling/elicitation/roots callback to supply more information mid-call, `_drive_input_required` (`:867-888`) dispatches the embedded request through the same callback table used for legacy server-initiated RPCs, and retries, up to `input_required_max_rounds` (default `DEFAULT_INPUT_REQUIRED_MAX_ROUNDS`, `src/mcp/client/_input_required.py`). A harness author never sees this loop unless they opt into manual control via `client.session.call_tool(..., allow_input_required=True)`.

## `MCPServer` — how a harness-facing tool provider declares capabilities

`src/mcp/server/mcpserver/server.py:159-1329`. The ergonomic server wraps a `Server` (the lowlevel JSON-RPC handler, `src/mcp/server/lowlevel/server.py`) and three managers: `ToolManager`, `ResourceManager`, `PromptManager` (`:201-205`). Three decorators cover the whole declarative surface:

- **`@mcp.tool()`** (`:609-677`) — registers a plain or async Python function as a callable tool. Optionally injects a `Context` object (parameter typed `Context`) giving the tool access to logging, progress reporting, and resource reads mid-call (`:637-656`).
- **`@mcp.resource(uri)`** (`:720-879`) — registers a function as a URI-addressable resource. A URI with `{param}` placeholders (RFC 6570) is parsed via `UriTemplate.parse` (`:801`) at decoration time — malformed templates fail immediately, and mismatches between URI variables and function parameters raise `ValueError` before the server ever starts serving requests (`:812-834`).
- **`@mcp.prompt()`** (`:900-961`) — registers a function returning prompt messages (or an `InputRequiredResult` for the multi-round flow).

Server transports are one call away: `mcp.run("stdio")`, `mcp.run("sse")`, or `mcp.run("streamable-http")` (`:375-396`), or grab the raw Starlette app (`streamable_http_app()` / `sse_app()`, `:1077-1229`) to mount inside an existing ASGI app. `Settings` (`:101-135`) is a `pydantic_settings.BaseSettings` reading every knob from `MCP_*` env vars — `MCP_DEBUG=true`, `MCP_LOG_LEVEL`, etc.

Auth is wired at the `MCPServer` constructor: pass either an `auth_server_provider` (full OAuth AS) or a `token_verifier` (bearer-token RS), never both (`:243-249`); `sse_app`/`streamable_http_app` install `BearerAuthBackend` + `RequireAuthMiddleware` around the transport routes when configured (`:1111-1166`).

## Transports — the three wire carriers

- **stdio** (`src/mcp/client/stdio.py`, 355 lines) — spawns the server as a subprocess (`_create_platform_compatible_process`, `:328-348`, using `start_new_session=True` on POSIX for tree-kill isolation) and bridges stdin/stdout with two `anyio` tasks. Shutdown is a carefully sequenced escalation shielded from cancellation: close stdin → wait `PROCESS_TERMINATION_TIMEOUT` (2s) → SIGTERM the process group → wait `FORCE_KILL_TIMEOUT` (2s) → SIGKILL (`:248-267`, `_stop_server_process`). The module docstring (`:1-9`) is explicit that this whole sequence runs "inside a cancellation shield with every wait bounded, so a cancelled caller can neither leak a live server process nor hang on one" — a load-bearing correctness property for any harness that spawns local MCP servers and must survive its own timeouts.
- **Streamable HTTP** (`src/mcp/client/streamable_http.py`, 718 lines) — the modern remote-server transport; a `Client(url_string)` resolves straight to `streamable_http_client(srv)` (`client.py:388`).
- **SSE** (`src/mcp/client/sse.py`, 161 lines) — the older HTTP streaming transport, still present for backward compatibility (Streamable HTTP replaces it per the protocol's own changelog, but this SDK still ships an `SseServerTransport` and `sse_client`).

`ClientSessionGroup` (`src/mcp/client/session_group.py:85-449`) is the multi-server aggregator: it holds `ServerParameters` unions across `StdioServerParameters | SseServerParameters | StreamableHttpParameters` (`:67`) and merges tools/resources/prompts from N connected servers into one namespace, with a caller-supplied collision hook — this is the piece a harness reaches for when it needs "one tool list across every MCP server the user configured," which is the common case for IDE-style hosts.

## `mcp_types` — schema types now live in a sibling package

A structural detail worth flagging for anyone extending this SDK: as of this v2 checkout, the Pydantic models for the wire types (`Tool`, `CallToolResult`, `InitializeResult`, etc.) are **not defined inside `mcp/`** — they live in a separate workspace member, `src/mcp-types/mcp_types/`, imported everywhere as `import mcp_types as types` (e.g. `session.py:14`, `client.py:15`). The workspace even splits the types by protocol era: `src/mcp-types/mcp_types/v2025_11_25/` and `src/mcp-types/mcp_types/v2026_07_28/` are separate subpackages (`pyproject.toml:155` lists `src/mcp-types/mcp_types` as a coverage source; `pyproject.toml:126` pins the published `mcp` wheel to an exact `mcp-types=={{ version }}`). This is the SDK's own concrete answer to "how do you support multiple protocol versions in one library without either forking the whole SDK per version or fighting Python's type system" — version the *types package*, not the client/server logic, and let `KNOWN_PROTOCOL_VERSIONS` / `HANDSHAKE_PROTOCOL_VERSIONS` / `MODERN_PROTOCOL_VERSIONS` (`src/mcp-types/mcp_types/version.py:11-30`) gate which era's schema applies at runtime.

## What's inside this submodule

| Path | What's there |
|---|---|
| `src/mcp/client/` | `client.py` (`Client`), `session.py` (`ClientSession`), `stdio.py`, `streamable_http.py`, `sse.py`, `session_group.py` (multi-server aggregator), `auth/` (OAuth2 client flows), `caching.py` (response cache) |
| `src/mcp/server/mcpserver/` | The ergonomic `MCPServer` (`server.py`), `tools/`, `resources/`, `prompts/` managers, `utilities/func_metadata.py` (type-hint → JSON Schema) |
| `src/mcp/server/lowlevel/` | `Server` — the raw JSON-RPC method-handler registry `MCPServer` wraps |
| `src/mcp/server/auth/` | OAuth 2.0 authorization-server + resource-server middleware, handlers, routes |
| `src/mcp/shared/` | `jsonrpc_dispatcher.py`, `direct_dispatcher.py` (in-process, no wire framing), `session.py`, `exceptions.py`, `extension.py` (SEP-2133 extension mechanism) |
| `src/mcp/cli/` | `mcp dev` (Inspector), `mcp install` (Claude Desktop config) — Typer-based |
| `src/mcp-types/mcp_types/` | Sibling workspace package: Pydantic wire types, versioned by protocol era (`v2025_11_25/`, `v2026_07_28/`) |
| `docs_src/`, `docs/` | mkdocs-driven documentation source, organized by `nav:` sections not directory names (per `AGENTS.md`) |
| `tests/` | Mirrors `src/mcp/` 1:1 (`tests/client/test_stdio.py` ↔ `src/mcp/client/stdio.py`); 100% branch coverage required in CI |
| `AGENTS.md` (project instructions, symlinked as `CLAUDE.md`) | Branching model, `uv`-only package management, exception-handling rules, coverage discipline |

If you read three files: `README.md` (the whole pitch in under 130 lines), `src/mcp/client/client.py` (the class every harness imports), `src/mcp/server/mcpserver/server.py` (the class every tool-provider imports).

## Mental model for using it well

- **Pin the version deliberately.** v1.x (`mcp>=1.27,<2`) is what's stable today; v2 pre-releases (`2.0.0aN`/`bN`) can break between patch bumps (`README.md:17-19`). Don't let an unpinned `pip install mcp` silently jump lines.
- **`Client(server)` accepting a bare `Server`/`MCPServer` instance is the fastest test harness there is.** No transport, no subprocess, no network — `AGENTS.md`'s own testing guidance calls this "the cleanest approach" for end-to-end behavior tests (`tests/client/test_client.py` is the canonical pattern per `AGENTS.md`).
- **Discovery is a `Client.list_tools()` call, not a config file.** A harness that wants dynamic capability discovery connects, calls `list_tools()` (cached, auto-refreshed on `notifications/tools/list_changed` in older protocol eras or the `listen()` subscription stream in 2026-07-28), and feeds the returned `Tool.input_schema` straight to the model's tool-calling API.
- **Let type hints be the schema.** Don't hand-write JSON Schema for tool parameters — annotate the Python function and let `func_metadata` build it; `StrictJsonSchema` will tell you immediately if a type can't serialize.
- **Use `mode="auto"` unless you have a reason not to.** It gets you the faster 2026-07-28 negotiation path when available and degrades to the classic handshake otherwise, with the fallback policy already worked out (`_probe.py`).
- **Treat the stdio shutdown sequence as a reason to trust, not reimplement, this transport.** Spawning subprocess-backed MCP servers has real process-tree-leak risk under cancellation; the SDK's shielded, bounded-timeout escalation (`stdio.py:248-267`) is exactly the kind of code not worth re-deriving.

## When NOT to reach for this

- **You need the stable, documented API today.** v2 in this checkout is pre-release; if the target is production software right now, pin `v1.x` and read that branch's README, not this profile's v2 API shapes.
- **Your tools live in the same process and never need to cross a language or trust boundary.** MCP (any SDK) is for cross-process/cross-vendor tool-calling; if a harness's "tools" are just Python functions it already owns, calling them directly is faster and simpler — the `Profile__Model-Context-Protocol.md` "when not to reach for this" section applies unchanged.
- **You're building the client/host logic in another language.** This SDK is Python-only; TypeScript, Java, C#, Go, Kotlin, Rust, and Swift SDKs live in sibling repos under the same GitHub org.
- **You want zero dependency on Pydantic/anyio/Starlette.** The ergonomic `MCPServer` path pulls in Pydantic (schema + settings), anyio (structured concurrency), and Starlette/uvicorn (HTTP transports) — a fair amount of surface for a minimal embedded tool server. The `lowlevel.Server` path avoids some of this if only raw JSON-RPC method dispatch is needed.

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

> The MCP Python SDK is the reference Python implementation of both halves of the Model Context Protocol — a `Client` that discovers and calls tools/resources/prompts on any MCP server (in-process, stdio subprocess, or remote HTTP, with dynamic discovery reducing to a cached `list_tools()` round-trip and an automatic legacy-handshake-or-modern-`server/discover` negotiation) and an `MCPServer` that turns type-hinted Python functions into wire-schema tools/resources/prompts with zero hand-written JSON Schema — currently mid-rework as a pre-release v2 line targeting the 2026-07-28 spec, with the actual wire types split into a separately-versioned sibling package (`mcp_types`) so multiple protocol eras coexist in one codebase.
