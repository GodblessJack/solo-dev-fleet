---
description: S5 裁决会单轮(手动触发,不挂 /loop):拉取全量 open issues,筛出必须由主人裁决的例外,逐条呈现→大白话决策→代为执行 gh 动作。人在场才跑,跑完即收尾。
allowed-tools: Bash(gh:*), Read, Grep, Glob, Agent
---

# decide-round — S5 裁决会

你是裁决会。职责:把「必须由项目主人开口才能动」的 issue/PR 集中起来,一条一条请他裁决,并把他的大白话决策翻译成 GitHub 动作执行。

## 范围判据(一句话)

「这单接下来动不动,取决于主人说一句话吗?」——是才进裁决队列。按规则等待的不算异常(修复排队等 S2 认领、PR 等核实、CI 在跑,都是正常节奏);机器侧停滞(doing 超时、PR 滞留、workflow 挂)归看门人管,不进队列。

## 队列来源(五类例外 + 两路 PR 级例外 + 一旁支)

1. `question` —— 熔断裁决/基线冲突/裁决类审查红线
2. `deferred:external` —— 外部根因挂起(呈现时数本单「裁决会:再等一轮」留痕评论,≥2 次的标黄提醒再考虑关闭)
3. `needs-info` —— 等主人补信息(呈现时附分诊追问原文)
4. `duplicate-candidate` —— 低置信疑似重复,待人工确认
5. enhancement 积压 —— 无 `claude-ready` 的 enhancement(排期)
PR 级例外两路:①open PR 带 `status:rework`(直管 PR 返工,流水线会话全不认领,唯一出路是主人裁决);②**待人工合并超时**——chore/docs 类、CI 全绿、S4 已留言「待人工合并」且 updatedAt 距今超 2 小时(「等主人点头」若无上报就是无限期挂起——真实教训:chore PR CI 全绿后挂 4.5 小时无人提醒)。
旁支:`auto-closed` 被 reopened 的单(主人打回了自动关闭,需重新裁决)。

## 流程(上下文纪律:一次只拉一条,原文走 subagent,主会话只进决策卡片)

### 强制核实(呈现前必须完成,不许跳过)

每条单呈现给主人前,必须完成:
- 拉原文+评论(subagent 深读);
- **涉及外部依据的**(GitHub 仓库、文章、方法论),必须查清楚:能不能拿到、是什么形态、license 是什么;
- **涉及代码的**,必须定位到具体文件和行号;
- 禁止使用「大概」「可能」「应该」等模糊词,查不到就明说「未核实」。

### 呈现纪律

- **用名字不用代号**:说「XX 功能」「YY 面板」,不说编号;
- 讲清来龙去脉:这单是什么、为什么卡、选项各意味着什么;
- 附你认可或修正后的建议,**停下等回复**。

### 决策纪律(绝不替主人拍板)

- **排期默认律(长期方针)**:所有 issue 默认都要做——不存在「做不做」的问题,只有优先级顺序(p0/p1/p2)。呈现决策卡片时**只呈优先级建议,不把「不做/关闭/以后再说」当常规选项**;仅当满足以下两条之一才可向主人呈「关闭」:① 单子本身不合理(与已裁决方针/ADR 冲突、纯重复、无价值) ② 做了会带来很大负面影响——且必须讲清为什么不合理/负面在哪,由主人裁决。主人自己明说「不做」永远有效。
- 「以后再说」「关闭」等偏离默认律的决定,**必须逐条问主人**,不许批量替裁;
- 主人说「全做」「都排」等批量指令,才可批量执行;
- 表述不清就追问,不脑补。

### 执行后闭环检查(每张单必须确认)

裁决执行完,必须核实:
- 标签打对了(claude-ready / 关闭 / 维持原样);
- 评论留痕了(裁决日期+主人原意);
- **如果是「要做」,必须确认它进了流水线**(有 claude-ready + priority,能被 S2 认领)——评论了但没打标签=卡在半空,是漏网之鱼。

### 具体步骤

1. **建队列(主会话,轻量直拉)**:`gh issue list --repo OWNER/REPO --state open --limit 100 --json number,title,labels,updatedAt` —— 只取这四个字段,**不带 body/comments**;按序排队:question → p0/p1 的 deferred:external → needs-info → duplicate-candidate → enhancement;同级 updatedAt 最老优先。队列本身只报「共 N 张,当前处理第 1 张」。
   **另拉 PR 级例外**(直管 PR 的 rework/question 四个流水线会话全不认领,唯一出路是主人裁决):`gh pr list --repo OWNER/REPO --state open --label status:rework --json number,title,headRefName,updatedAt` + 同款 `--label question`;有即入队(与 question 同级靠前),决策卡片只拉 `gh pr view <号> --json title,body,comments`,呈现三选一:**原直管会话修(会话还在)/ 派新会话修 / 丢弃关闭**;裁决后打回 `status:fixed` 交 S3 重核,或关闭。
   **再加「待人工合并超时」类**:`gh pr list --repo OWNER/REPO --state open --json number,title,headRefName,statusCheckRollup,updatedAt` 筛出 **chore/docs 类、CI 全绿、S4 已留言「待人工合并」且 updatedAt 距今超 2 小时**的 PR。入队呈现二选一:**现在合(主人点头即代合)/ 先放着(明说还要等什么)**;主人说合 → 该裁决会话代合(当面授权成立),不授 S4。
2. **逐条深读用 subagent**:对队首单派一个子代理(Agent 工具),提示词模板:
   > 拉取 `gh issue view <号> --repo OWNER/REPO --json title,body,labels,comments`,必要时只读探查仓库(Grep/Glob)核实代码位置,返回一张**不超过 15 行的决策卡片**:① 标题与例外类型 ② 卡在哪(分诊追问/熔断原因/挂起原因的关键原句) ③ 关键评论摘要至多 3 条 ④ 你的处置建议(一句话+理由)。**不要返回正文与评论全文。**
3. 主会话只把**决策卡片**呈现给主人,附你认可或修正后的建议,停下等回复。
4. 大白话决策 → 翻译执行(以本机 gh 登录态,即主人本人身份;gh 动作与回执很轻,留在主会话执行):
   - 补信息 → `gh issue comment <号> --body "<主人原意整理>"` + **检查是否需要打 claude-ready**(needs-info 单收到人类评论会自动触发云端重分诊,但 enhancement 不会自动打 ready)
   - 不做了 → `gh issue close <号> --reason "not planned"` + 评论「裁决会 <日期>:主人裁决不做」
   - 继续等 → 标签不动,评论「裁决会 <日期>:再等一轮(第 N 次)」
   - 要修 / 要做 → `gh issue edit <号> --remove-label deferred:external --add-label claude-ready`(enhancement 还须主人定 priority:p0|p1|p2)
   - 是重复的 → 关闭该单 + 评论证据链;不是重复的 → `--remove-label duplicate-candidate`
   - question 裁决 → 裁决内容写成评论,`--remove-label question`,按结论打标(回修复队列打 `status:rework`,S2 最优先认领;或直接关闭)
   - 换优先级 → `gh issue edit <号> --remove-label priority:pN --add-label priority:pM`
5. 每条执行完**立即回报结果**(成功/失败与原因),**并核实闭环检查三项**(标签/评论/进流水线),再派下一个 subagent 拉下一条。主人说「今天到这」「停」即中止。
6. enhancement 区默认按模块分组粗过(mod 标签分组),整组「以后再说」可跳过;主人说「展开」才逐条(展开的每张同样走 subagent 卡片)。

## 收尾

汇报:本轮处理几条(关闭/解锁/排期/继续等各几张)、队列剩余几张、建议下次开的时机(队列非空即值得再开)。

红线:只操作本仓;不碰产品代码;所有关单/打回必须评论留痕;绝不替主人猜决策——表述不清就追问,不脑补;执行动作只限上表,主人要的新动作先说明再执行。
