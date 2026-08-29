> [English](../overview.md) · [中文](overview.md)

# 会话模型与事件（overview）

## 分层

```
L0 raw blob 层     content-addressed，保留 harness 原生字节（import 即快照）
L1 canonical event log  统一 envelope，append-only，唯一持久化真源
L2 projections    读模型（chat 渲染 / resume-export / 派生索引），可随时重建
```

## 会话层级

```
Session ──1:N──> Run ──1:N──> Turn ──1:N──> Message ──> ordered ContentBlocks
                         └──> ToolCall / ToolResult（稳定 ID，关联到 run/turn/message）
```

- **Session**：一次会话，`nativeSessionId`（harness 原生 id）+ `id`（内容寻址的 canonical id）。
- **Run**：一次"执行单元"（harness 的 run / turn 边界语义各不相同，见 fidelity）。
- **Turn**：一轮用户输入 → agent 产出。
- **Message**：`role ∈ {system, user, assistant, tool, unknown}` + 有序内容块。
- **Tool**：`tool-call` + `tool-result` 配对，结果存工具实体的 `result`（canonical 表示，与 message 块形状解耦）。

## Session lineage DAG

session 不是平表。关系类型：

```
relation { id, sourceSessionId, targetSessionId, type, typeData, createdAt }
type ∈ { fork, resume, handoff, subagent, compact }
```

- **fork**：同一会话分叉出两条线。
- **resume**：同一 harness 内继续。
- **handoff**：跨 harness 导出→导入（**新建 session + handoff relation**，不伪装同一 session）。
- **subagent**：spawn 自某个 run 的某个 tool call（`typeData{parentRunId, parentToolCallId}`）。
- **compact**：compaction 产生的旧→新血缘。

## 事件 envelope

```ts
AgentEventEnvelope {
  schemaVersion, eventId, type, sessionId, timestamp, entityRevision?,
  source: { layer: L1|L2|L3, adapter, nativeEventId?, authenticated, generation? },
  authority: authoritative|self_reported|inferred, confidence: 0..1,
  correlation?: { agentSessionId, runId, turnId, messageId, toolCallId, toolResultId, parentId, clientActionId },
  payload,
}
```

事件类型 27 种：`session.*`（started/ended/bound）、`run.*`（started/settled）、`turn.*`、`message.*`（started/updated/completed）、`tool.*`、`approval.*`、`artifact.observed`、`task.updated`、`action.*`、`capabilities.changed`、`input.observed`、`status.changed`、`unknown.observed`。

**关键语义**：
- `entityRevision` + delta：`message.updated` 只能应用于明确的 entity revision，completed/settled 是 sticky terminal。
- `authority/confidence`：L1（一手记录）> L2（代理转述）> L3（推断）；赢者决定 canonical payload。
- `unknown.observed`：不认识的事件原样保留（保真，不丢弃）。

## Identity 与幂等

- 事件 id 内容寻址（`stableId(sessionId, nativeEventId, type)`）→ 同一来源 re-import 幂等。
- Session 的 canonical id = `sha256(harness ‖ native_session_id ‖ source_sha256)`（长度前缀，防跨 harness 同 slug 撞 key）。
- 事件 id 全局幂等：重放/重复 delivery 用 eventId 去重。

详见 `schema/agent-session-contracts.ts`（`validateAgentEnvelope` 是 exact-schema 校验的事实源）。
