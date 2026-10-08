# 04 · 一张单的完整一生 + 所有异常通道

## 4.1 正常路径(bug,7 步全自动)

```
① 提单    S1 走查发现缺陷(或人/批注通道提单)
           ↓ S1 单:from:qa + claude-ready 双标齐,免分诊
           ↓ 人提的单:triage.yml AI 分诊(核实→查重→补全→打 bug+ready+priority+mod)
② 就绪    claude-ready 含义 = 验收标准可验证 + 定位有出处 + 依赖明确
③ 认领    S2 /fix-round:rework 单最优先,其次 bug(按 priority),再 enhancement
           打 status:doing,worktree 切分支 fix/<N>-<简述>
④ 实施    TDD(先写失败校验脚本→改→绿);lint+test+build 三绿才许提 PR
⑤ 交付    提 PR(Closes #N),摘 doing 打 status:fixed
           云端自动:ci.yml 硬门禁 + claude-review 五轴审查(VERDICT 行)
⑥ 核实    S3 /verify-round:独立 worktree 复现(不信自述)
           bug 重复现路径;enhancement 按验收标准逐条核;2.5 风险评估必做
           通过 → status:verified + 锚定行(head/基底 sha)
⑦ 合并    S4 /merge-round:fix/ 双门禁(CI 绿+verified)、feature/ 三门禁(+VERDICT: PASS)
           齐 → squash 自动合并 + 显式关单;下轮复核 main push CI
```

## 4.2 异常通道一览(每条都有归属人)

| 异常 | 触发 | 处置 | 归属人 |
|---|---|---|---|
| 打回返工 | S3 核实不过 / S4 审查 🔴(缺陷类) | 摘 verified 打 `status:rework` | S2 最优先认领,既有分支续做 force-push |
| 裁决类红线 | 审查 🔴 指向 ADR 冲突/职责归属/需求歧义 | 打 `question`,S2 不动手(禁自行选边) | S5 裁决会 |
| 三次熔断 | 同一单被打回满 3 次 | 打 `question`,不再自动处理 | S5 裁决会 |
| 组合态断链 | main push CI 红(合并复核发现) | 熔断:提 p0 单、本轮不再合;同批已合 PR 的 issue 打 rework+`circuit:collateral`+评论连坐说明 | S4 处置;修复件合并+main 回绿后 **S3 连坐恢复池**自动恢复 verified |
| 幽灵 verdict | force-push 后门禁读到旧 head 的 FAIL | verdict-head 绑定:评论时间必须晚于 head committer 时间;陈旧 verdict 触发重审(close/reopen,间隔 ≥12s 防去抖,每 PR 至多一次) | S4 |
| 审查零评论 | review run success 但没发 VERDICT | Verdict 哨兵:run 结束核对本 run 期间有结论评论,无则补发标记并判红 | claude-review.yml 自带 |
| CI 未触发 | PR 与 main 冲突(CONFLICTING) | GitHub 对冲突 PR 不跑任何 workflow;评论提示 + 归属:issue 单归 S2,直管归原会话,超 2h 无人 → rebase 兜底 | S4 上报,看门人兜底 |
| superseded 废件 | 合集 PR 被分拆件顶替 | open PR 关联 issue 已关且 main 有 `(#N)` 合并件 → 关 PR 留痕 | S4 每轮清理 |
| 待人工超时 | chore/docs PR CI 绿但 2h 无人合 | S4 显著标出 + 看门人上报 + S5 入裁决队列 | 主人 |
| 会话死亡 | 某环会话进程没了 | 三环心跳:缺失即上报(建 issue) | 看门人 |
| 看门人死亡 | 状态文件 mtime 超 3 个周期 | 主人肉眼核查,重启会话 | 主人(唯一人肉兜底) |
| 分诊失败 | triage run 红 | 自动开告警 issue(不依赖人看红叉) | workflow 自带 |
| 重复单 | 锚点与 open 单一致 | 高置信自动关(附证据链+「误关请重开」);低置信疑似不关闭待人确认 | triage |
| 已修复的过期反馈 | 行为已不存在 + 找到修复件 | 双客观条件齐备才自动关(打 auto-closed),否则降级疑似 | triage |

## 4.3 三道闸门(idea → issue → 实现)

- **设计闸门**:新想法先在主会话拷问(grill)成形,结论沉淀进 ADR/术语表后才允许拆 issue;issue 必须引用设计依据;
- **就绪闸门**:同时满足「验收标准可验证 + 设计依据有出处 + 依赖明确」才打 `claude-ready`;流水线只认领带此标的;
- **上下文闸门**:拷问→拆单在主会话同一窗口完成;实现一律起新会话(worktree 隔离);主会话接近上下文上限就交接(handoff),不硬撑。

## 4.4 提单规格(谁提单打什么标)

- **bug(S1 提)**:有复现路径+锚点自查过 → 提单时直接打齐 `from:qa + claude-ready + bug + priority:pN + mod:*`,免分诊,S2 直接认领。漏打 ready 不会悬空——triage 对「from:qa 缺 ready」的单照样兜底分诊;
- **bug(人/批注提)**:只写现象即可,triage 自动核实、追问、补全、打标;
- **enhancement(一律人提)**:打 `enhancement + mod:*`,**不打 claude-ready**——排期权在人,裁决会点头补 ready 才进流水线。改进建议不许 AI 会话自己塞进实施队列;
- **什么情况下不建 issue、直接提 PR**:机制层改动(workflow/命令/CLAUDE.md/ADR/测试基建)走 `chore/` PR;主人直管大功能走直管 PR 通道(见 docs/06)。产品代码**必须**走完整 issue 门禁。

## 4.5 分支命名即路由

S4 靠分支前缀决定门禁等级,**命名就是路由**:

| 分支 | 门禁 | 合并方式 |
|---|---|---|
| `fix/<N>-<简述>` | 双门禁(CI+verified) | 自动 |
| `feature/<N>-<简述>` | 三门禁(+VERDICT: PASS) | 自动 |
| `fix/<简述>`、`feature/<简述>`(不带号) | 直管 PR 通道:PR 级 verified + 新鲜度两查(feature 另加 VERDICT) | 自动 |
| `chore/<简述>`、`docs/<简述>` 及其他 | CI 绿后待人工,超 2h 上报 | 永不自动 |
