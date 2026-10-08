# 08 · 日常运维手册

## 8.1 起舰队(每天/每次开机后)

在项目仓根目录各起一个 Claude Code 会话(推荐各用一个终端窗口/tmux 窗格,或 bg 会话):

```
# 会话 1(S1 测试)
/loop 20m /qa-round
# 会话 2(S2 实施,在 worktree 里跑)
/loop 15m /fix-round
# 会话 3(S3 核实,在另一个 worktree)
/loop 15m /verify-round
# 会话 4(S4 合并员)
/loop 15m /merge-round
# 会话 5(看门人)
/loop 15m /monitor-round
```

S5 裁决会不挂 loop,需要时手动:`/decide-round`(人在场才跑)。

预建 worktree(S2/S3 用,与主目录隔离):

```bash
git worktree add .worktrees/fix --detach
git worktree add .worktrees/verify --detach
```

## 8.2 各环间隔(动态调度口径)

每环收尾时自己排下一轮(ScheduleWakeup),口径:

| 环 | 有活 | 空第 1 次 | 连续空 |
|---|---|---|---|
| S2 fix | 10 分钟 | 15 分钟 | 30 分钟 |
| S3 verify | 10 分钟 | 15 分钟 | 30 分钟 |
| S4 merge | 15 分钟(队列非空)/60 分钟(空) | — | — |
| 看门人 | 固定 15 分钟 | — | — |

ScheduleWakeup 不可用时用 CronCreate 一次性任务兜底。

## 8.3 健康检查(每天一眼)

1. **看门人状态文件**:`ls -l ~/.solo-dev-fleet/monitor-watchdog.json`——mtime 超 45 分钟 = 看门人死了,重启它的会话;
2. **runner**:`systemctl --user status actions-runner-*` + Actions 页看 canary 是否连红;
3. **四池**:
   ```bash
   gh issue list --label claude-ready          # S2 池
   gh issue list --label status:fixed          # S3 池(issue 面)
   gh pr list --label status:fixed             # S3 池(直管 PR 面)
   gh pr list --label status:verified          # S4 池
   gh issue list --label question              # S5 池(该开裁决会了)
   ```
4. **「等主人」超时**:chore/docs PR 待人工超 2h、needs-info 超期、deferred:external 积压——该开裁决会;
5. **main push CI**:最近一次合并的 push CI 必须绿。

## 8.4 常见救火

| 症状 | 处置 |
|---|---|
| 某环会话死了 | 看门人会开 `[看门人]` 上报单;重起该会话 `/loop <间隔> /<命令>` 即可,队列状态全在 GitHub 标签里,会话无状态 |
| PR 幽灵返工(读旧 verdict) | S4 已有 verdict-head 绑定;手动重审 = close/reopen(间隔 ≥12s,防去抖) |
| main push CI 红 | S4 已熔断处置;照 p0 单修复,修复件合并后连坐单由 S3 自动恢复 |
| triage 连挂 | 跑 model-health.yml 分层定位(L0 secrets / L1 curl / L2 CLI);修好后 `gh workflow run triage.yml -f issue_number=N` 补票 |
| 会话越权/走样 | 直接关掉该会话(开火权);把案例写成命令文件的新注释,防止复发 |
| 上下文将满 | 命令文件是一轮一加载的,天然免疫;主会话用交接文档(handoff)开新窗口 |

## 8.5 机制改进怎么落地

机制层改动(workflow/命令/CLAUDE.md/ADR)= `chore/` PR,永不自动合并,主人把关。流程:

1. 任何会话发现机制漏洞 → 报告主人/开单,**不顺手膨胀**(改代码的会话只做自己那单);
2. 机制设计在监测/主会话讨论成形,修订写进 ADR;
3. 起 `chore/<简述>` 分支改文档,提 PR(正文说清动机),主人审完手动合;
4. 合并后下轮各环自动加载新剧本——机制即改即生效。

## 8.6 成本口径

- GitHub Actions:自托管 runner 全免费;
- LLM token:六环常驻空轮也烧 token——闲时间隔拉长(30 分钟)就是为这个;分诊/审查按事件触发,不空转;
- 人的时间:健康检查每天 2 分钟,裁决会按积压触发(通常几天一次)。
