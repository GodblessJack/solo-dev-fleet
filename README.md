# solo-dev-fleet

> 一个人 + 一支 AI 会话舰队 + GitHub:把「提单 → 分诊 → 修复 → 独立核实 → 审查 → 合并」整条软件生产线交给 6 个 Claude Code 会话角色和 6 个 GitHub Actions workflow 自动运转,人只保留三样东西:**裁决权、chore 合并权、开火权**。
>
> *One human commanding a fleet of AI sessions: a complete, battle-tested GitHub-native pipeline where AI agents triage issues, implement fixes, independently verify them, review PRs and auto-merge — while the human keeps only three powers: deciding, merging chores, and pulling the plug.*

这套机制在一个真实私有项目上全速运转,累计数百张 issue、上百个 PR 由流水线自动闭环。本仓是它的**完整可复制形态**:所有角色命令、workflow、模板、部署指南、架构决策记录,以及每一条机制背后的血泪教训(全部写在注释和文档里)。拿到手,一个下午就能让自己的仓库跑起来。

## 它解决什么问题

一个人用 AI 写代码,真正的瓶颈不是「写」,而是:

- AI 干的活**没人复核**——同一个会话既实现又自证,坏代码直接进 main;
- issue 越攒越多,**没人分诊**、没人查重、没人排期;
- PR 挂着**没人合并**,或者更糟——合并了没人验证组合态,main 悄悄断链;
- 自动化会话**悄悄死掉**,几天没人发现;
- 每个「等主人看看」的状态**没有上报通道**,等于无限期挂起。

solo-dev-fleet 把每一环都装上归属人和上报通道:**每个状态都必须有归属人,「等主人」也必须带上报,否则等于无人管。**

## 舰队编成(5+1 个会话角色)

| 角色 | 命令 | 职责 | 一句话 |
|---|---|---|---|
| S1 测试 | `/qa-round` | 周期性走查产品,发现缺陷提 bug 单 | 找问题的 |
| S2 实施 | `/fix-round` | 认领就绪单,TDD 修复,提 PR | 干活的 |
| S3 核实 | `/verify-round` | **独立**复现验证 S2 的 PR,通过打 verified | 挑刺的(干活的不能是打分的) |
| S4 合并员 | `/merge-round` | 多门禁(CI + verified + AI 审查)通过即自动 squash 合并 | 把关的 |
| S5 裁决会 | `/decide-round` | 必须由人开口的例外,逐条呈现给主人裁决 | 人的入口 |
| 看门人 | `/monitor-round` | 三环存活心跳 + 「等主人」超时上报 + 自报心跳 | 盯着所有人的 |

前四环由 `/loop` 驱动常驻自调度(动态间隔:忙 10 分钟、闲 15-30 分钟);S5 手动触发,人在场才跑;看门人 15 分钟一巡。

## 云端层(6 个 GitHub Actions)

| workflow | 作用 |
|---|---|
| `ci.yml` | **唯一硬门禁**:lint + test + build,main 和所有 PR 必过 |
| `triage.yml` | AI 分诊台:新 issue 自动核实、三层查重、补全、打标签 |
| `claude-review.yml` | PR 五轴 AI 审查,产出机器可读 `VERDICT: PASS/FAIL`(feature/ 合并门禁之一) |
| `claude.yml` | `@claude` 遥控:任何 issue/PR 评论 @claude 触发云端实现 |
| `canary.yml` | runner 心跳:每 30 分钟 3 并发金丝雀,失败自动开 issue |
| `model-health.yml` | 模型层探针:换端点/key 后一键验证 L0→L1→L2 |

## 一张图看懂流转

```
人/测试会话提 issue ──→ AI 分诊台(triage.yml)──→ claude-ready
                                                        │
看门人巡逻 ↑                                   S2 /fix-round 认领(TDD,提 PR)
   │                                                        │
   │                                          ci.yml 硬门禁 + claude-review 五轴审查
   │                                                        │
   │                                   S3 /verify-round 独立核实(复现,不自述)
   │                                                 │ 通过            │ 打回
   │                                          status:verified      status:rework
   │                                                 │           (S2 最优先返工;
   │                                          S4 /merge-round    裁决类红线转 question)
   │                                          门禁全齐自动合并         │
   │                                                 │         3 次熔断 → question
   │                                       main push CI 复核          │
   │                                       红 → 熔断+连坐处置    S5 /decide-round
   └──────────────── 异常/超时/「等主人」全部上报 ────────────→ 主人裁决
```

直管大功能(不建 issue 的 worktree 独立开发)有专用通道:PR 正文「验收标准」段 + PR 级标签 + 新鲜度两查,同样全自动,见 [docs/06-direct-pr-channel.md](docs/06-direct-pr-channel.md)。

## 快速开始

1. **建仓**:你的项目仓(私有即可,GitHub 免费计划够用——本套机制就是按免费计划边界设计的);
2. **建标签**:照 [docs/02-github-setup.md](docs/02-github-setup.md) 的标签词汇表一键创建;
3. **装模板**:把 `templates/` 下的 `CLAUDE.md`、`pull_request_template.md` 拷进你的仓并按占位符改;
4. **装 workflow**:把 `workflows/` 拷进 `.github/workflows/`,配置 Secrets;
5. **装 runner**:照 [docs/03-self-hosted-runner.md](docs/03-self-hosted-runner.md) 装 1-3 台自托管 runner(私有仓用 GitHub 托管 runner 会烧分钟数,自托管免费);
6. **放舰队**:照 [docs/08-operations.md](docs/08-operations.md) 起 6 个 Claude Code 会话,各挂 `/loop`。

## 设计原则(每条都是踩坑换来的)

1. **实现者 ≠ 验证者**:GitHub 平台禁止 PR 作者自批,Anthropic 官方文档原话 "the agent doing the work isn't the one grading it"——所以 S2 和 S3 必须是两个会话。
2. **每个状态都要有归属人**:「等主人」也必须带上报通道和超时,否则等于无人管。
3. **组合态才算数**:PR 的绿是「分支 + 当次 base」的绿;squash 合并后必须核 main push CI,红即熔断。
4. **自动化的是发现与上报,不是替人拍板**:裁决权永远在主人手里,裁决会只是把所有「需要你一句话」的事集中到一处。
5. **机制与产物同仓**:每条机制文档与其载体同 PR 合入,文档即代码。

## 文档导航

| 文档 | 内容 |
|---|---|
| [docs/00-overview.md](docs/00-overview.md) | 概念全景:角色、状态机、设计律 |
| [docs/01-concepts-and-roles.md](docs/01-concepts-and-roles.md) | 六个角色逐个讲:谁干什么、互不越界 |
| [docs/02-github-setup.md](docs/02-github-setup.md) | 建仓、标签、Secrets、workflow 安装 |
| [docs/03-self-hosted-runner.md](docs/03-self-hosted-runner.md) | 自托管 runner:安装、systemd 保活、多机并发坑 |
| [docs/04-pipeline-lifecycle.md](docs/04-pipeline-lifecycle.md) | 一张单的完整一生 + 所有异常通道 |
| [docs/05-claude-md-template.md](docs/05-claude-md-template.md) | CLAUDE.md 项目宪法怎么写 |
| [docs/06-direct-pr-channel.md](docs/06-direct-pr-channel.md) | 直管 PR 通道(不建 issue 的大功能) |
| [docs/07-labels.md](docs/07-labels.md) | 标签词汇表全文 |
| [docs/08-operations.md](docs/08-operations.md) | 日常运维:起舰队、看状态、救火 |
| [docs/09-faq.md](docs/09-faq.md) | 踩坑实录:GraphQL 弃用、去抖、幽灵 verdict… |
| [adr/0001](adr/0001-ai-collab-pipeline.md) | 流水线总架构决策(含全部修订史) |
| [adr/0002](adr/0002-direct-pr-verification-channel.md) | 直管 PR 核实合并通道决策 |

## License

MIT(见 [LICENSE](LICENSE))。命令文件与 workflow 中的经验教训注释是这套机制的灵魂,转载请保留出处。
