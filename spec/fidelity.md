# 保真度矩阵（fidelity）

## 声明式保真，不是承诺式保真

ASP 的核心立场：**转换保真度是声明出来的，不是打包票的。** 每个 importer/exporter 都要声明每个轴（axis）的保真等级与丢失清单（loss），roundtrip 保证按轴声明。

## 保真等级

| level | 含义 |
|---|---|
| `preserved` | 往返不丢失 |
| `partial` | 降级（如合并、重编码） |
| `evidence-only` | 只留在 evidence，不进 canonical 投影 |
| `dropped` | 丢弃，且有声明的原因 |
| `not-in-source` | 源 harness 根本不记录该轴 |

## 实测矩阵（4 harness）

| axis | pi | dimagent | claude | codex |
|---|---|---|---|---|
| messages/order | preserved | preserved | preserved（block-append journal 合并） | preserved（双流去重） |
| content blocks | preserved | preserved | preserved | preserved |
| tool chain | preserved | preserved | preserved | preserved |
| thinking | preserved(签名) | partial | preserved(**signature**) | partial(summary→thinking；**encrypted_content→unknown passthrough**) |
| run/turn 边界 | partial | preserved/partial | partial | preserved(turn_context) |
| model/usage | evidence-only | evidence-only | preserved | evidence-only |
| compaction | n/a | dropped | not-in-source | not-in-source |
| approvals | not-in-source | not-in-source | not-in-source | not-in-source |

## Resume-target profiles

handoff 保真 = 导出器知道"目标 harness resume 需要哪些字段"。缺必需字段 → **拒绝导出**（而非静默降级）；非必需 → 声明 loss。

| 目标 | resume 必需 |
|---|---|
| pi | entry 树（id/parentId）+ session header |
| Claude Code | uuid/parentUuid 链 + thinking signature + tool_use id |
| Codex | response_item 全序 + encrypted_content + call_id 配对 |
| opencode | message/part 全集 + git snapshot 引用 |
| dimagent | parts 全集 + compaction_states 游标 |

## Evidence 层（roundtrip 保真的来源）

bundle 的 `evidence` 是完整有序事件流（WAL 角色），`pivot` 快照永远可从它重放。**roundtrip 保真来自 evidence 层，而非投影层**——exporter 优先用 importer 存下的原生负载（`session_meta`、`turn_context`、reasoning `{summary, encrypted_content}`）逐字还原，缺失才合成。
