---
title: "Agent Harnesses — Answers to the Plan's Targeted Questions"
lede: "Phase 2 of the study plan. Not a survey of the study — three specific questions, answered by citing the profiles already written, in service of the Phase 3 synthesis decision."
date_created: 2026-07-13
authors:
  - Michael Staton
augmented_with:
  - Claude Code on Claude Sonnet 5
semantic_version: 0.0.0.1
tags:
  - Inquiry
  - Agent-Harnesses
  - MCP
  - Skills
status: Draft
---

# Targeted Questions — Agent Harnesses

Per `ai-labs/context-v/plans/Study-Agent-Harnesses-and-Conversational-UI-Before-Cross-Product-Shell.md`, Phase 2. Answers cite the profiles in `context-v/profiles/`, not the upstream repos directly — the profiles already carry the `file:line` citations.

## Q1 — How does opencode/goose decide which MCP servers + tools are in scope for a given session — is there a live enable/disable model we can copy for "swap context"?

**opencode** (`Profile__Opencode.md`, "Deep dive 4"): tools from built-ins + plugins + MCP servers converge into **one registry**, then `Permission.visibleTools()` filters the visible set per-agent via **wildcard pattern rules** — not a static allow-list. Scoping is a filter function applied at read-time over a unified registry, which means "swap context" could be implemented as swapping the wildcard rule set, not rebuilding the registry.

**goose** (`Profile__Goose.md`): every extension — including in-process `Platform`/`Builtin` variants that don't even spawn a subprocess — speaks the same `McpClientTrait` (`ExtensionConfig`, `crates/goose/src/agents/extension.rs:161-304`), scoped by a per-extension `available_tools` allowlist, gated further by a **three-tier runtime permission model**: `AlwaysAllow` / `AskBefore` / `NeverAllow` (`crates/goose/src/config/permission.rs:19-22`).

**Verdict for our shell:** goose's three-tier gate is the cleaner model to copy — it separates "is this tool discoverable" (extension config) from "is this tool allowed to fire without confirmation" (permission tier), which maps directly onto "swap which product's MCP servers are visible" (discoverability) vs. "does the user need to approve this specific call" (per-capability trust, echoing augment-it's own `requiresUserConfirmation` capability flag per the walking-skeleton plan).

## Q2 — How does OpenHands's event-stream/trajectory log map onto a cross-product trace/audit log?

**OpenHands** (`Profile__OpenHands.md`): trajectory persistence is a real pluggable abstraction — `EventService` (ABC, `openhands/app_server/event/event_service.py`) with four interchangeable backends (filesystem — one JSON file per event via `model_dump_json`/`model_validate_json`; SQL; AWS; GCS). Notably, the checked-out repo itself is no longer the agent — it re-imports `Event`/`Skill` types from a separate `openhands-sdk` package, i.e. OpenHands already went through the "extract the core into a reusable package, keep the app as a thin control plane" move we're contemplating for `@lossless/in-app-agent`.

**Verdict for our shell:** the `EventService` ABC-with-swappable-backend shape is a good template for a cross-product trace log — one event schema, pluggable sink (start with filesystem JSON per event, matching memopop-orchestrator's existing `.logs/runs/{job_id}__*.jsonl` pattern per the walking-skeleton plan), upgradeable to SQL later without touching the event schema.

## Q3 — How does continue's `.continue/` config loading handle per-project skill/rule swaps, and does anything replace it now that it's archived?

**continue** (`Profile__Continue.md`, flagged archived/read-only in its own README): `ConfigHandler` scans `.continue/{assistants,agents,configs}/*.yaml` as independent "profiles"; `setSelectedProfileId` switches the active one **mid-session, no restart**, persisted per-workspace. The honest caveat, from the code itself: switching profiles forces a **full MCP-manager reload**, not a cheap pointer swap. Separately, `MCPManagerSingleton.setConnections()` reconciles by **diffing** server identity (command+args+env for stdio, url for sse/http) — only tearing down/rebuilding what actually changed, plus an independent `setEnabled(id, bool)` per-server toggle.

**Verdict for our shell:** two independently useful patterns survive even though continue itself is dead: (1) profile-swap triggers a **full reload of the tool/MCP layer** — don't assume a cheap adapter pointer-swap is sufficient, budget for real teardown/rebuild cost on context switch; (2) the **diff-based reconciliation** (compare by connection identity, only touch what changed) is the right shape for "switching from augment-it context to dididecks context" when some MCP servers might be shared across both.

## Related

- `ai-labs/context-v/plans/Study-Agent-Harnesses-and-Conversational-UI-Before-Cross-Product-Shell.md`
- `context-v/profiles/Profile__Opencode.md`, `Profile__Goose.md`, `Profile__OpenHands.md`, `Profile__Continue.md`
- `ai-labs/context-v/explorations/Cross-Product-Native-Agent-Shell.md` — where these verdicts get used
