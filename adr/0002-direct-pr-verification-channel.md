# ADR-0002:直管 PR 的核实与合并通道——PR 级验收标准 + status:verified + 新鲜度两查

日期:2026-10;本文件为脱敏开源版,隐去项目细节,保留全部决策与修订史
状态:已裁决,在生产中运转

## 背景

主人直管的大功能/重大改进(本地开 worktree 会话独立实现 + 反复自测,完工提 PR)不走 issue 流程——分支名不带 issue 号。ADR-0001 的自动合并门禁把「status:verified」锚在 issue 上,S3 无对象可核、S4 门禁查不到,这类 PR 只能停在人工队列:曾有三个直管 PR(CI 全绿、AI 审查 PASS)堆积 1-3 天,verify/decide/merge 三个会话按设计都不认领,最终主人当面说「合并」才落地。

根因:**「独立核实」这道闸只给走 issue 流程的工作装了,恰恰最大的改动反而没有**——同一个会话既实现又自测,违反「实现者 ≠ 验证者」。

## 调研对照(为什么这么设计)

对业界一手来源的调研结论:

- **实现者不能自批是全行业铁律**:GitHub 平台禁止 PR 作者批准自己的 PR;Google「评审的终点是另一名工程师同意」;Meta「每个改动必须被评审,没有例外」;Anthropic 官方文档原话 "the agent doing the work isn't the one grading it"——正是 S3 独立核实存在的理由,直管 PR 不应豁免。
- **验收标准进 PR 是成熟惯例**:Meta/Phabricator 的 Test Plan 默认必填,质量标准原文「不熟悉该变更的人也能照着复现」——本 ADR 的 PR 正文「## 验收标准」段即其同构。
- **验证信号必须有失效语义**:GitHub 对 approval/check 都有 stale dismissal(新提交即作废);标签无平台级失效——因此 S4 新鲜度两查自建等价物。
- **合并前对着最新 main 重验(strict up-to-date / merge queue)防组合态断链**:GitHub 原生 merge queue 需 Enterprise Cloud 计划、branch protection 免费计划私有仓不可用(实测 403 "Upgrade to GitHub Pro")——平台级轮子全买不到,只能自建会话规则等价物:S3 核实时 merge 最新 main + S4 基底新鲜度查 + update-branch 重跑 CI。

**被否的备选**:①「授权标签」通道——所有会话共用主人 gh 账号,标签无法区分主人亲手打还是会话擅自打 = 开「机器人自我授权」口子(有会话越权合并前科),否决;②直管工作开工前建 issue 写死验收标准——与「边做边调方向」的节奏冲突,否决;③维持现状人工合——堆积与瓶颈留在主人,否决。

## 决策

### 1. 直管 PR 的生命周期(标签挂 PR 本身)

```
直管会话:实现 + 自测 → 提 PR(正文带「## 验收标准」段)→ 打 PR 级 status:fixed
    ↓
S3 认领(先清 issue 池再清直管池):独立 worktree、merge 最新 main、按验收清单逐条复现 + 风险评估
    ↓ 通过                                      ↓ 不通过
PR 打 status:verified                           PR 打 status:rework(归原直管会话/主人修,
+ 核实评论带锚定行:                                不归 S2——S2 只认 issue;修完重打 fixed 回队列)
  核实锚定:head <sha> / 基底 main <sha>         打回满 3 次熔断打 question
    ↓
S4 门禁:CI 绿 + PR 级 verified + 新鲜度两查(feature/ 另加 AI 审查 VERDICT: PASS)
    ↓ 全过 → 自动 squash 合并(无关联 issue,无单可关)
```

### 2. 验收标准段(承 Phabricator Test Plan)

- PR 模板固化「## 验收标准」段(网页建 PR 自动预填;gh CLI 建的由创建会话按模板结构写)。
- 标准每条 = 操作步骤 + 期望结果,达到「不熟悉本改动的人照着能复现」。
- S3 对缺段/不可执行的直管 PR 退回补写,**不自行编造验收标准**(核实基准只能来自作者陈述)。

### 3. 新鲜度两查(S4 侧,自建 stale dismissal + strict up-to-date)

1. **head 锚定查**:当前 headRefOid == 核实锚定 head 才有效;有功能性新提交 → 摘 verified 打回 status:fixed(S3 重核)。唯一例外:S4 自己的 update-branch 基底同步 merge 不作废。
2. **基底新鲜度查**:锚定 main ≠ 当前 main tip → `gh pr update-branch` 触发 CI 在组合态重跑,下轮凭新 CI 判。

### 4. 不变的边界

- chore/docs PR 永不自动合(机制类改动人工把关)。
- issue 通道(fix/<N>、feature/<N>)行为完全不变;PR 通道只覆盖不带号的直管 PR。
- 主人的当面授权永远高于自动机制(紧急单仍可当面说「合」插队)。
- S3 红线不变:核实会话永不 push 分支、永不执行合并。

## 后果

- 正面:直管大功能进 main 前有第二双眼睛(实现者≠验证者对齐业界铁律);组合态在合并前对最新 main 重验;堆积根除(verified 后 S4 全自动,不再等主人);返工路径明确(PR 级 rework + 熔断)。
- 代价:每个直管 PR 多等一轮独立核实(半小时到数小时);验收标准写得含糊会被 S3 退回,头一两个 PR 有磨合成本;S3 在 main 频繁前进时可能多核一轮(新鲜度作废重排)。

## 修订记录(同日连续三次,主人逐条指认漏洞)

- **修订 1:PR 级 rework 的认领真空**。初版把直管 PR 返工写成「归原直管会话/主人」但没有任何机制保证它被看见——S2/S3 只认领 issue 与 PR 级 fixed,S4 不改代码,S5 只扫 issue,四个会话对 PR 级 rework 全不认领;原直管会话通常已关,返工 PR 会静默挂死。修订:S4 打回动作补明确(摘 verified→打 rework→评论「待主人指定返工人」);S5 扫描面扩到 PR;看门人巡检显式上报。**不采纳「S2 自动认领 PR 级 rework」**:返工处置权归主人;自动化的是发现与上报,不是替主人决定谁来修。
- **修订 2:CONFLICTING 直管 PR 的归属**。冲突 PR 不触发任何 workflow(CI/审查全停),S3 无从核、S4 不动代码——初版没写谁解冲突。归属规则:原直管会话在世且活跃 → 等它解(防撞车);缺席或挂起超 2 小时 → 看门人 rebase 兜底;issue 通道单归 S2;chore 类同此兜底(解冲突只为恢复 CI,合并仍待主人)。
- **修订 3:「待人工合并」无限期挂起**。chore PR「永不自动合、等主人点头」若无上报机制 = 主人不刷 GitHub 就永远不知道有东西在等——实例:chore PR CI 全绿后挂 4.5 小时无人提醒。修订:S5 扫描面加「chore/docs + CI 全绿 + 待人工超 2 小时」入裁决队列(二选一:现在合→裁决会话当面授权代合/先放→明说等什么);S4 汇报显著标出;看门人巡检同项上报。**「等主人」的状态必须带上报,否则等于无人管**——三个修订同一条设计律:每个状态都要有归属人。
