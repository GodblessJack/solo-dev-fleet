# 07 · 标签词汇表

标签 = 这套系统的状态机载体。GitHub 免费计划没有自定义字段,标签是唯一全员(人、gh CLI、Actions workflow)可读写的状态通道。

## 类型

| 标签 | 含义 |
|---|---|
| `bug` | 确认的缺陷(分诊核实过或 QA 按构造提单) |
| `enhancement` | 功能/改进建议。**排期由人定**——无 claude-ready 不进流水线 |
| `question` | 必须主人开口:三次熔断/需求歧义/ADR 冲突/裁决类审查红线 |

## 来源(谁提的)

| 标签 | 含义 |
|---|---|
| `from:qa` | S1 测试会话提的(双标齐 `+claude-ready` 免分诊) |
| `from:human` | 人提的 |
| `from:annotation` | 产品内批注通道提的 |
| `from:canary` | runner 心跳告警单 |

## 状态机(同一时刻只挂一个 status:*)

| 标签 | 含义 | 谁打 |
|---|---|---|
| `status:doing` | S2 实施中(认领锁) | S2 认领时 |
| `status:fixed` | PR 已提,待核实 | S2 提 PR 时(摘 doing) |
| `status:verified` | S3 独立核实通过,待合并 | S3(须附锚定行) |
| `status:rework` | 被打回,返工态 | S3/S4 打回时 |

状态流转:`(无) → doing → fixed → verified → (关单)`;打回:`fixed/verified → rework → doing → fixed → …`;熔断:rework 满 3 次 → `question`。

**为什么打回进 rework 而不是 doing**:doing 是「实施中」锁,S2 不认领 doing 单(防止抢别人正在做的),打回到 doing 会造成无人认领的死锁。rework 是「待返工」,S2 最优先认领。

## 就绪与优先级

| 标签 | 含义 |
|---|---|
| `claude-ready` | 就绪闸门通过:验收标准可验证 + 设计依据有出处 + 依赖明确。流水线只认领带此标的 |
| `priority:p0` | 阻断主流程/数据损坏 |
| `priority:p1` | 功能明显不符但有绕行 |
| `priority:p2` | 体验/文案/边界问题 |

## 例外

| 标签 | 含义 | 归属人 |
|---|---|---|
| `needs-info` | 信息不足,等提单人补料;人类新评论自动触发重分诊 | 提单人 |
| `duplicate-candidate` | 疑似重复:高置信已自动关闭(留痕),低置信待人确认 | triage/S5 |
| `auto-closed` | AI 自动关闭留痕(「误关请重开」);被 reopened 后永不再自动关 | triage |
| `deferred:external` | 根因在外部服务/额度/第三方,挂起 | S5 定期清点 |
| `circuit:collateral` | S4 熔断连坐标记:PR 已合但组合态断链,非改动本身缺陷 | S3 连坐恢复池 |

## 模块(mod:*)

按项目自定义 5-8 个,建议覆盖:前端、后端、文档/技能、基础设施、外加你的核心业务域 2-4 个。模块标签是**查重锚点的第一维**(锚点 = mod + 代码位置/工具名 + 复现路径),提单必带一个。

## PR 级标签(直管通道)

`status:fixed` / `status:verified` / `status:rework` / `question` 同款标签挂在 **PR 本身**上,用于无 issue 号的直管 PR(见 docs/06)。issue 通道与 PR 通道语义一致、对象不同。
