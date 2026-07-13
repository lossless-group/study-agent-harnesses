---
name: OpenHands Profile
slug: openhands
upstream: https://github.com/OpenHands/OpenHands
package: N/A (npm: @openhands/agent-canvas; PyPI deps: openhands-sdk, openhands-agent-server, openhands-tools)
license: MIT (enterprise/ carved out under a separate enterprise/LICENSE)
maintainer: OpenHands (formerly OpenDevin / All-Hands-AI)
study: studies/agent-harnesses
profile_path: studies/agent-harnesses/OpenHands
profile_kind: control-plane frontend + app-server (formerly monolithic agent runtime, now split)
date_created: 2026-07-13
---

# OpenHands — Profile

A profile of OpenHands as it lives in this study (`studies/agent-harnesses/OpenHands/`, pinned at `5f9906f`, 2026-07-13). Cites pinned paths so you can jump to source rather than trust paraphrase. Read alongside [`Profile__Aider.md`](./Profile__Aider.md) and [`Profile__Opencode.md`](./Profile__Opencode.md) — OpenHands is the outlier of the three: **the repo you actually get when you pin `OpenHands/OpenHands` is no longer the agent**, it's the control-plane UI and server that talks to the agent.

## TL;DR

The README says it plainly, and it changes everything about how this profile has to be written (`README.md:14`, banner note directly under the description):

> The code in this repo is moving! See [Agent Canvas transition FAQ]... The source code for OpenHands Agent and Agent Server lives in [OpenHands/software-agent-sdk](https://github.com/OpenHands/software-agent-sdk). The source code for Agent Canvas lives in [OpenHands/agent-canvas](https://github.com/OpenHands/agent-canvas).

What's pinned in this study's checkout is **Agent Canvas**: "the self-hosted developer control center for coding agents and automations" (`README.md:6`) — a frontend + FastAPI app-server that can drive OpenHands' own agent, or Claude Code, Codex, Gemini, or "any ACP-compatible agent" (`README.md:8`), across local/Docker/VM/cloud backends. The CodeAct agent loop, the micro-agent (now "Skill") execution engine, and the actual event-stream *producer* do not live in this checkout at all — they are pip/npm dependencies pinned in `pyproject.toml:` `"openhands-agent-server==1.36.0"`, `"openhands-sdk==1.36.0"`, `"openhands-tools==1.36.0"` (verified: `grep openhands-sdk pyproject.toml`). This repo imports `Event` and `Skill` types from that external `openhands.sdk` package (`openhands/app_server/event/event_service.py:11` — `from openhands.sdk import Event`; `openhands/app_server/app_conversation/skill_loader.py:24` — `from openhands.sdk.skills import KeywordTrigger, Skill, TaskTrigger`) rather than defining them.

What remains here, and is real, cited, load-bearing architecture in its own right: an **`EventService` abstraction with four interchangeable persistence backends** (filesystem, SQL, AWS, Google Cloud — `openhands/app_server/event/{filesystem,event}_service.py`, `aws_event_service.py`, `google_cloud_event_service.py`), an **event-callback/webhook subsystem** for external integrations (`openhands/app_server/event_callback/`), a **FastMCP-based MCP router** that both hosts OpenHands' own tools (`create_pr`, `create_mr`, `create_bitbucket_pr`, ...) and proxies third-party MCP servers into the sandbox (Tavily, mounted under a namespace — `openhands/app_server/mcp/mcp_router.py:49-76`), and an explicit **skill-loading proxy** that documents itself as "a thin proxy" over the agent-server (`openhands/app_server/app_conversation/skill_loader.py:1-10`).

If you want one sentence: **the `OpenHands/OpenHands` submodule pinned in this study is Agent Canvas — a self-hosted control-plane (FastAPI app-server + React frontend) that persists agent trajectories as one `Event`-per-file (or SQL/AWS/GCS row) via a pluggable `EventService`, proxies MCP tool calls (its own git-provider tools plus mounted third-party MCP servers like Tavily) into agent sandboxes, and drives OpenHands' own CodeAct agent or any ACP-compatible agent (Claude Code, Codex, Gemini) — while the actual CodeAct loop, micro-agent/Skill execution, and event *production* now live in the separate `OpenHands/software-agent-sdk` repo as a versioned dependency, not in this checkout.**

## Why this exists — from monolithic agent to control-plane + swappable backend

This is the single most important thing to understand before reading any code in this checkout: **OpenHands underwent an architectural split**, and the split itself is evidence about how far agent-harness design has moved since the original OpenDevin/CodeAct paper (arXiv:2407.16741, cited verbatim in `.openhands/microagents/glossary.md` — *"CodeAct Agent — A generalist agent in OpenHands designed to perform tasks by editing and executing code"*).

1. **The agent runtime became a versioned dependency, not owned source.** `pyproject.toml` pins `openhands-sdk==1.36.0`, `openhands-agent-server==1.36.0`, `openhands-tools==1.36.0` as exact-version PyPI packages (`exclude-newer-package = { openhands-agent-server = false, openhands-sdk = false, openhands-tools = false }`). Nothing under `openhands/` in this checkout implements the CodeAct think-act-observe loop; `openhands/app_server/` only *calls into* an Agent Server process running that loop, per the architecture section (`README.md`, "Architecture" — *"Agent Canvas is powered by the OpenHands Agent Server, a REST API for running multiple agents on a single machine"*).
2. **Multi-backend, multi-agent-brand from day one.** The README's positioning line is explicit: *"Run OpenHands, Claude Code, Codex, Gemini, or any ACP-compatible agent across local, remote, and cloud backends"* (`README.md:8`). This is a deliberate widening from "OpenHands is an agent" to "OpenHands is a canvas any agent plugs into" — the frontend has first-class ACP (Agent-Client Protocol) plumbing (`frontend/src/hooks/use-acp-credential-form.ts`, `frontend/src/hooks/use-switch-acp-model.ts`, `frontend/src/types/v1/core/events/acp-tool-call-event.ts`, `tests/unit/app_server/test_webhook_router_acp.py`).
3. **The event stream is the integration surface, not an implementation detail.** Because the actual agent lives in a separate process (possibly on a different host — Docker, VM, or OpenHands Cloud, per `README.md` Option 2/Option 3 and the self-hosting guide link), the only way Agent Canvas can show you what the agent did is by consuming and persisting an `Event` stream that crosses a process/network boundary. That's why `EventService` has four backend implementations instead of one in-memory list — filesystem for local single-node use, SQL/AWS/GCS for anything shared or hosted.

## The event stream — `EventService`, four backends, one `Event` type

The abstract contract (`openhands/app_server/event/event_service.py:18-70`):

```python
class EventService(ABC):
    """Event Service for getting events."""

    @abstractmethod
    async def get_event(self, conversation_id: UUID, event_id: UUID) -> Event | None: ...

    @abstractmethod
    async def search_events(
        self, conversation_id: UUID, kind__eq: EventKind | None = None,
        timestamp__gte: datetime | None = None, timestamp__lt: datetime | None = None,
        sort_order: EventSortOrder = EventSortOrder.TIMESTAMP,
        page_id: str | None = None, limit: int = 100,
    ) -> EventPage: ...

    @abstractmethod
    async def count_events(self, conversation_id: UUID, ...) -> int: ...

    async def iter_events_for_export(self, conversation_id: UUID) -> AsyncGenerator[Event, None]:
        """Iterate all events for a conversation in export order."""
        events = page_iterator(self.search_events, conversation_id=conversation_id)
        async for event in events:
            yield event

    @abstractmethod
    async def save_event(self, conversation_id: UUID, event: Event): ...
```

`Event` itself is imported, not defined here (`event_service.py:11` — `from openhands.sdk import Event`), confirming trajectories are typed by the external SDK. `EventKind` (`openhands/app_server/event_callback/event_callback_models.py`) is this repo's own enum used purely for *filtering/querying* events, not for defining what an event is.

**Filesystem backend** — one JSON file per event (`openhands/app_server/event/filesystem_event_service.py:17-35`):

```python
@dataclass
class FilesystemEventService(EventServiceBase):
    """Event service based on file system"""
    limit: int = 500

    def _load_event(self, path: Path) -> Event | None:
        content = path.read_text()
        content = Event.model_validate_json(content)
        return content

    def _store_event(self, path: Path, event: Event):
        path.parent.mkdir(parents=True, exist_ok=True)
        content = event.model_dump_json(indent=2)
        path.write_text(content)
```

Every event round-trips through Pydantic's `model_dump_json`/`model_validate_json` — the on-disk trajectory is literally the `Event` model serialized 1:1, one file per event, directory-scoped per conversation (`_search_paths` globs `f'{prefix}/*'`). This is the local/laptop persistence path (Option 1/3 in the README's quickstart — running without a sandbox or from source).

**SQL, AWS, and Google Cloud backends** exist as siblings implementing the same `EventService` ABC (`openhands/app_server/event/{event_store.py,aws_event_service.py,google_cloud_event_service.py}`, confirmed present via directory listing) — the same pattern repeats in `event_callback/sql_event_callback_service.py` and `app_conversation/sql_app_conversation_info_service.py`. The architectural point: **which storage backend is active is an injected dependency (`EventServiceInjector`, `DiscriminatedUnionMixin` + `Injector[EventService]`, `event_service.py:73-74`), selected at deploy time**, not a hardcoded choice — the same conversation history can live as flat files on a laptop or as rows in a hosted Postgres/S3-backed deployment without the frontend caring which.

**The frontend mirrors this duality.** `frontend/src/stores/use-event-store.ts:9-11` explicitly carries two event shapes side by side during a schema migration: *"While we transition to v1 events, our store can handle both v0 and v1 events"* — `OHEvent = (OpenHandsEvent | OpenHandsParsedEvent) & { isFromPlanningAgent?: boolean }`. Events are sorted defensively by timestamp with fallback ordering for events missing timestamps (`use-event-store.ts:22-34`), which matters because events can arrive out of order across a network hop from a remote Agent Server.

## Skills (formerly micro-agents) — loaded through the agent-server, not here

`.openhands/microagents/` is this repo's *own* skill configuration for using OpenHands on itself (directory still named `microagents/` even though the code now calls them Skills) — two files, `documentation.md` and `glossary.md`. `documentation.md` frontmatter shows the still-current trigger schema:

```yaml
---
name: documentation
type: knowledge
version: 1.0.0
agent: CodeActAgent
triggers:
- documentation
- docs
- document
---
```

`glossary.md` is itself evidence of the terminology, defining `Agent Controller` ("manages the agent's lifecycle, handles its state, and coordinates interactions between the agent and various tools"), `Agent Delegation` ("the ability of an agent to hand off specific tasks to other specialized agents for better task completion"), and `CodeAct Agent` ("[a generalist agent in OpenHands](https://arxiv.org/abs/2407.16741) designed to perform tasks by editing and executing code") — but these definitions describe behavior implemented in `software-agent-sdk`, not in this checkout.

The loading code that *is* here confirms the hand-off explicitly. `openhands/app_server/app_conversation/skill_loader.py:1-10`:

> Utilities for loading skills for V1 conversations. This module provides functions to load skills from the agent-server, which centralizes all skill loading logic. The app-server acts as a thin proxy that: (1) Builds the org_config with authentication information, (2) Builds the sandbox_config with exposed URLs, (3) Calls the agent-server's `/api/skills` endpoint. All source-specific skill loading is handled by the agent-server.

And it imports the trigger types from the external SDK, not local definitions (`skill_loader.py:24` — `from openhands.sdk.skills import KeywordTrigger, Skill, TaskTrigger`). So the multi-agent delegation model this task asked about — CodeActAgent + micro-agents — is **fully real and fully documented in this checkout's own glossary**, but its *implementation* is one dependency hop away in `software-agent-sdk`; this repo's job is authentication, sandbox exposure, and proxying the `/api/skills` call.

## MCP integration — OpenHands as both MCP host and MCP proxy

`openhands/app_server/mcp/mcp_router.py` runs a `FastMCP` server (`mcp_server = FastMCP('mcp', mask_error_details=True)`, line 43) that does two distinct things:

1. **Hosts OpenHands' own tools directly** — `create_pr`, `create_mr`, `create_bitbucket_pr`, `create_bitbucket_data_center_pr`, `create_azure_devops_pr` (lines 147-488), each pulling per-user provider tokens via `get_provider_tokens`/`get_access_token`/`get_user_id` from request context, and each — on success — calling `save_pr_metadata()` (lines 98-144) to append the resulting PR/MR number back onto the `AppConversationInfo` record. Every tool also tries to append a **"continue refining the PR" link back to this exact OpenHands conversation** into the PR body when running in SaaS mode (`get_conversation_link()`, lines 78-95) — a small but telling detail: the tool call closes a loop back to its own originating conversation.
2. **Proxies a third-party MCP server into the sandbox** — `init_tavily_proxy()` (lines 49-76) builds a `fastmcp.Client` against Tavily's hosted MCP endpoint (`StreamableHttpTransport(url=f'https://mcp.tavily.com/mcp/?tavilyApiKey={tavily_api_key}')`), wraps it with `create_proxy()`, and **mounts it under a `tavily` namespace** so its tools surface as `tavily_*` — explicitly so "sandboxes can use Tavily search without the API key being exposed" (docstring, lines 50-53). This is the cleanest example in this checkout of MCP-as-secret-boundary: the sandbox process never sees the real Tavily key, only a proxied, namespaced tool surface the app-server brokers on its behalf.

Frontend-side MCP configuration is a full settings surface, not just a config file: `frontend/src/routes/mcp-settings.tsx`, `frontend/src/components/features/settings/mcp-settings/{mcp-server-list.tsx,mcp-server-list-item.tsx,mcp-server-form.tsx}`, mutation hooks `use-add-mcp-server.ts`/`use-update-mcp-server.ts`/`use-delete-mcp-server.ts`, and dedicated event-message rendering for MCP tool calls in chat (`frontend/src/components/features/chat/mcp-observation-content.tsx`, `event-message-components/mcp-event-message.tsx`). MCP servers are persisted per-org and per-user (`enterprise/migrations/versions/036_add_mcp_config_to_user_settings.py`, `103_add_mcp_config_to_org_member.py`), confirming MCP config is account-scoped state in the enterprise tier, not just a local file.

## Webhooks and event callbacks — the outward-facing trace

`openhands/app_server/event_callback/README.md:3` states the module's purpose directly: *"Manages webhooks and event callbacks for external system integration... allowing external systems to receive notifications when specific events occur within OpenHands conversations."* Components named in that README: `EventCallbackService` (abstract CRUD), `SqlEventCallbackService` (SQL-backed implementation), `EventWebhookRouter` (FastAPI router), with stated features including "event filtering by type and conversation," "callback result tracking and status monitoring," and "retry logic for failed webhook deliveries." This is the mechanism behind the README's own pitch of automations that "publish to Slack" or "decompose GitHub issues into tasks" (`README.md`, opening paragraph) — the event stream isn't just for the UI, it's the trigger source for an entire automation layer, backed by its own migration history (`enterprise/migrations/versions/117_add_event_callback_composite_index.py`).

## What's inside this submodule

| Path | What's there |
|---|---|
| `README.md` | States the repo split explicitly — Agent/Agent Server code is in `software-agent-sdk`; this repo is Agent Canvas |
| `pyproject.toml` | Pins `openhands-sdk`, `openhands-agent-server`, `openhands-tools` as exact-version PyPI deps — the agent runtime is consumed, not owned |
| `openhands/app_server/event/` | `EventService` ABC + filesystem/SQL/AWS/GCS backends; `event_router.py` (REST surface) |
| `openhands/app_server/event_callback/` | Webhook/callback subsystem — registration, filtering, retry, SQL storage |
| `openhands/app_server/mcp/mcp_router.py` | FastMCP server hosting OpenHands' own git-provider tools + Tavily proxy-mount |
| `openhands/app_server/app_conversation/` | Conversation lifecycle, `skill_loader.py` (thin proxy to agent-server's `/api/skills`), git integration |
| `openhands/app_server/integrations/` | GitHub/GitLab/Bitbucket/Bitbucket-DC/Azure DevOps service clients |
| `openhands/app_server/sandbox/` | Sandbox provisioning models (Docker/VM/cloud backend abstraction) |
| `.openhands/microagents/` | This repo's own two Skills (`documentation.md`, `glossary.md`) — the glossary is itself the clearest source-grounded definition of CodeAct/delegation terminology left in this checkout |
| `frontend/src/types/v1/core/events/` | TypeScript event type definitions mirroring the SDK's `Event` union, including ACP tool-call events |
| `frontend/src/stores/use-event-store.ts` | Zustand store bridging v0/v1 event schemas during the migration, timestamp-based ordering with defensive fallback |
| `frontend/src/components/features/settings/mcp-settings/` | Full MCP server CRUD UI |
| `frontend/src/hooks/use-acp-credential-form.ts`, `use-switch-acp-model.ts` | ACP (Agent-Client Protocol) credential + model-switch plumbing — the "any ACP agent" surface |
| `enterprise/` | Separately licensed (`enterprise/LICENSE`) — analytics, org-scoped MCP config, sharing services (AWS/GCS/filesystem-backed shared-event export), automation event service |
| `openhands/analytics/EVENTS.md` | Analytics event taxonomy (product telemetry, distinct from agent trajectory events) |
| `tests/unit/mcp/test_mcp_integration.py`, `tests/unit/app_server/test_webhook_router_acp.py` | Test coverage confirming MCP + ACP are actively exercised, not vestigial |

If you read three files: `README.md` in full (the transition banner changes how every other file should be read), `openhands/app_server/event/event_service.py` (the `EventService` ABC — the whole trajectory-persistence contract fits here), and `openhands/app_server/mcp/mcp_router.py` (both MCP-hosting and MCP-proxying patterns in one file).

## Mental model for using it well

- **Don't go looking for the CodeAct loop in this checkout.** It isn't here. If the question is "how does the think-act-observe cycle work," the answer lives in `OpenHands/software-agent-sdk`, pinned separately if this study ever needs that level of depth.
- **Treat `EventService` as the reusable pattern, not the `Event` schema itself.** The one-ABC/four-backends shape (filesystem for local, SQL/AWS/GCS for hosted) is a clean template for "same trajectory-consumer code, swappable persistence" — independent of what OpenHands specifically puts inside an `Event`.
- **MCP-as-secret-boundary is the reusable idea from `mcp_router.py`.** Mounting a proxied third-party MCP server under a namespace so the sandbox never sees the real API key is a pattern worth copying verbatim for any harness that needs to broker a paid API into an untrusted execution environment.
- **ACP is the "bring your own agent" seam.** If the goal is a control-plane that can front multiple agent *brands* (not just OpenHands' own), the ACP credential/model-switch hooks (`frontend/src/hooks/use-acp-credential-form.ts`, `use-switch-acp-model.ts`) are the reference shape for that boundary.
- **Skills are triggered, versioned, and agent-scoped, per the still-current frontmatter** (`name`, `type: knowledge`, `version`, `agent: CodeActAgent`, `triggers: [...]`) even though the loading implementation moved out — the schema itself is legible from `.openhands/microagents/documentation.md` alone.

## When NOT to reach for this

- **You want a single-repo, single-process agent to read end-to-end.** This checkout is now two services talking over HTTP/websocket, with the actual "brain" one `pip install` away. If the goal is "read one coherent agent loop," Aider (`Profile__Aider.md`) or opencode (`Profile__Opencode.md`) are the better single-repo reads in this study.
- **You need the event/trajectory *schema* itself, not just the storage pattern.** `Event`, `EventKind`'s upstream definitions, and the CodeAct action/observation types are defined in `software-agent-sdk`, not here — this repo only imports and stores them.
- **You're evaluating "how good is the agent at coding tasks."** That's an `software-agent-sdk` + benchmark question; this repo's tests are about webhook delivery, MCP routing, and conversation lifecycle, not agent task performance.
- **You want a lightweight CLI tool.** OpenHands/Agent Canvas is explicitly a "developer control center" — FastAPI app-server, React frontend, sandbox provisioning, optional enterprise analytics. The minimum viable install is `npm install -g @openhands/agent-canvas` plus `uv`, not a single binary.

## How this compares to the rest of the study

| Axis | OpenHands (Agent Canvas) | opencode | Aider |
|---|---|---|---|
| **Shape** | Split: control-plane app-server + frontend, agent runtime is an external versioned dependency (`software-agent-sdk`) | CLI-first harness, owns its full stack including MCP client | CLI, git-native pair-programmer, no server at all |
| **Where the agent loop lives** | **Not in this repo** — `openhands-sdk`/`openhands-agent-server` pip packages | In-repo, single process | In-repo, single process |
| **Trajectory persistence** | `EventService` ABC — filesystem (1 JSON file/event), SQL, AWS, GCS backends, selected by deploy config | Structured, readable session storage (JSON/DB-backed) | Flat append-only markdown chat-history file; git commits as the real trace |
| **MCP support** | **Both host and proxy** — hosts its own git-provider tools as MCP tools, and mounts third-party MCP servers (Tavily) under a namespace to hide API keys from the sandbox | Native — opencode owns its MCP client | **None** — no MCP anywhere in source or docs |
| **Multi-agent / delegation model** | Documented in `.openhands/microagents/glossary.md` (Agent Delegation, Agent Hub) but implemented upstream; this repo's Skills are proxied via `/api/skills` | N/A noted in this study | N/A — no tool-call loop at all |
| **Bring-your-own-agent** | **Yes, explicitly** — ACP support fronts Claude Code, Codex, Gemini, or any ACP-compatible agent, in addition to OpenHands' own | No — opencode is its own agent | No — Aider is its own agent |
| **Automation surface** | Webhook/event-callback subsystem with retry logic; drives Slack/GitHub/scheduled automations | Not a focus | Not a focus |
| **Best fit** | Teams wanting a self-hosted, multi-backend control plane that can run several agent brands and wire automations off agent events | Terminal-first agent work needing a real, swappable MCP toolset in one process | Git-repo-centric solo/pair coding where commit history as audit trail is valuable |

The crucial axis OpenHands owns in this study is **the control-plane/agent-runtime split itself** — no other entry has deliberately separated "the thing that runs the agent" from "the thing that shows you what the agent did and lets you swap which agent it is," with the boundary enforced by a versioned package dependency rather than an in-process interface. That is a different answer to this study's underlying question (how should a harness be shaped) than either opencode's "own the whole stack in one CLI" or Aider's "no stack at all, just git."

## One-line summary

> The `OpenHands/OpenHands` submodule pinned in this study is Agent Canvas, not the OpenHands agent — a self-hosted FastAPI control-plane + React frontend that persists agent trajectories through a pluggable `EventService` (filesystem/SQL/AWS/GCS backends, one `Event`-per-file locally), hosts and proxies MCP tools (its own git-provider tools plus a namespace-mounted Tavily proxy that hides API keys from the sandbox), and drives OpenHands' own CodeAct agent or any ACP-compatible agent (Claude Code, Codex, Gemini) — while the CodeAct loop and micro-agent/Skill execution that made OpenHands famous now live one versioned dependency away, in the separate `OpenHands/software-agent-sdk` repo, imported here (`from openhands.sdk import Event`, `from openhands.sdk.skills import Skill`) rather than owned.
