---
title: 用 iPhone 远程操控 Linux 上的 Codex CLI
date: 2026-08-13 10:00:00
updated: 2026-08-13 10:00:00
tags:
  - Codex
  - Linux
  - SSH
  - tmux
categories: devops
keywords: iPhone Codex CLI SSH tmux Mosh ZeroTier 远程开发
description: 通过 ZeroTier、SSH、tmux 和 Mosh，用 iPhone 安全地连接 Linux，并随时恢复同一个 Codex CLI 会话。
---

我想解决的问题很直接：Codex CLI 和项目都运行在 Linux 上，出门后再用 iPhone 查看进度、继续输入指令，而且断开连接不能终止正在进行的工作。

最适合这个场景的并不是在手机上安装 Codex，也不是把完整 Linux 桌面投到小屏幕上，而是让 iPhone 充当远程终端：

```text
iPhone
  -> ZeroTier 私有网络
  -> SSH 或 Mosh
  -> Linux
  -> tmux
  -> Codex CLI
```

项目文件、Git 凭据和 API Key 始终留在 Linux 上，手机只负责连接和输入。离开电脑前建立的 Codex 会话，也可以在手机上原样恢复。

<!-- more -->

## 为什么选择 SSH + tmux

远程桌面当然也能操作 Codex，但桌面界面是为大屏设计的。在 iPhone 上经常需要反复缩放、定位窗口和调出键盘，真正输入命令时并不方便。

SSH 更适合以终端为主的 Codex CLI，而 tmux 解决了连接中断的问题：

- iPhone 只显示终端，触控和键盘操作更直接。
- Codex 在 Linux 上运行，手机断网不会结束进程。
- 电脑和手机可以先后接入同一个会话。
- 代码、密钥和开发环境不需要同步到手机。

如果移动网络经常在 Wi-Fi 和 5G 之间切换，还可以用 Mosh 替代 SSH 承载交互。Mosh 负责应对弱网和地址变化，tmux 负责保留服务端会话，两者解决的是不同问题。

## 一、准备 Linux 主机

先确认 Linux 已安装并能正常运行 Codex CLI，然后安装 SSH 服务、tmux 和可选的 Mosh。以 Ubuntu 或 Debian 为例：

```bash
sudo apt update
sudo apt install openssh-server tmux mosh
sudo systemctl enable --now ssh
```

在 Linux 本机进入项目目录，确认 Codex 可以启动：

```bash
cd ~/project
codex
```

如果使用 API Key，建议只在 Linux 上配置，不要保存到 iPhone 的笔记、快捷指令或 SSH 主机备注中。例如可以在当前 shell 中注入：

```bash
export OPENAI_API_KEY="你的 API Key"
```

长期使用时应放进权限受控的环境配置或 secret manager。无论采用哪种方式，都不要把真实密钥提交到 Git 仓库。

## 二、用 ZeroTier 建立私有连接

不建议把 Linux 的 SSH 端口直接暴露到公网。我的环境已经使用 ZeroTier，因此只需要让 iPhone 和 Linux 加入同一个 ZeroTier 网络，并记下 Linux 的 ZeroTier IP，例如：

```text
10.147.x.x
```

在 Linux 上可以确认 ZeroTier 分配的地址：

```bash
zerotier-cli listnetworks
```

随后保持 iPhone 上的 ZeroTier 已连接。SSH 客户端中的目标主机填写这个私有 IP，而不是家中宽带的公网 IP。

Tailscale、WireGuard 等私有组网也能完成同样的工作，后面的 SSH 和 tmux 用法不变。

## 三、配置 iPhone SSH 客户端

这类工作流中，我更倾向使用 Blink Shell，因为它更像完整的移动终端，并且适合 `SSH + Mosh + tmux` 的长期交互。

另外两个常见选择是：

- Termius：主机管理和快速连接更直观，适合管理多台服务器。
- Secure ShellFish：与 Files、快捷指令等 Apple 生态的整合更好。

无论使用哪个客户端，都建议生成独立的 SSH 密钥，并把公钥安装到 Linux。可以在电脑上执行：

```bash
ssh-copy-id user@10.147.x.x
```

这里的 `user` 应是普通 Linux 用户，不要直接使用 root。验证密钥登录正常后，再根据自己的服务器配置关闭密码登录。

在 iPhone 客户端中保存以下信息：

```text
Host: Linux 的 ZeroTier IP
Port: 22
User: Linux 普通用户名
Authentication: SSH 私钥
```

首次连接时，要核对客户端显示的主机指纹与 Linux 上的指纹一致，确认后再信任主机。

## 四、启动并恢复 Codex 会话

第一次连接 Linux 后，可以直接在目标项目中创建 tmux 会话：

```bash
tmux new-session -s codex -c ~/project
codex
```

需要暂时离开时，先按 `Ctrl-b`，松开后再按 `d`。这只会把当前客户端从 tmux 分离，Codex 仍在 Linux 上运行。

以后无论从 iPhone 还是电脑重新连接，只要执行：

```bash
tmux attach -t codex
```

就能回到同一个工作目录和 Codex 进程。完整的手机端操作通常只有两步：

```bash
ssh user@10.147.x.x
tmux attach -t codex
```

如果忘记会话名称，可以先执行 `tmux ls`。如果还没有 `codex` 会话，可以用下面这条命令创建；已经存在时则直接接入：

```bash
tmux new-session -A -s codex -c ~/project
```

进入 tmux 后再运行 `codex` 即可。

## 五、弱网环境改用 Mosh

纯 SSH 在切换 Wi-Fi、5G 或短暂失去信号后，连接可能卡住或断开。服务器和 iPhone 客户端都支持 Mosh 时，可以改用：

```bash
mosh user@10.147.x.x
tmux attach -t codex
```

Mosh 初次建立连接仍需要 SSH，之后使用 UDP 通信。如果连不上，应检查：

- Linux 是否已经安装 `mosh`。
- iPhone 客户端是否支持 Mosh。
- 主机防火墙或 ZeroTier 规则是否允许 Mosh 使用的 UDP 端口。
- 服务端是否能从登录环境中找到 `mosh-server`。

即便使用 Mosh，也仍然应该保留 tmux。Mosh 能改善网络漫游体验，但手机应用被系统结束、设备重启或会话异常时，只有 tmux 能确保 Codex 进程仍留在服务器上。

## 六、日常使用方式

在电脑前工作时：

```bash
ssh user@10.147.x.x
tmux new-session -A -s codex -c ~/project
```

出门后在 iPhone 上：

```bash
mosh user@10.147.x.x
tmux attach -t codex
```

回到电脑后，再次接入同一个会话即可。通常不要让电脑和手机同时操作同一 tmux 窗口，以免屏幕尺寸变化和同时输入互相干扰；切换设备前先执行 tmux detach，体验更稳定。

手机比较适合查看 Codex 进度、补充任务要求、确认测试结果、查看小范围 diff，以及处理边界清楚的小修改。大规模代码审查、生产变更和涉及密钥或数据删除的操作，仍应回到电脑上完成。

## 七、安全检查

这套方案把开发机的终端开放给手机，基础安全配置不能省略：

- 使用 ZeroTier、Tailscale 或 WireGuard，不直接暴露 SSH 到公网。
- 使用 SSH 密钥，确认可用后再关闭密码登录。
- 为远程开发使用普通用户，不以 root 身份运行 Codex。
- 为 iPhone 设置锁屏密码和生物识别，并保护 SSH 客户端中的私钥。
- API Key、Git 凭据和项目代码只保留在 Linux。
- 批准 Codex 命令前检查操作目录和影响范围。
- 修改代码前检查 `git status`，重要任务使用独立分支或 worktree。

## 总结

对 API 模式和 Linux 开发环境来说，最实用的手机远程 Codex 方案是：

```text
iPhone + Blink Shell
  -> ZeroTier
  -> Mosh/SSH
  -> tmux
  -> Codex CLI
```

SSH 或 Mosh 负责从手机进入 Linux，tmux 负责让会话持续存在，Codex CLI 则始终在拥有代码和完整开发环境的 Linux 上工作。这样不需要在手机保存 API Key，也不需要勉强操作缩小后的桌面界面，换设备后仍能继续同一个 Codex 会话。
