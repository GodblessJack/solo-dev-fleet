---
description: S4 合并员会话单轮:扫描 open PR,一轮至多合并一个(fix/ 双门禁、feature/ 三门禁——verified 两通道:issue 号通道或无 issue 号直管 PR 的 PR 级 verified 通道含新鲜度两查;FIFO 取最早门禁齐者),合并后隔轮核 main push CI——红即熔断回滚处置不再合,chore/docs PR 待人工合并;收尾把 main 同步回本地主目录。由 /loop 驱动。
allowed-tools: Bash(gh:*), Bash(git:*)
---

# merge-round — S4 合并员单轮

你是合并员会话。gh 操作 + 本地 main 同步;不 checkout 分支、不构建、不改代码。

**给 PR 改标签一律走 issues API,禁用 `gh pr edit --add-label/--remove-label`**:gh 的 PR 编辑走 GraphQL,会对 projects(classic) 弃用字段报错退出(gh ≥2.60 实测),命令链断裂、标签没打上。正确处方:
```
gh api repos/OWNER/REPO/issues/<PR号>/labels -f 'labels[]=<标签>'        # 加
gh api -X DELETE repos/OWNER/REPO/issues/<PR号>/labels/<标签>            # 删
```

## 流程

**一轮至多合并一个(精确和高质量是最高准则,覆盖「一轮全收」)**:每轮最多新合**一个** PR;合并对象按 FIFO——从门禁全齐的 PR 中取创建时间**最早**者;其余门禁齐的 PR 留待下轮。理由:同轮连环合 A→B→C 时,B/C 的绿是在不含前者的旧 base 上跑的(组合态断链真实事故);逐个+隔轮复核才能把每个合并都钉在「最新 main + push CI 已验证」上。

0. **上轮合并复核(上轮合过 PR 则先做)**:查上次合并 push 触发的 main CI(`gh run list --workflow ci.yml --branch main --limit 1 --json status,conclusion,displayTitle`):
   - 还在跑(status≠completed)→ 本轮不合新 PR,自排 wakeup 600s 等结论;
   - conclusion=success → 继续下面流程;
   - conclusion=failure → 组合态断链:立即提 p0 issue(附失败 job 名与「squash 单提交,git revert 一键回滚」配方)、该 PR 关联 issue 摘 verified 打 `status:rework` **并加 `circuit:collateral` 标签**(熔断连坐的结构化标记——rework 态不进 S3 的 fixed 扫描面,处置条件满足后靠该标签被 S3 连坐恢复池发现自动复核恢复;不加标签的连坐单曾停滞 13 小时无人恢复——真实事故;标签不存在则先 `gh label create circuit:collateral --description "熔断连坐标记,归 S3 连坐恢复池" --color FBCA04` 再打——仓内无此标签时加标会报错,与摘 verified 拼同一命令会让整条失败连 rework 都没打上)、**issue 评论留连坐说明**(固定字样:「熔断连坐打回,非改动本身缺陷;处置顺序:修复件合并 → main 回绿 → 本单由 S3 复核恢复」——S3 连坐恢复池对存量无标签单靠这些字样做后备识别,信号源在 issue 评论流,PR 评论留证不能替代)、PR 评论留证据、汇报显著标出,**本轮不再合并任何 PR**。
1. `gh pr list --repo OWNER/REPO --state open --json number,title,headRefName,labels,statusCheckRollup,createdAt`
2. 逐个分类处理:

### 前置:无 checks 的 PR(防静默卡死)

若某 PR 的 `statusCheckRollup` 为空,先 `gh pr view <号> --json mergeable`:**mergeable=CONFLICTING → GitHub 对冲突 PR 不触发任何 workflow,CI/review 永远不会来**,不处理就是无限死等。S4 无代码权限,做两件事:PR 评论「与 main 冲突导致 CI 未触发,需 rebase/解冲突后重推」+ 汇报显著标出。**冲突处理归属**:issue 通道单归 S2;直管 PR 归原直管会话(在世且活跃则等它,防与其推送撞车);两类归属者缺席或挂起**超 2 小时无动作 → 看门人 rebase 解冲突重推兜底**(chore 类冲突同此——解冲突只为恢复 CI,合并与否仍待主人)。S4 汇报里注明「已按归属指派/已@看门人」。mergeable=MERGEABLE 但无 checks 属事件偶发丢失,空提交或 close/reopen 重触发即可(间隔 ≥12s 防去抖;触发后必查新 run 已创建),同样汇报。

### fix/ PR(自动合并,双门禁缺一不可)

门禁一·CI:`statusCheckRollup` 中 ci 的 gate job 必须绿;红 → 评论提示失败 job 后跳过;进行中 → 跳过等下轮。
门禁二·独立核实(两条通道,按分支名二选一):
- **issue 通道**(分支名 `fix/<N>-...`):`gh issue view <N> --json labels` 确认有 `status:verified`,且无 `status:rework`/`status:doing`/`question`/`needs-info`。
- **PR 通道**(分支名 `fix/` 但不带 `<N>-`,即直管 PR):PR 标签有 `status:verified` 且无 `status:rework`,并过**新鲜度两查**(见下节)。

两门禁都过 → **仅当它是本轮门禁全齐候选中 createdAt 最早者**才执行合并(其余留待下轮):`gh pr merge <号> --squash --delete-branch`,评论记录「双门禁通过,已自动合并」;issue 通道**必须显式关单**:`gh issue close <issue号> --comment "PR #<号> 已 squash 合并,关闭"`——squash 提交信息不保证带 Closes 引用,不显式关单 issue 会永远挂着(真实事故:PR 合并后 issue 无人关);PR 通道无关联 issue,评论记「直管通道双门禁通过,已自动合并」即可,无单可关。
门禁二不过(未 verified) → 不碰,那是 S3 的职责;在汇报中列出即可。

### PR 通道新鲜度两查(直管 PR 必做;业界 stale dismissal + strict up-to-date 的自建等价物,免费计划无平台级功能)

取 PR 评论里**最新一条 S3 核实评论**的锚定行 `核实锚定:head <sha> / 基底 main <sha>`(S3 打 PR 级 verified 必附;找不到锚定行 = 无效核实,评论提示 S3 补锚定,跳过):

1. **head 锚定查**(verified 不随新提交自动失效——标签没有平台级失效语义,必须自己查):
   - 当前 `headRefOid` == 锚定 head → 本查通过;
   - 不等 → `gh pr view <号> --json commits` 看新增提交:**唯一**新增是基底同步 merge(message 形如 `Merge branch 'main' into`,且 PR 评论里有本会话先前轮次的「已 update-branch 同步基底」留痕)→ verified 仍有效,继续第 2 查;否则(有功能性新提交)→ **verified 作废**:issues API 摘 `status:verified` 加 `status:fixed`,评论「verified 因新提交作废(核实锚定 head ≠ 当前 head),已重回 status:fixed,S3 将重核」,本轮跳过。
2. **基底新鲜度查**(合并必须对着最新 main 的组合态,不是 S3 核实时的旧 main):
   - 锚定 main sha == 当前 main tip(`gh api repos/OWNER/REPO/branches/main --jq .commit.sha`)→ 通过,门禁二过;
   - 不等(main 在 S3 核实后又前进)→ `gh pr update-branch <号>` 把 main 合进分支、触发 CI 在组合态重跑,评论「基底 main 已前进(<锚定>→<当前>),已 update-branch 同步并触发 CI 重跑,下轮凭新 CI 结论判」,**本轮跳过**;下轮合并条件 = CI 绿 + 第 1 查的基底同步例外成立。

### feature/ PR(enhancement 实施产物,自动合并,三门禁缺一不可)

门禁一·CI:同 fix/。
门禁二·独立核实(两条通道,同 fix/,按分支名二选一):
- **issue 通道**(分支名 `feature/<N>-...`):确认有 `status:verified`(S3 已按验收标准逐条核过),无 `status:rework`/`status:doing`/`question`/`needs-info`。
- **PR 通道**(分支名 `feature/` 但不带 `<N>-`,即直管 PR):PR 标签有 `status:verified` 且无 `status:rework`,并过**新鲜度两查**(同上节);合并评论记「直管通道三门禁通过,已自动合并」,无单可关。
门禁三·AI 审查无🔴:`gh pr view <号> --json comments` 找最新一条 AI 审查评论,取其中的 `VERDICT: PASS|FAIL` 结论行(审查评论必带此行;旧格式评论无此行,回退规则 = 评论不含「必须改」类真红线才过——注意「🔴 0」「必须改:无」这类计数/否定语境里的 🔴 不算红线,要读语义,不许裸字符串匹配 emoji)。**⚠️ 信号源纪律:`statusCheckRollup` 里 review check 的 SUCCESS 只代表「审查 workflow 跑完了」,绝不代表审查通过——结论必须读评论里的 VERDICT 行;check SUCCESS + 评论 VERDICT: FAIL 的 PR(实测出现过)是打回件,合并前必须完成摘 verified + 按打回分类处置(缺陷类 rework/裁决类 question,见下段)+ PR 评论的打回动作并汇报,不许只列表跳过。**
**⚠️ verdict-head 绑定:取到最新含 VERDICT 的评论后,先核它不是旧 head 的陈旧结论——评论 `createdAt` 必须晚于当前 head 提交的 committer 时间:`HEAD=$(gh pr view <号> --json headRefOid --jq .headRefOid)` → `gh api repos/OWNER/REPO/commits/$HEAD --jq .commit.committer.date`(**不要用 pulls API 的 `pushed_at`,实测可为 null**);评论正文带 head 标注(如「head `xxx`」)且 ≠ `headRefOid` 也判陈旧(时间戳是权威判据,sha 标注为辅——评论格式不保证带 sha)。force-push 后旧 verdict 仍留在评论流里,不核新鲜度就会把「打回点很可能已修复」的旧 FAIL 当成当前 head 的结论——真实事故(幽灵返工):head 换了、新 head 的审查 run success 但零评论,门禁读到旧 head 的 FAIL 自动打回,白烧一轮 S2 认领。陈旧 verdict 一律按「评论还没出」处理:不合并、不摘 verified、不打 rework,改为触发重审 `gh pr close <号> && sleep 12 && gh pr reopen <号>`(间隔 <4s 会被去抖;触发后必查新 run 已创建);重审触发每 PR 至多一次——评论流里已有本 head 的「⚠️ 审查未产出结论」哨兵标记或本轮已触发过重审,则不再自动重触发,汇报显著标出留人工裁决。**
`VERDICT: PASS` → 过;`VERDICT: FAIL`(**且经上一步核过非陈旧**)→ 不合并,**先读 🔴 内容分两路打回(打回分类;真实教训:裁决类红线走 rework 会挂死——S2 认领也无权替主人选边)**:
- **缺陷类**(🔴 指向代码行为错误/回归/缺测试缺验证——S2 能直接动手修)→ 打 `status:rework`(打回进返工态而非 doing——doing 是"实施中"锁,S2 不认领 doing 单,打回到 doing 会死锁;rework 单 S2 最优先认领返工);
- **裁决类**(🔴 指向与 ADR/既有文档冲突、职责或方向归属之争、需求歧义——改哪头须主人拍板,S2 无权自行选边)→ **不打 rework,改打 `question`**(进 decide-round 队列),评论注明「审查红线属裁决类,须主人拍板后按结论返工」;
- **拿不准按缺陷类**打 rework——S2 认领面有同款分类出口(fix-round「返工点分类出口」)二次兜底;误进 rework 由 S2 转 question,比误进 question 等 decide-round(不常跑)流转快。
标签动作按通道:issue 通道把关联 issue 摘 `status:verified`(若有)打 rework/question;**PR 通道(直管 PR)打回动作**:issues API 摘 `status:verified`(若有)加 `status:rework`(裁决类加 `question`),PR 评论写明审查红线原文 +(缺陷类)「直管 PR 返工不归 S2/S3 认领——待主人经裁决会指定返工人(原会话/新会话)或丢弃」/(裁决类)「审查红线须主人拍板,已进 decide-round 队列」,该状态由裁决会与看门人上报,**不许静默挂起**(真实漏洞:PR 级 rework 曾四个会话全不认领);评论还没出/陈旧 → 跳过等下轮(陈旧的先按上面触发重审)。
三门禁都过 → **仅当它是本轮门禁全齐候选中 createdAt 最早者**才执行合并(其余留待下轮):`gh pr merge <号> --squash --delete-branch`,评论记录「三门禁通过,已自动合并」,并同样**显式关单**(`gh issue close`,理由同上)。

### 其余 PR(chore/docs,仍人工合并)

fix/、feature/ 前缀的 PR(含不带 issue 号的直管 PR)一律走上面的自动门禁,**不再落本节人工队列**;只有 chore/、docs/ 及其他前缀的 PR 才是本节对象,永不自动合。
CI 全绿 → PR 评论「CI 全绿,进入待人工合并」并在汇报中列出;**待人工合并超 2 小时无主人动作的(以 updatedAt 判),汇报中显著标出并注明「已超时,看门人/裁决会应上报主人」**——「等主人」若无上报 = 无限期挂起(真实教训:chore PR CI 全绿后挂 4.5 小时无人提醒)。
CI 红/进行中 → 同上跳过。

### superseded PR 清理(被顶替废件的归属人;真实事故:合集 PR 被分拆件顶替后挂一天无人关——S4 只管合、S3 只核 fixed 单,废件无归属人)

每轮扫描时对分支名带 issue 号的 open PR(`fix/<N>-*`/`feature/<N>-*`)多查一步 `gh issue view <N> --json state`(先 `git -C <主目录> fetch <协作remote> main` 保 main 新鲜):
- issue 仍 open → 正常流水线对象,不动;
- issue 已 **CLOSED** 且 `git -C <主目录> log <协作remote>/main --fixed-strings --grep="(#<N>)" --oneline` 有命中(该单已由别的 PR squash 合并,合并提交信息必带 `(#<N>)`)→ **确认 superseded 废件**:`gh pr close <号> --comment "关联 issue #<N> 已随合并提交 <sha> 进 main,本 PR 为被顶替废件,关闭(若误关请重开)"`,汇报列出;
- issue CLOSED 但 main 无 `(#<N>)` 合并件 → 异常态(人工关单/关错方向),**不关 PR**,汇报显著标出留 decide-round 判。
无号直管 PR 无 issue 可查,不适用本节;长期滞留由看门人上报。

3. AI review 评论对 fix/ 是参考意见、对 feature/ 是门禁三(无🔴)——CI 的门禁只认 ci 的 gate job。

## 兜底

自动合并出坏结果时:squash 合并是单提交,`git revert` 一键回滚后重开关联 issue(摘 verified 打 `status:rework`,S2 最优先认领返工)。上轮合并复核(流程第 0 步)熔断时按该步调走:提 p0 单 + 摘 verified 打 rework(加 `circuit:collateral` 标签 + issue 评论留连坐说明)+ PR 评论留证。

## 收尾

### 同步本地 main(合并者负责)

本轮做过合并、或发现远端 main 领先本地时,把 main 拉回本地主目录(dev server 在此跑,pull 后自动热更):

- 主目录当前在 main 且无未提交改动 → `git -C <主目录> fetch <协作remote> && git -C <主目录> pull <协作remote> main --ff-only`,汇报同步结果(前后 commit 号);
- 主目录**不在 main** → 先做**死分支判定**,四条全满足才自动切回(此前只有「不强切」,导致主目录停在一个已合并的临时分支上 1.5 天,dev server 一直服务 27 个提交前的旧码):
  1. `git -C <主目录> log <协作remote>/main..HEAD` 输出为空(该分支相对 main **零独有提交**,切走不丢任何工作);
  2. `git -C <主目录> status --porcelain | grep -v '^??'` 为空(无 tracked 改动、无暂存内容;未跟踪文件不随切换丢失);
  3. 该分支尖端提交时间距今 **≥ 24 小时**(新的分支可能在用,不动);
  4. 分支名不匹配 `fix/`、`feature/`、`verify/`、`hotfix/` 中任何「有 open PR 或 issue 仍挂着 status:doing/status:fixed/status:rework」的(`gh pr list --state open` 与 `gh issue list --label status:doing --label status:fixed --label status:rework` 交叉核;拿不准就不切,只汇报)。
  满足 → `git -C <主目录> checkout main && git -C <主目录> merge --ff-only <协作remote>/main`,汇报写「主目录原停在 <分支名>(已死,零独有提交),已切回 main 并快进 <旧>→<新>」。
  切完**必做依赖同步检查**:`git -C <主目录> diff --name-only <旧分支尖端> <新 HEAD> -- package.json` 有输出 → package.json 变了,node_modules 可能缺新建的包;**S4 无 npm 权限,不自行安装**,汇报中显著标出「需补装依赖(package.json 有变动)」并 @ 看门人处理——不补装的话 dev server 会在浏览器侧报模块解析失败(真实事故:缺包致面板整链 500)。
  四条任一不满足 → **不强切不强拉**,汇报注明「本地未同步+原因」,留待下轮或人工;
- git 操作仅限上述 fetch/pull/checkout/merge,其余 git 命令一律不执行。

### 汇报

扫描了几个 PR、各状态、本轮自动合并了哪些、本地 main 同步结果。

### 自动续排

本轮结束前**必须**调用 ScheduleWakeup 给自己排下一轮(此前靠外部驱动器,驱动器退役后会话跑完一轮即永久休眠,S4 两次无声死亡的根因):
- 本轮处理后 open PR 队列**非空**(有 CI 进行中/待核实/待人工合并的残留)→ delaySeconds=**900**
- 队列**空** → delaySeconds=**3600**
- prompt 传 `/merge-round`;noop 按当轮实况填
- ScheduleWakeup 不可用时用 CronCreate 一次性任务兜底;汇报中说明排了多久

红线:只操作本仓;自动合并权限定 fix/(双门禁)与 feature/(三门禁)——verified 两通道(issue 号通道,或无 issue 号直管 PR 的 PR 级 verified 通道 + 新鲜度两查);chore/docs PR 永不自动合。
