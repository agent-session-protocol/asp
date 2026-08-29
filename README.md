# Agent Session Protocol (ASP)

> **Agent Session Protocol (ASP) 是 agent session 的存储与迁移协议——[ACP](https://agentclientprotocol.com) 的 session 层姊妹；ACP 管实时（编辑器↔agent 的当下交互），ASP 管会话的持久与跨 runtime 迁移。**

ASP 只描述**协议层**：会话如何被描述、事件如何建模、保真度如何声明、不透明负载如何往返。它**不是**存储引擎、不是转换工具、也不是实时控制协议——那些是[实现层](../../universal-session-log)的事。

## 一句话可引用定义

> ASP 定义 agent session 的 canonical schema、事件语义、保真度矩阵与不透明负载往返约定，使任意 harness 产生的会话可以被持久化、回放、并在 harness 之间迁移。

## 仓库布局

```
asp/
├── schema/
│   ├── agent-session-contracts.ts      # canonical schema：envelope、事件类型、内容块、校验（TS 参考实现）
│   └── agent-session-materializer.ts   # evidence → snapshot 的投影语义
└── spec/
    ├── overview.md        # 会话模型、事件、correlation、lineage、identity
    ├── content-blocks.md  # 内容块 + 不透明负载（opaque passthrough）
    ├── fidelity.md        # 保真度矩阵 + loss 声明 + resume-profile
    └── acp-mcp.md         # 与 ACP / MCP 的关系与映射
```

## 三个核心设计原则

1. **正确性只来自 append log + evidence**：事件日志是唯一真源，快照永远可由 evidence 重放重建。
2. **不透明负载一等化**：claude 的 thinking `signature`、codex 的 `encrypted_content` 这类"不可解析但必须原样往返"的字节，用 typed `unknown` block 保真，不在转换时 normalize 掉。
3. **转换保真靠 evidence 层**：roundtrip 时优先用 importer 存下的原生负载逐字还原，缺失才合成并声明 loss。

## 实现

存储优先的参考实现见 [`agent-session-protocol/universal-session-log`](https://github.com/agent-session-protocol/universal-session-log)（USL：`usl-core` 存储引擎 + `usl-capture` 捕获 + `usl-convert` 跨 harness 转换）。

## 状态

**Draft（v1.0 schema）**。schema 以 TypeScript 为当前参考实现，JSON Schema 制品与 SDK 生成在路线图上。
