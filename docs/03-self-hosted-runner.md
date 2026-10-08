# 03 · 自托管 Runner

## 为什么必须自托管

私有仓用 GitHub 托管 runner 有每月分钟数配额(免费计划 2000 分钟),AI 分诊/审查动辄十几分钟一场,几天就烧穿。自托管 runner 跑在 own 机器上,**分钟数免费、不限量**,还能直接复用本机已装好的工具链(node、Claude CLI、ffmpeg 之类)。

**安全边界**:自托管 runner 只用于**私有仓 + 不接受外部 PR** 的场景。fork PR 能在你的机器上执行任意代码——公开仓千万别这么干。

## 3.1 安装(Linux,systemd 保活)

```bash
# 一台机器可以装多个 runner 实例(目录分开),建议 2-3 个并发
mkdir -p ~/actions-runner && cd ~/actions-runner
# 到 GitHub 仓 Settings → Actions → Runners → New self-hosted runner 拿最新包与 token
curl -o runner.tar.gz -L https://github.com/actions/runner/releases/download/v2.328.0/actions-runner-linux-x64-2.328.0.tar.gz
tar xzf runner.tar.gz
./config.sh --url https://github.com/OWNER/REPO --token <注册token> --name <机器名-1> --unattended

# systemd 用户级保活(关键:runner 自带 runsvc.sh 是手工脚本,不是包自带服务)
mkdir -p ~/.config/systemd/user
cat > ~/.config/systemd/user/actions-runner-1.service <<'EOF'
[Unit]
Description=GitHub Actions Runner 1
After=network-online.target

[Service]
Type=simple
WorkingDirectory=%h/actions-runner
ExecStart=%h/actions-runner/runsvc.sh
Restart=always
RestartSec=30

[Install]
WantedBy=default.target
EOF
systemctl --user daemon-reload
systemctl --user enable --now actions-runner-1
loginctl enable-linger $USER   # 关 ssh 后用户级服务照样活
```

多实例:复制目录为 `~/actions-runner-2`、`~/actions-runner-3`,各自 config(不同 --name),各写一个 service。**每台 runner 默认单并发**,N 个实例 = N 并发。

## 3.2 多机/多实例共享 HOME 的坑(实测)

- **npm 全局安装会撞 ETXTBSY**:多个 runner 同时 `npm install -g` 同一个 CLI,正在执行的 bin 被覆盖 → 文本忙。解法:安装命令包 `flock`(见 workflow 里的 `flock /tmp/claude-npm-install.lock -c '...'`),且装前先检测版本已在位就跳过;
- **systemd 用户服务拿不到新加的用户组**:把用户加进 docker 组后,已运行的 user manager 还是旧组快照——需要 `systemctl --user daemon-reload` + 重启会话,或干脆重启机器;
- **缓存/锁目录**:`/tmp` 下的锁文件全实例共享,命名带上用途前缀。

## 3.3 心跳:canary.yml(不信任 online 状态)

runner 在 GitHub 上显示 online 只说明进程活着,不说明「能干活」和「挂了有人拉起」。`canary.yml` 每 30 分钟发 3 个并发 job:

- 每台 runner 单并发 → 3 个 job 必然分布到 3 台实例,哪台挂了它的心跳就断;
- 失败自动开 issue(`from:canary + priority:p1`)——**红了没人看的信号等于没有信号**(曾有分诊 workflow 连挂 4 天无人发现,因为红叉躺在 Actions 页没人看);
- GitHub 的 schedule 实测会缺席/迟到 30 分钟以上,所以用 30 分钟一场的高频换取「最坏空窗 1 小时」;要求更高的场景,把心跳调度放到本机 systemd timer。

## 3.4 Claude Code CLI 的安装姿势

`anthropics/claude-code-action` 默认从 claude.ai 下载 CLI 安装器——某些出口 IP 会被区域拦截页挡住(curl 拿回 HTML,bash 解析炸)。成熟做法:

```bash
CLAUDE_BIN=<你的 node bin 目录>/claude
if [ -x "$CLAUDE_BIN" ] && "$CLAUDE_BIN" --version 2>/dev/null | grep -q "<版本>"; then
  echo "已在位,跳过安装(防多 runner 并发装同一 bin 撞 ETXTBSY)"
else
  flock /tmp/claude-npm-install.lock -c 'npm install -g @anthropic-ai/claude-code@<版本>'
fi
```

然后在 action 输入里指定 `path_to_claude_code_executable`,action 检测到即跳过自带下载。版本与 action 内置期望对齐,升级时全 workflow 一起升。

另两个实测坑:
- action 默认的 GitHub App token 交换在自托管 runner 上可能 403——**显式传 `github_token: ${{ secrets.GITHUB_TOKEN }}`** 绕开;
- `claude.yml` 不要用 `cancel-in-progress: true`——后到的评论会顶掉正在干活的 Claude。
