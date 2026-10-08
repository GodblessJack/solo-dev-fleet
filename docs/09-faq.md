# 09 · 踩坑实录(FAQ)

每条都是真实事故换来的,按主题分组。命令文件和 workflow 注释里有更多现场记录。

## GitHub API / gh CLI

**`gh pr edit --add-label/--remove-label` 会莫名失败退出**
gh 的 PR 编辑走 GraphQL,会对 projects(classic) 弃用字段报错(gh ≥2.60),整条命令链断裂——标签没打上,后续 && 链全没执行。**凡是给 PR 改标签,一律走 issues API**:
```bash
gh api repos/OWNER/REPO/issues/<PR号>/labels -f 'labels[]=status:verified'          # 加
gh api -X DELETE repos/OWNER/REPO/issues/<PR号>/labels/status:fixed                  # 删
```
(PR 就是 issue,labels 端点通用。)改完必查终态再走的 `&&` 链。

**close/reopen 重触发 CI/审查有两个坑**
①冲突(CONFLICTING)的 PR 永不触发任何 workflow,close/reopen 也救不了——只能 rebase;②两次操作间隔 ≤4 秒会被 GitHub 去抖,`gh pr close N && sleep 12 && gh pr reopen N`;触发后必查新 run 真的创建了。

**API 响应里 `pushed_at` 可能为 null**
判 verdict 新鲜度要用 commit 的 committer.date(`gh api repos/OWNER/REPO/commits/<sha> --jq .commit.committer.date`),别用 pulls API 的 pushed_at。

## CI / 合并

**「两个各自 CI 绿的 PR 合体后 main 红了」**
PR 的绿是「分支+当次 base」的绿,base 前进后 GitHub 不重算。所以 S4 每次合并后必核 main push CI,红即熔断。同理,S3 核实交互风险单(异步化/共享状态/多文件重构)必须先 merge 最新 main 再 build。

**squash 合并后本地分支永远显示「未合并」**
正常——squash 产生新提交,原分支提交不在 main 历史上。别因此怀疑合并失败。

**tsc 增量构建会「假绿」**
`tail` 看输出文案会被增量缓存骗过;判 build 只看退出码,且定期清增量(`tsc -b --clean` 或删 tsbuildinfo)做一次全量验证。另外 tsc 错误带 ANSI 色码,`grep " error TS"` 数行数不可靠。

## Claude Code / workflow

**claude-code-action 安装器被区域拦截**
action 自带 installer 从 claude.ai 下载,部分出口 IP 拿到的是「App unavailable in region」HTML——改走 npm 预装 + `path_to_claude_code_executable` 指定路径,并用 flock 防多 runner 并发装同一 bin 撞 ETXTBSY。

**run success ≠ 有结论**
审查 run 报 success(is_error:false)但可能零评论(轮数截断/发评论失败)。claude-review.yml 的「Verdict 哨兵」在 run 结束核对本 run 期间有 VERDICT 评论,无则判红——run 状态与「无结论」保持一致,门禁才不会沿用旧结论。

**轮数上限会把干完的活判失败**
分诊/审查「先出结论再深挖」:结论和标签先落地发评论,再按需深挖;为穷尽细节拖到最后,撞 max-turns 上限整轮作废。

**GitHub schedule 会翘班**
cron 触发实测会缺席/迟到 30 分钟以上。心跳类调度用高频(30 分钟一场)换覆盖;要求更高的放本机 systemd timer。

## 流水线机制

**「清单为空」可能是标签断点,不是真没活**
某环报「没有可认领的单」而它确实在诚实汇报——先查上游标签有没有流转到位(提了 PR 没打 fixed?核实过没打 verified?),再查是不是文档口径矛盾(曾有「认领条件把 rework 列为排除项」与「rework 最优先认领」自相矛盾,字面执行 = 返工单永远无人认领)。

**会话共用主人 gh 登录态 = 没有「机器人可自我授权」的通道**
任何「授权标签」都无法区分人打的还是机器人自己打的。授权只发生在当面(裁决会/当面一句话),机器人只做发现与上报。

**每个状态都要写归属人,包括「等主人」**
「等主人看看」没有上报通道和超时 = 无限期挂起。chore 待人工 2h、PR 级 rework、冲突 PR——每类都要么有会话认领,要么有看门人上报,要么进裁决会队列。
