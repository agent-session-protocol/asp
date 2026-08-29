# Agent Session Protocol (ASP)

> **Agent Session Protocol (ASP) is the storage and migration protocol for agent sessions — the session-layer counterpart to [ACP](https://agentclientprotocol.com). ACP handles the live present (editor↔agent interaction); ASP handles the durable past and cross-runtime migration of sessions.**
>
> [English](README.md) · [中文](README.zh-CN.md)

ASP describes only the **protocol layer**: how a session is modeled, how events are structured, how fidelity is declared, and how opaque payloads round-trip. It is **not** a storage engine, a conversion tool, or a live control protocol — those belong to the [implementation layer](../../universal-session-log).

## One-line definition

> ASP defines the canonical schema, event semantics, fidelity matrix, and opaque-payload round-trip conventions for agent sessions, so that any session produced by any harness can be persisted, replayed, and migrated across harnesses.

## Repository layout

```
asp/
├── schema/
│   ├── agent-session-contracts.ts      # canonical schema: envelope, event types, content blocks, validation (TS reference)
│   └── agent-session-materializer.ts   # evidence → snapshot projection semantics
└── spec/
    ├── overview.md        # session model, events, correlation, lineage, identity
    ├── content-blocks.md  # content blocks + opaque passthrough
    ├── fidelity.md        # fidelity matrix + loss declarations + resume profiles
    └── acp-mcp.md         # relationship and mapping to ACP / MCP
```

## Three core principles

1. **Correctness comes from the append log + evidence alone.** The event log is the single source of truth; snapshots are always re-derivable from it.
2. **Opaque payloads are first-class.** Bytes that cannot be parsed but must round-trip verbatim (e.g. Claude's thinking `signature`, Codex's `encrypted_content`) are carried in a typed `unknown` block — never normalized away.
3. **Round-trip fidelity comes from the evidence layer.** Converters replay the native payloads recorded by the importer verbatim, and only synthesize (with declared loss) when those are absent.

## Reference implementation

See [`agent-session-protocol/universal-session-log`](https://github.com/agent-session-protocol/universal-session-log) (USL: `usl-core` storage engine + `usl-capture` + `usl-convert`).

## Status

**Draft (schema v1.0)**. The schema currently ships as a TypeScript reference; a JSON Schema artifact and SDK generation are on the roadmap.
