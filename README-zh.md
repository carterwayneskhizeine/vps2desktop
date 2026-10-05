# vps2desktop

[English](README.md) | 简体中文

把一台只有 SSH 的全新 **Ubuntu x86_64 VPS**，变成一台完整可用的**远程桌面（RDP）机器**——干活的不是你，而是你选定的 AI agent。

你在勾选页选组件（Xfce 桌面、中文输入、浏览器、编辑器、AI CLI……），把清单加连接信息交给任意 AI agent（Claude Code、Codex、OpenCode、pi……），它会把这台机器变成 root 直连、RDP 可用的生产力工作站，所有组件在安装时一律拉取**官方最新版**。

## 工作流程

```
1. git clone 本仓库
2. 用浏览器打开 checklist.html
   → 选预设或逐项勾选（依赖自动连带勾选）
   → 可选填写连接信息（host / port / user / password）
   → 导出 manifest.yaml，放到 machines/<别名>/manifest.yaml
3. 对你的 AI agent 说："deploy machines/<别名>"
   → 它按 SKILL.md 执行：预检 → root 化 → 逐组件安装+验收 → 记录
```

日常更新：`git pull`，然后让 agent *check updates*——它读取 `catalog/CHANGELOG.md`，对照你机器的 manifest，逐项询问你是否安装新内容。

## 会装什么

`catalog/` 里的一切都是可选的。亮点：Xfce + xrdp（PipeWire 音频可用）、fcitx5 中文输入、Chrome / VS Code（root 安全包装）、nvm + Node LTS / uv / Anaconda，以及 AI CLI 全家桶：Claude Code、Codex、OpenCode、pi、cc-switch、aichat、agent-browser、tavily。

## 环境要求

- 一台 **Ubuntu 24.04+ x86_64** 的 VPS（其他系统：agent 会拒绝并明确告知）
- 任意能读本仓库并执行 `ssh` 的 AI coding agent（Claude Code、Codex、OpenCode、pi……）
- agent 运行所需的 POSIX shell（Linux / macOS / WSL / Git Bash）

## 安全须知（请先读）

- 部署会把机器标准化为 **root 密码 SSH 登录**（见 `docs/adr/0003-root-password-login.md`）。`fail2ban` 组件用来缓解由此带来的爆破风险。
- 你的凭据只存在于 `machines/<别名>/manifest.yaml`，该目录被 **gitignore**，永不离开你的机器。你在 `checklist.html` 里输入的一切不会被发送到任何地方——它是单个离线 HTML 文件。
- 厂商提供的原始用户（如 `ubuntu`）保持原样不动，作为兜底退路。

## 仓库结构

| 路径 | 用途 |
|---|---|
| `SKILL.md` | 你的 AI agent 遵循的协议（deploy / status / update-check 三种模式） |
| `checklist.html` | 离线单文件组件勾选页；导出你的 manifest |
| `catalog/` | 每个组件一篇文档：安装命令、守卫、验收、已知的坑 |
| `catalog/INDEX.yaml` | 组件注册表：分组、依赖、预设 |
| `machines/` | 你的本地机器状态（被 gitignore——由你/agent 创建） |
| `docs/adr/` | 架构决策记录 |
| `CONTEXT.md` | 本仓库术语表 |

## 参与贡献

发现了一个所有人都会踩的坑？欢迎提 issue 或 PR 改进 `catalog/` 里的组件文档——见 `CONTRIBUTING.md`。只在你机器上出现的特有问题请写在你本地的 `machines/<别名>/notes.md`，不要进仓库。

## 许可证

[MIT](LICENSE)
