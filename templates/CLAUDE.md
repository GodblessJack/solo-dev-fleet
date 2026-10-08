# CLAUDE.md — <项目名>(单人 AI 舰队宪法模板)

> 用法:把本文件拷到你仓根目录,替换所有 `<...>` 占位符,按项目删改硬性约束。
> 设计意图逐段讲解见 docs/05-claude-md-template.md。

<一句话项目简介:是什么、技术栈、运行形态>

# 沟通原则(最高优先级)

## 对用户汇报与提问的规则

1. **禁止高度概括后隐藏细节的表述**。不许抛出几个抽象关键字就让用户做选择——用户听不懂,也无法分辨。
2. **提问和选项必须基于深度调查之后、还原事实**:讲清现有流程是怎么一步步走的、改完之后流程变成什么样、结果是什么、来龙去脉是什么。
3. **多方案对比必须逐方案展开**:具体要改哪些文件/哪些环节、用户的实际体验会发生什么变化、有什么风险和副作用、与其他方案的区别和相互影响——让用户不依赖 AI 也能独立判断。
4. 技术术语和代码标识符保留原文,但周边解释必须完整。
5. **完工汇报必须直观、量化**:做完工作后直接说做了什么——改了哪几个文件、建了哪几个 issue(带编号链接)、推了哪个 commit(带 hash);紧接着列接下来要做什么、谁来做。禁止行话包装,禁止与任务无关的描述和感想。

这条原则优先级高于省篇幅、高于简洁偏好。

---

## 必读文档(动手前先读)

- `CONTEXT.md` — 领域术语表(如有)。界面/代码/文档文案必须遵守术语口径
- `docs/adr/` — 架构决策记录;改动与 ADR 冲突时必须先提出讨论,不许先斩后奏

## 工程命令

- `<构建命令>`(**提交前必须通过**)
- `<测试命令>`(**提交前必须通过**)
- `<lint 命令>`

## 硬性约束

- **红线:永不 pull/push/fetch 上游 fork 源 `<上游remote>`(如适用);协作 remote 是 `<协作remote>`**
- **优化无损律:任何优化都不能牺牲功能和性能。**凡「省」类改动(上下文折叠/压缩/预算收缩、缓存截断、降采样、字段裁剪)必须附**功能无损证明**——受影响消费方清单 + 开/关对照证据(缺证明 S3 不放行);优化与功能冲突时功能优先,宁可放弃优化。**落盘/存档链路禁止不可逆有损变换**:有损只允许发生在发给模型的临时视图,存档保原文或可还原引用
- 交付物零「模拟/原型/待定/占位」字样,按最终质量标准做
- 界面与文档文案遵守项目术语口径
- 凭证只进 `.env.local` / 项目 keystore,绝不入库、不进 issue/PR 正文
- **无由不烧资源**:昂贵的操作(渲染/导出/真调付费 API)默认不执行;一旦执行,必须对产物做完对应验证并报结果

## Git 与 GitHub 工作流

- 账本仓:`<OWNER/REPO>`(私有);GitHub 操作一律 `gh` CLI
- `main` 只接受 PR 合入(squash);修复走 `fix/<issue号>-<简述>`,功能走 `feature/<issue号>-<简述>`,基建/文档走 `chore/<简述>`(可无 issue 号但 PR 正文说清动机)。**直管功能 PR(主人直派 worktree 会话做的大功能,不建 issue)**:分支 `feature/<简述>`(不带号),PR 正文必带「## 验收标准」段(结构见 `.github/pull_request_template.md`,每条 = 步骤 + 期望结果,写到不熟悉的人能照着复现),自测完打 PR 级 `status:fixed` 交付 S3 核实——详见 docs/06(直管 PR 通道)
- **机制层例外**:workflow/Secrets/`.claude/commands`/CLAUDE.md/ADR/测试基建等机制设计类改动,AI 可不建 issue 直接改;**但落地一律走 PR**——机制类开 `chore/` PR(可无 issue 号,PR 正文说清动机)。产品代码仍走完整 PR 门禁
- 提交信息:中文,格式 `<type>: <说明> (#<issue号>)`,type ∈ feat/fix/chore/docs/refactor/test
- 分支/合并操作本地进行;push 只推 `<协作remote>`,每次 push 前确认目标 remote 名

### 三道闸门(idea → issue → 实现)

- **设计闸门**:新想法先在主会话拷问成形,结论沉淀到 `CONTEXT.md`/`docs/adr/` 后才允许拆 issue;issue 必须有「设计依据」引用
- **就绪闸门**:issue 同时满足「验收标准可验证 + 设计依据有出处 + 依赖明确」才打 `claude-ready`;流水线会话只认领带此标的
- **上下文闸门**:拷问→拆 issue 在主会话同一窗口完成;实现一律起新会话(worktree 隔离);主会话接近上下文上限时用交接文档(handoff)交接,不硬撑

### 流水线会话(单轮命令 + /loop 驱动)

- S1 测试 `/qa-round`(主目录,端口 <XXXX>):双轨走查——工具/API 层 e2e(mock 档高频、真调档按预算)+ UI 浏览器走查;提单前必须用结构化锚点(mod 标签+代码位置/工具名+复现路径)查 open issues,已存在则评论补证据不新建
- S2 实施 `/fix-round`(worktree `.worktrees/fix`):认领 `bug` 或 `enhancement` + `claude-ready` 且无 `status:doing` 的单(**`status:rework` 返工单最优先**,bug 次之;issue 同一时刻只挂一个 status 标签——提 PR 摘 doing 打 fixed);代码类 TDD(先写失败校验),文档/技能类以验收标准逐条对照;build+test+lint 全过才提 PR;根因聚合单须过完全部关联复现路径;验收标准不具体的 enhancement 退回待排期不许自由发挥
- S3 核实 `/verify-round`(worktree `.worktrees/verify`,端口 <XXXX>):独立复现验证(bug 重复现路径,enhancement 按验收标准逐条核;**无 issue 号的直管 PR 走 PR 通道:PR 正文「## 验收标准」段为核对清单,通过打 PR 级 verified 并附 head/基底锚定行**);通过打 `status:verified`,不通过退回 `status:rework`(返工态,S2 最优先认领;直管 PR 的 rework 归原直管会话/主人,不归 S2);打回 3 次熔断打 `question`
- S4 合并员 `/merge-round`(纯 gh):`fix/` PR 双门禁(CI 绿 + `status:verified`)、`feature/` PR 三门禁(加 AI 审查无🔴)通过即自动 squash 合并;verified 两通道——issue 号通道,或无 issue 号直管 PR 的 **PR 级 verified + 新鲜度两查**(head 锚定查防陈旧核实、基底新鲜度查防组合态断链);chore/docs PR 待人工合并。**每次合并后须核本批 push 触发的 main CI 结论:红即视为 p0 阻断,立即提单回报并优先修复**——PR 的绿是「分支 + 当次 base」的绿,base 前进后 GitHub 不会重算;squash 后的组合态只有 main push CI 能判
- S5 裁决会 `/decide-round`(手动触发,不挂 /loop):拉全量 open issues 筛出必须主人裁决的例外(question 熔断/deferred:external/needs-info/疑似重复/enhancement 排期/PR 级 rework/chore 待人工超时),逐条呈现→大白话裁决→代执行关单/摘标签/排期;按规则等待不算异常,机器侧停滞归看门人
- 看门人 `/monitor-round`(常驻 /loop):三环(S2/S3/S4)存活心跳 + chore/docs PR 待人工超时上报 + 自报心跳状态文件;只观察与上报,不催单不代合不代裁决

### 查重三层防线(全入口通用)

提单时锚点自查 → 分诊时双轨查重(锚点一致=高置信/现象疑似=低置信)→ 同根因不同验证路径走「根因聚合」不关闭。高置信重复允许 AI 自动合并关闭(附证据链+「误关请重开」),低置信疑似一律待人确认。

## 批注通道(本地,可选)

产品内「添加批注」→ 画框/截图/上下文捕获 → POST 到本地 dev server → 本机 `gh` 建 issue(`from:annotation`+`priority:p1`),截图落仓内指定目录(进 .gitignore 例外)。
