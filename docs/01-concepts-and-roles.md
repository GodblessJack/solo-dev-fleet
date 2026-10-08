# 01 · 六个角色逐个讲

每个角色 = 一个常驻 Claude Code 会话,行为完全由 `claude-commands/` 下的单轮命令文件定义。把命令文件拷进你仓的 `.claude/commands/`,会话里敲 `/loop <间隔> /<命令名>` 即上岗。

## S1 测试 `/qa-round` —— 找问题的

**职责**:周期性走查产品,发现缺陷提 bug 单。一轮只走查一个模块/一条路径。

- 双轨走查:工具/API 层 e2e(高频,mock 档)+ UI 浏览器走查(低频);
- 预算闸门:真调外部服务按天计数(贵的每天 ≤2 次,其余 ≤10 次),超预算自动降 mock 档;
- **提单前必须锚点自查**(查重第一层):结构化锚点 = `mod:*` 模块标签 + 涉及文件/工具名 + 复现路径;已有同锚点 open 单 → 评论补证据不新建;
- 提单即打 `bug + from:qa + claude-ready + mod:* + priority:*`——按构造即可就绪,免分诊直接进 S2 认领面;
- 纪律:一轮一个模块、不提 enhancement(改进建议由人提)、证据先行、同根因多处表现只提一单(根因聚合)。

## S2 实施 `/fix-round` —— 干活的

**职责**:认领就绪单,TDD 修复,提 PR。一轮一单,worktree 隔离。

- 认领面:`bug|enhancement + claude-ready`,无 doing/fixed/verified/question/needs-info/deferred/collateral;**rework 返工单最优先**(打回/烂尾先归位);
- 代码类 TDD:先写一个会失败的校验脚本,确认红 → 再改 → 确认绿;文档类以「验收标准逐条对照 + 加载链检查」代替;
- 三道本地门禁(lint + test + build)全绿才许提 PR——这就是 CI 门禁的本地镜像;
- 提 PR 即推进标签:摘 doing 打 fixed;PR 正文必须 `Closes #N`;
- **不自由发挥**:验收标准不具体的单退回排期;返工点属「裁决类」(ADR 冲突/职责归属/需求歧义)→ 不动手,打 question 转裁决会——**裁决权不是实施权**。

## S3 核实 `/verify-round` —— 挑刺的

**职责**:独立复现验证 S2 的 PR。不轻信自述,一切以复现为准。

- 三个池子顺序清:A. issue 池(status:fixed)→ B. 直管 PR 池(PR 级 fixed)→ C. 连坐恢复池(rework + circuit:collateral,`--state all`);
- bug 重复现路径,enhancement 按验收标准逐条核(每条「过/不过 + 证据」),直管 PR 以 PR 正文「验收标准」段为清单;
- **2.5 风险评估**:打 verified 前必须回答「这个修复可能弄坏什么,并验证没弄坏」——共享点影响面 grep、组合态 build(merge 最新 main 再构建)、回归方向核、相邻路径抽查;
- 通过:打 verified + **锚定行**(`核实锚定:head <sha> / 基底 main <sha>`);不通过:打 rework(缺陷类);
- 打回 3 次熔断 → question 转裁决会;红线:**永不 push、永不合并**(合并统一收口 S4,便于审计兜底)。

## S4 合并员 `/merge-round` —— 把关的

**职责**:扫 open PR,门禁齐自动 squash 合并,一轮至多合一个(FIFO 取最早门禁齐者)。纯 gh 操作,不改代码。

- `fix/` 双门禁:CI 绿 + verified;`feature/` 三门禁:加 claude-review 最新 `VERDICT: PASS`;
- **verdict 新鲜度绑定**:verdict 评论必须晚于 head 提交时间,否则触发重审(close/reopen),防把旧 head 的 FAIL 当成当前结论(幽灵返工);
- **打回分类两路**:审查红线属缺陷类 → rework(S2 返工);属裁决类 → question(裁决会);拿不准按缺陷类;
- **上轮合并复核**:本轮先核上次合并 push 触发的 main CI——红 = 组合态断链,熔断:提 p0 单 + 连坐打回 + 本轮不再合;
- **superseded 清理**:open PR 的关联 issue 已关且 main 有合并件 → 关废件 PR;
- chore/docs PR 永不自动合;待人工超 2 小时显著标出上报;
- 收尾把 main 同步回本地主目录(dev server 热更)。

## S5 裁决会 `/decide-round` —— 人的入口

**职责**:把「必须主人开口才能动」的单集中起来,逐条呈现 → 大白话裁决 → 翻译成 gh 动作执行。手动触发,人在场才跑。

- 范围判据一句话:「这单接下来动不动,取决于主人一句话吗?」按规则等待的不算异常,机器侧停滞归看门人;
- 队列来源:question / deferred:external / needs-info / duplicate-candidate / enhancement 待排期 / PR 级 rework / chore 待人工合并超时;
- **排期默认律**:所有 issue 默认都要做,只排优先级;「不做/关闭」必须逐条问,不许批量替裁;
- 呈现纪律:说名字不说编号、讲清来龙去脉、附建议、停下等回复;
- 执行后闭环检查:标签打对 + 评论留痕 + 「要做」的单确认进了流水线(有 ready + priority)。

## 看门人 `/monitor-round` —— 盯着所有人的

**职责**:三环(S2/S3/S4)存活心跳 + 「等主人」超时上报 + 自报心跳。只观察与上报,不催单不代合不代裁决。

- 三环心跳:会话在列且未停滞(busy 超 90 分钟疑卡死);缺失即上报;
- chore/docs PR 待人工超时(>2h)上报——「等主人」若无上报 = 无限期挂起;
- 上报通道 = 建 issue(标题带 `[看门人]` 前缀,24 小时同键去重防刷屏);
- **自报心跳**:每轮写状态文件(带时间戳),mtime 超 3 个巡检周期未更新 = 看门人自己死了,主人重启——看门人死了没人报它自己,这条是唯一兜底。

## 角色边界(互不越界)

| | 提单 | 改代码 | push | 打 verified | 合并 | 裁决 |
|---|---|---|---|---|---|---|
| S1 | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |
| S2 | ❌ | ✅ | ✅ 分支 | ❌ | ❌ | ❌ |
| S3 | ❌ | ❌ | ❌ | ✅ | ❌ | ❌ |
| S4 | ✅ 异常单 | ❌ | ❌ | ❌ | ✅ fix/feature | ❌ |
| S5 | ❌ | ❌ | ❌ | ❌ | ✅ 主人当面授权的 chore | ❌(翻译主人的话) |
| 看门人 | ✅ 上报单 | ❌ | ❌ | ❌ | ❌ | ❌ |
