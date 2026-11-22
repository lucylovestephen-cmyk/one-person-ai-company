# 一人公司 · 多 Agent 协作规则

## 团队

- **Lucy**：老板，最终决策；不插手 agent 的具体管理，只和 Muse 对话、只看结果。
- **Muse**：只负责和 Lucy 对话，只对结果负责；直接管理其他 agent（派活、验收、协调）。
- **DSH**：手，VPS 后台执行（靠网关接任务）。
- **WorkBuddy**：执行，照文档干活（Windows 本机）。
- **Codex**：方案和验收，出规划、验 WorkBuddy 的活。
- **Postiz**：发行，只负责定时上传（工具，不是会话方）。
- **Claude**：另一 AI，咨询用，靠 Lucy 传话。

## 默认模式：点对点流水线（不开群聊）

流程：Muse 派活 → Codex 出规划文档 → WorkBuddy 按文档执行 → Codex 按文档验收 → Muse 收结果。

- 规划只出文档，不写代码；开发只按文档干活，不做规划。
- 文档是契约：消息里只传“做什么、依据哪份文档”，不传全量上下文。
- 谁定标准谁验收：Codex 定标准、Codex 验收；干活的不验自己的活。
- Lucy 只在起点介入一次，中间全自动。

## 消息格式（统一信封）

每条跨会话消息必须带：message_id、group_id、round、agent_id、from、to、type、doc_path、version、summary、evidence_paths。

- type：派活 / 交付 / 返工 / 通过。
- group_id：不属于任何群的消息填 default；群里的消息填该群的 id。
- round：群聊里的轮次；点对点消息填 0。
- evidence_paths：依据的产物路径（文档、文件），没有就空。
- message_id 去重：同一条消息重复触发只处理一次。
- 不管从哪个通道来（网关、跨会话消息、对话），入库后都能追溯。

## 返工回路

- 验收不通过 → 发回返工，带逐条不通过的依据（通过 / 不通过 + 逐条依据），不只一句“看起来没问题”。
- 最多返工 3 次，超过就停下来通知 Lucy（人工介入）。

## 人工闸门（两个）

1. 规划文档定稿前（Lucy 点头）。
2. 最终验收通过后、合并或发布前（Lucy 点头）。

## 群聊（例外才开）

默认不开群聊。只在以下情况开：规划和开发对需求理解反复对不上，需要当面对齐；有第三方（测试、设计）要同时参与决策；Lucy 想旁听一次评审。

群聊注册字段：group_id、moderator、members、max_rounds、status、output。

- 调度：round_robin（轮流）或 leader_decides（群主决定）。默认由 Lucy 点名（leader 指定），成员不被点名不发言。
- 硬性终止：max_rounds 默认 8，到顶强制产出结论，或升级给 Muse；光靠 Lucy 判断完成不够。
- 终止：到 max_rounds 强制产出结论，写入 decision.md，再发 group.decision_made。
- 失败：达不成一致发 group.deadlocked，交给 Lucy。
- 知识库按群分命名空间：group/<group_id>/，里面放议题、产物路径、最近 N 轮摘要和最终 decision.md。成员只读这一份，不读全量对话。
- Lucy 只看摘要和结论：她是唯一看到全部发言的人，细节留在知识库，必要时再按路径取。
- Lucy 既主持又可能参与决策：如果涉及她自己产出的东西，裁决交给 Muse 或另一个成员。
- 验收独立：结论落盘后继续由独立验收 Agent 按标准判，群里的共识不等于验收通过。

## 共享上下文

- 共享知识库（网关上）：所有 agent 通过网关 API 读写。
- 团队协调对话（“一人公司 · AI 团队”）：Lucy 的指挥口，Muse 在里面派活。
- DSH 靠网关，WorkBuddy / Codex / Claude 靠 Lucy 传话。

## 现在不做的事

- 不改网关加群聊调度（评估结论：流水线已够用，群聊是例外；等真需要对齐时再加）。
