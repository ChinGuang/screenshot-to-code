# Proposal: Resume interrupted generations

Status: draft for review
Author: platform team

## Context

A generation runs as a WebSocket session on `/generate-code` (`backend/routes/generate_code.py`). It creates up to 4 variants in parallel, and each variant runs an agent loop (`backend/agent/engine.py`) that streams code to the browser with `setCode`, `toolStart` and `toolResult` messages.

If the connection drops (laptop sleep, Wi-Fi change, closing the tab by mistake), the work is lost. The user has to start again and pay for the whole generation a second time. Generations often take 1–3 minutes and can cost up to a few dollars, so this happens more than we would like.

## Goals

- If the browser disconnects, a generation can be resumed from where it stopped, with no repeated LLM calls for work already done.
- Resuming works after a page reload, and from another tab.
- The existing cost limit keeps protecting users: no generation can spend more than `GENERATION_MAX_COST_USD`.

## Non-goals

- Resuming after the backend process restarts. (Snapshots on disk would make this possible later.)
- Sharing a generation between different users.

## Design

### 1. Generation ids

When a generation starts, the backend assigns it an id and sends it to the frontend in a new `generationStarted` message. Ids are a counter stored in `{data dir}/generations/next_id`, so they are short and easy to read in logs (`gen-1042`, `gen-1043`, …). The frontend keeps the current id in `localStorage`.

### 2. Snapshots

For each variant, the engine writes a snapshot to `{data dir}/generations/{id}/variant-{n}.json`. A snapshot contains everything needed to continue the agent loop:

- the original request `params` (input images, stack, prompt, settings, model choice),
- the full message history sent to the model so far (including image content),
- the current file state (the latest HTML),
- the list of completed tool calls and their results.

To make sure no progress is lost, the snapshot is rewritten after every streamed event: every `setCode` chunk, every tool start and every tool result.

### 3. Resuming

A new WebSocket route, `/resume-generation`, takes `{ generationId }`. It loads the snapshots of all unfinished variants and, for each one:

1. rebuilds the provider session from the saved message history,
2. replays the saved file state to the browser with `setCode`,
3. continues the agent loop from the last saved event.

If the last saved event was a tool start without a matching tool result, the tool call is simply run again so the loop can continue.

The frontend calls `/resume-generation` automatically when it reconnects and finds an unfinished generation id in `localStorage`. Because the id is in `localStorage`, any open tab of the app can pick it up.

### 4. Cost limit

The engine already aborts a variant when its spend exceeds `GENERATION_MAX_COST_USD` (`$3.00`). Resumed variants go through the same engine code, so the limit applies to resumed generations automatically and no change is needed.

### 5. API

| Route | Purpose |
| --- | --- |
| `WS /generate-code` | Unchanged, plus the new `generationStarted` message |
| `WS /resume-generation` | Resume an unfinished generation by id |
| `GET /api/generations/{id}` | Return the snapshot of each variant, for debugging and for the frontend to show progress before resuming |

## Rollout

1. Ship snapshots behind a `RESUME_ENABLED` flag, writing only.
2. Enable `/resume-generation` for local development.
3. Enable by default in the next release.

## Alternatives considered

- **Keep the WebSocket session alive on the server for 5 minutes after a disconnect, then reattach.** Simpler, but it doesn't survive a page reload, and memory use grows with abandoned sessions.
- **Snapshot only after each finished turn.** Fewer writes, but a drop in the middle of a long streamed turn would lose up to a minute of work.

## Open questions

- Should snapshots be compressed?
- How should the UI show a generation that is being resumed in another tab?
