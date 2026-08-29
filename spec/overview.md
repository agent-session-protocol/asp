# Session Model & Events

> [English](overview.md) · [中文](zh-CN/overview.md)

## Layers

```
L0 raw blob          content-addressed native bytes (import = snapshot)
L1 canonical log     unified envelope, append-only, single durable source of truth
L2 projections       read models (chat render / resume-export / derived indexes), rebuildable
```

## Session hierarchy

```
Session ──1:N──> Run ──1:N──> Turn ──1:N──> Message ──> ordered ContentBlocks
                         └──> ToolCall / ToolResult (stable IDs, correlated to run/turn/message)
```

- **Session**: one conversation; `nativeSessionId` (harness-native) + `id` (content-addressed canonical id).
- **Run**: an execution unit (harnesses differ in how they delimit run/turn — see fidelity).
- **Turn**: one user input → agent output.
- **Message**: `role ∈ {system, user, assistant, tool, unknown}` + ordered content blocks.
- **Tool**: `tool-call` + `tool-result` pairing; the result lives on the tool entity's `result` (canonical, decoupled from message block shape).

## Session lineage DAG

Sessions are not a flat table. Relation types:

```
relation { id, sourceSessionId, targetSessionId, type, typeData, createdAt }
type ∈ { fork, resume, handoff, subagent, compact }
```

- **fork**: a session branches into two lines.
- **resume**: continuation within the same harness.
- **handoff**: export→import across harnesses (creates a **new session + handoff relation**, never masquerades as the same session).
- **subagent**: spawned from a run's tool call (`typeData{parentRunId, parentToolCallId}`).
- **compact**: old→new lineage produced by compaction.

## Event envelope

```ts
AgentEventEnvelope {
  schemaVersion, eventId, type, sessionId, timestamp, entityRevision?,
  source: { layer: L1|L2|L3, adapter, nativeEventId?, authenticated, generation? },
  authority: authoritative|self_reported|inferred, confidence: 0..1,
  correlation?: { agentSessionId, runId, turnId, messageId, toolCallId, toolResultId, parentId, clientActionId },
  payload,
}
```

27 event types: `session.*` (started/ended/bound), `run.*` (started/settled), `turn.*`, `message.*` (started/updated/completed), `tool.*`, `approval.*`, `artifact.observed`, `task.updated`, `action.*`, `capabilities.changed`, `input.observed`, `status.changed`, `unknown.observed`.

**Key semantics**:
- `entityRevision` + deltas: `message.updated` applies only to an explicit entity revision; completed/settled are sticky terminals.
- `authority/confidence`: L1 (first-hand) > L2 (relayed) > L3 (inferred); the winner determines the canonical payload.
- `unknown.observed`: unrecognized events are preserved verbatim (never dropped).

## Identity & idempotency

- Event ids are content-addressed (`stableId(sessionId, nativeEventId, type)`) → re-importing the same source is idempotent.
- Session canonical id = `sha256(harness ‖ native_session_id ‖ source_sha256)` (length-prefixed, to prevent cross-harness same-slug collisions).
- Event ids are globally idempotent: replays/duplicates are deduplicated by eventId.

See `schema/agent-session-contracts.ts` (`validateAgentEnvelope` is the source of truth for exact-schema validation).
