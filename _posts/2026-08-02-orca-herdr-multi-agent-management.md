---
title: AI能力集 -- Orca & Herdr：GUI和终端组合使用覆盖本地和远程开发环境
description: 通过 Herdr 终端复用 + Orca 可视化 IDE 组合，应对多 AI Agent 并行运行时的状态盲区与会话中断痛点
categories: [AI, AI能力集]
tags: [AI, Agent, 工具]
---

## 1. 引言

多 Agent 并行运行正在成为 AI 编程的新常态：Claude Code 写代码、Codex 做评审、Cursor CLI 跑测试……一不留神，终端窗口已经堆满，状态一塌糊涂。

传统方案有明显的局限：

- **Tmux/Screen**：能做终端分屏，但无法识别 AI Agent 的语义状态——分不清 Agent 是正在工作、卡住等待确认、还是已经结束
- **多窗口手动管理**：窗口切换繁琐，SSH 断开后会话丢失，Agent 进程也随之中断
- **纯 GUI 工具**：资源占用高，不适合服务器常驻

近期通过Orca 和 Herdr组合使用，效果很好：

- **Herdr**：终端 TUI 轻量化方案，AI 版 tmux，2026 年 6 月登顶 GitHub 趋势榜第一
- **Orca**：GUI 可视化多 Agent 工作台，GitHub Star 超 19k，主打一站式编排与调试

二者互补组合使用，正好覆盖终端重度用户和 GUI 偏好者的不同场景。

相关链接：
* [Orca 仓库](https://github.com/stablyai/orca)
* [Herdr 仓库](https://github.com/ogulcancelik/herdr)

---

## 2. 工具介绍

### 2.1. Herdr介绍：终端向，AI 专用终端复用器

Herdr 由 [@ogulcancelik](https://github.com/ogulcancelik) 开发，纯 Rust 实现，无 Electron、无浏览器、无后台遥测。

**基本信息：**
* 定位：面向 AI 智能体的终端多路复用器（Agent Multiplexer），对标经典终端工具 `tmux`，但原生适配 AI 编码 Agent（Claude Code、OpenAI Codex、Ollama、各类终端 AI 助手）
* 基于Rust开发，单文件二进制程序，无 Electron 重型依赖，极低内存占用
* 跨平台支持 **Linux /macOS**，**Windows**当前是实验性支持，部分特性可能支持不全（所以我Windows下换成了Orca，组合使用效果很好）

**核心功能：**

1、经典终端复用能力（兼容 tmux 使用习惯）
* 窗格分屏（水平 / 垂直分割）、标签页、工作区分组管理；
* 会话分离 / 后台常驻（detach）：关闭终端、SSH 断线后，AI 进程继续后台运行，任意终端重新挂载恢复会话；
* 鼠标拖拽调整窗格、tmux 风格前缀快捷键自定义

2、AI Agent 专属核心能力（差异化亮点）
* Agent 状态可视化：侧边栏直接标注每个窗格 AI 状态：Working（生成代码）、Blocked（等待用户确认 / 输入）、Done、Idle，不用切进窗格判断 AI 是否卡死。
* AI Agent 自身可以调用 Herdr 接口：自动新建窗格、读取其他窗格日志、等待其他 Agent 任务完成，实现Agent 互相调度、多智能体流水线，适合自动化编码工作流。
* 会话持久化落地：重启电脑依然复原全部 AI 会话、历史对话上下文、分屏布局

### 2.2. Orca介绍：图形 IDE 向，多 Agent 集成开发环境

StablyAI Orca 是硅谷创业公司Stably（YC 孵化）在 2026 年推出的开源多 AI 编程智能体编排 IDE（Agentic IDE / ADE 智能体开发环境），核心作用是统一调度多款 AI 编程 Agent（Claude Code、OpenAI Codex、Cursor、Gemini、GitHub Copilot 等）并行开发，依靠Git Worktree 隔离机制解决多 AI 同时写代码互相覆盖、Git 冲突、上下文污染的行业痛点。

主打一站式编排、调试、监控多 Agent 协同任务。

**基础信息：**
* 开发语言：TypeScript
* 支持macOS / Windows / Linux 全平台
* 配套移动端：iOS（App Store）、Android（APK）远程控制

**核心功能：**
* 多 Agent 兼容调度：原生适配市面主流命令行 AI 编程 Agent，Claude Code、OpenAI Codex、Cursor CLI、OpenCode、Gemini CLI、Copilot CLI、Cline 等
* 高性能终端系统（Ghostty 内核）：WebGL 硬件加速无限分屏终端；支持终端分屏布局自定义，适合调试多进程服务、后端集群
* 移动端远程管控：手机 App 与桌面端实时配对
* 开发工作流深度集成：GitHub / Linear 原生对接；SSH 远程 Worktree；文件拖拽交互等等
* Orca CLI 命令行工具，支持脚本化自动化编排，支持以Skill由Agent调度

### 3.1. Orca vs Herdr 对比

| 对比维度 | Herdr | Orca |
|---------|-------|------|
| 界面形态 | 终端 TUI（纯命令行） | 桌面 GUI 图形界面 |
| 开发语言 | Rust（极致轻量） | TypeScript |
| 核心定位 | AI 专用终端多路复用器 | 多 Agent 集成开发 IDE |
| 资源占用 | 极低，适合服务器常驻 | 偏高，本地桌面运行 |
| 上手难度 | 需要熟悉终端快捷键 | 开箱即用，IDE 逻辑零学习成本 |
| 核心优势 | 会话保活、SSH 远程、资源节约 | 可视化编排、调试、多 Agent 任务流水线 |
| 典型场景 | 服务器长时间跑 AI 脚本、纯终端工作流 | 本地多 Agent 对比测试、复杂 AI 任务编排 |

## 4. 实际使用

### 4.1. 安装

直接到Github上下载发布产物即可，均有Windows版本。

orca和herdr中都可以创建多个worktree来进行并行AI任务，若要使用herdr的鼠标右键功能，在orca里有点冲突，可组合prefix前缀键使用。

### 4.2. 实际使用界面

![orca使用界面](/images/2026-08-02-orca.png)

![herdr使用界面](/images/2026-08-02-herdr-remote.png)

### 4.3. orca远程headless启动

支持远程服务器启动orca服务端，我本地有一台PC机安装了Linux系统，上面跑着OpenClaw、Hermes等，准备在这台运行一个server端，和手机配对，这样在手机上可以控制orca。

orca项目里有篇文档专门介绍headless方式安装和配对：`docs/reference/headless-linux-server.md`。

1、下载Linux版本的服务包： orca-linux.AppImage

AppImage 的特点就是不需要安装——它是一个自包含的可执行文件。下载后赋予执行权限就能跑。

不过有两个前提：
* FUSE 依赖（默认模式）
* Xvfb：无桌面环境需要它来渲染，Orca会自动启动，但需要先安装

2、远程服务补全依赖，此处补充提示我的机器缺少的依赖

`dnf install -y xorg-x11-server-Xvfb`

3、启动

> 注意：用 root 运行需要加 `--no-sandbox` 参数，否则会报错：Running as root without --no-sandbox is not supported. 

方式1：直接`./AppImage`运行

```
[root@xdlinux ➜ /home ]$  ./orca-linux.AppImage --no-sandbox serve --port 6768
...
[codex-real-home-hooks] trust grant unavailable (error); entry rolled back, managed lane kept
[codex-trust-grant] falling back to self-computed trust (reason=retry-cached, host=native)
[serve] orca CLI install: installed (/root/.local/bin/orca-ide)
[serve] bare orca dispatcher installed: /root/.local/bin/orca -> /home/orca-linux.AppImage
Orca server ready
Bound endpoint: ws://0.0.0.0:6768
Advertised endpoint: ws://127.0.0.1:6768
Web client URL: http://127.0.0.1:6768/web-index.html#pairing=orca%3A%2F%2Fpair%3Fcode%3DeyJ2IjoyLCJlbmRwb2ludCI6IndzOi8vMTI3LjAuMC4xOjY3NjgiLCJkZXZpY2VUb2tlbiI6IjliODczODc3YjZiOTIwYWIwZGYwMTgwYjcwMjhkY2Q0NjVhMGU2MDRlNTRhMGI5YiIsInB1YmxpY0tleUI2NCI6IndzMWphY3Jwd205bThrd29EMG1uZng5YTMvVTYyaXB3OXJnY3l6RXl1Qnc9Iiwic2NvcGUiOiJydW50aW1lIn0
Pairing URL: orca://pair?code=eyJ2IjoyLCJlbmRwb2ludCI6IndzOi8vMTI3LjAuMC4xOjY3NjgiLCJkZXZpY2VUb2tlbiI6IjliODczODc3YjZiOTIwYWIwZGYwMTgwYjcwMjhkY2Q0NjVhMGU2MDRlNTRhMGI
...
```

方式2：提取后运行，`./squashfs-root/AppRun --no-sandbox serve --port 6768`

如果你不想装 FUSE，用 extraction 模式完全不依赖 FUSE。
* 提取：`./orca-linux.AppImage --appimage-extract`
* 之后直接运行提取出的`AppRun`：`./squashfs-root/AppRun serve --port 6768`

提取出来的`squashfs-root/`目录（默认目录就是这个）可以留在任何位置，不影响运行。

```sh
[root@xdlinux ➜ /home ]$ ./orca-linux.AppImage --appimage-extract
squashfs-root/.DirIcon
squashfs-root/AppRun
squashfs-root/LICENSE.electron.txt
squashfs-root/LICENSES.chromium.html
...
```

提取出来的内容结构如下：

```sh
[root@xdlinux ➜ squashfs-root ]$ ls -ltrh
total 266M
-rw-r--r--  1 root root 1.1K Aug  2 22:38 LICENSE.electron.txt
-rwxr-xr-x  1 root root 3.5K Aug  2 22:38 AppRun
-rw-r--r--  1 root root  20M Aug  2 22:38 LICENSES.chromium.html
-rwxr-xr-x  1 root root  15K Aug  2 22:38 chrome-sandbox
-rw-r--r--  1 root root 118K Aug  2 22:38 chrome_100_percent.pak
-rw-r--r--  1 root root 194K Aug  2 22:38 chrome_200_percent.pak
-rwxr-xr-x  1 root root 1.8M Aug  2 22:38 chrome_crashpad_handler
-rw-r--r--  1 root root  11M Aug  2 22:38 icudtl.dat
-rwxr-xr-x  1 root root 254K Aug  2 22:38 libEGL.so
-rwxr-xr-x  1 root root 6.2M Aug  2 22:38 libGLESv2.so
-rwxr-xr-x  1 root root 2.7M Aug  2 22:38 libffmpeg.so
-rwxr-xr-x  1 root root 4.5M Aug  2 22:38 libvk_swiftshader.so
-rwxr-xr-x  1 root root 2.4M Aug  2 22:38 libvulkan.so.1
drwx------  2 root root 4.0K Aug  2 22:38 locales
lrwxrwxrwx  1 root root   49 Aug  2 22:38 orca-ide.png -> usr/share/icons/hicolor/512x512/apps/orca-ide.png
-rw-r--r--  1 root root  221 Aug  2 22:38 orca-ide.desktop
-rwxr-xr-x  1 root root 210M Aug  2 22:38 orca-ide
drwx------ 10 root root 4.0K Aug  2 22:38 resources
-rw-r--r--  1 root root 7.0M Aug  2 22:38 resources.pak
-rw-r--r--  1 root root 357K Aug  2 22:38 snapshot_blob.bin
drwx------  4 root root   30 Aug  2 22:38 usr
-rw-r--r--  1 root root  107 Aug  2 22:38 vk_swiftshader_icd.json
-rw-r--r--  1 root root 723K Aug  2 22:38 v8_context_snapshot.bin
```

4、配对方式：Tailscale

内核：[tailscale](https://github.com/tailscale/tailscale)，各平台的应用包基本时在此基础上包装了一层。
* 对应的安卓版本在：[tailscale-android](https://github.com/tailscale/tailscale-android/releases)。
* Linux安装：`curl -fsSL https://tailscale.com/install.sh | sh`

```sh
[root@xdlinux ➜ /home ]$ curl -fsSL https://tailscale.com/install.sh | sh
Installing Tailscale for fedora, using method dnf
+ '[' 3 = 3 ']'
+ dnf install -y 'dnf-command(config-manager)'
Last metadata expiration check: 3:45:59 ago on Sun 02 Aug 2026 07:18:19 PM CST.
Package dnf-plugins-core-4.3.0-24.el9_7.noarch is already installed.
Dependencies resolved.
...
```


