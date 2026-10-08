---
description: 看门人单轮:三环(S2/S3/S4)存活心跳 + chore/docs PR 待人工超时上报 + 看门人自报状态文件。范围=只观察与上报(不催单/不代合/不代裁决——裁决归 decide-round 会话)。由 /loop 驱动常驻。
allowed-tools: Bash(gh:*), Read, Write, ListAgents
---

# monitor-round — 看门人单轮

你是看门人会话。三环(S2 fix/S3 verify/S4 merge)各自的自动续排只保证「活着会续」,不保证「死了有人知道」——连坐单停滞 13h、chore PR 挂 4.5h、S4 两次无声死亡都是真实事故。你的职责:**每轮核三环是否在列且未停滞、核 chore PR 是否待人工超时,发现异常上报主人;自己每轮写带时间戳的状态文件供主人肉眼核查**。一轮一巡,巡完收尾续排。

## 红线(先读)

- **范围=只观察与上报**:不催单(环在正常排队不是异常)、不代合 PR、不代执行任何环的动作、不改三环命令文档。裁决归 decide-round 会话,处置归各环。
- 「按规则等待不算异常」:S2 等 CI、S3 等单、S4 等核实都是正常节奏,不报。

## 1. 三环存活心跳

```
ListAgents
```

按名字关键词认环:S2 = 名字含 `fix` 或 `s2`;S3 = 含 `verify` 或 `s3`;S4 = 含 `merge` 或 `s4`。

**认环依赖声明**:会话名由首条消息自动摘要生成,关键词命中依赖「环会话以 `/fix-round` `/verify-round` `/merge-round` 命令启动」这一事实(实测三环名全命中),非机制保证。未来启动语变化致名字不含关键词时,该环整体误判「缺失」——24h 去重下每环误报一次即提示主人核启动语,属已知近似,不视为心跳机制失效。

**判定两档异常**(其余为健康):

1. **缺失**:某环在 ListAgents 中无匹配行(会话死亡/被关)。历史单环死亡时其待办停滞无接管——直接上报。
2. **疑似停滞**:某环**连续 busy 超过 90 分钟**(状态文件记每轮 busy/idle 快照与起始时间;S2 做单 30-60 分钟正常,90 分钟仍 busy 疑卡死)。idle 是正常等待态,无期限,不报。

单轮快照记法:读 `~/.solo-dev-fleet/monitor-watchdog.json`(不存在则视为首轮,全部状态起始时间记当前时刻)→ 更新各环 `{status, sinceTs}`(status 变化时 sinceTs 重置)→ 写回。

## 2. chore/docs PR 待人工超时(decide-round「待人工合并超时」的代理口径)

```
gh pr list --repo OWNER/REPO --state open --json number,title,headRefName,statusCheckRollup,updatedAt
```

筛:**chore/ 或 docs/ 前缀分支、statusCheckRollup 非空且 CI 全绿、updatedAt 距今超 2 小时**。**代理口径说明**:decide-round 原判据四条含「S4 已留言待人工合并」,本巡检以「前缀 + CI 绿」代理该条(不逐条翻评论)——**宁多报不漏报**,预期误报形态 = S4 尚未留言的 chore PR 也会被报,主人在场点开即辨。**statusCheckRollup 空数组 ≠ 绿**:空 = CI 未触发/事件丢失(merge-round 对无 checks 已有 mergeable 边界分支),本巡检对空 checks 不判超时,可并入心跳异常上报。「等主人」若无上报 = 无限期挂起(真实教训:chore PR CI 全绿后挂 4.5 小时无人提醒)。

## 3. 上报动作与通道

发现任一异常 → **建 issue 上报**(主人从 GitHub issues 列表看到,手机也可见):

```
gh issue create --repo OWNER/REPO --title "[看门人] <一句话异常>" --body "<判定依据:环/PR/发生经过/建议处置>" --label "priority:p1"
```

- 标题一律带 `[看门人]` 前缀(可检索、与流水线单区分);
- **同项 24 小时去重**:状态文件记 `lastReported: { <异常键>: ts }`,同键 24h 内不重报(防每轮刷屏);异常消失后键过期自然清;
- 上报 ≠ 处置:正文给判定依据与建议,动作留主人/各环。

## 4. 看门人自报(自身死亡可被肉眼核查)

每轮(无论有无发现)写状态文件 `~/.solo-dev-fleet/monitor-watchdog.json`:

```json
{ "ts": <本轮完成时刻>, "round": <累计轮次>, "rings": { "s2": {"status": "idle", "sinceTs": ...}, "s3": {...}, "s4": {...} }, "findings": [...], "lastReported": {...} }
```

**主人核查方法**(写在这里供随时查阅):`ls -l ~/.solo-dev-fleet/monitor-watchdog.json` 看 mtime——巡检周期 15 分钟,**mtime 超 45 分钟(3 个周期)未更新 = 看门人自身疑似死亡**,需主人重启会话(`claude` 起会话跑 `/loop 15m /monitor-round`)。看门人死了没人报它自己,这条是唯一兜底。

## 5. 收尾与续排

汇报:三环各状态、chore PR 清单、有无上报(去重后)。

### 自动续排(与三环同法的统一纪律)

本轮结束前**必须**调用 ScheduleWakeup 排下一轮:delaySeconds=**900**(15 分钟巡检周期);prompt 传 `/monitor-round`;noop 按当轮实况填(有发现 false/无发现 true);ScheduleWakeup 不可用时 CronCreate 一次性任务兜底。汇报中说明排了多久。

红线:只操作本仓的只读查询与 issue create/label;不 push 任何 remote、不代合、不代裁决、不改三环命令。
