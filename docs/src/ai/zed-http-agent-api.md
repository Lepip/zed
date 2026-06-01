---
title: Zed HTTP Agent API (Draft)
description: HTTP API contract for creating, prompting, monitoring, and closing Zed agent sessions from an external orchestrator.
---

# Zed HTTP Agent API

This page defines the HTTP API contract expected by the tracker service for managing Zed agent sessions programmatically. It covers session lifecycle endpoints, webhook events, status mapping, and rollout behavior.

Version: **1.0.0-draft**  
Target: tracker autonomous milestone (Phase 8)

> **Note:** This specification includes tracker-side integration expectations. Some items are marked as planned and may not be implemented in your current build.

## Overview {#overview}

The tracker calls the Zed HTTP API for agent lifecycle operations:

- create session
- send prompt
- poll status
- close session

MCP remains the path for agent actions (`create_task`, `attach_artifact`, and similar tool calls).

```text
Tracker (Next.js)                Zed Fork (HTTP :8765)
┌──────────────────┐            ┌──────────────────────────┐
│ zed-runtime.ts   │───POST───→│ /agents                  │
│                  │───POST───→│ /agents/:id/prompt       │
│                  │───GET────→│ /agents/:id              │
│                  │───DELETE─→│ /agents/:id?reason=…     │
│                  │←─webhook──│ (Zed → tracker)          │
│                  │───GET────→│ /healthz                 │
└──────────────────┘            └──────────────────────────┘
                              webhook → POST /api/zed/events
```

### Completion model (MVP)

| Responsibility                                  | Owner                                                                      |
| ----------------------------------------------- | -------------------------------------------------------------------------- |
| Launch session, send work prompt, close session | Tracker via Zed HTTP (`zed-runtime`)                                       |
| Observe prompt start and finish, session exit   | Tracker via webhooks (primary)                                             |
| Final dispatch and task transition handling     | Tracker `reportHandoffResult` service, invoked on `agent.prompt_completed` |
| Mid-task tool actions (claim, chat, artifacts)  | Worker Zed agent via MCP (`TRACKER_MCP_AGENT_ID`)                          |

## Environment variables (tracker) {#environment-variables}

| Variable                  | Required               | Purpose                                                              |
| ------------------------- | ---------------------- | -------------------------------------------------------------------- |
| `ZED_HTTP_URL`            | No                     | Base URL for the Zed fork, for example `http://localhost:8765`       |
| `TRACKER_URL`             | In HTTP mode           | Public tracker base URL used to build `{TRACKER_URL}/api/zed/events` |
| `ZED_WEBHOOK_SECRET`      | Recommended            | Shared secret used by tracker and Zed webhook header                 |
| `OPERATOR_TOKEN`          | Yes (existing)         | Passed through agent `env` for tracker API calls                     |
| `TRACKER_WORKDIR_WINDOWS` | Recommended on Windows | Unified operation directory for Windows                              |
| `TRACKER_WORKDIR_LINUX`   | Recommended on Linux   | Unified operation directory for Linux                                |
| `TRACKER_WORKDIR`         | Optional fallback      | Cross-platform fallback workdir                                      |
| `ZED_WORKDIR`             | Legacy fallback        | Backward-compatible fallback for older setups                        |

Zed HTTP paths below are relative to `ZED_HTTP_URL`.

## Required endpoints {#required-endpoints}

### Health check {#health-check}

`GET /healthz`

Response: `200 OK` with `{ "ok": true }` (or equivalent).

The tracker caches the health result for 60 seconds and does not call this before every operation.

### Create agent session {#create-agent-session}

`POST /agents`

Request body:

```json
{
  "workdir": "/absolute/path/to/tracker",
  "model": "gpt-5.5",
  "thinking": true,
  "reasoning_effort": "medium",
  "mcp_servers": ["tracker"],
  "env": {
    "TRACKER_MCP_AGENT_ID": "agent-clx123abc",
    "OPERATOR_TOKEN": "abc123..."
  },
  "webhook_url": "http://localhost:3000/api/zed/events",
  "webhook_secret": "same-as-ZED_WEBHOOK_SECRET",
  "metadata": {
    "tracker_agent_id": "agent-clx123abc",
    "tracker_task_id": "task-456",
    "tracker_dispatch_id": "dispatch-789"
  }
}
```

Response (`201 Created`):

```json
{
  "id": "zed-session-abc123",
  "status": "idle",
  "created_at": "2026-05-29T12:00:00Z"
}
```

Error responses should use standard HTTP status codes with a JSON error body such as `{ "error": "..." }`.

### Send prompt to agent {#send-prompt}

`POST /agents/:id/prompt`

Request body:

```json
{
  "prompt": "You are a worker agent. ...",
  "timeout_seconds": 900
}
```

#### MVP mode: asynchronous (required)

- return `202 Accepted` immediately
- run prompt in background
- emit `agent.prompt_completed` webhook when done

Example `202` response:

```json
{
  "accepted": true,
  "status": "working",
  "session_id": "zed-session-abc123",
  "output": "Latest assistant output text so far (optional)"
}
```

The tracker must use webhooks as the completion signal.

#### Optional sync mode (debug only)

If `Prefer: respond-sync` (or `?sync=true`) is sent, Zed may block and respond with:

```json
{
  "output": "Full agent response text...",
  "done": true,
  "status": "completed",
  "usage": {
    "input_tokens": 1234,
    "output_tokens": 567
  },
  "duration_ms": 45200,
  "error": null
}
```

#### Optional streaming mode

`GET /agents/:id/stream` may provide SSE events:

```text
data: {"type":"thought","content":"..."}
data: {"type":"output","content":"..."}
data: {"type":"done","output":"...","usage":{...}}
```

### Get agent status {#get-agent-status}

`GET /agents/:id`

Example response:

```json
{
  "id": "zed-session-abc123",
  "status": "working",
  "workdir": "/path/to/tracker",
  "model": "gpt-5.5",
  "created_at": "2026-05-29T12:00:00Z",
  "last_active_at": "2026-05-29T12:05:00Z",
  "usage": {
    "input_tokens": 5000,
    "output_tokens": 1200
  },
  "metadata": {
    "tracker_dispatch_id": "dispatch-789"
  },
  "current_prompt": "...",
  "current_output": "..."
}
```

This endpoint is primarily for reconciliation if webhook delivery is missed.

### Close or cancel session {#close-or-cancel-session}

`DELETE /agents/:id?reason=dispatch%20completed`

Zed may also accept an optional JSON body `{ "reason": "..." }` for clients that send DELETE bodies.

Response: `200 OK` or `204 No Content`.

After close, `GET /agents/:id` should return `404`.

### Lifecycle webhooks (Zed -> tracker) {#lifecycle-webhooks}

If `webhook_url` is supplied at create time, Zed should send lifecycle events:

`POST {webhook_url}`

Headers:

- `Content-Type: application/json`
- `X-Zed-Webhook-Secret: {webhook_secret}`

Event examples:

```json
{
  "event": "agent.started",
  "session_id": "zed-session-abc123",
  "timestamp": "2026-05-29T12:00:01Z",
  "metadata": {
    "tracker_agent_id": "...",
    "tracker_dispatch_id": "..."
  }
}
```

```json
{
  "event": "agent.prompt_started",
  "session_id": "zed-session-abc123",
  "timestamp": "2026-05-29T12:00:05Z",
  "metadata": {}
}
```

```json
{
  "event": "agent.prompt_completed",
  "session_id": "zed-session-abc123",
  "timestamp": "2026-05-29T12:05:00Z",
  "status": "completed",
  "output": "Agent final response text...",
  "usage": {
    "input_tokens": 1234,
    "output_tokens": 567
  },
  "error": null,
  "metadata": {}
}
```

```json
{
  "event": "agent.exited",
  "session_id": "zed-session-abc123",
  "timestamp": "2026-05-29T12:06:00Z",
  "reason": "session_closed",
  "metadata": {}
}
```

```json
{
  "event": "agent.error",
  "session_id": "zed-session-abc123",
  "timestamp": "2026-05-29T12:03:00Z",
  "error": "MCP server disconnected",
  "recoverable": true,
  "metadata": {}
}
```

Delivery guidance:

- fire-and-forget
- retry up to 3 times with 1 second backoff on non-2xx
- support per-session `webhook_url`

## Nice-to-have endpoints {#nice-to-have-endpoints}

### List models {#list-models}

`GET /models`

```json
{
  "models": [
    {
      "id": "gpt-5.5",
      "provider": "openai",
      "available": true
    }
  ]
}
```

### List sessions {#list-sessions}

`GET /agents`

Returns an array of session objects (same shape as `GET /agents/:id`).

## Cap enforcement and identity (1:1 model) {#cap-enforcement-and-identity}

When HTTP mode is enabled:

`1 started dispatch -> 1 tracker Agent row -> 1 Zed session (zedSessionId)`

Queued dispatches do not consume cap slots until promoted to `started` and a Zed session is created.

## Status vocabulary mapping {#status-vocabulary-mapping}

| Zed session status | Prompt completion status | Tracker `Agent.status` | Notes                              |
| ------------------ | ------------------------ | ---------------------- | ---------------------------------- |
| `idle`             | —                        | `idle`                 | Ready for prompt                   |
| `working`          | —                        | `working`              | Prompt in flight                   |
| `done`             | `completed`              | `idle` after handoff   | Prompt completed                   |
| `error`            | `error`                  | `failed` or `idle`     | Depends on recoverability          |
| —                  | `timeout`                | `idle`                 | Dispatch usually failed or retried |

Treat `agent.prompt_completed.status` as authoritative for dispatch outcome.

## State reconciliation {#state-reconciliation}

If webhooks are missed:

1. poll `GET /agents/:id` for active `zedSessionId` values
2. synthesize completion handling from Zed state
3. if `404`, clear `zedSessionId` and normalize tracker state

## Tracker integration expectations {#tracker-integration-expectations}

When `ZED_HTTP_URL` is configured, tracker should:

1. cache `/healthz` for 60 seconds
2. launch Zed runtime on dispatch transition to `started`
3. include webhook URL and secret in create payload
4. pass `TRACKER_MCP_AGENT_ID` and `OPERATOR_TOKEN` in `env`
5. send prepared handoff prompt to `POST /agents/:id/prompt`
6. consume `agent.prompt_completed` webhook as completion signal
7. run `reportHandoffResult` from webhook payload
8. close session after handoff processing
9. validate `X-Zed-Webhook-Secret` in tracker webhook route
10. fall back to legacy behavior if `ZED_HTTP_URL` is unset

## Non-requirements {#non-requirements}

Out of scope for MVP:

- long-blocking prompt responses in production path
- bidirectional streaming as a hard requirement
- file upload or download over this API
- agent-to-agent HTTP messaging
- API authentication for localhost-only Zed MVP

## Backward compatibility {#backward-compatibility}

With `ZED_HTTP_URL` unset:

- existing DB and scheduler flow remains unchanged
- workers can continue using pull-based task discovery

With `ZED_HTTP_URL` set:

- runtime creates and tracks Zed sessions
- webhooks drive completion and session lifecycle updates

## Implementation order (tracker side) {#implementation-order}

| Sequence | Task                                                               | Depends on |
| -------- | ------------------------------------------------------------------ | ---------- |
| 1        | Add env vars (`ZED_HTTP_URL`, `TRACKER_URL`, `ZED_WEBHOOK_SECRET`) | —          |
| 2        | Add Prisma field `Agent.zedSessionId`                              | —          |
| 3        | Add webhook route `POST /api/zed/events` with secret validation    | 1          |
| 4        | Add service `src/server/services/zed-runtime.ts`                   | 1, 2       |
| 5        | Wire create and queue processing to `zed-runtime.launch`           | 4          |
| 6        | Trigger `reportHandoffResult` on `agent.prompt_completed`          | 3, 4       |
| 7        | Update cap logic and reconciliation on `agent.exited`              | 2, 3       |
| 8        | Add task chat system messages for lifecycle events                 | 3          |
| 9        | Add reconciliation job using `GET /agents/:id`                     | 4          |
| 10       | Update architecture and API docs                                   | —          |

## Deprecation note {#deprecation-note}

`ORCHESTRATOR.md` remains as fallback guidance until HTTP mode is stable in production.
