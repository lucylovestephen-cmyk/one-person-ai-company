# 一人公司 · 多 Agent 协作规则

> 版本 v2 · 2026-11-22（以 210 行完整版为底：五处修订 + 三处通道更新）
> 落地工具：`tools/pipeline.py`（台账，零依赖，可离线）
> 边界：**不改网关做群聊调度。** 网关只负责任务队列和共享知识库。

---

## 1. 团队

| 成员 | 角色 | 干什么 | 通道 |
| --- | --- | --- | --- |
| **Lucy** | 老板 | 最终决策；两个闸门只有她能点头 | 人 |
| **Muse** | 大脑 + 视频，群主 | 编排、派活、汇总、决定停 | 协调对话 |
| **DSH** | 手 | VPS 后台执行 | 网关（任务队列） |
| **WorkBuddy** | 执行 | 照文档干活 | Windows 本机 |
| **Codex** | 方案 + 验收 | 出规划、验 WorkBuddy 的活 | OpenAI API（重要检验时；无上下文，只派独立任务） |
| **Postiz** | 发行 | 只负责定时上传 —— **工具，不是会话方**；由 Muse 触发 | 无 |
| **Claude** | 顾问 | 另一 AI，咨询用 | Anthropic API（重要检验时；无上下文，只派独立任务） |

角色边界是硬的：

- 规划只出文档，不写代码。
- 开发只按文档干活，不做规划。
- 谁定标准谁验收：Codex 定标准、Codex 验收。干活的不验自己的活。

---

## 2. 默认模式：点对点流水线（不开群聊）

```
Muse 派活 → Codex 出规划文档 → 〔闸门 1：Lucy 点头〕
      → Muse/Codex 派开发 → WorkBuddy 按文档执行 → 交付
      → Codex 按文档验收 → 通过 / 返工（≤3 次）
      → 〔闸门 2：Lucy 点头〕 → Muse 触发 Postiz 发行 → Muse 收结果
```

**消息里只传"做什么、依据哪份文档"，不传全量上下文。** 文档是契约。

Lucy 在两个人工闸门点头；中间由 Muse 派活、汇总、决定停。Codex 与 WorkBuddy 之间目前靠 Lucy 传话（直连通道接通后改为自动），所以现在还不是全自动。

### 状态机（台账视角）

```
assigned ──gate.plan approved──▶ planned ──派活给执行者──▶ developing
   ▲                                                        │
   │                                                        ▼
escalated ◀──第 4 次返工── rework ◀──返工── delivered ◀──交付──┘
   │                          │                 │
   │（交人）                   └──▶ rework ──交付──▶ delivered ──通过──▶ passed
```

任何一次 `返工` 都要求带逐条依据；第 4 次直接 `escalated`，停在那儿等人。

---

## 3. 消息信封（固定格式）

每条跨会话消息必须带统一信封：

| 字段 | 必填 | 说明 |
| --- | --- | --- |
| `message_id` | ✅ | 去重键。**同一条消息重复触发只处理一次** |
| `group_id` | — | 不属于任何群填 `default` |
| `round` | — | 群聊轮次；点对点填 `0` |
| `from` → `to` | ✅ | 谁给谁 |
| `type` | ✅ | `派活 / 交付 / 返工 / 通过 / 群聊结论 / 僵局` |
| `doc_path` | 派活必填 | 依据的文档（契约本身） |
| `version` | 派活必填 | 文档版本；与已定稿版本不一致直接拒收 |
| `summary` | ✅ | 一句话说清做什么 / 结果如何 |
| `evidence_paths` | 交付、返工必填 | 产物路径；没有就无法验收 |
| `task_id` | 建议 | 缺省时按 `doc_path` 归并到同一任务 |

```json
{
  "message_id": "m-20261122-004",
  "group_id": "default",
  "round": 0,
  "from": "codex",
  "to": "workbuddy",
  "type": "派活",
  "doc_path": "docs/plan-gateway.md",
  "version": "v1",
  "summary": "按 v1 实现网关台账对接，只改 tools/",
  "evidence_paths": []
}
```

**不管从哪个通道来**（网关、跨会话消息、对话），入库后都能追溯：`pipeline.py task <id>` 能列出每一跳的时间、类型、从谁到谁、依据哪份文档。

---

## 4. 返工回路

验收不通过 → 发回返工，**带逐条不通过的依据**：

- 通过 / 不通过 + 逐条依据，不接受"看起来没问题"。
- 最多 3 次；第 4 次自动升级给 Lucy，工具直接返回退出码 4 并落 `escalation` 事件。

```bash
# 验收单作为证据
python3 tools/pipeline.py send --json '{
  "message_id": "m-005", "from": "codex", "to": "workbuddy",
  "type": "返工", "task_id": "m-001",
  "summary": "3 条不通过：版本号、缺干跑、没写回滚",
  "evidence_paths": ["reports/review-m-004.md"]
}'
```

---

## 5. 人工闸门（两个）

| 闸门 | 时机 | 谁 | 工具表现 |
| --- | --- | --- | --- |
| `plan` | 规划文档定稿前 | Lucy | 未批准时，给执行者派活直接被拒（退出码 3） |
| `release` | 最终验收通过后、合并或发布前 | Lucy | `gate --gate release --approve` 才放行；放行后由 Muse 触发 Postiz 发行 |

```bash
python3 tools/pipeline.py gate m-001 --gate plan --approve --by lucy --note "方向对，开工"
```

闸门不是流程装饰：`plan` 没批，`workbuddy`/`dsh` 的派活会被拒，这条规则由代码保证，不靠记性。

---

## 6. 群聊（例外才开）

只在三种情况开：

1. 规划和开发对需求理解反复对不上，需要当面对齐。
2. 有第三方（测试、设计）要同时参与决策。
3. Lucy 想旁听一次评审。

注册字段：`group_id`、`moderator`、`members`、`max_rounds`、`status`、`output`。
调度：`round_robin`（轮流）或 `leader_decides`（群主决定）。**默认 `leader_decides`，由 Muse 点名；不被点名不发言。**
硬终止：`max_rounds` 默认 **8**，到顶强制产出结论，写入 `decision.md`，同时通知 Lucy —— 光靠 Muse 判断"完成了"不够。
终止：到 `max_rounds` 强制产出结论 → 写 `decision.md` → 发 `group.decision_made`。
失败：达不成一致 → 发 `group.deadlocked` → 交给 Lucy。

> ⚠️ 这一节是**规则**，不是当前实现。网关里没有群聊调度（见第 10 节）。

---

## 7. 共享上下文

- **共享知识库（网关上）**：所有 agent 通过网关 API 读写。
- **团队协调对话（"一人公司 · AI 团队"）**：Lucy 的指挥口，Muse 在里面派活。
- **DSH 靠网关；WorkBuddy 靠 Windows 网关（建成后）；Codex / Claude 靠 API（重要检验时开）；直连没通之前，临时靠 Lucy 传话。**

命名空间按群分：`group/<group_id>/`，里面放：议题、产物路径、最近 N 轮摘要、最终 `decision.md`。
**成员只读这一份，不读全量对话。** Muse 只看摘要和结论 —— 她是唯一看到全部发言的人，细节留在知识库，需要时按路径取。

---

## 8. 两条容易踩的边界

**Muse 既主持又可能参与决策。** 如果议题涉及她自己产出的东西，裁决交给 Lucy —— 台账里用 `acceptor` 字段把验收人钉住。

**群里达成共识 ≠ 验收通过。** 结论落盘后，仍由独立验收 Agent 按标准判。

---

## 9. 现在不做的事

- ❌ 不改网关加群聊调度。评估结论：**流水线已经够用**，群聊是例外，等真需要对齐时再加。
- ❌ 不引入 MindRoom / SwarmChat 一类重框架。
- ❌ 不为"5 个 agent"这种经验值做设计。（5 是经验值，不是标准；两个够用就用两个。）
- ❌ 不把 112MB/会话、1GB 内存这类数字当普适结论 —— 那是那台 VPS 的实测值，换机器要重测。

---

## 10. 落地对照：规则 → 代码

| 规则 | 由什么保证 |
| --- | --- |
| 信封必填字段、类型合法 | `normalize()`，缺字段退出码 2 |
| 同一 `message_id` 只处理一次 | `messages.message_id` 主键 + 退出码 5 |
| 文档是契约、版本要对齐 | `version` 与 `plan_version` 不一致拒收 |
| 闸门 1 未过不能开工 | 派活给 `workbuddy/dsh` 时检查 `gate_plan` |
| 交付必须有产物 | `deliver` 要求 `evidence_paths` 非空 |
| 返工必须带逐条依据 | `rework` 要求 `evidence_paths` 非空 |
| 返工 ≤ 3 次 | 第 4 次 `escalated` + 退出码 4 |
| 谁定标准谁验收 | `pass` 的 `from` 必须是 `acceptor` |
| 任何通道都能追溯 | 一张 `messages` 表 + `task <id>` 全貌 + `export` |

### 命令

```bash
python3 tools/pipeline.py send --file msg.json     # 投递（幂等）
python3 tools/pipeline.py inbox --agent workbuddy --claim
python3 tools/pipeline.py task m-001               # 全貌：状态、闸门、每一跳
python3 tools/pipeline.py gate m-001 --gate plan --approve --by lucy
python3 tools/pipeline.py tasks --state escalated
python3 tools/pipeline.py status
python3 tools/pipeline.py export --out ledger.md
python3 tools/pipeline.py doctor
```

默认库：`$MUSE_DSH_HOME/data/pipeline.db`（没有则 `./data/pipeline.db`）。`--db` 可指定，便于每台机器各存一份再合并。

---

## 11. 还没定的

1. `pipeline.db` 放在哪台机器上作为**唯一事实来源**？（放 VPS 最简单：三台机器都能连网关。）
2. 台账要不要自动镜像进网关知识库（`group/<group_id>/`），还是只存路径、结论再手工沉淀？

（第 3 条已定：`release` 闸门之后由 Muse 触发 Postiz 发行。）
