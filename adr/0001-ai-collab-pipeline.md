# ADR-0001:AI 协作开发流水线——GitHub 账本 + 六会话 + 云端六 workflow

日期:2026-09(初版);本文件为脱敏开源版,隐去项目细节,保留全部决策与修订史
状态:已裁决,在生产中运转

## 背景

要建立一套「1 个人 + N 个 AI 会话 + GitHub」的协作开发体系。经逐项拷问确认,决定:

**① 代码托管**:私有仓作为代码主仓与唯一账本(GitHub Issues/PR/Projects);GitHub 免费计划够用——merge queue/branch protection 等付费能力全部用会话规则自建等价物。**② 批注通道**:提单用户只有项目主人,故走纯本地小回路——产品内标注 POST 到本地 dev server,后端用本机 gh 登录态直接建 issue,不建对外部署基座。**③ 测试口径**:两层,工具/API 层 e2e 为主(mock 档高频 + 真调档低频按日预算),UI 浏览器走查为辅——缺陷分布证明问题集中在工具层。**④ 云端层全量**:ci(唯一硬门禁)+ triage + claude-review + claude 遥控 + canary 心跳 + model-health 探针;裸装 Claude CLI 用 `claude -p`,云端模型端点复用本地会话端点(Secrets 配 `ANTHROPIC_BASE_URL`+`ANTHROPIC_API_KEY`,模型名用 API 裸名)。**⑤ 合并权渐进放权**:一期核实通过只打 verified 等人工合,验证后切自动。**⑥ 运行强度**:会话常驻,`/loop` 自适应间隔(忙 5-15 分钟/闲 30-60 分钟)。**⑦ 多入口查重三层防线**:提单时结构化锚点(mod 标签+代码位置/工具名+复现路径)自查;分诊时双轨查重(锚点一致=高置信,仅现象疑似=低置信);同根因不同验证路径的单不关闭,聚合到根因 issue 的现象清单;高置信重复允许 AI 自动合并关闭(附证据链+「误关请重开」),低置信疑似一律待人确认。

## Considered Options(被拒绝的替代方案)

- **GitHub 只当账本、代码不上云**:PR 流/CI 门禁/云端 triage-review 整体垮掉,体系砍掉一半。
- **本地主仓 + 脱敏镜像**:脱敏边界难维护、镜像滞后导致核实失真,维护成本最高收益最不确定。
- **批注通道走对外部署的 Serverless Function**:前提是对外部署站点,local-first 产品无外部测试者,不需要;接口设计保留升级可能。
- **只测 UI 层**:与缺陷分布严重不匹配(缺陷集中在工具/API 层)。
- **合并权一开始全自动**:体系未验证期坏代码可能直接进 main;永远人工把关则主人成为流水线瓶颈。渐进放权替代。
- **纯文本相似度查重**:多入口下同根因描述语言完全不同(工具报错 vs 界面现象 vs 大白话),文本查重基本失效——结构化锚点替代。
- **云端用官方 Anthropic API**:兼容性最省心但费用最高;复用当前会话端点一致性最好。

## 决策(角色与门禁)

六会话:S1 `/qa-round`(测试提单)、S2 `/fix-round`(worktree 实施)、S3 `/verify-round`(worktree 独立核实)、S4 `/merge-round`(纯 gh 合并)、S5 `/decide-round`(手动裁决会)、看门人 `/monitor-round`(常驻巡检)。单轮命令 + `/loop` 驱动,不常驻业务进程。

合并门禁:`fix/` 双门禁(CI gate 绿 + status:verified);`feature/` 三门禁(+ AI 审查 VERDICT: PASS);chore/docs 永不自动合;无 issue 号直管 PR 走 ADR-0002 的 PR 级通道。

## Consequences

- 标签词汇表:`from:qa/from:human/from:annotation/from:canary`、`bug/enhancement/question`、`priority:p0-p2`、`status:doing/fixed/verified/rework`、`claude-ready`、`needs-info/duplicate-candidate/auto-closed`、`deferred:external`、`circuit:collateral`、`mod:*`(按项目自定义)。
- 人工介入点收敛为:question 裁决、enhancement 排期、低置信重复确认、chore/docs PR 合并、误关重开。
- 文档纪律:每条机制文档与其载体同 PR 合入;CLAUDE.md 扩为项目宪法,沟通原则置顶。
- 已知局限:本地截图在 GitHub 网页不可见(本地路径方案);真调档 e2e 有预算上限,覆盖率受限;常驻 /loop 空轮烧 token。

## 修订记录(全部真实事故驱动)

- **新增 S5 裁决会**:补「例外处理入口」。手动触发的决策会议,与自动巡逻分工(巡逻=机器健康自动报,裁决会=人的决定集中消化)。判据「这单接下来动不动,取决于主人一句话」。
- **决策⑤提前放权(fix/ PR 验证通过即自动合并)**:原「verified 等人工合」在实测中暴露要害——测试会话测的是 main,修复滞留 PR 期间每轮重复撞同一 bug,白烧测试轮次;且 PR 队列仅 1-2 个在飞,人工合并成为纯瓶颈。
- **enhancement 实施链开通(S2 扩认领 + feature/ 三门禁)**:复盘发现设计洞——一批 enhancement 打了 claude-ready 却无任何角色认领(原实施会话只认 bug),就绪闸门的「ready 即可被流水线认领」承诺落空。
- **打回进 rework 而非 doing**:doing 是「实施中」锁,S2 不认领 doing 单(防抢活),打回到 doing 造成过无人认领死锁;rework 是「待返工」,S2 最优先认领。
- **熔断连坐恢复通道(circuit:collateral)**:S4 熔断(main push CI 红)把已合并 PR 的 issue 打回 rework 后,处置条件满足时无任何扫描面能自动恢复——曾有连坐单停滞 13 小时。修订:S4 熔断加 collateral 标签+固定字样评论;S3 增连坐恢复池(`--state all`,因 S4 合并时已显式关单);S2 认领面对 collateral 单一律跳过。
- **打回分类双路**:S4 门禁三打回原一律 rework——但审查红线有两类:缺陷类(S2 能修)与裁决类(ADR 冲突/职责归属/需求歧义,须主人拍板);裁决类红线走了 rework,S2 认领也无权选边,单挂死一天。修订:S4 打回先分类,裁决类打 question 进裁决会;S2 认领面加同款分类出口(禁自行选边)。
- **superseded PR 清理归属**:合集 PR 被分拆件顶替后挂一天无人关——S4 只管合、S3 只核 fixed 单,废件无归属人。修订:S4 每轮对带号 open PR 做 superseded 判定(关联 issue 已关且 main 有合并件 → 关 PR 留痕)。
- **S2 认领条件自相矛盾修正**:可认领条件的排除清单误列 status:rework,与「rework 最优先认领」矛盾,字面执行 = 返工单永远无人认领。
- **新增看门人 monitor-round**:三环各自的自动续排只保证「活着会续」,不保证「死了有人知道」——S4 两次无声死亡、连坐单停滞 13h、chore PR 挂 4.5h 都是实录。看门人=三环心跳+「等主人」超时上报+自报心跳文件(自己死了靠 mtime 被主人肉眼发现)。
- **「等主人」状态必须带上报**:chore PR 待人工合并无超时上报 = 无限期挂起(4.5 小时实例)。修订:S4 显著标出、看门人上报、S5 入裁决队列。设计律落成明文:**每个状态都要有归属人**。
