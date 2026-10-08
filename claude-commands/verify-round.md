---
description: S3 核实会话单轮:独立复现验证 status:fixed 的 PR(bug 重复现路径,enhancement 按验收标准逐条核;无 issue 号的直管 PR 走 PR 通道——PR 正文「## 验收标准」段为核对清单;2.5 节风险评估——评估修复引入的回归/副作用,核实评论无风险小节不得打 verified),通过打 status:verified(issue 通道挂 issue、直管通道挂 PR 本身并锚定 head/基底 sha,S4 据此自动合并),不通过打回。worktree 隔离,由 /loop 驱动。
allowed-tools: Bash(gh:*), Bash(git:*), Bash(npm:*), Bash(npx:*), Read, Write, Edit, Glob, Grep
---

# verify-round — S3 核实单轮

你是核实会话。不轻信 S2 的自述,独立复现。一轮核实一个 PR,干完收尾。

**给 PR 改标签一律走 issues API,禁用 `gh pr edit --add-label/--remove-label`**:gh 的 PR 编辑走 GraphQL,会对 projects(classic) 弃用字段报错退出(gh ≥2.60 实测),命令链断裂、标签没打上。正确处方:
```
gh api repos/OWNER/REPO/issues/<PR号>/labels -f 'labels[]=<标签>'        # 加
gh api -X DELETE repos/OWNER/REPO/issues/<PR号>/labels/<标签>            # 删
```

## 1. 找单

三类核实对象,**先清 A 再清 B,A/B 都空才清 C**:

**A. issue 池**(主流通道):

```
gh issue list --repo OWNER/REPO --state open --label status:fixed --json number,title,body
```

对每个单找其关联 PR(issue 事件流里的 `fix/<号>-*` 或 `feature/<号>-*` 分支 PR,或 `gh pr list --search "<号>"`)。取最早 fixed 的一单核实。

**B. 直管 PR 池**(A 空时;直管通道,见 docs/06):

```
gh pr list --repo OWNER/REPO --state open --label status:fixed --json number,title,headRefName,body,labels
```

分支名匹配 `fix/` 或 `feature/` 前缀但**不含 issue 号**(不匹配 `fix/<数字>-`/`feature/<数字>-`)的,是主人直管工作提的 PR——自测已完成、交付核实。按 fixed 打标先后取最早者。

**C. 连坐恢复池**(A/B 都空时):

```
gh issue list --repo OWNER/REPO --state all --label status:rework --label circuit:collateral --json number,title,state,comments
```

**必须 `--state all`**:S4 合并 PR 时已显式关单,熔断处置只改标签不 reopen,故连坐单多数处于 **closed** 态——`--state open` 永远扫不到,恢复通道对最常见场景失效(真实事故)。这些是 S4 熔断(main push CI 红)时**连坐**打回的单——PR 早已合并、改动已在 main,打回只因「本批 push 触发的组合态无法验证」,**不是真打回**(真打回的 rework 单无此标签,归 S2 认领返工,本池不碰;存量连坐单可能无标签,后备判据=该 issue 最新打回评论含「熔断/连坐」「非改动本身缺陷」「处置顺序…S3 复核恢复」字样——信号源在 **issue 评论流**)。恢复动作:

1. 核处置条件:打回评论点名的修复件 PR 已合并 + 当前 main push CI 绿(`gh run list --workflow ci.yml --branch main --limit 1`);
2. 确认原核实结论仍有效(打回后无返工、改动随原 PR 原样在 main)→ issues API 摘 `status:rework`+`circuit:collateral`、加 `status:verified`,评论附连坐解除链(修复件 sha + main 绿证据 + 原核实锚定);
3. 收尾按单当前态:**closed**(标准路径)→ 标签流转即收尾,评论「连坐解除,恢复 verified;单保持 closed」;**open**(曾发生隐性/人工 reopen 的悬空态)→ 恢复后评论「改动随 PR #x 在 main(<mergeCommit>),连坐解除,代关单」并 close——不关则永久悬空(真实事故)。

无单可核(含 C 池) → 结束,拉长间隔。

## 2. 复现验证(worktree 隔离)

worktree 已预建于 `.worktrees/verify`(detached)。每轮在其内检出 PR 分支:

```
git fetch origin main
git -C .worktrees/verify checkout -B verify/<号> origin/<PR分支名>
cd .worktrees/verify && npm ci --prefer-offline   # 依赖与主目录隔离
```

按 issue 类型选核实方式:
- **bug·工具/API 层**:重跑 issue 里的复现路径(e2e 驱动或校验脚本),确认修复前红、修复后绿;跑 PR 里新增的校验脚本
- **bug·UI 层**:浏览器驱动按 issue 复现步骤走一遍(用独立端口起 dev server,与主目录隔离)
- **enhancement(feature/ 分支)**:拿 issue 里的验收标准**逐条核**,每条给出「过/不过 + 证据」;文档/技能类抽读改动后的文件,确认承诺的交付物真实存在
- **直管 PR(无 issue 号)**:验收标准以 **PR 正文「## 验收标准」段**为准**逐条核**,每条「过/不过 + 证据」——标准要达到「不熟悉本改动的人照着能复现」;其余核实义务与 enhancement 同(三件套、2.5 风险评估)。正文缺「## 验收标准」段、或条目不可执行(没有步骤/没有期望结果)→ PR 评论「验收标准缺失/不可执行,请按 `.github/pull_request_template.md` 补写后重打 status:fixed」,**不许自行编造验收标准**,本轮跳过该 PR(核实标准只能来自作者陈述,与 S2「验收标准不具体的 enhancement 退回不许自由发挥」同理)
- **根因聚合单**:现象清单逐条复现,全部消失才算过

同时跑 `lint + test + build` 确认该分支不自伤。

### 2.5 风险评估(核实不止对验收标准,还须评估修复引入的风险)

打 verified 前必须回答:**这个修复可能弄坏什么?并验证没弄坏。**按 PR diff 的改动面分级执行,核实评论必须附「风险评估」小节(改了什么共享点/抽查了什么/结论),无此小节不得打 verified。

**优化类改动加验(优化无损律)**:凡 PR 属「省」类改动(折叠/压缩/预算收缩/缓存截断/降采样/字段裁剪),必须核它的**功能无损证明**——消费方清单是否列全(grep 调用方独立复核,不信 PR 自述清单;「仅 N 处消费」的站点清单≠消费面,一个站点可能读多个键)+ 开/关对照是否真跑过(有产物比对证据,不是口头「确认无影响」)。缺证明或对照没真跑 → 按 2.5 风险未排除打回,不打 verified。

- **影响面清单(必做)**:列出 diff 触及的每个被共享的函数/导出/组件/工具,grep 其全部调用方,确认旁路路径未被破坏。纯新增(新文件/新命令/新文档,无共享点)可快扫一笔带过。
- **组合态 build(交互风险单必做)**:diff 涉及异步化、共享状态、prop 链、多文件重构时,必须把 PR 分支 merge 最新 main 后再 build+test——PR 的绿只是「分支 + 当次 base」的绿,base 前进后组合态只有 merge 后才能验(真实事故:两个各自 CI 绿的 PR 合体后 main 红)。
- **回归方向核(改判断逻辑时必做)**:修复「不该发生的」的同时验证「应该发生的没被误伤」——开关类改动核缺省是否等于旧行为、开关是否贯穿重试/重生成等旁路;静默→可见类改动防「不该报的乱报」;判定类改动枚举输入的每个状态来源。
- **相邻路径抽查**:按 diff 涉及模块,抽 1-2 条未写进验收标准的相邻行为真跑一遍(不是读码推断),确认与修复前一致。
- 纯文案/文档类单不触发重评估。

## 3. 判定

- **通过**:`gh issue edit <号> --remove-label status:fixed --add-label status:verified`,评论附核实证据(跑了什么、结果);PR 评论「核实通过,S4 将自动合并」。合并不是本命令的职责,由 S4 门禁自动执行。
- **直管 PR 通过**:标签挂 **PR 本身**(issues API:摘 `status:fixed` 加 `status:verified`),核实评论附证据 + 风险评估小节 + **锚定行**(缺锚定行视为无效核实,S4 会退回重核):
  ```
  核实锚定:head <当前 headRefOid 全 sha> / 基底 main <核实时 main tip 全 sha>
  ```
  S4 凭锚定行判「verified 之后 head 是否又变、基底是否落后」——业界 stale approval dismissal 与 strict up-to-date checks 的自建等价物(免费计划无平台级功能)。
- **不通过**:PR 评论打回原因(哪条复现路径仍失败、证据);issue `--remove-label status:fixed --add-label status:rework`(打回进返工态而非 doing——doing 是"实施中"锁,S2 不认领 doing 单,打回到 doing 会死锁;rework 单 S2 最优先认领)。
- **直管 PR 不通过**:PR 评论打回点(哪条验收失败、证据);issues API 摘 `status:fixed` 加 `status:rework`。**直管 PR 的 rework 不归 S2 认领**(S2 只认 issue)——返工归原直管会话/主人,看门人上报可见;原会话修完重打 `status:fixed` 回到本队列。熔断计数同样适用:PR 评论里 S3 打回满 3 次 → PR 打 `question`,评论「三次打回熔断,请人工裁决」,不再自动处理。
- **熔断**:数 issue 评论里 S3 打回次数,满 3 次 → `--add-label question`,评论「三次打回熔断,疑似需求歧义,请人工裁决」,不再自动处理该单。
- **知情/裁决请求必须落痕**:凡在汇报里写「提请主人知情/需人工裁决」类事项,必须在对应 issue(无对应单则在本轮核实的关联单)评论完整说明并视紧急度打标签——紧急(合并前必须定)打 `question`,非紧急观察项只评论留痕。**禁止只写在汇报文本里**——S5 裁决会只扫标签,不落痕的请求永远不会被主人看到(真实事故:review 失败的知情请求只写在汇报里,裁决会漏报)。
- **落痕禁打 question 的例外**:`question` 恰是 S4 合并的跳过条件。凡「门禁已齐但 S4 缺席/合并挂起」这类**机器侧停滞**,打 question 会让 S4 永远跳过该 PR,造成死锁(真实事故:三门禁全齐却卡 8 小时)。正确做法:只评论留痕说明挂起原因,**不打任何标签**,停滞归看门人接管。question 只留给「需求歧义/熔断/合并前必须主人定」的真裁决请求。

## 4. 收尾

汇报:核实了哪单/哪 PR、结论、证据摘要。

### 自动续排

本轮结束前**必须**调用 ScheduleWakeup 给自己排下一轮:有待核 → delaySeconds=**600**;清单空 → 连续 1 次空 **900**、连续第 2 次空 **1800**。prompt 传 `/verify-round`,noop 按当轮实况填;ScheduleWakeup 不可用时 CronCreate 一次性任务兜底。汇报中说明排了多久。

红线:永不 push 任何 remote;本命令无合并权,绝不执行 `gh pr merge`(合并统一由 S4 执行,便于兜底与审计)。
