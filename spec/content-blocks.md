# Content Blocks & Opaque Payloads

> [English](content-blocks.md) · [中文](zh-CN/content-blocks.md)

## ContentBlock types

```ts
ContentBlock =
  | { type: "text"; text }
  | { type: "thinking"; text; signature? }        // resume-critical: signature must round-trip
  | { type: "tool-call"; toolCallId; name; arguments }
  | { type: "tool-result"; toolCallId; toolResultId; content; isError }
  | { type: "image"; mimeType; data?; uri?; alt? }
  | { type: "reference"; uri; title?; mediaType? }
  | { type: "error"; message; code?; detail? }
  | { type: "unknown"; nativeType; value }        // opaque-payload container
```

## Opaque passthrough — a first-class citizen

Agent ecosystems contain plenty of bytes that cannot be parsed but must round-trip verbatim:

- Claude's thinking `signature` (required by the Anthropic API on resume)
- Codex's reasoning `encrypted_content` (OpenAI's server-side verification chain, ~1KB blob)

Normalizing such payloads destroys resume capability. ASP's rules:

1. **Carry them in a typed `unknown` block**: `{ type: "unknown", nativeType, value }`, where `nativeType` names the source (e.g. `codex.encrypted_reasoning`).
2. **canonical→native→canonical must be idempotent**: no importer/exporter re-encoding path may double-wrap an `unknown` block — otherwise `nativeType` is clobbered to `"unknown"` and the payload is lost.

> Field lesson: e-core's `normalizeContent` and pi's importer/exporter both double-wrapped canonical `unknown` blocks, dropping Codex's `encrypted_content` `nativeType` on a pi round-trip. All three were fixed; the rule is now baked into schema validation.

## Divergence from MCP / ACP

| ASP | MCP / ACP |
|---|---|
| `thinking` (with signature) is a first-class block | ACP v2 demotes thinking to config + stats; no dedicated block |
| `unknown` is a fidelity container (resume-critical) | `Other` block (`_`-prefixed) preserves raw payload — semantically identical |

## Canonical representation of tool results

Tool-result content exists both on the **tool entity's `result`** and in message blocks. Rule: **the tool entity's `result` is authoritative** (canonical, decoupled from message block shape) — harnesses disagree on the block shape for tool results (pi uses `[text]`, Codex uses `[tool-result]`), so exporters must read from the tool entity, never from the message blocks.
