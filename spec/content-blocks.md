# 内容块与不透明负载

## ContentBlock 类型

```ts
ContentBlock =
  | { type: "text"; text }
  | { type: "thinking"; text; signature? }        // resume 关键：signature 需原样往返
  | { type: "tool-call"; toolCallId; name; arguments }
  | { type: "tool-result"; toolCallId; toolResultId; content; isError }
  | { type: "image"; mimeType; data?; uri?; alt? }
  | { type: "reference"; uri; title?; mediaType? }
  | { type: "error"; message; code?; detail? }
  | { type: "unknown"; nativeType; value }        // 不透明负载容器
```

## 不透明负载（opaque passthrough）——一等公民

agent 生态里有大量**不可解析但必须原样往返**的字节：

- claude 的 thinking `signature`（Anthropic API 侧 resume 校验必需）
- codex 的 reasoning `encrypted_content`（OpenAI 服务端校验链，~1KB blob）

这类负载若被 normalize 即丢失 resume 能力。ASP 的规则：

1. **typed `unknown` block 承载**：`{ type: "unknown", nativeType, value }`，`nativeType` 标注来源（如 `codex.encrypted_reasoning`）。
2. **canonical→native→canonical 必须幂等**：任何 importer/exporter 重编码路径都**不得二次包裹** `unknown` block——否则 `nativeType` 被覆盖为 `"unknown"`，负载丢失。

> 实测教训：e-core 的 `normalizeContent` 与 pi 的 importer/exporter 都曾二次包裹 canonical `unknown` block，导致 codex `encrypted_content` 经 pi 一趟丢 `nativeType`。三处已修，规则固化进 schema 校验。

## 与 MCP/ACP 的差异

| ASP | MCP / ACP |
|---|---|
| `thinking`（含 signature）是一等块 | ACP v2 把 thinking 降级为 config + 统计，无常设块 |
| `unknown` 是保真容器（resume 关键） | `Other` 块（`_` 前缀）保留原始 payload，语义一致 |

## tool result 的 canonical 表示

`tool-result` 的内容同时存在于**工具实体 `result`** 与 message 块。规则：**工具结果一律以工具实体的 `result` 为准**（canonical 表示，与 message 块形状解耦）——因为不同 harness 对 tool result 的块形状不一致（pi 用 `[text]` 块，codex 用 `[tool-result]` 块），exporter 必须从工具实体取，不能依赖 message 块形状。
