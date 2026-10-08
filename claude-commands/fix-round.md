---
description: S2 实施会话单轮:认领一个 bug 或已排期的 enhancement,TDD/验收标准驱动实施,提 PR。worktree 隔离,由 /loop 驱动,一轮一单。
allowed-tools: Bash(gh:*), Bash(git:*), Bash(npm:*), Bash(npx:*), Read, Write, Edit, Glob, Grep
---

# fix-round — S2 实施单轮

你是实施会话。一轮只做一单,做完收尾。

## 1. 认领

```
gh issue list --repo OWNER/REPO --state open \
  --label claude-ready --json number,title,labels,body
```

可认领条件(全部满足):有 `bug` 或 `enhancement`;有 `claude-ready`;无 `status:doing`、`status:fixed`、`status:verified`、`question`、`needs-info`、`deferred:external`、`circuit:collateral`(**`status:rework` 不是排除项——返工单恰是最优先认领对象**,见下行排序;此处曾误列 rework 为排除项,与「rework 最优先」自相矛盾,字面执行 = 返工单永远无人认领——真实事故)。
排序:**`status:rework` 返工单最优先**(打回/烂尾单先归位,再开新工;**但挂 `circuit:collateral` 的除外——那是 S4 熔断连坐标记,不是真打回,归 S3 连坐恢复池恢复;本会话认领面一律跳过,误认领会走「checkout 既有 PR 分支续做」而该 PR 已 squash 合并且分支已删,流程走不通,还与 S3 恢复池撞车**);其次 bug 优先于 enhancement;同类内按 priority(p0>p1>p2),再按 issue 号小者优先。无可认领单 → 本轮结束,建议拉长 /loop 间隔。

**返工单(rework)流程**(打回进返工态而非 doing——doing 是"实施中"锁,S2 不认领 doing 单,打回到 doing 会造成无人认领的死锁):认领前必读该单的打回出处——issue/PR 评论里 S3 的「核实打回」或 S4 的「VERDICT: FAIL 打回」或看门人的「烂尾接管」评论,针对打回点返工;分支用**既有 PR 分支**(worktree 里 checkout 该分支续做),修完 force-push 更新同一个 PR,不重开 PR。认领动作:评论「S2 返工认领,接手打回点:<一句话>」+ `gh issue edit <号> --remove-label status:rework --add-label status:doing`。

**返工点分类出口**:读打回出处时同步判定——打回点属**缺陷类**(行为错误/回归/缺验证,本会话能直接修)→ 照常认领返工;属**裁决类**(与 ADR/既有文档冲突、职责或方向归属之争、需求歧义——改哪头须主人拍板)→ **不认领不动手**,`gh issue edit <号> --remove-label status:rework --add-label question` + 评论「返工点属裁决类(引红线一句),须主人拍板,已转 decide-round 队列」,转下一单。**禁止自行选边改 ADR/职责归属**——那是裁决权不是实施权(真实事故:审查红线要求统一某角色职责定义,属裁决类却走了 rework,S2 认领也无权选边,单挂死一天)。拿不准按缺陷类认领,但实施中发现其实要动 ADR/职责归属 → 立即停手,摘 doing 打 question 按裁决类处理。

**enhancement 开工前校验(就绪闸门的执行层)**:认领 enhancement 后先读验收标准。验收标准不具体/无法验证的(如只写「优化一下 XX」)→ 不动手,评论追问具体验收口径、`--remove-label claude-ready` 退回待排期,转向下一单。禁止对着模糊需求自由发挥。

**优化无损律**:凡「省」类改动(上下文折叠/压缩/预算收缩、缓存截断、降采样、字段裁剪),实施时必须先列**受影响消费方清单**(谁在读这份数据、读它的哪个字段)并设计**开/关对照**(同一输入,优化开/关各跑一遍,产物逐项比对);PR 正文须附「功能无损证明」=消费方清单+对照结果。落盘/存档链路**禁止做不可逆有损变换**——有损只允许发生在发给模型的临时视图,存档保原文或可还原引用(真实事故:折叠先于落盘执行,档案原文永久丢失)。

认领动作:评论「S2 认领,分支 fix|feature/<号>-<简述>」+ `gh issue edit <号> --add-label status:doing`。

## 2. 实施(worktree 隔离)

worktree 已预建于 `.worktrees/fix`(detached)。每轮在其内切出分支:

```
git fetch origin main   # 用你仓的协作 remote 名
git -C .worktrees/fix checkout -B <fix|feature>/<号>-<简述> origin/main
cd .worktrees/fix
```

分支前缀:bug 用 `fix/`,enhancement 用 `feature/`。**命名即路由**:S4 靠前缀与是否带号决定门禁等级。

**质量纪律分两类**:

- **代码类**:TDD——先写一个会失败的校验脚本(按你仓测试形态:单元测试/集成脚本/e2e),确认红 → 再改 → 确认绿。
- **文档/技能类**:无可执行测试,验收证据 = 加载链/引用完整性检查全绿 + PR 正文逐条对照 issue 验收标准写明「交付了什么、在哪些文件」。

**根因聚合单**(正文含「关联现象清单」):实施必须把清单里每个现象的复现路径全部过一遍,少一条都不许提 PR。

**疑难单**(被打回过 2 次):必须先产出最小复现脚本/步骤,无稳定复现不许猜着改。

## 3. 提 PR

- `lint + test + build` 三项全绿才许提交(这就是 CI 门禁的本地镜像;判 build 只看退出码——tsc 类增量构建会假绿,tail 看文案不可靠)
- 提交信息:`fix: <说明> (#<号>)` 或 `feat: <说明> (#<号>)`
- `git push <协作remote> <分支名>`
- `gh pr create --repo OWNER/REPO --title "<fix|feat>: <说明> (#<号>)" --body "实施说明/验收标准逐条对照/测试证据。Closes #<号>"`
- issue 状态推进为 `status:fixed`:`gh issue edit <号> --remove-label status:doing --add-label status:fixed`(单标签口径:同一时刻只挂一个 status 标签,doing 在提 PR 时摘掉,追溯靠评论;enhancement 同此口径)

## 4. 收尾

汇报:做了哪单、分支、PR 号、证据摘要。

### 自动续排

本轮结束前**必须**调用 ScheduleWakeup 给自己排下一轮(此前只写「建议间隔」无排程指令,会话跑完一轮即永久休眠——真实事故):池里有单 → delaySeconds=**600**;清单空 → **1800**。prompt 传 `/fix-round`,noop 按当轮实况填;ScheduleWakeup 不可用时 CronCreate 一次性任务兜底。汇报中说明排了多久。

红线:push 只推协作 remote,永不碰上游 fork 源(如适用)。
