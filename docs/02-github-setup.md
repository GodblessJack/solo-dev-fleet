# 02 · GitHub 侧配置

## 2.1 仓库

- 私有仓即可,**GitHub 免费计划够用**——整套机制就是按免费计划边界设计的(merge queue、branch protection 这些付费功能全部用会话规则自建了等价物);
- main 只接受 PR 合入(squash)——靠纪律,不靠 branch protection(免费计划私有仓不可用);
- 协作约定:所有 AI 会话共用主人的 gh 登录态(`gh auth login`)。

## 2.2 标签(一键创建)

```bash
REPO=OWNER/REPO  # 改成你的仓

# 类型
gh label create bug           -R $REPO --color D73A4A --description "确认的缺陷"
gh label create enhancement   -R $REPO --color A2EEEF --description "功能/改进(排期由人定)"
gh label create question      -R $REPO --color D876E3 --description "必须主人开口:熔断/歧义/ADR冲突"
# 来源
gh label create from:qa         -R $REPO --color 0E8A16 --description "S1 测试会话提的单"
gh label create from:human      -R $REPO --color 0E8A16 --description "人提的单"
gh label create from:annotation -R $REPO --color 0E8A16 --description "产品内批注通道提的单"
gh label create from:canary     -R $REPO --color 0E8A16 --description "runner 心跳告警单"
# 状态机(同一时刻只挂一个 status:*)
gh label create status:doing    -R $REPO --color FBCA04 --description "S2 实施中"
gh label create status:fixed    -R $REPO --color FBCA04 --description "PR 已提,待 S3 核实"
gh label create status:verified -R $REPO --color 0E8A16 --description "S3 独立核实通过,待 S4 合并"
gh label create status:rework   -R $REPO --color E99695 --description "被打回,S2 最优先返工"
# 就绪与优先级
gh label create claude-ready -R $REPO --color 1D76DB --description "就绪闸门三要素齐备,流水线可认领"
gh label create priority:p0  -R $REPO --color B60205 --description "阻断主流程/数据损坏"
gh label create priority:p1  -R $REPO --color D93F0B --description "功能明显不符但有绕行"
gh label create priority:p2  -R $REPO --color FBCA04 --description "体验/文案/边界"
# 例外
gh label create needs-info          -R $REPO --color FEF2C0 --description "信息不足,等提单人补料"
gh label create duplicate-candidate -R $REPO --color FEF2C0 --description "疑似重复,待确认"
gh label create auto-closed         -R $REPO --color FEF2C0 --description "AI 自动关闭留痕(误关请重开)"
gh label create deferred:external   -R $REPO --color FEF2C0 --description "根因在外部服务/额度,挂起"
gh label create circuit:collateral  -R $REPO --color FBCA04 --description "S4 熔断连坐标记,归 S3 恢复池"
# 模块(按你的项目改,建议 5-8 个)
gh label create mod:frontend -R $REPO --color C5DEF5 --description "前端/UI"
gh label create mod:backend  -R $REPO --color C5DEF5 --description "服务端/API"
gh label create mod:skills   -R $REPO --color C5DEF5 --description "文档/技能/命令"
gh label create mod:infra    -R $REPO --color C5DEF5 --description "构建/CI/部署"
```

**为什么标签是状态机**:GitHub 免费计划没有自定义字段,标签是唯一全员(人 + gh CLI + Actions)可读写的状态载体。代价是「标签无失效语义」——verified 不随新提交自动作废,所以配套了锚定行 + 新鲜度两查(见 docs/06)。

## 2.3 Secrets(Actions 用)

仓库 Settings → Secrets and variables → Actions:

| Secret | 用途 |
|---|---|
| `ANTHROPIC_BASE_URL` | Anthropic 兼容端点(官方或第三方) |
| `ANTHROPIC_API_KEY` | 端点 key |
| `ANTHROPIC_MODEL` | 主模型名(API 裸名) |
| `ANTHROPIC_DEFAULT_HAIKU_MODEL` / `..._SONNET_MODEL` / `..._OPUS_MODEL` | 三档模型名 |

**模型名收进 Secrets 而不是写死在 workflow 里**:切模型只改 secret,不动代码、不留提交记录。

## 2.4 装 workflow

把本仓 `workflows/` 下 6 个文件拷进你仓的 `.github/workflows/`:

| 文件 | 触发 | 作用 |
|---|---|---|
| `ci.yml` | push main + 所有 PR | 唯一硬门禁(lint+test+build)——按你的技术栈改 run 命令 |
| `triage.yml` | issue 新建/评论/手动 | AI 分诊台 |
| `claude-review.yml` | PR opened/synchronize/reopened | 五轴 AI 审查 + VERDICT 哨兵 |
| `claude.yml` | 评论含 @claude | 云端遥控实现 |
| `canary.yml` | 每 30 分钟 | runner 心跳(3 并发 slot) |
| `model-health.yml` | 手动 | 模型端点 L0/L1/L2 探针 |

拷完先跑一次 `model-health.yml`(Actions 页手动触发),L0→L2 全绿再继续。

## 2.5 装模板与命令

```bash
# 在你的项目仓根目录
cp /path/to/solo-dev-fleet/templates/CLAUDE.md ./CLAUDE.md                 # 按占位符改
cp /path/to/solo-dev-fleet/templates/pull_request_template.md ./.github/pull_request_template.md
mkdir -p .claude/commands
cp /path/to/solo-dev-fleet/claude-commands/*.md ./.claude/commands/
```

命令文件里的 `OWNER/REPO` 全局替换成你的仓(`grep -rl 'OWNER/REPO' .claude/commands | xargs sed -i 's|OWNER/REPO|你的仓|g'`)。

## 2.6 批注通道(可选)

产品内嵌「添加批注」按钮 → 截图+上下文 POST 到本地 dev server → 后端用本机 gh 建 issue(`from:annotation + priority:p1`),截图存仓内目录。适合「边用产品边捉虫」的单人节奏:不用离开产品就能提单,单自动进分诊流水线。实现要点:通道代码必须全实例通用(任何 dev server 实例都能收),截图落盘路径进 .gitignore 例外。
