# 06 · 直管 PR 通道(不建 issue 的大功能)

> 本章对应 [adr/0002-direct-pr-verification-channel.md](../adr/0002-direct-pr-verification-channel.md),是整套机制里最新、也最容易被忽略的一块。

## 6.1 问题:最大的改动反而没有独立核实

主人的工作流里有一类活不适合走 issue:**大功能/重大改进**——在本地开 worktree 会话独立开发,边做边调方向,自己反复测试,完工直接提 PR。这类 PR 分支名不带 issue 号。

原流水线只认 issue:核实标签挂在 issue 上、合并门禁查 issue 状态。于是直管 PR 三会话全不认领——核实会话没单可核、合并门禁查无对象、裁决会只扫 issue。**同一个会话既实现又自测,恰恰是最大的改动没有第二双眼睛**,违反「实现者 ≠ 验证者」铁律。

## 6.2 业界对照(调研结论)

- **实现者不能自批是全行业铁律**:GitHub 平台禁止 PR 作者批准自己的 PR;Google/Meta 成文规定;Anthropic 官方文档 "the agent doing the work isn't the one grading it";
- **验收标准进 PR 是成熟惯例**:Meta/Phabricator 的 Test Plan 默认必填,质量标准原文「不熟悉该变更的人也能照着复现」——直管 PR 的「验收标准」段即其同构;
- **验证信号必须有失效语义**:GitHub 对 approval/check 有 stale dismissal;标签没有 → 自建锚定行 + 新鲜度两查;
- **合并前对最新 main 重验**(strict up-to-date/merge queue):GitHub 原生 merge queue 需 Enterprise Cloud,branch protection 免费计划私有仓不可用 → 自建等价物。

**被否决的备选**:①「授权标签」通道——所有会话共用主人 gh 登录态,标签无法区分「人打的」还是「机器人自己打的」,等于机器人自我授权(有越权合并前科);②直管工作开工前建 issue 写死验收标准——与「边做边调方向」的节奏冲突;③维持人工合——堆积在主人。

## 6.3 生命周期(PR 级标签,与 issue 通道平行)

```
直管会话:实现 + 自测 → 提 PR(正文必带「## 验收标准」段)→ 打 PR 级 status:fixed
    ↓
S3 认领(先清 issue 池再清直管池):独立 worktree、merge 最新 main、按验收清单逐条复现 + 风险评估
    ↓ 通过                                        ↓ 不通过
PR 级 status:verified                            PR 级 status:rework
+ 核实评论带锚定行:                                (归原直管会话/主人,不归 S2;
  核实锚定:head <sha> / 基底 main <sha>           修完重打 fixed;3 次熔断 → question)
    ↓
S4 门禁:CI 绿 + PR 级 verified + 新鲜度两查(feature/ 另加 VERDICT: PASS)
    ↓ 全过 → 自动 squash 合并(无关联 issue,无单可关)
```

## 6.4 验收标准段规则(Phabricator Test Plan 同构)

- PR 模板固化「## 验收标准」段;每条 = 操作步骤 + 期望结果,达到「不熟悉本改动的人照着能复现」;
- S3 对缺段/不可执行的直管 PR 退回补写,**不自行编造验收标准**——核实基准只能来自作者陈述。

## 6.5 新鲜度两查(S4 侧,免费计划的自建等价物)

1. **head 锚定查**:当前 head == 锚定 head → 有效;唯一例外是 S4 自己 update-branch 的基底同步 merge;有功能性新提交 → verified 作废,摘回 fixed 重核;
2. **基底新鲜度查**:锚定 main ≠ 当前 main tip → `gh pr update-branch` 触发 CI 在组合态重跑,本轮跳过,下轮凭新 CI 判。

## 6.6 直管 PR 的例外处置(每条都有归属人)

| 状态 | 归属人 |
|---|---|
| PR 级 rework | 原直管会话(在世)/主人经裁决会指定返工人/丢弃——**不归 S2**(S2 只认 issue);S5 扫描面含 PR 级 rework,看门人上报 |
| 冲突(CONFLICTING) | 原会话优先(防撞车);缺席超 2h → 看门人 rebase 兜底 |
| 三次熔断 | PR 级 question → S5 裁决会 |

## 6.7 不变的边界

- chore/docs PR 永不自动合(机制类人工把关);
- issue 通道行为完全不变,PR 通道只覆盖不带号的直管 PR;
- 主人当面授权永远高于自动机制(紧急当面说「合」即可插队);
- S3 红线不变:核实会话永不 push、永不合并。
