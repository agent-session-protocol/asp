# Relationship to ACP / MCP

> [English](acp-mcp.md) · [中文](zh-CN/acp-mcp.md)

## Positioning: three layers, three jobs

| protocol | owns | timeline | essence |
|---|---|---|---|
| **ACP** (Agent Client Protocol) | editor↔agent live control (prompt/steer/cancel/permission) | present | wire protocol |
| **MCP** (Model Context Protocol) | agent↔tool invocation | present | tool protocol |
| **ASP** (Agent Session Protocol) | session persistence, replay, cross-runtime migration | past | storage & migration protocol |

**Complementary, not competing**: ACP owns the live, MCP owns tools, ASP owns the durable past — the missing "durable back" of the ACP/MCP ecosystem.

## Relationship to ACP

ACP v2 (2026-07-20 draft) happens to be isomorphic with ASP on key points:

- ACP's "beyond the turn + multi-client observation" ⇄ ASP's append-only event log + snapshot/cursor
- ACP's "stable IDs + patch semantics (omitted/null/value/chunk-append)" ⇄ ASP's `entityRevision` + delta merge
- ACP's `Other` block ("receivers must preserve raw payload") ⇄ ASP's typed `unknown` block

**Integration directions**:
1. **ACP tap** (write path): an ASP implementation acts as an ACP client, subscribing to `session/update` → mapping to ASP events → storing. This is the best live capture (harness-neutral, semantic, no FUSE/privilege issues).
2. **ACP agent adapter** (read path): an ASP implementation acts as an ACP agent, replaying stored sessions to an editor via `session/resume` + `session/update`.

**Divergence**: ACP v2 removed the thinking content block (demoted to `thought_level` config + `thought_tokens`); ASP treats thinking (with signature) as a resume-critical first-class block. Over an ACP tap, thinking can only be passively preserved in an `Other` block — an explicit fidelity trade-off.

## Relationship to MCP

ACP v2's `ContentBlock` reuses MCP's schema (Text/Image/Audio/ResourceLink/Resource/Other). ASP's ContentBlock naming is its own (thinking/tool-call/tool-result are resume-critical, absent from MCP); mapping below.

## Content-block mapping

| ASP ContentBlock | ACP/MCP ContentBlock | note |
|---|---|---|
| `text` | `Text` | direct |
| `image` | `Image` / `Audio` | direct (audio via unknown or extension) |
| `reference` | `ResourceLink` / `Resource` | direct |
| `unknown` | `Other` (`_`-prefixed) | **both mandate opaque passthrough — semantically identical** |
| `thinking`(+signature) | (none) | removed in ACP v2; only via `Other` |
| `tool-call`/`tool-result` | `ToolCall*` | maps to tool events + tool entity |
