# Fidelity Matrix

> [English](fidelity.md) · [中文](zh-CN/fidelity.md)

## Declared fidelity, not promised fidelity

ASP's core stance: **conversion fidelity is declared, not guaranteed.** Every importer/exporter declares, per axis, a fidelity level and a loss list; round-trip guarantees are per-axis.

## Fidelity levels

| level | meaning |
|---|---|
| `preserved` | survives round-trip |
| `partial` | degraded (merged, re-encoded) |
| `evidence-only` | kept in evidence only, not in the canonical projection |
| `dropped` | dropped, with a declared reason |
| `not-in-source` | the source harness does not record this axis |

## Measured matrix (4 harnesses)

| axis | pi | dimagent | claude | codex |
|---|---|---|---|---|
| messages/order | preserved | preserved | preserved (block-append journal merged) | preserved (dual-stream dedup) |
| content blocks | preserved | preserved | preserved | preserved |
| tool chain | preserved | preserved | preserved | preserved |
| thinking | preserved (signature) | partial | preserved (**signature**) | partial (summary→thinking; **encrypted_content→unknown passthrough**) |
| run/turn boundaries | partial | preserved/partial | partial | preserved (turn_context) |
| model/usage | evidence-only | evidence-only | preserved | evidence-only |
| compaction | n/a | dropped | not-in-source | not-in-source |
| approvals | not-in-source | not-in-source | not-in-source | not-in-source |

## Resume-target profiles

Handoff fidelity means the exporter knows "which fields the target harness needs to resume". Missing required fields → **refuse to export** (never silently degrade); non-required → declare loss.

| target | resume requirements |
|---|---|
| pi | entry tree (id/parentId) + session header |
| Claude Code | uuid/parentUuid chain + thinking signature + tool_use id |
| Codex | full response_item order + encrypted_content + call_id pairing |
| opencode | full message/part set + git snapshot reference |
| dimagent | full parts + compaction_states cursor |

## The evidence layer (source of round-trip fidelity)

A bundle's `evidence` is the full ordered event stream (WAL role); the `pivot` snapshot is always re-derivable from it. **Round-trip fidelity comes from the evidence layer, not the projection** — exporters replay the native payloads recorded by the importer (e.g. `session_meta`, `turn_context`, reasoning `{summary, encrypted_content}`) verbatim, and only synthesize when absent.
