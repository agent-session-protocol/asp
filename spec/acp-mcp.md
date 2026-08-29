# 与 ACP / MCP 的关系

## 定位：三层各司其职

| 协议 | 管什么 | 时间轴 | 本质 |
|---|---|---|---|
| **ACP**（Agent Client Protocol） | 编辑器↔agent 的实时控制（prompt/steer/cancel/permission） | 当下 | wire protocol |
| **MCP**（Model Context Protocol） | agent↔工具的调用 | 当下 | tool protocol |
| **ASP**（Agent Session Protocol） | 会话的持久、回放、跨 runtime 迁移 | 历史 | 存储与迁移协议 |

**互补不竞争**：ACP 管实时，MCP 管工具，ASP 管会话的持久与迁移。ASP 是 ACP/MCP 生态缺失的那层——"durable back"。

## 与 ACP 的关系

ACP v2（2026-07-20 draft）的关键设计恰好与 ASP 同构：

- ACP 的"超越 turn + 多 client 观察" ⇄ ASP 的 append-only 事件日志 + snapshot/cursor
- ACP 的"稳定 ID + patch 语义（省略/null/值/chunk-append）" ⇄ ASP 的 `entityRevision` + delta 合并
- ACP 的 `Other` 块（"receiver 必须保留原始 payload"）⇄ ASP 的 typed `unknown` block

**集成方向**：
1. **ACP tap**（写路径）：ASP 实现方充当 ACP client，订阅 `session/update` → 映射成 ASP 事件 → 入存储。这是最优 live capture（harness 中立、语义级、无 FUSE/特权问题）。
2. **ACP agent 适配器**（读路径）：ASP 实现方充当 ACP agent，把存储的会话经 `session/resume` + `session/update` 重放给编辑器。

**分歧点**：ACP v2 移除了 thinking content block（降级为 `thought_level` 配置 + `thought_tokens` 统计）；ASP 把 thinking（含 signature）当 resume 关键一等块。走 ACP tap 时 thinking 只能进 `Other` 被动保留，是显式的保真度折损点。

## 与 MCP 的关系

ACP v2 的 `ContentBlock` 复用 MCP 的 schema（Text/Image/Audio/ResourceLink/Resource/Other）。ASP 的 ContentBlock 命名自成一格（thinking/tool-call/tool-result 是 resume 关键，MCP 没有），映射见下。

## 内容块映射

| ASP ContentBlock | ACP/MCP ContentBlock | 备注 |
|---|---|---|
| `text` | `Text` | 直映射 |
| `image` | `Image` / `Audio` | 直映射（audio 用 unknown 或扩展） |
| `reference` | `ResourceLink` / `Resource` | 直映射 |
| `unknown` | `Other`（`_` 前缀） | **双方都要求 opaque passthrough，语义一致** |
| `thinking`(+signature) | （无） | ACP v2 移除，只能进 `Other` |
| `tool-call`/`tool-result` | `ToolCall*` | 映射到 tool 事件 + 工具实体 |
